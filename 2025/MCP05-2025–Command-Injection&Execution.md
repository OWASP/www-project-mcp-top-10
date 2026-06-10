---

layout: col-sidebar
title: "MCP05:2025 – Command Injection & Execution"

---

### Description
Command injection in MCP environments occurs when an AI agent constructs and executes system commands, shell scripts, API calls, or code snippets using untrusted input whether from user prompts, retrieved context, or third-party data sources without proper validation or sanitization. Unlike traditional command injection where attackers directly control input fields, MCP-based command injection is mediated through the model layer: the agent interprets natural language instructions and translates them into executable operations. This creates a unique attack surface where:

##### Prompt-driven execution: 
Instructions hidden in prompts, documents, or context can cause the agent to generate malicious commands that appear syntactically valid.

##### Dynamic command construction: 
Agents often build shell commands, SQL queries, or API requests by concatenating parameters derived from context, making them vulnerable to injection if boundaries aren't enforced.

##### Tool-mediated execution: 
MCP tools that wrap system calls, database operations, or file system access become injection vectors if they pass unsanitized agent outputs directly to interpreters.

##### Chained execution: 
A seemingly benign command can be chained with malicious operators (&&, |, ;, backticks) to execute arbitrary code. Because agents operate autonomously and often with elevated privileges to perform their intended functions, successful command injection can lead to complete system compromise, data exfiltration, or lateral movement across interconnected services. 

##### Tool-mediated server-side request forgery (SSRF):
MCP tools that fetch URLs, call webhooks, convert remote documents, or proxy API
requests give the model network execution authority. If a destination is selected
from a prompt, retrieved document, tool result, or other untrusted context, an
attacker may cause the tool to request an unintended internal or privileged
service. This remains an SSRF risk even when the tool does not execute shell
commands.

The destination must be validated as data at the tool boundary. A model deciding
that a URL "looks safe" is not a security control. Validation also has to account
for redirects, DNS changes between validation and connection, IPv4 and IPv6
representations, and non-HTTP schemes supported by the underlying client.

### Impact
- Arbitrary code execution: Attackers gain the ability to run shell commands, scripts, or binaries on the host system with the agent's privileges.
- Data exfiltration: Sensitive files, databases, or environment variables can be read and transmitted to attacker-controlled endpoints.
- Internal service access: Network-enabled tools can reach services that are not exposed to the original user, including administrative APIs and cloud metadata services.
- Credential exposure: Responses from internal or link-local endpoints can disclose workload credentials, tokens, configuration, or infrastructure details.
- System compromise: Installation of backdoors, rootkits, or persistent access mechanisms.
- Privilege escalation: Exploiting SUID binaries, sudo misconfigurations, or service accounts to gain higher-level access.
- Denial of service: Resource exhaustion through fork bombs, infinite loops, or system shutdowns.
- Lateral movement: Using compromised MCP servers as pivot points to attack internal infrastructure, databases, or cloud resources.
- Supply chain poisoning: Injecting malicious code into build pipelines, CI/CD systems, or deployment artifacts.
- Regulatory violations: Unauthorized system modifications or data access leading to compliance breaches (PCI DSS, HIPAA, SOC 2).

### Is the Application Vulnerable? (Checklist)
Your MCP environment is likely vulnerable if:
- Agents construct shell commands by concatenating user input, prompts, or retrieved data without escaping or parameterization.
- Tool implementations pass agent outputs directly to exec(), system(), eval(), subprocess.run(shell=True), or similar unsafe execution functions.
- No input validation exists for parameters before they're incorporated into system calls, SQL queries, or API requests.
- Models generate code (bash, Python, PowerShell) that is automatically executed without sandboxing or human review.
- File path operations accept unsanitized input, allowing directory traversal (../../../etc/passwd) or overwriting critical files.
- API or database calls are constructed using string interpolation rather than parameterized queries or safe APIs.
- Agent outputs are not constrained to allowlists of permitted commands, arguments, or file paths.
- URL-fetching or webhook tools accept a complete destination from model-controlled or retrieved content.
- Destination checks occur only before DNS resolution or only on the first URL in a redirect chain.
- Outbound tools can connect to loopback, private, link-local, multicast, or otherwise reserved address ranges.
- The HTTP client accepts unnecessary schemes or automatically follows redirects without revalidation.
- The MCP server has unrestricted network egress or can directly reach cloud metadata and internal control-plane services.
- Special characters (;, |, &, $(), backticks, >, <, &&, ||) in agent-generated parameters are not stripped or escaped.
- Environment variables or secrets can be accessed through command substitution ($VAR, $(cmd), backticks).
- No runtime sandboxing isolates tool execution from the host system or critical resources.
- Tools run with excessive privileges (root, admin, or service accounts with broad permissions).
- Execution occurs across different contexts (e.g., generating commands on one server that execute on another without re-validation).

### How to Prevent (Defensive Design & Governance)
1. Enforce Command Boundaries
- Use allowlists for permitted commands, arguments, and file paths.
- Reject shell metacharacters (; | & $() <> && || \ ``).
- Normalize and validate all file paths to block traversal.

2. Adopt Safe Execution Patterns
- Never use shell=True, eval(), exec(), or string-built commands.
- Always execute with structured parameters (e.g., subprocess.run(['ls', 'logs'])).
- Disable direct execution of model-generated code unless manually reviewed.

3. Sandbox All Tools
- Run tools inside containers, micro-VMs, gVisor/Kata, or jailed users.
- Enforce timeouts, resource limits, and read-only file systems.
Isolate high-risk tools (file system, network, DB) into separate sandboxes.

