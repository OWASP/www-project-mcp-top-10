---

layout: col-sidebar
title: "MCP03:2025 - Tool Poisoning"

---

### Description
Schema poisoning occurs when an adversary tampers with the contract or schema definitions that govern agent-to-tool interactions in an MCP ecosystem. Schemas define the shape, types, and semantics of requests and responses — effectively the “language” agents use to call tools. If an attacker can modify a schema (or its metadata) so that a benign-sounding operation maps to a destructive action, agents that trust and follow the schema may inadvertently execute dangerous commands.
Schema attacks are a supply-chain style compromise: the attacker doesn’t exploit a code bug directly, they change the contract so legitimate agents behave incorrectly while passing superficial validation.

### Impact

- Data loss or corruption: benign workflows cause irreversible deletion or alteration.
- Privilege abuse: agents may gain unintended capabilities if schema fields map to higher-risk operations.
- Silent policy bypass: validation checks that match schema constraints may be bypassed because the schema itself is malicious.
- Widespread compromise: a single poisoned schema distributed across many agents/tenants can multiply the blast radius.
- Erosion of trust & auditability: logs and traces will show “valid” actions invoked per contract even though the contract was malicious.

### Is the Application Vulnerable? (Checklist)

Your MCP deployment may be vulnerable if any of the following are true:
- Schemas, manifests, or tool descriptors are fetched dynamically from remote locations without integrity checks.
- There is a writable schema registry or repository that lacks RBAC, code-review, or approvals.
- Schema edits are promoted to production automatically via CI/CD without signed commits or attestations.
- Agents accept and act on schema changes at runtime without operator confirmation.
- There is no provenance or version binding stored with the schema (who changed it, when, why).
- No testing or contract verification exists that asserts semantic invariants (e.g., archive must not map to DELETE).

If schemas are treated as configuration files that can be changed without formal governance, treat them as a high-value attack vector.

### How to Prevent (Controls & Best Practices)

1. Signed Schemas & Manifest Integrity
- Digitally sign schemas and tool manifests (e.g., JWS / COSE / PKI-backed signatures). Agents must verify signatures before accepting or using a schema.
- Use content-addressable identifiers (hashes) for schema versions and validate against trusted hashes.

2. Immutable Schema Registry & Version Control
- Store schemas in an immutable version-controlled system (Git with signed commits) or an append-only ledger.
- Enforce branch protections, required code review, and multi-person approval for schema changes.

3. Strong Access Controls & Separation of Duties
- Apply least-privilege RBAC to the schema registry; separate the role that can propose a change from the role that approves and publishes it.
- Use short-lived tokens for deployment pipelines and require human approvals for critical schema releases.


4. Policy-as-Code for Semantic Constraints
- Encode semantic invariants as policy checks (e.g., using OPA/Rego): archive actions cannot map to HTTP DELETE unless explicitly approved.
- Run these policy checks in CI and in a runtime policy decision point (PDP) before execution.

5. Schema Provenance & Metadata
- Each schema/version should include provenance metadata: author, signature, hash, timestamp, and approved-by.
- Agents should log the schema hash and provenance metadata used for each invocation for audit and forensic purposes.

6. Runtime Enforcement & Guardrails
- Don’t allow agents to interpret schema changes as immediate action drivers without revalidation.
- Require a “schema attestation” that binds the schema hash to a specific agent identity and session.
- Implement runtime sanity checks: if an operation’s semantic impact exceeds a threshold (e.g., destructive verbs, data volume), pause execution and require human approval.

### Remediation

- Revoke or block the promoted schema version (remove from registry or mark as compromised).
- Roll back agents to the last known-good schema hash and force revalidation.
- Rotate any tokens or credentials that may have been abused.
- Conduct forensic analysis: which agents used the poisoned schema, what actions executed, which data changed or was removed.
- Patch CI/CD and registry processes to require signed commits and multi-party approvals where missing.

### Example Attack Scenarios

#### Scenario 1 — Compromised CI Pipeline Promotes Malicious Schema
 An attacker compromises a CI/CD runner used to publish schemas and pushes a malicious schema that remaps archive to DELETE. Because the registry auto-promotes approved jobs, agents across production begin issuing destructive calls.

#### Scenario 2 — Dependency Supply-Chain Tampering
 A dependency providing tool manifests is trojaned. When consumers fetch manifests during startup, they ingest tampered schemas that alter semantics for a widely used tool.

#### Scenario 3 — Insider Abuse via Registry Write Access
 An insider with write access to the schema registry modifies a schema to escalate abilities of a specific agent, enabling unauthorized data access and exfiltration.

#### Scenario 4 — Man-in-the-Middle Rewriting Schemas in Transit
 Schemas served over unsecured channels are rewritten in transit by an attacker (or misconfigured proxy), altering operation verbs so that benign requests become destructive.
References & Further Reading



