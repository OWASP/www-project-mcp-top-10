---

layout: col-sidebar
title: "Recommended Control: Client-Side Tool Risk Gating for MCP Hosts"

---

## Overview

Client-side tool risk gating is a host-side control pattern: the MCP host (client) vets every tool before the model can see or call it, using only what the existing protocol already exposes. It requires **no cooperation from upstream servers**, no new protocol fields, and no signing infrastructure, so it can be deployed today against any unmodified MCP server. It is complementary to cryptographic approaches such as signed tool definitions: signatures prove *origin* when the upstream adopts them; client-side gating bounds *capability and change* even when the upstream is uncooperative or compromised.

The pattern combines four mechanisms:

1. **Ordered risk scoring** - every tool resolves to an ordered risk level (L0-L5) from observable inputs, with floor rules that short-circuit to the top level on known-dangerous signals.
2. **Hard exposure ceiling** - tools above a configured maximum level are withheld from the model entirely (deny by default, no approval path).
3. **Definition fingerprint pinning (TOFU)** - a content hash of each tool definition is recorded on first sight; a silent redefinition ("rug pull") flips the tool into a blocked re-review state.
4. **Human-in-the-loop call gating** - designated tools pause each invocation for explicit human approval, with the risk accounting for that mitigation made explicit rather than silent.

A working open-source reference implementation exists in Spring AI Playground (Apache-2.0, incubating in the Spring AI Community); concrete classes are linked per mechanism below.

## Risk Coverage

| MCP Risk | Mitigation | Mechanism |
|----------|------------|-----------|
| **MCP02**: Privilege Escalation via Scope Creep | Capabilities resolve to an ordered level via a monotonic max-merge (never silently lowered), and a composition-level ceiling refuses any tool whose level exceeds the configured maximum | Risk scoring + exposure ceiling |
| **MCP03**: Tool Poisoning | Description scan for injection signatures (advisory chip); definition fingerprint detects rug-pull redefinitions and withholds the tool pending human re-review (enforced) | Poisoning scan + TOFU pinning |
| **MCP06**: Intent Flow Subversion | Blast-radius limiting only: per-call human approval on gated tools and the capability ceiling bound what a hijacked plan can reach; the injection itself is not detected | HITL gate + ceiling |
| **MCP08**: Lack of Audit and Telemetry | Every risk computation and gate decision emits a structured signal (server risk, tool risk, floor override, poisoning hit, fingerprint mismatch, composition lifecycle) plus per-call metrics tagged with the resolved level | Risk signal sink |
| **MCP09**: Shadow MCP Servers | No silent auto-discovery (explicit registration only); an unauthenticated unknown-host server floors to the top level; post-connection definition changes are caught by the fingerprint ledger | Floor overrides + TOFU pinning |
| **MCP10**: Context Injection & Over-Sharing | A sends-user-data signal raises the score of data-exfiltrating tools; per-conversation tool selection and exposure modes limit what the model sees | Scoring + exposure controls |

**Not addressed** (separate controls required):

- **MCP01**: Token Mismanagement - credential storage and masking are a separate concern.
- **MCP04**: Supply Chain Attacks - this pattern pins tool *definitions*, not packages or server binaries; pair with dependency and artifact controls.
- **MCP05**: Command Injection - requires an execution sandbox at the tool-runtime level.
- **MCP07**: Insufficient Authentication - transport authentication is a separate layer (for example an OAuth2 resource server in front of the endpoint).

## Mechanisms

### 1. Ordered risk scoring with floor overrides

Each tool resolves to one of six ordered levels (L0 Verified, L1 Safe, L2 Low, L3 Moderate, L4 High, L5 Critical). Two independent scoring models feed the same scale and never mix:

