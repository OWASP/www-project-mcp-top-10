---

layout: col-sidebar
title: "MCP07:2025 – Insufficient Authentication & Authorization"

---

### Description
Inadequate authentication and authorization occur when MCP servers, tools, or agents fail to properly verify identities or enforce access controls during interactions. Since MCP ecosystems often involve multiple agents, users, and services exchanging data and executing actions, weak or missing identity validation exposes critical attack paths.

An MCP authorization flow can cross four distinct principals: the user, the MCP client, the MCP server, and the downstream resource server or API. A secure design authenticates and authorizes each boundary without treating one bearer token, session cookie, or unverified identity claim as authority for the entire chain. The audit trail should preserve who requested an action, which client and server handled it, who authorized it, and which downstream identity executed it.

###### Insecure authentication typically manifests as:
- Missing or optional API key or token validation
- Hard-coded shared secrets across agents
- Use of static credentials in configuration files or logs
- Insecure token issuance (no expiry, weak entropy, or non-scoped tokens)
- Acceptance of a token that was not issued for the MCP server
- Token passthrough from the MCP client to a downstream API
- Missing issuer, audience, resource, expiry, or scope validation

###### Authorization flaws occur when:
- Agents or users can perform actions beyond their intended privileges
- Access control checks rely solely on client-side enforcement
- MCP servers trust unverified “caller identity” metadata
- Tool endpoints don’t validate permission scopes per user or agent
- OAuth proxy servers reuse one static upstream client identity without binding consent to the initiating MCP client
- Authorization is applied to one endpoint or transport route but omitted from another
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
- The MCP server accepts a token with the wrong audience or forwards the client's token to an upstream API
- OAuth consent is remembered only for the server's static upstream client ID, allowing a newly registered MCP client to skip meaningful consent
- Redirect URIs are matched by prefix or pattern instead of exact pre-registration
- OAuth authorization requests do not bind the response to the initiating client and browser session with a validated `state` value
- The organization cannot distinguish the requester, authorizer, MCP client, MCP server, and downstream principal in policy decisions and logs
- No expiration or rotation policies for authentication credentials
If you cannot determine “who did what, and with what authority”, your system is already vulnerable.


### How to Prevent (Secure Implementation Guidance)
1. Strong Authentication for All Entities
- Authenticate remote HTTP requests using the MCP authorization specification or another approved mechanism appropriate to the deployment. Use mTLS or other sender-constrained tokens where supported and required by the threat model.
- Use short-lived, scoped tokens tied to the intended resource and permissions.
- Validate issuer, audience, resource, expiry, and scope on every protected request. Never trust unverified client-provided identity claims.
- Reject tokens that were not issued for the MCP server. Do not pass an MCP client token through to a downstream API; obtain a separate downstream token for that resource.

2. Implement Fine-Grained Authorization
- Adopt RBAC (roles) or ABAC (attributes) models: Example: “Agent X may read customer data but not execute tools.”
- Evaluate permissions per request, not per session.
- Deny-by-default: any unrecognized agent or scope should be blocked automatically.

3. Token Lifecycle Management
- Enforce expiration, rotation, and revocation policies for all tokens.
- Store tokens securely (vaulted or encrypted).
- Detect and block replayed or duplicated tokens.
- Keep MCP client grants and downstream credentials separately revocable. Revoking one MCP connection should not require rotating a credential shared by unrelated users or tenants.

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

8. Bind OAuth Clients, Redirects, and Consent
- Register and validate exact redirect URIs. Do not use prefix matching or open redirectors.
- Generate and validate a `state` value for every authorization request and bind it to the initiating MCP client and browser session.
- Prefer Client ID Metadata Documents for client identification on MCP protocol revision `2026-07-28` and newer. When Dynamic Client Registration is supported for backward compatibility, apply the same client identity, redirect, and consent controls.
- When an MCP server proxies authorization to a third-party API using a static upstream client ID, obtain consent for each newly introduced MCP client. A remembered consent cookie for the static upstream client is not sufficient proof of consent for a new MCP client.
- Present the identity of the requesting MCP client and the requested downstream access before authorization.

9. Preserve Identity and Consent Across the Chain
- Record the requester and authorizer separately when they differ.
- Bind policy decisions to the MCP client, MCP server, downstream resource, user or service principal, tenant, and operation.
- Do not replace the user's identity with a shared server identity where downstream authorization or accountability requires user attribution.
- Include the identity chain, consent decision, policy version, and downstream destination in redacted audit records.



