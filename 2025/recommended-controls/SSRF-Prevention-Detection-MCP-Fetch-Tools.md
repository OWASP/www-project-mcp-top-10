---
layout: col-sidebar
title: "Recommended Control: SSRF Prevention and Detection for MCP Fetch/Scrape Tools"
---

# Recommended Control: SSRF Prevention and Detection for MCP Fetch/Scrape Tools

**Applies to:** MCP05:2025 – Command Injection & Execution (also relevant to MCP07 – OAuth/Authorization Discovery and MCP09 – Remote Transport Security, see cross-references below)
**Contributed by:** Syed Anas Mohiuddin
**References:** IETF Internet-Draft draft-mohiuddin-mcp-security-considerations-00; mcp-safeguard (https://pypi.org/project/mcp-safeguard/)

---

## Overview

An MCP server that exposes a tool accepting a `url` parameter and passes it to an HTTP client
without validating the resolved IP address is vulnerable to Server-Side Request Forgery (SSRF).
Because MCP tool arguments originate from LLM reasoning, an attacker who can influence the
content the model reads — via prompt injection — can steer such a tool to request internal,
loopback, link-local, or cloud-metadata endpoints.

This control addresses three related but distinct mechanisms, which are easy to conflate and
require different handling:

- **Server-side request forgery** — the server itself makes an attacker-influenced outbound
  request. This is the core threat this document addresses.
- **Redirect-based SSRF** — one way an SSRF request reaches a prohibited destination: the initial
  URL is safe, but a 3xx response steers the request to an unsafe target on a later hop.
- **DNS time-of-check-to-time-of-use (TOCTOU) / "DNS rebinding"** — the resolved destination
  changes between the validation step and the connection step, even when the hostname itself
  never changes and no redirect occurs. This defeats naive "resolve, check, then let the HTTP
  client resolve again and connect" implementations, including an earlier version of this
  document's own sample code.

A separate, client-side variant of DNS rebinding targets a browser making requests to a local MCP
HTTP server; it is a different attack surface from the server-side cases above and is out of scope
for this control.

---

## Prevalence

*[Author's note for the OWASP maintainers: the previous version of this section claimed "a
substantial fraction" of surveyed servers were vulnerable without citing a sample size or
methodology. That claim is being withdrawn pending either (a) a documented, reproducible survey
with real numbers, or (b) reframing as an informal, non-statistical observation. Do not restore
language implying a rigorous survey unless one actually exists and can be cited.]*

In informal testing performed while developing mcp-safeguard, multiple popular MCP fetch/scrape
server implementations exhibited no SSRF protection at all when their URL tool was invoked against
a listener under the researcher's control. This is not presented as a statistically rigorous
prevalence figure. Two failure modes recur consistently in the servers examined:

1. **No validation** — the URL is passed to the HTTP client without any checks
2. **Scheme-only validation** — the code checks for `http` or `https` but never checks the
   resolved host or IP, allowing `http://169.254.169.254/` to pass unchecked

---

## Example Attack Scenario

1. An agent is deployed with an MCP fetch or scrape server.
2. The agent processes external content containing a prompt-injection payload that instructs it
   to fetch an internal URL.
3. The model invokes the fetch tool with `http://169.254.169.254/latest/meta-data/` or
   `https://10.0.0.1/internal-admin`.
4. The server fetches it and returns the body into the model's context — giving the attacker
   internal service access or cloud credential exposure without the user observing anything wrong.

---

## Impact

- **Credential theft:** Cloud instance-metadata services (AWS IMDSv1, GCP metadata) return
  short-lived IAM credentials to any caller on the local network. An SSRF-vulnerable MCP server
  running in a cloud VM can expose these credentials to the model's context.
- **Internal service access:** Admin APIs, Kubernetes API server, and internal dashboards that
  are not exposed to the public internet become reachable via the MCP server's network position.
- **Port and host enumeration:** Response-body and timing differences leak internal network
  topology.

---

## Recommended Control: Validated, Pinned Fetch

Prefer a vetted network-policy component or egress proxy (e.g., an allowlisted forward proxy, or a
cloud provider's VPC-level egress control) as the primary mitigation where operationally feasible —
it removes the need to get hand-rolled IP validation correct in every service. Where that is not
available, an MCP server MUST NOT simply check a hostname's resolved address and then let a
general-purpose HTTP client re-resolve and connect to that hostname independently — the two
resolutions can return different addresses (DNS TOCTOU / rebinding), silently defeating the check.
The validated address must be the same address the connection is made to.

Before issuing any outbound HTTP request, the MCP server MUST:

1. **Reject non-HTTP/HTTPS schemes** — block `file://`, `gopher://`, `ftp://`, `dict://`, etc.
2. **Resolve the hostname to all its addresses** (both A and AAAA records — a resolver that only
   returns IPv4, such as `gethostbyname()` in Python, silently drops any IPv6-based bypass check),
   and reject the hostname if *any* returned address falls in a blocked range:
   - Loopback: `127.0.0.0/8`, `::1/128`
   - Private (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - Link-local / cloud metadata: `169.254.0.0/16`, `fe80::/10`
   - CGNAT: `100.64.0.0/10`
   - IPv4-mapped IPv6 (`::ffff:x.x.x.x`) — validate the mapped IPv4 address, not the prefix
3. **Connect directly to the validated address**, not to the hostname — while still sending the
   original `Host` header and, for HTTPS, the original hostname as the TLS SNI/`server_hostname`,
   so routing and certificate validation remain correct. This is what pins the check to the
   connection and closes the TOCTOU gap in step 2.
4. **Re-validate on every redirect hop.** Do not let the HTTP client follow redirects
   automatically. Treat each `Location:` header as a new URL and repeat steps 1–3 before following
   it. A redirect chain that resolves safely on hop one and unsafely on hop two must be caught.
5. **Apply an allowlist** where the use case permits — restrict to approved domains rather than
   relying solely on a blocklist.

### Minimal Node.js Implementation

```javascript
const dns = require('dns').promises;
const https = require('https');
const http = require('http');
const ipRangeCheck = require('ip-range-check');

const BLOCKED_RANGES = [
  '127.0.0.0/8', '::1/128',
  '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
  '169.254.0.0/16', 'fe80::/10',
  '100.64.0.0/10',
];

function normalizeIpv4Mapped(address) {
  const m = /^::ffff:(\d+\.\d+\.\d+\.\d+)$/.exec(address);
  return m ? m[1] : address;
}

// Resolves BOTH A and AAAA records and rejects the hostname if any of them
// is unsafe — a resolver that only checks the first/IPv4 result can be
// bypassed by a record the check never looked at.
async function resolveAllOrReject(hostname) {
  const records = await dns.lookup(hostname, { all: true, verbatim: true });
  for (const { address } of records) {
    if (ipRangeCheck(normalizeIpv4Mapped(address), BLOCKED_RANGES)) {
      return null;
    }
  }
  return records;
}

// Fetches a URL while pinning the TCP connection to a pre-validated IP
// (closing the DNS TOCTOU gap), preserving the correct Host header and TLS
// SNI, and re-validating the destination on every redirect hop rather than
// trusting a single check before the first request.
async function safeFetch(urlStr, maxRedirects = 5) {
  let current = new URL(urlStr);
  for (let hop = 0; hop <= maxRedirects; hop++) {
    if (!['http:', 'https:'].includes(current.protocol)) {
      throw new Error('blocked scheme');
    }
    const records = await resolveAllOrReject(current.hostname);
    if (!records) throw new Error('blocked destination');
    const pinnedIp = records[0].address;
    const client = current.protocol === 'https:' ? https : http;

    const response = await new Promise((resolve, reject) => {
      const req = client.request({
        host: pinnedIp, // connect to the address we validated, not the hostname
        servername: current.protocol === 'https:' ? current.hostname : undefined, // correct TLS SNI
        headers: { Host: current.hostname }, // correct Host header
        path: current.pathname + current.search,
        port: current.port || (current.protocol === 'https:' ? 443 : 80),
      }, resolve);
      req.on('error', reject);
      req.end();
    });

    if ([301, 302, 303, 307, 308].includes(response.statusCode)) {
      const location = response.headers.location;
      if (!location) throw new Error('redirect with no location');
      current = new URL(location, current); // re-validated at the top of the next loop
      continue;
    }
    return response;
  }
  throw new Error('too many redirects');
}
```

### Minimal Python Implementation

```python
import ipaddress
import socket
import ssl
import http.client
from urllib.parse import urlparse, urljoin

BLOCKED_RANGES = [
    "127.0.0.0/8", "::1/128",
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "169.254.0.0/16", "fe80::/10",
    "100.64.0.0/10",
]

def _is_blocked(ip) -> bool:
    if isinstance(ip, ipaddress.IPv6Address) and ip.ipv4_mapped:
        ip = ip.ipv4_mapped  # validate the mapped v4 address, not the ::ffff: prefix itself
    return any(ip in ipaddress.ip_network(r, strict=False) for r in BLOCKED_RANGES)

# getaddrinfo() returns BOTH A and AAAA records; gethostbyname() is IPv4-only
# and silently makes any IPv6 entry in BLOCKED_RANGES unreachable dead code.
def resolve_all_or_reject(hostname: str):
    infos = socket.getaddrinfo(hostname, None)
    addrs = [ipaddress.ip_address(info[4][0]) for info in infos]
    if any(_is_blocked(a) for a in addrs):
        return None
    return addrs

# Connects directly to a pre-validated address (closing the DNS TOCTOU gap),
# preserves the Host header and TLS SNI, and re-validates on every redirect
# hop instead of trusting a single pre-flight check.
def safe_fetch(url: str, max_redirects: int = 5):
    for _ in range(max_redirects + 1):
        parsed = urlparse(url)
        if parsed.scheme not in ("http", "https"):
            raise ValueError("blocked scheme")
        addrs = resolve_all_or_reject(parsed.hostname)
        if not addrs:
            raise ValueError("blocked destination")
        pinned_ip = str(addrs[0])
        port = parsed.port or (443 if parsed.scheme == "https" else 80)

        if parsed.scheme == "https":
            ctx = ssl.create_default_context()
            raw_sock = socket.create_connection((pinned_ip, port))
            sock = ctx.wrap_socket(raw_sock, server_hostname=parsed.hostname)  # correct SNI
            conn = http.client.HTTPConnection(pinned_ip, port)
            conn.sock = sock
        else:
            conn = http.client.HTTPConnection(pinned_ip, port)

        conn.request("GET", parsed.path or "/", headers={"Host": parsed.hostname})
        resp = conn.getresponse()

        if resp.status in (301, 302, 303, 307, 308):
            location = resp.getheader("Location")
            if not location:
                raise ValueError("redirect with no location")
            url = urljoin(url, location)  # re-validated at the top of the next loop
            continue
        return resp
    raise ValueError("too many redirects")
```

---

## Detection

Automated detection of this SSRF pattern across MCP server tool definitions is available in
**mcp-safeguard**, an open-source scanner purpose-built for the MCP ecosystem:

- PyPI: https://pypi.org/project/mcp-safeguard/
- Install: `pip install mcp-safeguard`

The scanner implements runtime verification (connecting to a local listener to confirm the
server actually initiates the connection) in addition to static rule checks.

---

## Cross-references

This control is not limited to MCP05's command-injection scope. The same unvalidated-destination
pattern applies to:

- **MCP07 – OAuth/Authorization Discovery**, where an authorization-server or metadata-document
  URL discovered from untrusted input can be redirected the same way.
- **MCP09 – Remote Transport Security**, where a remote MCP server endpoint reachable over
  SSE/streamable-HTTP is itself a destination that should go through the same validation before
  any credential is attached to it.

---

## References

- IETF Internet-Draft: *Security Considerations for Model Context Protocol Implementations*
  (draft-mohiuddin-mcp-security-considerations-00)
- OWASP SSRF Prevention Cheat Sheet
- MCP05:2025 – Command Injection & Execution
