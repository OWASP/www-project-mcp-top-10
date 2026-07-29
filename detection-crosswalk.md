---
title: Detection Crosswalk
layout: null
tab: true
order: 5
tags: mcptopten detection crosswalk
---

## Detection Crosswalk: MCP Top 10 → Executable Detection Rules

The MCP Top 10 catalogs **what can go wrong** at the Model Context Protocol layer. This page maps each risk to **what detects it** — categories of runtime detection rules an implementer can run against MCP tool calls, tool responses, and agent I/O.

The mapping below uses [Agent Threat Rules (ATR)](https://github.com/Agent-Threat-Rule/agent-threat-rules) as a worked example: an open, MIT-licensed detection format (YAML rules plus a reference engine, "like Sigma, but for AI agents"). ATR is one implementation of the idea; the crosswalk is written so any executable rule format could occupy the same column. It is a reference for defenders building detection coverage, **not** an endorsement of any single tool by the OWASP project.

### How to read the coverage column

- **Direct** — the risk maps onto a dedicated ATR rule category built for exactly this pattern.
- **Partial** — the risk is a sub-case of a broader category; coverage is real but narrower than the risk's full scope.
- **Detective control, not preventive** — ATR is a detection layer, so for risks about *missing* controls (e.g. audit gaps) the value ATR adds is the telemetry signal itself, not prevention.

### Crosswalk

| MCP Top 10 risk | ATR detection category | Coverage | Notes |
| :-- | :-- | :-- | :-- |
| MCP01 — Token Mismanagement & Secret Exposure | `context-exfiltration` (+ `prompt-injection` for token retrieval via injection) | Direct | Rules match secret/token exposure in tool arguments and responses; the injection path the risk describes is covered by the prompt-injection category. |
| MCP02 — Privilege Escalation via Scope Creep | `privilege-escalation` (+ `excessive-autonomy`) | Direct | Scope-expansion and unintended-action patterns. |
| MCP03 — Tool Poisoning | `tool-poisoning` | Direct | The risk's named sub-techniques (rug pulls, schema poisoning, tool shadowing) are the category's core. |
| MCP04 — Software Supply Chain Attacks & Dependency Tampering | `skill-compromise` | Direct | Compromised/tampered skill and dependency patterns. |
| MCP05 — Command Injection & Execution | `tool-poisoning` (subset) | Partial | Command-execution abuse via tool arguments is covered as a sub-case; not a standalone category. |
| MCP06 — Intent-Flow Subversion | `agent-manipulation` | Direct | Goal-hijacking / intent-redirection patterns. |
| MCP07 — Insufficient Authentication & Authorization | `privilege-escalation` (subset) | Partial | Detection surfaces the *consequences* of weak authz (unauthorized actions); it does not verify the auth mechanism itself. |
| MCP08 — Lack of Audit and Telemetry | — | Detective control, not preventive | ATR does not prevent this risk; a rule engine watching tool calls *is* a telemetry source, so deploying detection is itself a partial mitigation for the audit gap. |
| MCP09 — Shadow MCP Servers | `tool-poisoning` + `skill-compromise` | Partial | The tool-shadowing / rogue-tool patterns are covered; discovery of unregistered servers at the network layer is out of scope for a content-detection ruleset. |
| MCP10 — Context Injection & Over-Sharing | `prompt-injection` + `context-exfiltration` | Direct | Injected context and over-shared data patterns. |

### Notes on scope and honesty

- **Coverage is not 10/10 by design.** Two risks map only partially (MCP05, MCP07, MCP09) and one is a detective-not-preventive control (MCP08). A crosswalk that claimed full coverage would be misleading; detection complements, it does not replace, the preventive controls each MCP Top 10 entry recommends.
- **Rule counts change.** Category-to-risk mapping is stable, but the number of rules in each category moves as the ruleset evolves. For an authoritative, current count, see the linked ATR repository rather than a number frozen in this page.
- **Format-neutral.** The ATR column is a concrete, runnable example. The intent of this page is to give MCP Top 10 readers a template for "risk → executable detection," which any comparable open rule format can fill.
