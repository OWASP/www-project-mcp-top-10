---

layout: col-sidebar
title: "Recommended Control: Verifiable Authorization Lineage"

---

## Overview

Verifiable authorization lineage is a **record-layer** control: every consequential agent action is
bound, in the record itself, to the human-delegated authorization it acted under — so that a party
who does **not** operate the system can re-derive, offline, what the agent was authorized to do and
what it did.

It addresses a gap left open when audit is treated purely as operator-side observability. Structured,
tamper-evident, SIEM-forwarded logs answer *"can **we** see what happened and respond."* They do not
answer *"can an **external** party — an auditor, a counterparty, a regulator, or an incident responder
at a different organization — verify this without trusting us."* Integrity alone does not close the
gap: a record the operator holds, hashes and signs is still verified **against the operator**. In
multi-agent and cross-organizational MCP topologies the relying party is frequently not the operator,
which is exactly where operator-held records lose evidentiary value.

The control is complementary to, not a substitute for, runtime authorization. Runtime controls
**prevent** an out-of-scope action; this control leaves behind portable **proof** of the authority the
action was taken under. Prevent and prove are different properties and neither implies the other.

Four mechanisms, each verifiable from exported bytes plus public trust anchors:

1. **Human-rooted lineage** — every action traces, hash by hash, to an authorization a human granted.
2. **Grantee binding** — the action was performed by the agent the grant *names*, not by any bearer of it.
3. **Offline scope re-evaluation** — the granted scope is re-checked by the verifier, not read back from the operator's decision log.
4. **Graded attribution, reported at the floor** — the record states *what backs* the human attribution, and a verifier never reports a grade stronger than its weakest root.

## Risk Coverage

| MCP Risk | Mitigation | Mechanism |
|----------|------------|-----------|
| **MCP08**: Lack of Audit and Telemetry | Adds independent verifiability to integrity: the record is re-derivable by a party that trusts no operator, and states the residual trust it carries rather than implying none | Human-rooted lineage + graded attribution + append-only hash chain |
| **MCP02**: Privilege Escalation via Scope Creep | Delegated authority is strictly narrowing by construction — effective scope is the intersection down the delegation path — and a sub-delegation that exceeds what its issuer could grant fails verification. Escalation is detectable from the record alone, after the fact | Offline scope re-evaluation + delegator-authority check |
| **MCP07**: Insufficient Authentication & Authorization | Binds an action to the *named* grantee, so a delegation identifier is not usable as a bearer credential; preserves the acting agent, the delegating human and the granted scope in one record rather than across correlated logs | Grantee binding + human-rooted lineage |
| **MCP06**: Intent Flow Subversion | Blast-radius and evidence only: a successful subversion is bounded by the recorded scope, and the action is attributable to the authority it claimed. The injection itself is not detected | Offline scope re-evaluation |

**Not addressed** (separate controls required):

- **MCP01**: Token Mismanagement — credential issuance, storage and rotation are a separate layer. This control consumes identity artifacts, it does not mint them.
- **MCP03**: Tool Poisoning — tool definitions are not in the authorization record; pair with definition pinning.
- **MCP04**: Supply Chain Attacks — records actions, not provenance of servers or packages.
- **MCP05**: Command Injection — requires an execution sandbox at the tool runtime.
- **MCP09**: Shadow MCP Servers — an unregistered server produces no records; discovery and inventory are prerequisite controls.
- **MCP10**: Context Injection & Over-Sharing — payload confidentiality is out of scope; records may be content-addressed rather than carrying content.

## Mechanisms

### 1. Human-rooted lineage

Every non-root record references its authorizing grant by **content hash**, never by a foreign key into
a mutable table, and the reference chain terminates at an authorization designating a human principal.
Verification walks the references and recomputes each hash, so a grant cannot be substituted,
back-dated or re-pointed after the fact. Delegation depth is bounded to keep the walk terminating.

### 2. Grantee binding

