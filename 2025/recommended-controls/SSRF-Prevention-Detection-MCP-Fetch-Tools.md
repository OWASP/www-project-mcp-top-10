---
layout: col-sidebar
title: "Recommended Control: SSRF Prevention and Detection for MCP Fetch/Scrape Tools"
---

# Recommended Control: SSRF Prevention and Detection for MCP Fetch/Scrape Tools

**Applies to:** MCP05:2025 – Command Injection & Execution  
**Contributed by:** Syed Anas Mohiuddin  
**References:** IETF Internet-Draft draft-mohiuddin-mcp-security-considerations-00; mcp-safeguard (https://pypi.org/project/mcp-safeguard/)

---

## Overview

An MCP server that exposes a tool accepting a `url` parameter and passes it to an HTTP client
without validating the resolved IP address is vulnerable to Server-Side Request Forgery (SSRF).
Because MCP tool arguments originate from LLM reasoning, an attacker who can influence the
content the model reads — via prompt injection — can steer such a tool to request internal,
loopback, link-local, or cloud-metadata endpoints.

This differs from classic SSRF in that the "user input" is the model's own tool call. Any
untrusted content the model ingests (a web page, document, or email) becomes a potential
injection vector for internal network access.

---

## Prevalence

In a runtime-verified survey of MCP fetch and scrape servers across the npm and PyPI ecosystems,
SSRF was found to be widespread. A substantial fraction of runnable fetch servers tested connected
to an attacker-controlled internal listener when their URL tool was invoked — confirming no SSRF
protection at all. Every instance was confirmed by runtime verification (actual TCP connection to a
127.0.0.1 listener under the researcher's control), not static heuristics.

Two failure modes recur consistently:

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

## Recommended Control: Pre-Flight URL Validation

Before issuing any outbound HTTP request, the MCP server MUST:

1. **Reject non-HTTP/HTTPS schemes** — block `file://`, `gopher://`, `ftp://`, `dict://`, etc.
2. **Resolve the hostname via DNS**, then reject any resolved address in the following ranges:
   - Loopback: `127.0.0.0/8`, `::1/128`
   - Private (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - Link-local / cloud metadata: `169.254.0.0/16`, `fe80::/10`
   - CGNAT: `100.64.0.0/10`
   - IPv4-mapped IPv6 (`::ffff:x.x.x.x`) — validate the mapped IPv4 address, not the prefix
3. **Re-validate after every redirect** — DNS-rebinding and redirect-chain attacks bypass
   validation performed only before the first request. Every `Location:` header must be
   re-checked against the blocklist before following.
4. **Apply an allowlist** where the use case permits — restrict to approved domains rather than
   relying solely on a blocklist.

### Minimal Node.js Implementation

```javascript
const dns = require('dns').promises;
const ipRangeCheck = require('ip-range-check');

const BLOCKED_RANGES = [
  '127.0.0.0/8', '::1/128',
  '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
  '169.254.0.0/16', 'fe80::/10',
  '100.64.0.0/10',
];

async function isSafeUrl(urlStr) {
  let url;
  try { url = new URL(urlStr); } catch { return false; }
  if (!['http:', 'https:'].includes(url.protocol)) return false;
  const { address } = await dns.lookup(url.hostname);
  return !ipRangeCheck(address, BLOCKED_RANGES);
}
```

### Minimal Python Implementation

```python
import ipaddress
import socket
from urllib.parse import urlparse

BLOCKED_RANGES = [
    "127.0.0.0/8", "::1/128",
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "169.254.0.0/16", "fe80::/10",
    "100.64.0.0/10",
]

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https"):
        return False
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    except (socket.gaierror, ValueError):
        return False
    return not any(
        ip in ipaddress.ip_network(r, strict=False) for r in BLOCKED_RANGES
    )
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

## References

- IETF Internet-Draft: *Security Considerations for Model Context Protocol Implementations*
  (draft-mohiuddin-mcp-security-considerations-00)
- OWASP SSRF Prevention Cheat Sheet
- MCP05:2025 – Command Injection & Execution
