---

layout: col-sidebar
title: "MCP10:2025 – Context Injection & Over-Sharing"

---

#### Description
In MCP-based systems, context acts as the working memory for agents — storing prompts, retrieved documents, intermediate reasoning, and interaction history. When this context is shared, persistently stored, or insufficiently scoped, sensitive information from one session, agent, or user can leak into another. Context Injection occurs when malicious or unintended content is embedded into this shared memory, influencing how future requests are processed. Over-Sharing happens when context is reused across agents or workflows that should be isolated (e.g., customer support and marketing). Together, these issues cause private or sensitive information to propagate beyond its intended boundaries, leading to privacy violations, regulatory exposure, and corrupted agent behavior.

This risk is comparable to: 
- Slack bots leaking private channel messages
- AI meeting summarizers exposing confidential conversations
- Session bleed across multi-tenant SaaS apps
But amplified by the autonomous, context-persistent nature of agentic AI.

### Impact
- Cross-agent and cross-user data leakage
- Violation of privacy regulations (GDPR, HIPAA, PCI DSS)
- Unauthorized exposure of trade secrets and internal strategy
- Persistent contamination of model behavior due to injected context
- Loss of trust in AI systems and internal tools
- Legal, financial, and reputational damage

In multi-tenant or multi-department systems, this risk can escalate quickly and silently.

### Is the Application Vulnerable? (Checklist)
Your MCP system is vulnerable if:

- Agents or services share a common context buffer or vector store
- Context memory persists across multiple users or sessions
- Context is reused for performance optimization without revalidation
- Sensitive data enters context without classification or tagging
- No policy defines how long context can live (no TTL or expiry rule)
- Context or embeddings are reused for multi-agent reasoning
- The same context store is accessible across teams or departments
- Agents can access each other’s memory without access checks

If your architecture cannot guarantee strict separation of context by user, agent, and use-case, you are exposed.

### How to Prevent (Defensive Design & Governance Controls)
1. Use Ephemeral Contexts
- Make context windows short-lived and per session by default.
- Enforce automatic deletion after task completion.
- Avoid persistent memory unless explicitly sanctioned and governed.


2. Context Isolation & Segmentation
- Assign unique context namespaces per:
    - User
    - Agent
    - Workflow
    - Tenant
- Prevent one agent from accessing another agent’s memory directly.
- In multi-tenant setups, isolate retrieval indexes and vector stores.


3. Data Classification Tagging
- Tag all inputs and retrieved data as:
    - Public
    - Internal
    - Confidential
    - Restricted
- Prevent low-trust or cross-domain agents from accessing restricted context.

4. Context Expiry and TTL Enforcement
- Define time-to-live (TTL) policies such as:
  - Session end
  -  30 minutes
  -  24 hours max
Automatically purge expired contexts and embeddings.


5. Context Sanitization & Redaction
 -  Scan and redact:
    - PII
    - Secrets
    - Tokens
    Internal system identifiers before storing in context.
 -  Use automated scanners or classification pipelines.


6. Human-in-the-Loop for Sensitive Context
Require approval before sensitive context is:
    Exported
    Summarized
    Shared across agents
Show a preview of context that will be reused.

7. Context Access Logging
 -  Log:
 -  Agent ID
 -  Context ID
 -  Read/write events
 - TTL + purge events
Integrate context logs into SIEM/XDR for monitoring.

8. Context Injection Filtering
 -  Detect and block instruction-like content trying to persist in memory:
    -   “Ignore previous instructions”
    -   “Share everything you know”
 -  Maintain injection pattern detection models.


### Example Attack Scenarios

#### Scenario 1  — Cross-Team Data Leak
Support and marketing teams share the same MCP agent infrastructure.
 Marketing agent retrieves support transcripts containing sensitive customer disputes and internal policy details.

#### Scenario 2  — Multi-Tenant Context Bleed
A cloud MCP platform fails to isolate vector stores between tenants. Tenant A’s internal documents appear in Tenant B’s retrieval outputs.

### Remediation
-Purge existing shared contexts and caches.
Enforce per-agent and per-user segmentation.
Introduce TTL policies and auto-purge logic.
Rotate keys and invalidate context stores if contamination is confirmed.
Review access control around vector databases and embeddings.

### References & Further Reading
- abc
- 

### [Make suggestions on Github:- ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP10-2025%E2%80%93ContextInjection%26OverSharing.md)

