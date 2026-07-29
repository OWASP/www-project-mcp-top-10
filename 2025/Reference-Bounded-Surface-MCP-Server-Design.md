---

layout: col-sidebar
title: "Reference: Bounded-Surface MCP Server Design"

---

> **Document Type:** Reference / Appendix — This is not a new Top 10 entry. It describes a defensive design pattern that structurally mitigates multiple OWASP MCP Top 10 categories at design time.

---

### Overview

The **Bounded-Surface Isolation** pattern describes MCP servers that expose only a tightly scoped set of domain-specific tools, deliberately omitting capabilities such as content retrieval, filesystem access, web browsing, and arbitrary command execution. The goal is to eliminate entire vulnerability classes structurally — at design time — rather than relying exclusively on runtime detection and mitigation.

This document defines the pattern, maps it to the OWASP MCP Top 10, provides a concrete reference implementation (AgentPay MCP), and offers an actionable design checklist for MCP server authors.

---

### 1. Pattern Description: Bounded Surface Isolation

#### Definition

A **bounded-surface MCP server** is one that:

1. Exposes **only** the tools required to accomplish its specific domain function.
2. Provides **no** content retrieval, web browsing, or arbitrary read access to external data sources.
3. Provides **no** filesystem read/write capability.
4. Provides **no** shell, subprocess, or operating system command execution.
5. Returns **only** structured data (e.g., JSON) — never arbitrary freeform text sourced from untrusted external content.
6. Enforces authorization **independent of the AI context window** — i.e., business rules are validated outside the LLM, not delegated to model reasoning.

#### Why It Matters

Most MCP security controls operate at runtime: input validation, output filtering, anomaly detection, policy enforcement. These controls are necessary and important. However, they share a common dependency: they must be applied correctly at every execution, for every tool call, across every session.

Bounded-surface design eliminates the dependency by removing capabilities entirely. A server that has no `read_file` tool cannot be exploited via filesystem traversal — regardless of what an attacker injects into the AI context. A server that has no shell tool cannot be leveraged for command injection — regardless of how the model interprets a prompt.

This is the distinction between **design-time elimination** and **runtime mitigation**. Both are required in a mature security posture; bounded surface handles the former.

---

### 2. OWASP MCP Top 10 Category Mapping

The table below shows how bounded-surface design affects each relevant Top 10 category.

| Category | Bounded Surface Impact |
|---|---|
| **MCP03:2025 — Tool Poisoning** | **Reduced surface.** No content-retrieval tools means there are no conduits through which adversarially crafted content can be passed into tool parameters or model context. The attack surface for schema/tool poisoning shrinks to the payment domain only. |
| **MCP05:2025 — Command Injection & Execution** | **Eliminated.** No shell, subprocess, or exec-equivalent tools are exposed. There is no code path by which an injected payload could reach an OS command interpreter. |
| **MCP06:2025 — Intent Flow Subversion** | **Reduced.** Domain-scoped tools provide a small, auditable action space. An attacker attempting to subvert agent intent must work within 7 enumerated payment operations rather than a broad, general-purpose API surface. |
| **MCP09:2025 — Shadow MCP Servers** | **Reduced.** A bounded server cannot be hijacked to perform arbitrary actions outside its domain. Even if a shadow server is registered that mimics the bounded server, the actions available are limited to the payment domain. |
| **MCP10:2025 — Context Injection & OverSharing** | **Eliminated** (for the content-injection vector). No tool returns arbitrary text sourced from external, attacker-controlled content. All responses are structured JSON produced by internal business logic — there is no pass-through of external content into the AI context window. |

---

### 3. Reference Implementation: AgentPay MCP

**AgentPay MCP** (Patent Pending) is an open-source MCP server that implements the bounded-surface pattern for AI agent payment operations. It is provided here as a concrete, inspectable reference implementation. The pattern itself is universal and not specific to payments.

