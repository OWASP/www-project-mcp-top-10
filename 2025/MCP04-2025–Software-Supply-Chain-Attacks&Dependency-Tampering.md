---

layout: col-sidebar
title: "MCP04:2025 – Software Supply Chain Attacks & Dependency Tampering"

---

### Description
MCP environments rely heavily on third-party components — SDKs, connectors, protocol servers, vector database clients, plugins, and model-side tool integrations. Because these software modules often run within trusted execution paths, a compromised dependency can alter agent behavior, introduce hidden backdoors, or modify protocol semantics without triggering detection.

Attackers may target:
- MCP server libraries,
- Third-party plugins,
- Dependency updates,
- Open-source model tooling,
- Build pipelines and package registries.

Once compromised, these components can perform malicious actions such as:
- Calling unsafe APIs,
- Exfiltrating context data,
- Inserting rogue schemas,
- Tampering with tool execution,
- Issuing silent privilege escalation.

This parallels traditional software supply-chain attacks (e.g., SolarWinds, Codecov), but is amplified by agentic automation — where malicious components influence autonomous workflows at scale.

### Impact
- Unauthorized access and code execution
- Context poisoning & data exfiltration
- Privilege escalation through manipulated tools/schemas
- Silent corruption of MCP logic and decisioning
- Cross-tenant compromise if shared connectors are affected
- Propagation into downstream systems (CI/CD, cloud infra)

Because compromised dependencies often appear legitimate, they can operate undetected for long periods.

### Is the Application Vulnerable? (Checklist)
Your MCP environment may be vulnerable if:
- The system installs MCP connectors or plugins without signing / provenance checks
- Dependencies are fetched automatically during runtime or build
- SBOM / dependency inventory is incomplete or unavailable
- Teams use “latest” or floating version references
- There is no dependency integrity verification (hash, signature, attestation)
- No sandboxing isolates third-party components
- Vendors/maintainers have no formal security process
- Open-source components are directly modified and redistributed
- Plugin code is allowed to perform network calls without review
- Your organization has no verified reverse-DNS entry in the Official MCP Registry while unverified third-party packages or hosted proxies claim your brand name (*Official Registry Vacuum*)
- Developers or autonomous agents install unverified npm/PyPI MCP packages maintained by personal/free-email accounts lacking organizational domain provenance (or treat self-asserted `homepage` links as proof of first-party ownership)
- Remote MCP client configurations permit third-party hosted proxies or URL templates that embed API keys or bearer tokens directly inside URL path segments
- Unscoped community MCP packages invoke unverified `postinstall` binary downloads, shadow unrelated CLI `bin` names, or occupy dormant namespace-squat placeholders (`coming-soon` stubs)

### How to Prevent

1. Signed Components & Provenance Verification
Require cryptographic signing for:
- SDKs
- Plugins
- Tool manifests
- Container images
- Validate signatures during install + startup

2. Build SBOM / CBOM Visibility
Generate SBOM (software bill of materials) and CBOM (cryptographic bill of materials) snapshots for each MCP server + plugin package
Store SBOM alongside deployments for auditing + incident response

Track:
- Versions
- Hashes
- Licenses
- Provenance metadata

3. Version Pinning & Approved Registries
- Pin component versions — avoid “latest”
- Use internal package mirrors or registries
- Block direct downloads from the public internet

4. Dependency Scanning
- Apply SCA (software composition analysis) + code scanning tools to detect:
- Known CVEs
- Malicious indicators
- Poisoned transitive dependencies


5. Sandbox Third-Party Plugins
- Run plugins in constrained environments (e.g., WASM, container isolation)
- Restrict filesystem + network access
  
6. Supply-Chain Governance
- Maintain vendor risk profiles
- Require suppliers to provide signed attestations
- Review open-source maintainers’ security maturity

7. External Brand-Namespace Provenance & Registry Claiming
- Claim your organization's reverse-DNS namespace (e.g., `com.example/*`) in the Official MCP Registry (`registry.modelcontextprotocol.io`) or publish a canonical security notice specifying approved first-party MCP servers
- Publish official MCP packages strictly under verified organization scopes (`@example/*`) with OIDC Trusted Publishers (npm/PyPI) rather than individual or free-email maintainer accounts; do not rely on self-asserted `homepage` metadata as provenance proof
- Enforce an enterprise MCP client allowlist pinned to verified first-party server URLs and scoped packages, rejecting unverified third-party hosted proxies, URL-path credential templates (`/{token}`), and unverified `postinstall` binary downloaders
- Run periodic passive external audits across public MCP registries, npm, and PyPI to identify high-signal namespace collisions, dormant placeholder clusters, and historical `0.0.1-security` holding packages targeting your brand

### Detection Guidance
Look for:
Hash/signature changes in installed packages
Plugins making calls to unknown domains
Silent installation of new dependencies
Unauthorized schema or configuration diffs
Sudden behavior drift in MCP agents
Presence of unverified third-party hosted MCP proxies or unverified npm/PyPI packages claiming the organization's brand when no official domain-anchored entry exists (*Official Registry Vacuum*)
Remote MCP endpoint templates that accept or transmit bearer tokens/API keys in URL path parameters rather than `Authorization` headers
Maintainer provenance mismatches (e.g., packages claiming enterprise brand ownership published by personal/free-email accounts with no link to verified organizational domains, or packages registering unrelated CLI `bin` names / unverified `postinstall` downloaders)
Historical npm Security holding packages (`0.0.1-security`) occupying brand-named `mcp-server-*` identifiers, indicating prior supply-chain abuse in your brand's namespace

### Example Attack Scenarios

#### Scenario 1 — Trojanized Plugin
A popular open-source connector gains a malicious update. It silently exfiltrates customer support transcripts to an adversary-controlled endpoint. 

#### Scenario 2 — Typo-squatted Plugin
Attackers may also publish typo‑squatted plugins that mimic the legitimate plugin’s name, tricking developers into installing the malicious version.

#### Scenario 3 — Registry Compromise
An MCP package registry is compromised and replaces specific versions of a library used for context ingestion. The modified library injects new instructions into shared context memory.

#### Scenario 4 — Dependency Confusion
An attacker publishes a dependency to a public registry with the same name as an internal MCP plugin. Because developers rely on default resolution behavior, their agents pull the attacker’s version giving attackers execution access.

#### Scenario 5 — Build Pipeline Attack
CI systems are compromised and append rogue instructions to MCP manifests, adding new privileged schema methods that call destructive APIs.

#### Scenario 6 — Brand-Namespace Impersonation & Official Registry Vacuum
An enterprise has not published an official domain-anchored server in the Official MCP Registry, even though developers and AI coding agents regularly search for integrations with its APIs. When an autonomous agent or engineer resolves the brand name across public registries, it encounters unverified third-party hosted proxies, dormant namespace-squatted placeholders, or unverified packages maintained via personal email accounts. Without strict client-side allowlists or domain provenance checks, credentials and operational context are routed through unvetted third-party infrastructure, transmitted inside URL path templates logged by intermediaries, or executed via unverified `postinstall` binary downloaders.

### References & Further Reading
*   [https://genai.owasp.org/llmrisk/llm032025-supply-chain/](https://genai.owasp.org/llmrisk/llm032025-supply-chain/)
*   [https://atlas.mitre.org/techniques/AML.T0010](https://atlas.mitre.org/techniques/AML.T0010)

### [Make suggestions on Github](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP04-2025%E2%80%93Software-Supply-Chain-Attacks%26Dependency-Tampering.md)

