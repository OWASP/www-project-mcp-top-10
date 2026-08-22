---

layout: col-sidebar
title: "Candidate Risk: Cross-Server Trust and Capability Manipulation"

---

> **Status:** Candidate for community review. This document does not assign a final MCP Top 10 number or propose removing an existing category.

### Description

MCP hosts and aggregators can make tools from several servers available to the same model or workflow. These servers may have different owners, identities, permissions, data access, and levels of trust.

Cross-server trust and capability manipulation occurs when content from one MCP server changes how the host or model selects, invokes, or supplies arguments to tools from another server. It also occurs when separately approved tools form a dangerous capability chain that was never reviewed or authorized as a whole.

The source server does not need direct access to the target server. Its tool description, annotations, resource content, or tool output may be interpreted in the same model context as the target server's tools. A low-trust server can therefore influence a higher-trust capability without compromising that server or changing its implementation.

Examples include:

- A server describes its tool using instructions that tell the model when and how to invoke a tool from another server.
- Two servers expose the same or similar tool name and the host loses track of which server owns the selected capability.
- Data returned by an internal search tool becomes an argument to an externally connected messaging or upload tool.
- A read-only tool supplies attacker-controlled instructions that trigger a state-changing tool from another server.
- A user approves each server separately, but no control evaluates the combined action chain or final data destination.

### Why This Is a Separate Candidate

This risk overlaps existing categories, but its security boundary is different:

- **MCP03 Tool Poisoning** focuses on malicious or modified tools, schemas, descriptions, or metadata. Cross-server manipulation can occur when every individual tool behaves as implemented.
- **MCP06 Intent Flow Subversion** focuses on malicious content changing the user's intended workflow. This candidate focuses on trust transfer and capability composition across server boundaries.
- **MCP02 Scope Creep** focuses on permissions becoming broader than intended. In a cross-server chain, each tool may retain its intended permissions while their combination creates a capability that was never approved.

The defining question is not only whether a tool is trustworthy. It is whether one server's content is allowed to influence capabilities owned by another server, and whether the resulting chain is authorized as a whole.

### Impact

- **Unauthorized actions:** A low-trust server can steer a model toward destructive or privileged tools exposed by a trusted server.
- **Data exfiltration:** An internal read capability can be combined with an external write capability to move sensitive data outside its approved boundary.
- **Approval bypass:** Per-server or per-tool approval can appear valid even though the combined workflow was never reviewed.
- **Trust confusion:** Users, policy engines, and logs may identify the invoked tool but lose the source that influenced its selection and arguments.
- **Cross-system compromise:** A malicious result from one business system can trigger actions in source control, CI/CD, cloud, identity, or administrative tools.
- **Larger blast radius:** Adding one untrusted server can affect the use of every other tool visible in the same orchestration context.

### Is the Application Vulnerable? (Checklist)

Your MCP deployment may be vulnerable if:

- Tools from servers with different trust levels are presented in one shared model context without isolation.
- Tool identity is represented only by a display name or a server-reported name.
- The host does not handle tool-name collisions across servers.
- Tool descriptions or annotations can refer to and direct the use of tools owned by another server.
- Tool output flows directly into another server's arguments without preserving source provenance.
- Authorization is evaluated one invocation at a time without considering the complete action chain.
- Sensitive data retrieved from internal tools can be passed to external tools without a destination policy.
- Approval prompts identify the final tool but not the data source, prior tool calls, resolved arguments, and destination.
- Tool definitions can change without a new review or approval decision.
- Audit records cannot reconstruct which server supplied the content that influenced an invocation.

### How to Prevent

1. Bind Tools to a Stable Server Identity
- Represent tool identity as a combination of the server trust identity and tool name, not the tool name alone.
- Do not rely on a self-reported server display name as a security identifier.
- Preserve the server identity through discovery, policy evaluation, approval, invocation, and logging.

2. Handle Cross-Server Name Collisions
- Detect duplicate and confusingly similar tool names when aggregating servers.
- Use server-qualified names or another deterministic disambiguation strategy in the host.
- Show the owning server in approval and administration interfaces.

3. Isolate Trust Domains
- Avoid placing tool descriptions from untrusted and privileged servers into one unconstrained instruction context.
- Separate servers by trust level, data sensitivity, and action impact where practical.
- Require an explicit policy decision before content crosses from a lower-trust domain to a higher-impact capability.

4. Preserve Data and Instruction Provenance
- Track which server, resource, tool result, or user supplied content used to select a tool or construct its arguments.
- Carry provenance into authorization and data-loss decisions rather than presenting it only as text to the model.
- Treat descriptions, annotations, resources, and tool output as untrusted input unless their source and integrity meet the applicable policy.