**Package:** `agentpay-mcp` (npm) — https://www.npmjs.com/package/agentpay-mcp  
**SDK:** `agentwallet-sdk` — https://www.npmjs.com/package/agentwallet-sdk

#### 3.1 Tool Inventory (Complete)

The server exposes exactly seven tools. No others exist in the codebase.

| Tool | Domain Function |
|---|---|
| `deploy_wallet` | Provision a smart-contract agent wallet with a `SpendingPolicy` |
| `get_wallet_info` | Return wallet address, balance, and policy metadata |
| `send_payment` | Execute an on-chain payment within policy limits |
| `check_spend_limit` | Query remaining spend capacity for the current period |
| `queue_approval` | Submit a transaction for human approval when limits are exceeded |
| `x402_pay` | Fulfill an HTTP 402 Payment Required challenge via the x402 protocol |
| `get_transaction_history` | Return structured transaction records for a wallet |

Every tool accepts structured input, performs a bounded domain operation, and returns structured JSON. No tool fetches arbitrary URLs, reads files, executes commands, or returns externally sourced freeform text.

#### 3.2 Absent Capabilities (By Design)

The following capabilities are **intentionally absent** from AgentPay MCP:

- No `read_file` / `write_file` / `list_directory`
- No `execute_command` / `run_shell` / `spawn_process`
- No `fetch_url` / `browse_web` / `search_web`
- No `read_email` / `read_document` / `query_database` (beyond own ledger)
- No generic `eval` or code execution path

This is not an omission — it is a deliberate security boundary. A conformant bounded-surface server treats these capabilities as out-of-scope for its domain and does not implement them.

#### 3.3 Transport and Process Isolation

AgentPay MCP communicates exclusively via the **stdio transport** (standard input/output), which means:

- It runs as a **separate OS process**, not as an in-process library loaded into the agent runtime.
- There is **no shared memory** between the MCP server and the host agent process.
- Process isolation is enforced by the OS, not by application code.
- Compromise of the MCP server process does not directly grant access to the agent's context window, memory, or other loaded MCP servers.

This satisfies the MCP specification's process isolation recommendation and is consistent with the least-privilege principle at the process boundary.

#### 3.4 Authorization Independent of AI Context

The server's `SpendingPolicy` is encoded in a **smart contract** (on-chain logic) that validates every payment transaction independently. The enforcement chain is:

```
AI Agent → agentpay-mcp (tool call) → SpendingPolicy smart contract → on-chain execution
```

Critically, the smart contract does **not** consult the AI context window, the model's reasoning, or the tool call parameters as a source of policy truth. The policy is defined at wallet deployment time, stored on-chain, and enforced by the contract at execution time.

This means that even if an attacker successfully manipulates the AI into requesting a payment that violates policy (e.g., via prompt injection), the **smart contract will reject the transaction** regardless of what the AI intended or what parameters were passed.

#### 3.5 CVE-2026-31841 (ContextCrush) Immunity Analysis

**CVE-2026-31841** (ContextCrush, disclosed by Noma Security) describes an attack class in which maliciously crafted content retrieved into the AI context window is used to overwrite, suppress, or manipulate prior instructions — effectively "crushing" the context with adversarial content.

AgentPay MCP is structurally immune to ContextCrush for the following reasons:

1. **No content retrieval.** The server has no tool that fetches external content into the AI context. There is no vector by which attacker-controlled text can be introduced into the context window through this server.

2. **Structured responses only.** All tool responses are JSON objects with a defined schema. A response cannot contain freeform text that could masquerade as system instructions.

3. **No pass-through semantics.** No tool accepts a URL, file path, or query and returns its raw content. All tools perform bounded computations on internal state.

4. **On-chain policy enforcement.** Even if ContextCrush succeeded in manipulating the AI's interpretation of its own policy, the on-chain `SpendingPolicy` would still reject out-of-bounds transactions.

Immunity is structural, not behavioral — it does not depend on the model's ability to resist the attack.

