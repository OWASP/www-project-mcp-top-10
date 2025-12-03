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

### Impact
- Arbitrary code execution: Attackers gain the ability to run shell commands, scripts, or binaries on the host system with the agent's privileges.
- Data exfiltration: Sensitive files, databases, or environment variables can be read and transmitted to attacker-controlled endpoints.
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


### Detection
Unusual commands: Detection of shell metacharacters (;, |, &, backticks) in tool parameters or logs.
Privilege escalation attempts: Execution of sudo, su, or SUID binaries by agent processes.
Unexpected network activity: Outbound connections from agent hosts to unknown domains.
File system anomalies: Access to sensitive paths (/etc/passwd, /root, /proc/, ~/.ssh).
Syscall anomalies: Abnormal patterns detected by Falco, auditd, or osquery (e.g., execve with suspicious args).
High resource consumption: CPU spikes, memory exhaustion, or disk I/O storms indicating malicious scripts.
Failed validation attempts: Repeated rejections of inputs containing metacharacters or forbidden commands.

### References & Further Reading
- abc

### [Make suggestions on Github:- ](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP10-2025%E2%80%93ContextInjection%26OverSharing.md)
