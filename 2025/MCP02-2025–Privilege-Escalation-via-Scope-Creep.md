---

layout: col-sidebar
title: "MCP02:2025 - Token Mismanagement and Secret Exposure"

---

### Description:
Scope creep occurs when temporary or narrowly scoped permissions granted to an MCP agent or tool are expanded over time—intentionally for convenience or accidentally through configuration drift—until the agent holds broad or administrative privileges.

Because MCP deployments frequently connect models to multiple systems (repositories, cloud APIs, ticketing, CI/CD), small, cumulative scope increases can transform a low-risk automation into a high-impact attack surface. Scope creep is especially dangerous in agentic systems because agents act autonomously: an over-privileged agent can make unlabeled changes, trigger deployments, or access sensitive data without human review.



### Impact:
Exposure of authentication tokens can lead to:
-Unauthorized modifications to code, infrastructure-as-code (IaC) manifests, or production configuration.
-Unreviewed deployments and potential introduction of backdoors or vulnerabilities.
-Full environment control when privileges allow service account impersonation, creation of new credentials, or management of identity resources.
-Regulatory and compliance exposure due to uncontrolled data access or change history gaps.
-Amplified incident blast radius because agents often have automated, repeatable execution paths.


### How to Detect?
Your MCP deployment may be vulnerable if any of the following are true:
-Permissions are modified manually in development or prod without automated change logs.
-Service/agent accounts are shared across teams or sessions (no per-agent identity).
-There is no enforced expiration for scopes or tokens.
-Ad-hoc testing changes are promoted to production without approval gates.
-There is limited visibility into which agent invoked which action (weak or missing attribution).
-No automated entitlement/permission review process exists.


### Remediation:

1. Least Privilege by Design
Define minimal permissions required per agent before deployment. Document intended actions and map them to explicit scopes.
Use fine-grained scopes (e.g., repo:write:branch=feature/* rather than repo:write).
2. Policy-as-Code & Automated Enforcement
Encode permission policies as code (Rego, OPA, IAM policies in Terraform) and enforce them in CI/CD pipelines.
Reject configurations that violate policy rules during PR checks.
3. Expiry-Based & Just-in-Time (JIT) Access
Issue time-limited scopes/tokens for sessions. Require revalidation for long-running or recurring tasks.
Use JIT elevation workflows with approval gates for any higher-risk action.
4. Per-Agent Identity & Credential Binding
Assign unique identities to agents and bind credentials to the agent and session context (no shared global service accounts).
Use token binding or attestation to prevent credential reuse outside the intended session.
5. Automated Entitlement Reviews & Drift Detection
Periodically (and on change) run entitlement audits to find scope expansions.
Alert on permission increases and requires a documented justification and approval.
6. Runtime Controls & Guardrails
Implement runtime policy enforcement (PDP/PIP) to block disallowed commands or tool calls.
Apply action whitelists, safe execution sandboxes, and require multi-step confirmation for high-impact operations.
7. Strong Change Management & Audit Trails
All permission changes must be tracked, reviewed, and linked to a change request or ticket.
Keep immutable, tamper-evident logs tying actions to agent identity and session.
8. Separation of Duties & Approval Flows
Separate the authority to grant permissions from the authority to deploy code or change production settings.
Require human-in-the-loop approvals for non-routine privilege grants.


### Example Attack Scenarios:

#### Scenario A — Accidental Escalation → Supply-chain Compromise
 A developer grants repo:write for a temporary test. Later, a malicious contributor creates a crafted PR that the over-privileged agent auto-merges into main. The merged code introduces a dependency that includes a malicious payload; CI deploys it automatically.

#### Scenario B — Credential Harvesting + Escalation
 An attacker discovers an agent's long-lived token in logs. Using that token, they grant the agent additional scopes via an exposed internal API. The agent then creates new service accounts and exfiltrates data to an external endpoint.
#### Scenario C — Automated Policy Bypass
 
 An organization allows unrestricted modifications to agent manifests via an internal tooling endpoint used by developers. An attacker uses social engineering to get temporary access to that tool and updates the manifest to include org:admin, enabling a full takeover.


### References & Further Reading
<<<TBA>>>
