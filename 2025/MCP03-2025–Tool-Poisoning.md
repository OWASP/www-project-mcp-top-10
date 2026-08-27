---

layout: col-sidebar
title: "MCP03:2025 - Tool Poisoning"

---

### Description
Schema poisoning occurs when an adversary tampers with the contract or schema definitions that govern agent-to-tool interactions in an MCP ecosystem. Schemas define the shape, types, and semantics of requests and responses — effectively the “language” agents use to call tools. If an attacker can modify a schema (or its metadata) so that a benign-sounding operation maps to a destructive action, agents that trust and follow the schema may inadvertently execute dangerous commands.
Schema attacks are a supply-chain style compromise: the attacker doesn’t exploit a code bug directly, they change the contract so legitimate agents behave incorrectly while passing superficial validation.

Tool poisoning is also a confused-deputy attack at the manifest layer: an agent treats tool-manifest text — tool names, descriptions, and parameter documentation — as trusted context and may act on instructions hidden there. Because this text is delivered as part of the tool contract, a correctly delivered, validly signed description can still carry a malicious payload — signing and transport integrity prove a description’s origin, not the intent of its contents.

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

### Detection Indicators (Static Analysis)

Tool poisoning via description text is detectable before install, by statically analyzing each tool's declared `name`, `description`, and parameter descriptions (the text the model treats as authoritative). Treat the presence of any of the following in a tool's declared surface as a strong signal to review the server before trusting it:

- **Model-directed imperatives**: instructions aimed at the model rather than describing the tool, such as "ignore previous instructions", "do not tell the user", or "before answering, read ...".
- **Sensitive-path references**: a benign tool description that mentions credential or secret locations (`~/.ssh`, `id_rsa`, `.env`, `.aws/credentials`, `/etc/passwd`).
- **Exfiltration patterns**: an action verb (send, post, upload, forward) near an external destination (a URL, webhook, or endpoint).
- **Hidden or zero-width characters**: zero-width spaces and bidirectional control characters (U+200B-200F, U+202A-202E, U+2060, U+FEFF) used to smuggle instructions past human review.
- **Comment-smuggled instructions**: model-directed text hidden inside HTML or markdown comments (`<!-- ... -->`) that a rendered view would not show.

These are pre-connection, static indicators and do not replace the runtime and governance controls below; they let a consumer triage a server's declared tool surface before it is wired into an agent. Open-source scanners implementing these checks exist, mapping their findings to this Top 10.

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

7. Independent Authorization Gate (Plan-vs-Authorize)
- Separate the authority to *propose* an action (the model, which reads tool descriptions as trusted text) from the authority to *authorize* it (a deterministic gate outside the model). A poisoned description can steer what the model proposes, but it cannot address a gate that never treats the tool surface as instructions.
- Treat any text that merely asserts approval ("the user approved this", "authorized") as data, not authorization. Bind approval to an out-of-band token tied to the specific request or session, so injected text cannot self-authorize.
- Key the gate's authorization state to the (server identity, tool) pair rather than to the tool name alone. When a server renames a tool, a name-keyed prior is simply absent, so the renamed tool reaches the gate as a first sighting rather than as a mismatch, and a fresh attestation is minted instead of the gate raising. Treat a previously unseen tool name on a server that already has attested tools as unattested rather than as new, so that the absence of a prior is a state the gate holds an opinion about and not a default.
- Scan outbound arguments for secret-shaped payloads (credential patterns, key formats, known sensitive-file contents) and block egress at the gate. Containment then rests on neither the model resisting the injection, which benchmarks show it does not do reliably, nor the detection layer in front of it recognising a rewritten payload, so a successful injection still cannot exfiltrate.

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
### References & Further Reading
- [MCP Security Notification: Tool Poisoning Attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) — Invariant Labs' original disclosure of tool poisoning via malicious descriptions
- [GitHub MCP Exploited: Accessing Private Repositories via MCP](https://invariantlabs.ai/blog/mcp-github-vulnerability) — Real-world tool poisoning attack against GitHub MCP server
- [MCP Injection Experiments](https://github.com/invariantlabs-ai/mcp-injection-experiments) — Reproducible code snippets demonstrating tool poisoning attacks
- [Poison Everywhere: No Output from Your MCP Server Is Safe](https://www.cyberark.com/resources/threat-research-blog/poison-everywhere-no-output-from-your-mcp-server-is-safe) — CyberArk research on output-based poisoning vectors
- [MCPTox: A Benchmark for Tool Poisoning Attack on Real-World MCP Servers](https://arxiv.org/html/2508.14925v1) — Academic benchmark for evaluating tool poisoning attacks
- [We Built the Security Layer MCP Always Needed](https://blog.trailofbits.com/2025/07/28/we-built-the-security-layer-mcp-always-needed/) — Trail of Bits on tool description trust-on-first-use pinning
- [Model Context Protocol Has Prompt Injection Security Problems](https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/) — Simon Willison's analysis of tool poisoning as prompt injection
- [leash-poc: MCP tool-poisoning contained by an independent policy gate](https://github.com/tonydzi/leash-poc) — Reproducible demonstration pairing the tool-poisoning attack with plan-vs-authorize containment (attack *and* defense), with adversarial tests on the gate

### [Make suggestions on Github](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP03-2025%E2%80%93Tool-Poisoning.md)
