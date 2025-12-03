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
- 

### [Make suggestions on Github ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP07-2025%E2%80%93Insufficient-Authentication%26Authorization.md)