### Example Attack Scenarios

#### Scenario 1 – Token Replay Attack
An attacker intercepts an API token used by one MCP agent. Because the token is static and not bound to a specific identity, they reuse it to perform admin-level actions on another server.

#### Scenario 2 – Cross-Agent Privilege Escalation
A misconfigured “Testing” agent has access to the same authorization scope as “Production.” A developer unintentionally executes tool commands against production data, causing a major incident.

#### Scenario 3 – Spoofed Identity in Unverified Agent
A malicious service registers as a fake MCP agent using an unprotected onboarding endpoint. Without certificate validation or signed manifests, it is treated as a legitimate internal agent.

#### Scenario 4 – Inherited Context Tokens
 An assistant agent inherits the parent’s credentials through shared context, allowing it to execute privileged functions intended only for admins.

#### Scenario 5 – OAuth Proxy Confused Deputy
An MCP server uses one static OAuth client ID to connect to a third-party API while allowing MCP clients to register dynamically. A user has previously approved the static upstream client, so the authorization server retains a consent cookie. An attacker registers a new MCP client and initiates authorization through the proxy. If the server does not bind the new client to a fresh consent decision and validate the exact redirect and `state`, the attacker may obtain an authorization code without the user meaningfully approving that client.

#### Scenario 6 – Token Passthrough
An MCP server accepts a bearer token presented by a client without checking that the token was issued for the MCP server. It forwards the token to a downstream API. This bypasses the server's intended request validation, rate limits, and audit boundary, and the downstream service cannot reliably attribute the action to the correct MCP client.

### Detection
- Tokens reused across multiple agents or IP addresses.
- Failed authentication attempts followed by successful privileged actions.
- Actions performed by unknown or unregistered agent IDs.
- Sudden increase in unauthorized “403” responses in logs.
- Tokens used after expiry timestamps.
- Tokens accepted with an unexpected issuer, audience, or resource.
- The same client token observed at both the MCP server and a downstream API.
- New dynamic clients completing authorization without a client-specific consent event.
- Authorization codes or callbacks that cannot be correlated with a validated `state` value and exact redirect URI.


### Immediate Remediation
- Revoke all compromised or static tokens immediately.
- Rotate all service credentials and enforce unique per-agent identities.
- Enable mTLS and strict API key binding.
- Stop token passthrough and issue separate audience-bound tokens for the MCP server and downstream resources.
- Revoke grants issued through unbound OAuth proxy flows and require client-specific consent.
- Audit existing agents, tools, and connectors for excessive privileges.
- Review and patch authorization middleware to enforce scope validation.
- Add temporary compensating controls: IP restrictions, manual approvals for sensitive actions.

### References & Further Reading
- [MCP Specification — Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices) — Official guidance on authentication, authorization, and transport security
- [MCP Security Best Practices: Confused Deputy and Token Passthrough](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) — Official attack descriptions and required mitigations
- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) — Authorization requirements for HTTP-based MCP transports in protocol revision 2026-07-28
- [RFC 9700 — OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700) — Redirect URI, authorization response, and token security guidance
- [MCP Security Vulnerabilities: How to Prevent Prompt Injection and Tool Poisoning](https://www.practical-devsecops.com/mcp-security-vulnerabilities/) — Analysis finding 38% of MCP servers lack authentication entirely
- [Microsoft & Anthropic MCP Servers at Risk of RCE, Cloud Takeovers](https://www.darkreading.com/application-security/microsoft-anthropic-mcp-servers-risk-takeovers) — Authorization bypass leading to cloud account compromise
- [Systematic Analysis of MCP Security](https://arxiv.org/html/2508.12538v1) — Academic analysis of authentication and authorization gaps across MCP implementations
- [Securing the Model Context Protocol: Risks, Controls, and Governance](https://arxiv.org/pdf/2511.20920) — Framework for MCP authentication and governance controls
- [Model Context Protocol Security: Critical Vulnerabilities Every CISO Must Address](https://www.esentire.com/blog/model-context-protocol-security-critical-vulnerabilities-every-ciso-should-address-in-2025) — eSentire analysis of MCP auth boundaries


### [Make suggestions on Github ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP07-2025%E2%80%93Insufficient-Authentication%26Authorization.md)