---

### 4. Design Checklist for Bounded-Surface MCP Servers

The following checklist can be applied to any MCP server during design review, code review, or security audit. A server that satisfies all items is a conformant bounded-surface implementation.

**Scope**
- [ ] The server exposes only tools required for its specific domain function
- [ ] The complete tool inventory has been documented and reviewed
- [ ] Tools added in future releases go through a domain-scope review gate

**Prohibited Capabilities**
- [ ] No content retrieval or web browsing tool is present
- [ ] No filesystem read or write tool is present
- [ ] No shell, subprocess, or OS command execution tool is present
- [ ] No generic code evaluation or dynamic execution path is present

**Response Shape**
- [ ] All tool responses are structured data (JSON with defined schema)
- [ ] No tool returns raw external content (URL body, file contents, search results)
- [ ] Response schemas are documented and validated at runtime

**Naming and Discoverability**
- [ ] Tool names are domain-specific (e.g., `send_payment`, not `send` or `execute`)
- [ ] Tool names do not collide with common utilities or generic-sounding operations
- [ ] Tool descriptions accurately describe scope limitations, not just capabilities

**Isolation**
- [ ] Server runs as a separate OS process (stdio or equivalent isolated transport)
- [ ] No shared memory or shared runtime with the host agent process
- [ ] Server process runs with minimal OS-level permissions (least privilege)

**Authorization**
- [ ] Authorization enforcement is implemented outside the AI context window
- [ ] Policy is not delegated to model reasoning (i.e., model cannot override policy by reasoning about it)
- [ ] Authorization logic has its own test suite, independent of AI behavior tests

**Review**
- [ ] A threat model exists that explicitly lists what is out of scope for this server
- [ ] The out-of-scope list is reviewed when new tools are proposed

---

### 5. Complementary Pattern: Runtime Execution Enforcement

Bounded-surface design is a **design-time** control. It reduces the attack surface that runtime controls must defend.

The complementary pattern — **Runtime Execution Enforcement** — operates at tool dispatch time and covers:

- Schema validation of tool call parameters before execution
- Parameter integrity verification (type, range, format, allowlist)
- Dispatch-time policy checks (rate limits, session context, caller identity)
- Audit logging of all tool invocations with structured metadata
- Anomaly detection on tool call sequences

A robust MCP security posture applies both patterns: bounded surface to eliminate structural attack classes, and runtime enforcement to harden the remaining surface.

> **Community Contribution Note:** A detailed section on runtime execution enforcement patterns for MCP servers is forthcoming from a community contributor (@mindburnlabs). That section will complement this reference document and provide implementation guidance for dispatch-time controls.

---

### 6. References

- **CVE-2026-31841 (ContextCrush)** — Noma Security disclosure. Prompt injection via context window manipulation in MCP-connected agents.
- **agentpay-mcp** — Reference implementation (npm): https://www.npmjs.com/package/agentpay-mcp
- **agentwallet-sdk** — Agent wallet SDK (npm): https://www.npmjs.com/package/agentwallet-sdk
- **x402 Protocol** — HTTP 402 Payment Required for AI agents: https://x402.org
- **MCP03:2025 — Tool Poisoning**: https://owasp.org/www-project-mcp-top-10/
- **MCP05:2025 — Command Injection & Execution**: https://owasp.org/www-project-mcp-top-10/
- **MCP06:2025 — Intent Flow Subversion**: https://owasp.org/www-project-mcp-top-10/
- **MCP09:2025 — Shadow MCP Servers**: https://owasp.org/www-project-mcp-top-10/
- **MCP10:2025 — Context Injection & OverSharing**: https://owasp.org/www-project-mcp-top-10/
- **Model Context Protocol Specification**: https://modelcontextprotocol.io

---

### [Suggest edits or additions on GitHub](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/Reference-Bounded-Surface-MCP-Server-Design.md)