The action record's principal MUST equal the grantee named in the authorization it roots in. This is
the difference between *"a valid delegation exists"* and *"this agent acted under it."* Without it, a
delegation identifier functions as a bearer credential: any agent presenting it inherits the authority,
while every relation check in the policy engine still reads green — the identifier is present in the
decision input and is evaluated, just never bound to the actor.

### 3. Offline scope re-evaluation

The granted scope is expressed as a machine-evaluable predicate — action vocabulary, resource clauses,
validity window, aggregate limits — and the verifier **re-evaluates** each action against the effective
scope, intersected down the delegation path. The permit decision is re-derived, not read back from the
operator's assertion that it was made.

**What a passing scope verdict does not assert.** It proves the action conformed to the predicate **as
recorded**. It does not assert that the predicate encoded the boundary the deployment intended. A
predicate that is correctly evaluated but under-constrained — resolving an object by global identifier
without binding it to the grantor's tenant, or computing the relation over a caller-supplied value —
yields a passing verdict on an action a correct runtime would have refused. Deployments MUST read a
pass as *"the action fell inside the recorded grant,"* never as *"the grant was correctly drawn."*

### 4. Graded attribution, reported at the floor

This is the mechanism we would most encourage implementers to adopt independently of any particular
protocol, because it is cheap and it is measurably absent from deployed identity platforms.

> An audit record SHOULD carry a machine-checkable statement of **what backs the human attribution** —
> operator assertion, a signature by a key under the principal's control, or a qualified
> identity-proofed signature — and a verifier SHOULD report it alongside the verdict. Where a record
> covers multiple authorizations, the reported grade SHOULD be the **weakest** present.

The second sentence is the anti-evasion clause. Without it, one strongly-bound grant masks a series of
unbacked ones inside the same tamper-evident log, and the record reads as a stronger claim than the
evidence supports.

The motivation is empirical. Naming a principal in a credential is **not** evidence that the principal
consented: on at least one major identity platform, a delegation created administratively — with no
prompt ever shown to the named human — yields access tokens indistinguishable in every identity claim
from one produced by an interactive, MFA-backed consent. A grading statement separates those two; a
record that merely names a principal does not.

### 5. Append-only structure

Records are hash-chained with a per-scope monotonic sequence, each record committing its predecessor.
Position in history is committed **inside** the hash preimage, so modification, insertion, deletion and
reordering are each detectable by recomputation. Corrections are new entries, never edits. An optional
RFC 3161 timestamp over a Merkle root of a covered range provides independent temporal evidence.

## Limitations

- **Capture completeness is not a property of the record.** No record system can self-certify that every action reached it; coverage is an instrumentation property.
- **Refused actions may leave no trace.** A record layer that emits on action does not, by itself, evidence what was *blocked*. Deny and step-up records are a distinct and largely unstandardized concern.
- **Identity issuance is out of scope.** The control grades and consumes identity artifacts; it does not establish who a keyholder legally is.
- **Runtime enforcement is out of scope.** See §3 above — this control records and re-checks predicates, it does not author them.

## Reference Implementation

**VAL (Verifiable Authorization Lineage)** — an open, vendor-neutral protocol specifying this control,
with a runnable offline verifier. Apache-2.0, zero runtime dependencies, transport-agnostic.

- Specification: <https://github.com/val-protocol/verifiable-authorization-lineage>
- Offline verifier: [`@val-protocol/chain-verifier`](https://www.npmjs.com/package/@val-protocol/chain-verifier)
- CLI: [`@val-protocol/chain-verifier-cli`](https://www.npmjs.com/package/@val-protocol/chain-verifier-cli)

```bash
npx @val-protocol/chain-verifier-cli --export=./chain.ndjson
# per-record PASS/FAIL, re-derived locally; no call to the operator at verification time
```

Offered as one concrete implementation of the control, not as a product recommendation. The mechanisms
above are protocol-independent and implementable in any append-only record format.