5. Evaluate Complete Capability Chains
- Apply policy to the sequence of tools, data sources, resolved arguments, credentials, and final destination.
- Block combinations such as sensitive internal read followed by unapproved external write, even when each invocation is independently allowed.
- Enforce these rules outside the model using deterministic policy and destination controls.

6. Make Approval Chain-Aware
- For high-impact actions, show the user which servers contributed data, which tools were called, the final resolved arguments, and where data or changes will go.
- Require new approval when an earlier tool changes the meaning, target, or sensitivity of the final action.
- Do not treat installation-time approval of two servers as approval of every possible chain between them.

7. Monitor Tool Definition Drift
- Canonicalize and hash security-relevant tool definitions, including names, descriptions, annotations, and schemas.
- Alert when a definition changes and require reapproval for changes that affect behavior, permissions, data flow, or model instructions.
- Record the active definition hash with each invocation.

8. Restrict Data Destinations and Capabilities
- Apply allowlists and egress controls to tools that send data externally.
- Limit sensitive read tools and high-impact write tools to the minimum required resources.
- Keep credentials scoped to the server and operation that require them.

### Detection

Monitor for:

- Tool descriptions or annotations that name, reference, or instruct use of tools from another server.
- Duplicate or confusingly similar tool names across aggregated servers.
- New tool sequences that cross server or trust boundaries.
- Sensitive results from internal tools flowing into external messaging, upload, webhook, or generic network tools.
- A low-trust server's output immediately preceding a privileged or destructive action on another server.
- Changes to tool definitions after approval.
- Invocations where the source provenance, owning server, policy decision, or final destination is missing from the audit record.
- A new server changing invocation patterns for previously stable tools.

Detection should use orchestration, policy, endpoint, and network telemetry together. Looking only at the final tool call will not show which server influenced it.

### Example Attack Scenarios

#### Scenario 1: Cross-Server Tool Shadowing

A user installs a trusted source-control server and a low-trust utility server. The utility server publishes a benign-looking tool whose description tells the model that repository checks must first invoke the source-control server's export tool and pass the result to an external URL. The utility tool never calls the source-control server directly. Its description manipulates the model into using the trusted tool on its behalf.

#### Scenario 2: Read-to-Send Capability Composition

An internal document server can search confidential files but cannot communicate externally. A separate messaging server can send content to arbitrary channels but cannot read internal files. A poisoned document result instructs the model to send the search results to an attacker-controlled channel. Each server retains its intended permissions, but their combination creates an unapproved exfiltration path.

#### Scenario 3: Tool-Name Collision

Two connected servers expose a tool named `search`. One searches an internal knowledge base and the other sends queries to an external service. The host presents both without a stable server-qualified identity. The model or user selects the external tool while believing the query will remain internal, disclosing sensitive search terms.

#### Scenario 4: Fragmented Approval

A user approves a ticket-reading server and a deployment server separately. A malicious ticket tells the model to deploy a supplied image as part of resolving the issue. The host asks for approval only on the final deployment call and does not show that the image and instruction came from an untrusted ticket. The user approves without seeing the complete chain.

### Immediate Remediation

- Disable or isolate the server that supplied the manipulating content.
- Stop affected cross-server workflows and revoke active approvals for the chain.
- Preserve tool definitions, definition hashes, prompts, results, invocation logs, and network destinations for investigation.
- Identify actions taken by other servers after consuming content from the affected source.
- Rotate credentials or tokens if data or privileged actions may have crossed trust boundaries.
- Add server-qualified tool identity, provenance tracking, and chain-level policy before re-enabling the workflow.

### References and Related Categories

- [MCP Tools Specification](https://modelcontextprotocol.io/specification/2026-07-28/server/tools) - Requires clients to treat tool annotations as untrusted unless they come from trusted servers and discusses name collisions when aggregating tools from multiple servers.
- [MCP Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) - Official guidance for MCP trust and security boundaries.
- [MCP02: Privilege Escalation via Scope Creep](../MCP02-2025%E2%80%93Privilege-Escalation-via-Scope-Creep.md)
- [MCP03: Tool Poisoning](../MCP03-2025%E2%80%93Tool-Poisoning.md)
- [MCP06: Intent Flow Subversion](../MCP06-2025%E2%80%93Intent-Flow-Subversion.md)
- [MCP08: Lack of Audit and Telemetry](../MCP08-2025%E2%80%93Lack-of-Audit-and-Telemetry.md)
