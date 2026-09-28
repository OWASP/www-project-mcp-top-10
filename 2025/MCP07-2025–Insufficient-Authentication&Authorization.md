---

layout: col-sidebar
title: "MCP07:2025 – Insufficient Authentication & Authorization"

---

### Description
Inadequate authentication and authorization occur when MCP servers, tools, or agents fail to properly verify identities or enforce access controls during interactions. Since MCP ecosystems often involve multiple agents, users, and services exchanging data and executing actions, weak or missing identity validation exposes critical attack paths.

###### Insecure authentication typically manifests as:
- Missing or optional API key or token validation
- Hard-coded shared secrets across agents
- Use of static credentials in configuration files or logs
- Insecure token issuance (no expiry, weak entropy, or non-scoped tokens)

###### Authorization flaws occur when:
- Agents or users can perform actions beyond their intended privileges
- Access control checks rely solely on client-side enforcement
- MCP servers trust unverified “caller identity” metadata
- Tool endpoints don’t validate permission scopes per user or agent
- Together, these weaknesses can lead to unauthorized access, privilege escalation, and data compromise—the same class of issues that historically dominated web and API security, now amplified by autonomous, interconnected agents.

### Impact
- Unauthorized actions or data access (e.g., triggering deployment, retrieving confidential data)
- Privilege escalation through token reuse or misconfigured scopes
- Cross-agent impersonation, where one agent acts as another
- Data leakage via over-permissive APIs or shared context tokens
- Service compromise, allowing attackers to chain actions through trusted connectors
- Regulatory & compliance exposure, especially when sensitive data is accessed without audit trails

### Is the Application Vulnerable? (Checklist)

You are likely exposed if any of the following apply:
- MCP servers don’t require mutual authentication between agents and tools
- Tokens or API keys are shared, static, or long-lived
- Authorization decisions rely on client input or context hints rather than server-side checks
- Tools or connectors don’t validate caller identity or scope before execution
- There is no role-based or attribute-based access control (RBAC / ABAC)
- Access logs lack identity correlation between agent and user actions
- Agents can reuse tokens or credentials issued to others
- No expiration or rotation policies for authentication credentials
- Revoked access remains usable through cached authorization decisions or existing sessions beyond the deployment's defined revocation propagation limit
If you cannot determine “who did what, and with what authority”, your system is already vulnerable.


### How to Prevent (Secure Implementation Guidance)
1. Strong Authentication for All Entities
- Require mutual TLS (mTLS) between MCP clients, agents, and servers.
- Use short-lived, scoped tokens (JWT/OAuth2-style) tied to specific sessions and permissions.
- Enforce token binding to agent identity (e.g., signed agent attestation).
- Validate every token on the server side — never trust client-provided claims.

2. Implement Fine-Grained Authorization
- Adopt RBAC (roles) or ABAC (attributes) models: Example: “Agent X may read customer data but not execute tools.”
- Evaluate permissions per request, not per session.
- Deny-by-default: any unrecognized agent or scope should be blocked automatically.

3. Token Lifecycle Management
- Enforce expiration, rotation, and revocation policies for all tokens.
- Store tokens securely (vaulted or encrypted).
- Detect and block replayed or duplicated tokens.
- Define and test the maximum delay between revoking a grant and denying subsequent protected operations at every enforcement point. Invalidate cached authorization decisions or bound their lifetime to meet that limit; an unexpired token or an existing session is not proof that access is still authorized.
- When using token introspection, account for stale cached responses and never cache beyond token expiration. If authorization freshness cannot be established within the defined limit, deny the protected operation rather than continuing to use an old allow decision.

4. Least Privilege Principle
- Minimize agent permissions — assign only what’s needed for the task.
- Split high-privilege operations into separate workflows requiring human review.
- Restrict admin or system tokens from being used in development or shared contexts.

5. Centralized Identity & Access Management
- Integrate MCP authentication with organizational IAM or OIDC providers.
- Require federated identity for all user-driven and system-driven actions.
- Centralize policy enforcement through a Policy Decision Point (PDP).

6. Logging, Monitoring & Auditing
- Log every authentication attempt and authorization decision.
- Detects repeated failed logins, invalid tokens, or cross-tenant token reuse.
- Feed these logs into a SIEM/XDR for anomaly detection and alerting.

7. Secure-by-Default Configurations
- Disable guest or anonymous access in all MCP endpoints.
- Prevent local testing servers from exposing endpoints publicly.
- Enforce environment-specific credentials for dev/test/prod.



### Example Attack Scenarios

#### Scenario 1 – Token Replay Attack
An attacker intercepts an API token used by one MCP agent. Because the token is static and not bound to a specific identity, they reuse it to perform admin-level actions on another server.

#### Scenario 2 – Cross-Agent Privilege Escalation
A misconfigured “Testing” agent has access to the same authorization scope as “Production.” A developer unintentionally executes tool commands against production data, causing a major incident.

#### Scenario 3 – Spoofed Identity in Unverified Agent
A malicious service registers as a fake MCP agent using an unprotected onboarding endpoint. Without certificate validation or signed manifests, it is treated as a legitimate internal agent.

