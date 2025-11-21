---
title: Top10
layout:  null
tab: true
order: 2
tags: mcptopten
---

## OWASP Top 10 for Model Context Protocol version [v0.1]

## MCP1:2025 – [Token Mismanagement & Secret Exposure](2025/MCP01-2025-Token-Mismanagement-and-Secret-Exposure)
Hard-coded credentials, long-lived tokens, and secrets stored in model memory or protocol logs can expose sensitive environments to unauthorized access.Attackers may retrieve these tokens through prompt injection, compromised context, or debug traces, leading to full compromise of connected systems. Short-lived, scoped credentials and secret-scanning controls are essential to mitigate this risk.

## MCP2:2025 – [Privilege Escalation via Scope Creep](2025/MCP02-2025–Privilege-Escalation-via-Scope-Creep)
Temporary or loosely defined permissions within MCP servers often expand over time, granting agents excessive capabilities. An attacker exploiting weak scope enforcement can perform unintended actions such as repository modification, system control, or data exfiltration. Enforce least-privilege design, automated scope expiry, and strict access reviews to prevent escalation.

## MCP03:2025 – [Tool Poisoning](2025/MCP03-2025–Tool-Poisoning)
Tool poisoning occurs when an adversary compromises the tools, plugins, or their outputs that an AI model depends on - injecting malicious, misleading, or biased context to manipulate model behavior. This category encompasses several sub-techniques, including rug pulls (malicious updates to trusted tools), schema poisoning (corrupting interface definitions to mislead the model), and tool shadowing (introducing fake or duplicate tools to intercept or alter interactions).

## MCP4:2025 – [Software Supply Chain Attacks & Dependency Tampering](2025/MCP04-2025–Software-Supply-Chain-Attacks&Dependency-Tampering)
MCP ecosystems depend on open-source packages, connectors, and model-side plug-ins that may contain malicious or vulnerable components. A compromised dependency can alter agent behavior or introduce execution-level backdoors. Implement signed components, dependency monitoring, and provenance tracking for all MCP modules.

## MCP5:2025 – [Command Injection & Execution](2025/MCP05-2025–Command-Injection&Execution)
Command injection in MCP environments occurs when an AI agent constructs and executes system commands, shell scripts, API calls, or code snippets using untrusted input—whether from user prompts, retrieved context, or third-party data sources—without proper validation or sanitization.

## MCP6:2025 – [Prompt Injection via Contextual Payloads](2025/MCP06-2025–Prompt-InjectionviaContextual-Payloads)
This risk is analogous to classic injection attacks (e.g., XSS, SQLi), but in the MCP world the “interpreter” is the model and the “payload” is text (or any content that becomes text after OCR/processing). Because models are designed to follow natural-language instructions, prompt injection attacks are both powerful and subtle.

## MCP07:2025 – [Insufficient Authentication & Authorization](2025/MCP07-2025–Insufficient-Authentication&Authorization)
Inadequate authentication and authorization occur when MCP servers, tools, or agents fail to properly verify identities or enforce access controls during interactions. Since MCP ecosystems often involve multiple agents, users, and services exchanging data and executing actions, weak or missing identity validation exposes critical attack paths.

## MCP8:2025 – [Lack of Audit and Telemetry](2025/MCP08-2025–Lack-of-Audit-and-Telemetry)
Without comprehensive activity logging and real-time alerting, unauthorized actions or data access may go undetected.
Limited telemetry from MCP servers and agents impedes investigation and incident response. Maintain detailed logs of tool invocations, context changes, and user-agent interactions with immutable audit trails.

## MCP9:2025 – [Shadow MCP Servers](2025/MCP09-2025–Shadow-MCP-Servers)
“Shadow MCP Servers” refer to unapproved or unsupervised deployments of Model Context Protocol instances that operate outside the organization’s formal security governance. Much like Shadow IT, these rogue MCP nodes are often spun up by developers, research teams, or data scientists for experimentation, testing, or convenience—frequently using default credentials, permissive configurations, or unsecured APIs.

## MCP10:2025 – [Context Injection & Over-Sharing](2025/MCP10-2025–ContextInjection&OverSharing)
In the Model Context Protocol (MCP), “context” represents the working memory that stores prompts, retrieved data, and intermediate outputs across agents or sessions. When context windows are shared, persistent, or insufficiently scoped, sensitive information from one task, user, or agent may be exposed to another. This phenomenon—known as context over-sharing—turns convenience into a liability.
