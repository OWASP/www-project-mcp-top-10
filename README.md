![OWASP Logo](assets/images/OWASP_Logo.png)

# [OWASP MCP Top 10]([https://owasp.org/www-project-smart-contract-top-10/](https://owasp.org/www-project-mcp-top-10/))

-----

## About the MCP Top 10

As AI systems become increasingly integrated into software supply chains, enterprise applications, and security infrastructure, the need for structured, secure, and interpretable model interaction layers is paramount. The Model Context Protocol (MCP) is emerging as a framework to define the operational, contextual, and behavioral boundaries of AI models. However, with the power and flexibility of MCPs comes a new class of vulnerabilities and attack surfaces that remain underexplored. This OWASP Top 10 for MCP outlines the most critical security concerns arising in the lifecycle of MCP-enabled systems—spanning from model misbinding, context spoofing, and prompt-state manipulation to insecure memory references and covert channel abuse. These risks are amplified in scenarios involving agentic AI, model chaining, multi-modal orchestration, and dynamic role assignment. By mapping the top 10 MCP-related vulnerabilities and offering concrete recommendations for secure design, implementation, and auditing practices, this project aims to equip AI developers, ML engineers, and security practitioners with the insights necessary to build context-aware and attack-resilient AI systems. The OWASP MCP Top 10 will serve as a living document, evolving alongside the pace of AI model capability and protocol innovation—anchored in real-world threats, research findings, and industry feedback.

## Road Map
* MCP01:2025 - [Token Mismanagement & Secret Exposure](2025/MCP01-2025-Token-Mismanagement&Secret-Exposure)
* MCP02:2025 - [Privilege Escalation via Scope Creep](2025/MCP02-2025–Privilege-Escalation-via-Scope-Creep)
* MCP03:2025 - [Tool Poisoning](2025/MCP03-2025–Tool-Poisoning)
* MCP04:2025 - [Software Supply Chain Attacks & Dependency Tampering](2025/MCP04-2025–Software-Supply-Chain-Attacks&Dependency-Tampering)
* MCP05:2025 - [Command Injection & Execution](2025/MCP05-2025–Command-Injection&Execution)
* MCP06:2025 - [Prompt Injection via Contextual Payloads](2025/MCP06-2025–Prompt-InjectionviaContextual-Payloads)
* MCP07:2025 - [Insufficient Authentication & Authorization](2025/MCP07-2025–Insufficient-Authentication&Authorization)
* MCP08:2025 - [ Lack of Audit and Telemetry](2025/MCP08-2025–Lack-of-Audit-and-Telemetry)
* MCP09:2025 - [Shadow MCP Servers](2025/MCP09-2025–Shadow-MCP-Servers)
* MCP10:2025 - [Context Injection & Over-Sharing](2025/MCP10-2025–ContextInjection&OverSharing)

### Overview

| Title | Description |
| -- | -- |
| MCP01 - Token Mismanagement & Secret Exposure | Hard-coded credentials, long-lived tokens, and secrets stored in model memory or protocol logs can expose sensitive environments to unauthorized access. Attackers may retrieve these tokens through prompt injection, compromised context, or debug traces, leading to full compromise of connected systems. |
| MCP02 - Privilege Escalation via Scope Creep| Temporary or loosely defined permissions within MCP servers often expand over time, granting agents excessive capabilities. An attacker exploiting weak scope enforcement can perform unintended actions such as repository modification, system control, or data exfiltration. |
| MCP03 - Tool Poisoning | Tool poisoning occurs when an adversary compromises the tools, plugins, or their outputs that an AI model depends on - injecting malicious, misleading, or biased context to manipulate model behavior. |
| MCP04 - Software Supply Chain Attacks & Dependency Tampering | A compromised dependency can alter agent behavior or introduce execution-level backdoors. |
| MCP05 - Command Injection & Execution | Command injection occurs when an AI agent constructs and executes system commands, shell scripts, API calls, or code snippets using untrusted input whether from user prompts, retrieved context, or third-party data sources without proper validation or sanitization. |
| MCP06 - Prompt Injection via Contextual Payloads | This risk is analogous to classic injection attacks (e.g., XSS, SQLi), but in the MCP world the “interpreter” is the model and the “payload” is text (or any content that becomes text after OCR/processing). Because models are designed to follow natural-language instructions, prompt injection attacks are both powerful and subtle. |
| MCP07 -  Insufficient Authentication & Authorization | Inadequate authentication and authorization occur when MCP servers, tools, or agents fail to properly verify identities or enforce access controls during interactions. Since MCP ecosystems often involve multiple agents, users, and services exchanging data and executing actions, weak or missing identity validation exposes critical attack paths. |
| MCP08 - Lack of Audit and Telemetry | Limited telemetry from MCP servers and agents impedes investigation and incident response. Maintain detailed logs of tool invocations, context changes, and user-agent interactions with immutable audit trails. |
| MCP09 - Shadow MCP Servers | “Shadow MCP Servers” refer to unapproved or unsupervised deployments of Model Context Protocol instances that operate outside the organization’s formal security governance.Much like Shadow IT, these rogue MCP nodes are often spun up by developers, research teams, or data scientists for experimentation, testing, or convenience frequently using default credentials, permissive configurations, or unsecured APIs. |
| MCP10 - Context Injection & Over-Sharing | In the Model Context Protocol (MCP), “context” represents the working memory that stores prompts, retrieved data, and intermediate outputs across agents or sessions. When context windows are shared, persistent, or insufficiently scoped, sensitive information from one task, user, or agent may be exposed to another. This phenomenon known as context over-sharing turns convenience into a liability.|

## Data Sources
--

## Licensing
The OWASP MCP Top 10 document is licensed under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons
Attribution-ShareAlike 4.0 license. Some rights reserved.

## Project Leaders
- [Vandana Verma Sehgal](mailto:vandana.verma@owasp.org) (Twitter: [@infosecvandana](https://x.com/Infosecvandana))