4. Apply Least Privilege
Run tools as non-root with minimal filesystem, API, and DB permissions.
Prevent agents from accessing environment variables or secrets by default.

5. Strong Validation at Tool Boundaries
Validate agent output against schemas before execution.
Use parameterized SQL/APIs — never interpolate input.
Reject unsafe patterns: chained commands, redirection, wildcards, command substitution.

6. Add Human-in-the-Loop for Sensitive Actions
Require approval for destructive, privileged, or system-modifying operations.
Log all tool calls with full parameters and maintain immutable audit trails.

7. Constrain Outbound Network Authority
- Prefer tool-specific destination identifiers over accepting arbitrary URLs.
- Allowlist required schemes, hosts, ports, and methods when the business flow has known destinations.
- Parse and canonicalize destinations with a maintained URL library; do not validate URLs with ad hoc substring or regular-expression checks.
- Resolve the destination and reject loopback, private, link-local, multicast, unspecified, and other non-public addresses unless a narrowly documented internal destination is explicitly allowed.
- Disable redirects where possible. Otherwise, repeat destination validation after every redirect and enforce a low redirect limit.
- Apply the same checks immediately before connection so DNS rebinding cannot change an approved hostname into a prohibited address.
- Enforce egress policy at the network layer so the tool process can reach only required services. Keep cloud metadata and internal control-plane endpoints unreachable.
- Do not return raw internal response bodies, headers, or connection errors to the model or user.
- Apply timeouts, response-size limits, and content-type restrictions to reduce denial-of-service and data-exposure impact.

### Example Attack Scenarios

#### Scenario 1 — Shell Metacharacter Injection
A user asks an MCP agent: "List files in the logs directory and also show me /etc/passwd"
The agent generates:
bash
ls logs; cat /etc/passwd
The tool executes this as a single shell command, exposing system account information.
Mitigation: Use parameterized execution (subprocess.run(['ls', 'logs'])) and reject compound commands.

#### Scenario 2 — API Parameter Injection
An attacker submits a prompt containing: "Search for user'; DROP TABLE users;-- in the database"
The agent constructs:
SELECT * FROM records WHERE name = 'user'; DROP TABLE users;--'
The SQL injection destroys the database.
Mitigation: Always use prepared statements; never interpolate user input into SQL strings.

#### Scenario 3 - Prompt-Directed Internal Request
An MCP document-conversion tool accepts a URL selected from retrieved content.
The content instructs the agent to fetch an internal service address. The tool
can reach that address even though the user cannot, and returns the response to
the model.

Mitigation: Do not expose arbitrary destinations when a bounded identifier is
sufficient. Validate the resolved address at connection time, revalidate every
redirect, block prohibited address ranges, and enforce independent network
egress controls.

### Detection
Unusual commands: Detection of shell metacharacters (;, |, &, backticks) in tool parameters or logs.
Privilege escalation attempts: Execution of sudo, su, or SUID binaries by agent processes.
Unexpected network activity: Outbound connections from agent hosts to unknown domains.
Prohibited destinations: Requests to loopback, private, link-local, metadata, or newly observed destinations.
Resolution changes: Hostnames that resolve to prohibited ranges, return mixed public/private answers, or change address classification between checks.
Redirect anomalies: Redirect chains that cross trust boundaries, change scheme or port, or repeatedly approach the configured limit.
Tool-intent mismatch: Network requests that do not match the initiating tool's documented destination set or user-approved action.
Response anomalies: Unusual internal headers, credential-shaped values, or unexpectedly large responses returned by URL-fetching tools.
File system anomalies: Access to sensitive paths (/etc/passwd, /root, /proc/, ~/.ssh).
Syscall anomalies: Abnormal patterns detected by Falco, auditd, or osquery (e.g., execve with suspicious args).
High resource consumption: CPU spikes, memory exhaustion, or disk I/O storms indicating malicious scripts.
Failed validation attempts: Repeated rejections of inputs containing metacharacters or forbidden commands.

### References & Further Reading
- [mcp-remote CVE-2025-6514 (CVSS 9.6)](https://composio.dev/blog/mcp-vulnerabilities-every-developer-should-know) — Arbitrary OS command execution when MCP clients connect to untrusted servers
- [Three Flaws in Anthropic MCP Git Server (CVE-2025-68143, CVE-2025-68144, CVE-2025-68145)](https://thehackernews.com/2026/01/three-flaws-in-anthropic-mcp-git-server.html) — Path validation bypass enabling file access and code execution
- [Systematic Analysis of MCP Security](https://arxiv.org/html/2508.12538v1) — Academic study finding 82% of MCP implementations use APIs prone to path traversal, 67% to code injection
- [A Security Engineer's Guide to MCP](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-mcp/) — Semgrep analysis of command injection patterns in MCP tool implementations
- [MCP Servers: The New Security Nightmare](https://equixly.com/blog/2025/03/29/mcp-server-new-security-nightmare/) — Analysis of shell execution risks in MCP server deployments
- [MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices) - Official MCP guidance covering SSRF risks and mitigations
- [OWASP Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) - Application- and network-layer SSRF defenses
- [CVE-2025-65512](https://nvd.nist.gov/vuln/detail/CVE-2025-65512) - SSRF in an MCP webpage-to-markdown tool involving hostname validation and redirect-chain bypasses
- [CVE-2025-65513](https://nvd.nist.gov/vuln/detail/CVE-2025-65513) - SSRF in an MCP fetch tool allowing private-IP validation bypass and internal resource access

### [Make suggestions on Github:- ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP10-2025%E2%80%93ContextInjection%26OverSharing.md)
