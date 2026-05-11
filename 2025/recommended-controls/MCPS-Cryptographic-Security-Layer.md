---

layout: col-sidebar
title: "Recommended Control: MCPS — Cryptographic Security Layer for MCP"

---

## Overview

MCPS (MCP Secure) is an open-source cryptographic security layer for the Model Context Protocol that addresses multiple risks identified in the OWASP MCP Top 10. It operates as an envelope around existing JSON-RPC messages — analogous to how TLS wraps HTTP — providing agent identity, message integrity, tool authenticity, and replay protection without modifying the core MCP protocol.

- **License**: MIT
- **Dependencies**: Zero (Node.js) / One (Python: `cryptography`)
- **Specification**: [SPEC.md](https://github.com/razashariff/mcps/blob/main/SPEC.md) (2,603 lines)
- **Installation**: `npm install mcp-secure` / `pip install mcp-secure`

## Risk Coverage

MCPS provides mitigations for 8 of the 10 OWASP MCP Top 10 risks:

| MCP Risk | MCPS Mitigation | Mechanism |
|----------|----------------|-----------|
| **MCP01**: Token Mismanagement | Agent Passports replace long-lived tokens with short-lived, cryptographically signed credentials bound to specific key pairs | Passport expiry + key binding |
| **MCP02**: Privilege Escalation | Trust Levels L0-L4 enforce minimum security requirements per server. Capability lists restrict agent permissions. | Trust level gating |
| **MCP03**: Tool Poisoning | Tool definitions are digitally signed by their author. Clients verify signatures before accepting tools. Schema hashing detects silent modifications between sessions (rug pull protection). | ECDSA tool signatures + schema pinning |
| **MCP04**: Supply Chain Attacks | Signed tool definitions with provenance metadata (author passport ID, timestamp) provide cryptographic proof of origin. | Tool signature chain |
| **MCP06**: Intent Flow Subversion | Signed tool descriptions prevent injection of malicious instructions via tampered tool metadata. Changes to signed descriptions are detected and rejected. | Tool integrity verification |
| **MCP07**: Insufficient Auth | Mutual passport-based authentication verifies both client and server identity. Trust Authority revocation enables real-time blacklisting of compromised agents. | Passport verification + revocation |
| **MCP08**: Lack of Audit | Every signed message envelope provides a cryptographic audit trail with non-repudiation. Message signatures bind content to a specific agent identity and timestamp. | Signed envelopes |
| **MCP09**: Shadow Servers | Trust level enforcement rejects connections from unverified agents. Servers can require minimum Trust Level L2+ (Trust Authority-verified passports) to prevent shadow server connections. | Trust level minimum enforcement |

**Not addressed** (application-layer concerns outside MCPS scope):
- **MCP05**: Command Injection — requires input sanitization at the tool implementation level
- **MCP10**: Context Injection — requires content-level inspection beyond transport security

## Technical Architecture

### Agent Passports

Cryptographic identity credentials binding an ECDSA P-256 key pair to an agent identity. Passports are issued by a Trust Authority (self-hostable) and include:

- Agent name and version
- Public key (JWK format)
- Issued/expiry timestamps
- Capability list
- Trust level assignment
- ECDSA signature from the issuing Trust Authority

### Message Signing

Every JSON-RPC message is wrapped in a signed envelope containing:

- Passport ID (binding message to agent identity)
- Timestamp (for freshness verification)
- Nonce (UUID v4, for replay protection)
- ECDSA signature over the canonical message + metadata

Recipients verify the signature, check the timestamp window (default 300 seconds), and reject replayed nonces.

### Tool Definition Signing

Tool authors sign tool definitions (name, description, inputSchema) with their private key. Clients:

1. Verify the signature against the author's passport
2. Compute and store the schema hash (pin)
3. On subsequent connections, compare schema hashes to detect unauthorized changes

### Trust Levels

| Level | Requirements |
|-------|-------------|
| L0 | No verification (current MCP behavior) |
| L1 | Messages signed, self-signed passports accepted |
| L2 | Messages signed, Trust Authority-verified passports required |
| L3 | L2 + tool definition signatures required |
| L4 | L3 + mutual authentication + real-time revocation checking |

## Self-Hostable Trust Authority

The Trust Authority component has **no external service dependency**. Any organization can operate its own Trust Authority by:

1. Generating an ECDSA P-256 key pair
2. Publishing the public key via HTTPS endpoint, JWK Set, or static configuration
3. Optionally: operating a revocation endpoint (CRL-style or OCSP-style)

This design follows the TLS Certificate Authority model — the protocol defines the interface, not a specific provider.

## References

- [MCPS Specification (SPEC.md)](https://github.com/razashariff/mcps/blob/main/SPEC.md)
- [GitHub Repository](https://github.com/razashariff/mcps)
- [npm Package](https://www.npmjs.com/package/mcp-secure)
- [PyPI Package](https://pypi.org/project/mcp-secure/)
- [Landing Page & Scan Results](https://mcp-secure.dev)
- [MCP SEP Submission](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2395)
- [TapAuth: 41% of MCP Servers Have No Auth](https://tapauth.ai/blog/518-mcp-servers-scanned-41-percent-no-auth)
- [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)

---

*Contributed by Raza Sharif, CyberSecAI Ltd — [contact@agentsign.dev](mailto:contact@agentsign.dev)*