#### Scenario 4 – Inherited Context Tokens
 An assistant agent inherits the parent’s credentials through shared context, allowing it to execute privileged functions intended only for admins.

#### Scenario 5 – Revocation Bypassed by Cached Authorization
In a synthetic deployment, an MCP client is authorized to call a document export tool. An administrator revokes that client's grant, but the MCP server keeps a cached allow decision for the existing session. The client continues requesting exports, and the server executes them using its still-valid downstream credential. The revocation was recorded, yet access continues beyond the deployment's defined propagation limit because enforcement never consults the updated grant state.

### Validating Revocation Enforcement
Use a synthetic resource and a downstream test double that records attempted operations. These checks validate the existing per-request authorization and token lifecycle controls:

1. With a valid grant, complete an authorized tool call to populate session and authorization caches. Confirm that an independent grant can also perform its intended operation.
2. Revoke the first MCP client grant while keeping the downstream test credential valid, and record when revocation is acknowledged. Retry the same protected operation with the previously issued credential, both through the retained session and after reconnecting. Exercise each server instance or worker that can reuse authorization state.
3. Verify that calls made after the defined propagation limit are denied **before downstream execution**, while the independent grant still works. A successful revocation response or an error returned after the tool has already executed is insufficient evidence. Record any successful calls during the propagation window as residual exposure.
4. Where enforcement relies on introspection or a remote policy service, make that dependency unavailable after warming the cache. Once the permitted freshness limit is exceeded, verify that protected calls are denied and do not reach the downstream test double.
5. Correlate the revocation event, request, grant identifier, authorization decision and reason, and downstream outcome in the audit trail. Include the requester, authorizer, MCP client, MCP server, and downstream principal where those identities are available from trusted authentication or authorization context. Use non-secret identifiers; do not record access tokens, refresh tokens, or downstream credentials.
6. Where enforcement resolves current grant state through an authority, keep that authority reachable but make the tested grant's state unresolvable (for example, simulate a missing grant record). Exercise both a request with no cached decision and a previously authorized request after its cached decision exceeds the permitted freshness limit. Verify denial before downstream execution, with no fallback to the old allow decision. Confirm that the independent grant still works, and record the denial reason using the audit checks above.

Revoking a refresh token or deleting a client-side credential does not by itself prove that an already-issued access token is rejected. Document whether enforcement uses online checks, cache invalidation, or token expiration, and measure the resulting delay. These checks cover subsequent calls; cancellation or rollback of work already in progress requires separate application-specific controls.

### Detection
- Tokens reused across multiple agents or IP addresses.
- Failed authentication attempts followed by successful privileged actions.
- Actions performed by unknown or unregistered agent IDs.
- Sudden increase in unauthorized “403” responses in logs.
- Tokens used after expiry timestamps.


### Immediate Remediation
- Revoke all compromised or static tokens immediately.
- Rotate all service credentials and enforce unique per-agent identities.
- Enable mTLS and strict API key binding.
- Audit existing agents, tools, and connectors for excessive privileges.
- Review and patch authorization middleware to enforce scope validation.
- Add temporary compensating controls: IP restrictions, manual approvals for sensitive actions.

### References & Further Reading
- [MCP Authorization Specification — Access Token Usage](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization#access-token-usage) — Authorization and token validation for HTTP requests
- [RFC 7009 — OAuth 2.0 Token Revocation, Sections 2.1 and 3](https://www.rfc-editor.org/rfc/rfc7009.html) — Revocation propagation, related tokens, and implementation trade-offs
- [RFC 7662 — OAuth 2.0 Token Introspection, Section 4](https://www.rfc-editor.org/rfc/rfc7662.html#section-4) — Stale authorization information and cache lifetime constraints
- [MCP Specification — Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices) — Official guidance on authentication, authorization, and transport security
- [MCP Security Vulnerabilities: How to Prevent Prompt Injection and Tool Poisoning](https://www.practical-devsecops.com/mcp-security-vulnerabilities/) — Analysis finding 38% of MCP servers lack authentication entirely
- [Microsoft & Anthropic MCP Servers at Risk of RCE, Cloud Takeovers](https://www.darkreading.com/application-security/microsoft-anthropic-mcp-servers-risk-takeovers) — Authorization bypass leading to cloud account compromise
- [Systematic Analysis of MCP Security](https://arxiv.org/html/2508.12538v1) — Academic analysis of authentication and authorization gaps across MCP implementations
- [Securing the Model Context Protocol: Risks, Controls, and Governance](https://arxiv.org/pdf/2511.20920) — Framework for MCP authentication and governance controls
- [Model Context Protocol Security: Critical Vulnerabilities Every CISO Must Address](https://www.esentire.com/blog/model-context-protocol-security-critical-vulnerabilities-every-ciso-should-address-in-2025) — eSentire analysis of MCP auth boundaries


### [Make suggestions on Github ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP07-2025%E2%80%93Insufficient-Authentication%26Authorization.md)



