---
title: Example
layout:  null
tab: true
order: 1
tags: example-tag
---

## OWASP Top 10 for Model Context Protocol version 0.1

## MCP1:2025 – Token Mismanagement & Secret Exposure
Hard-coded credentials, long-lived tokens, and secrets stored in model memory or protocol logs can expose sensitive environments to unauthorized access.Attackers may retrieve these tokens through prompt injection, compromised context, or debug traces, leading to full compromise of connected systems. Short-lived, scoped credentials and secret-scanning controls are essential to mitigate this risk.

## MCP2:2025 – Privilege Escalation via Scope Creep
Temporary or loosely defined permissions within MCP servers often expand over time, granting agents excessive capabilities. An attacker exploiting weak scope enforcement can perform unintended actions such as repository modification, system control, or data exfiltration. Enforce least-privilege design, automated scope expiry, and strict access reviews to prevent escalation.

## MCP3:2025 – Insecure MCP Protocol Implementations
Unverified or custom MCP server implementations frequently omit security validations such as authentication, schema enforcement, or command validation. This can result in remote code execution, unauthorized tool access, or schema manipulation. Adhering to secure implementation guidelines, version validation, and protocol conformance testing reduces these exposures.

## MCP4:2025 – Supply Chain & Dependency Tampering
MCP ecosystems depend on open-source packages, connectors, and model-side plug-ins that may contain malicious or vulnerable components. A compromised dependency can alter agent behavior or introduce execution-level backdoors. Implement signed components, dependency monitoring, and provenance tracking for all MCP modules.

## MCP5:2025 – Insecure Plugin Integrations Description MCP (Model Context Protocol) 
MCP Environments often extend their functionality through plugins—modular components that expose new tools, APIs, or context sources to agents. These plugins share the same MCP session or context, enabling them to exchange data, access logs, and communicate with the agent through a unified interface. When multiple plugins operate within a shared context without isolation or access boundaries, data and execution flows can inadvertently overlap. A malicious or poorly designed plugin can monitor or intercept requests from others, extract sensitive data, or alter outputs, leading to cross-plugin data leakage, privilege escalation, or malicious code execution.

## MCP6:2025 – Inadequate Authentication & Authorization
Weak identity management or missing authorization checks in tool-server interactions can allow impersonation or privilege misuse. Attackers may replay tokens, forge session identifiers, or impersonate trusted servers. Mutual authentication, token binding, and zero-trust service design are critical defenses.

## MCP7:2025 – Schema Poisoning
Schema poisoning occurs when an adversary tampers with the contract or schema definitions that govern agent-to-tool interactions in an MCP ecosystem. Schemas define the shape, types, and semantics of requests and responses — effectively the “language” agents use to call tools. If an attacker can modify a schema (or its metadata) so that a benign-sounding operation maps to a destructive action, agents that trust and follow the schema may inadvertently execute dangerous commands.

## MCP8:2025 – Lack of Audit and Telemetry
Without comprehensive activity logging and real-time alerting, unauthorized actions or data access may go undetected.
Limited telemetry from MCP servers and agents impedes investigation and incident response. Maintain detailed logs of tool invocations, context changes, and user-agent interactions with immutable audit trails.

## MCP9:2025 – Shadow MCP Servers
“Shadow MCP Servers” refer to unapproved or unsupervised deployments of Model Context Protocol instances that operate outside the organization’s formal security governance. Much like Shadow IT, these rogue MCP nodes are often spun up by developers, research teams, or data scientists for experimentation, testing, or convenience—frequently using default credentials, permissive configurations, or unsecured APIs.

## MCP10:2025 – Context Injection & Over-Sharing
In the Model Context Protocol (MCP), “context” represents the working memory that stores prompts, retrieved data, and intermediate outputs across agents or sessions. When context windows are shared, persistent, or insufficiently scoped, sensitive information from one task, user, or agent may be exposed to another. This phenomenon—known as context over-sharing—turns convenience into a liability.