- **Locally-authored tools** score from their *enforced* sandbox posture (network scope, filesystem access including a destructive-operation flag floored at L5, and host class permissions), merged with a **monotonic max**: a capability can raise the level, nothing silently lowers it.
- **External tools** score from what the upstream declares: MCP tool annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`), a side-effect scope, a sends-user-data flag, and a documentation-quality penalty. Server registrations score separately on transport, authentication, catalog trust, and documentation axes.

Floor rules short-circuit the arithmetic on known-dangerous signals: an irreversible verb in a tool name (`delete_`, `drop_`, `purge_`, `wipe_`), a destructive hint without an idempotent hint, or a remote unauthenticated server at an unknown host all pin the result to L5 regardless of the sum. Floors make the scoring resistant to gaming by omission: missing or vague metadata raises the score rather than lowering it.

Reference implementation: [`SandboxPostureCalculator`](https://github.com/spring-ai-community/spring-ai-playground/blob/main/src/main/java/org/springaicommunity/playground/service/tool/policy/SandboxPostureCalculator.java) (enforced posture), [`McpToolPublishRiskCalculator`](https://github.com/spring-ai-community/spring-ai-playground/blob/main/src/main/java/org/springaicommunity/playground/service/mcp/risk/McpToolPublishRiskCalculator.java) and [`McpServerRiskCalculator`](https://github.com/spring-ai-community/spring-ai-playground/blob/main/src/main/java/org/springaicommunity/playground/service/mcp/risk/McpServerRiskCalculator.java) (declared signals + floors).

### 2. Hard exposure ceiling

Scoring only matters if something enforces it. The exposure path carries a maximum risk level; enabling a tool whose resolved level exceeds the ceiling is **refused outright** - a deny with no approval path, checked by ordinal comparison at enable time. The model never sees a tool above the ceiling, which turns the risk level from advice into an access-control decision.

### 3. Definition fingerprint pinning (trust-on-first-use)

The host records a SHA-256 fingerprint of each tool's canonical definition (name, description, input schema) when the tool is first seen. On every recomposition the fingerprint is recomputed and compared:

- **Unchanged** - the tool re-exposes normally.
- **Mismatch** - the tool flips to an `AWAITING_REREVIEW` state and is **withheld from re-exposure** until a human explicitly re-approves it. A rug-pull redefinition therefore cannot ride in on a prior approval.

Two properties keep the control honest. First, a matching fingerprint shows only that the *declared interface* is unchanged, not that backend behavior is; the pin is change *detection*, not proof of trustworthiness, and the human re-review gate is where the trust decision lives. Second, TOFU trusts the first presentation by definition; the scoring floors (mechanism 1) are what confront a hostile first impression.

Reference implementation: [`McpToolHashLedger`](https://github.com/spring-ai-community/spring-ai-playground/blob/main/src/main/java/org/springaicommunity/playground/service/mcp/risk/McpToolHashLedger.java).

### 4. Human-in-the-loop call gating with explicit risk accounting

Tools designated as requiring approval pause **each invocation** for a human decision before the upstream call fires. Two details matter for a defensible design:

- The mitigation is **accounted for explicitly**: a gated tool displays a one-band-lower *effective* level beside its inherent one (for example `L5 -> L4`), so operators see both what the tool *could* do and what the gate reduces it to. Nothing is silently lowered.
- The credit is **conditional**: if the tool's description trips the poisoning scan, the mitigation credit is forfeited and the tool scores at its full inherent level - an attacker cannot use a poisoned description to talk the score down.

### 5. Structured audit signals

Every computation and gate decision emits a structured event (server risk computed, tool risk computed, floor override triggered, poisoning hit, fingerprint mismatch, composition lifecycle) and per-call metrics tagged with the resolved level, so the gating layer is itself auditable (MCP08) rather than a black box.

## Relationship to cryptographic controls

This pattern and signature-based controls (such as the MCPS cryptographic layer) solve adjacent problems and compose well. Signed tool definitions provide provenance and integrity *when the upstream participates*; client-side gating provides capability bounds and change detection *unconditionally*, including against servers that will never adopt signing. A host that implements both gets provenance where available and an enforced ceiling everywhere.

## Limitations

- The fingerprint covers the declared interface (name, description, input schema); upstream annotations are not covered on the live path, so an annotation-only redefinition is a stated blind spot.
- TOFU trusts first sight; only floors and scoring confront a malicious first presentation.
- The pattern does not inspect tool *results* at runtime; instruction injection through retrieved content (MCP06) is bounded, not detected.
- Ratings and ceilings are host-local policy; they do not attest anything to third parties.

## References

- Reference implementation (Apache-2.0): [Spring AI Playground](https://github.com/spring-ai-community/spring-ai-playground)
- Coverage mapping against this Top 10: [OWASP MCP Top 10 Coverage](https://spring-ai-community.github.io/spring-ai-playground/mcp-owasp-top-10/)
- Risk model documentation: [MCP Server Safety](https://spring-ai-community.github.io/spring-ai-playground/mcp-server-safety/)
- Level resolution algorithm: [Safe Tool Specification, Section 10.6](https://spring-ai-community.github.io/spring-ai-playground/safe-tool-specification/#106-risk-level)
- Interface-fingerprint interoperability discussion (MCP community): [modelcontextprotocol #1913 adoption evidence](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/1913#issuecomment-4796914875)

---

*Contributed by Jemin Huh, maintainer of Spring AI Playground (Spring AI Community).*
