---

layout: col-sidebar
title: "MCP08:2025 – Lack of Audit and Telemetry"

---

### Description
MCP (Model Context Protocol) systems often orchestrate complex, autonomous workflows — performing data retrieval, tool execution, and decision-making with minimal human intervention. When audit logging and telemetry are absent or poorly implemented, organizations lose visibility into what actions agents perform, what data they access, and how decisions are made. A lack of comprehensive logging not only undermines incident response and forensic analysis, but also obscures compliance violations, insider abuse, and model misbehavior. In AI-integrated environments, this gap becomes even more critical — an unmonitored agent can silently perform sensitive operations or exfiltrate data for weeks without detection.

### Impact
- No traceability for agent actions or context decisions — making root-cause investigation impossible.
- Compliance failure with regulatory frameworks (GDPR, PCI DSS, ISO 27001) that require activity and access logs.
- Delayed breach detection, increasing dwell time and damage from malicious or accidental misuse.
- Integrity loss, as organizations cannot verify whether an outcome or decision originated from valid sources.
- Operational blind spots, making it impossible to detect model drift, behavioral anomalies, or prompt injections in real time.
- Regulatory penalties and reputation damage from inability to demonstrate due diligence or data governance.

### Is the Application Vulnerable? (Checklist)

Your MCP environment is likely vulnerable if:
- Agent activity is not logged in a structured, centralized format (JSON, OpenTelemetry, etc.).
- Logs are stored locally, deleted frequently, or lack integrity protections.
- Tool invocations, prompt contents, and system events are not captured or correlated.
- The environment has no integration with SIEM/XDR or centralized monitoring platforms.
- The environment has no integration with SIEM/XDR or centralized monitoring platforms.
- The environment has no integration with SIEM/XDR or centralized monitoring platforms.
- Logs do not include user identity, timestamps, or schema versioning.
- There is no alerting for anomalous tool use, unauthorized API calls, or unexpected model behaviors.
- Privacy concerns led to overly broad log suppression instead of redaction or anonymization.
- Audit retention policies are undefined or do not align with compliance requirements.

### How to Prevent (Defensive Practices & Architecture Controls)
1. Implement Structured, Tamper-Evident Logging
Log all agent actions, tool invocations, schema versions, and context snapshots in a structured format (JSON, CEF, OTEL). Apply cryptographic hashing (HMAC, SHA-256) to log files for integrity. Store logs in append-only or write-once media (e.g., AWS S3 Object Lock, WORM storage).

  Include essential fields:
  - timestamp
  - agent_id
  - session_id
  - tool_invoked
  - parameters_used
  - response_summary
  - user_identity (if applicable)

2. Integrate with SIEM, XDR, or Centralized Monitoring
- Forward MCP logs to enterprise SIEM systems (Splunk, ELK, Sentinel, Chronicle, etc.) for correlation.
- Establish automated alert rules for high-risk activities (e.g., tool execution involving sensitive data).
- Use Extended Detection and Response (XDR) systems to correlate agent behaviors with network or endpoint signals.

3. Protect Sensitive Data in Logs
- Implement PII-safe logging: tokenize or mask user identifiers and redact sensitive fields before storage.
- Use field-level encryption for secrets, tokens, or confidential context entries.
- Apply data classification labels to log streams to govern retention and access.

4. Establish Behavioral Baselines
- Collect telemetry to build a behavioral profile of normal agent operations.
- Use anomaly detection or ML-based behavioral analytics to flag deviations (e.g., unexpected API calls, unusual output patterns).
- Regularly review and update baseline thresholds.

5. Enforce Access Control & Segregation of Duties
- Restrict who can access logs — separate operational monitoring from security investigations.
- Require dual authorization for log deletion or retention changes.
- Apply least privilege and auditing on logging subsystems themselves.

6. Implement Real-Time Observability
- Use OpenTelemetry or equivalent frameworks to trace requests across the MCP pipeline — from prompt creation to tool invocation.
- Tag every trace with session and schema identifiers to enable end-to-end correlation.
- Display agent performance and behavior dashboards for operational visibility.

7. Retention & Compliance Policies
- Align log retention with applicable frameworks (e.g., PCI DSS: 1 year minimum).
- Automatically archive or purge logs per retention schedule.
- Periodically verify that retention, encryption, and deletion processes function as intended.

8. Continuous Audit & Verification
- Conduct periodic audit drills to ensure investigators can reconstruct events from logs.
- Test integrity checks — attempt to tamper with logs and validate detection alerts.
- Implement audit trail self-verification, where logs cross-reference session data for consistency.


### Example Attack Scenarios

#### Scenario 1 – Silent Exfiltration
An MCP agent in a healthcare analytics system is compromised. It begins exporting small amounts of patient data via legitimate tool calls. Because detailed telemetry is disabled, no alerts are generated. The breach remains undetected for months.

#### Scenario 2 – Insider Manipulation
A developer disables telemetry for a testing session and uses the agent to extract pricing model data. Without audit trails, no accountability can be established, and the insider’s activity goes unnoticed.

#### Scenario 3 – Prompt Injection Leading to Data Theft
A malicious PDF introduces an instruction causing the agent to retrieve credentials and send them to an external domain. No logs exist for context transformations or network calls, preventing forensics or mitigation.

#### Scenario 4 – Drift Without Detection
A compliance bot slowly drifts in behavior after multiple retraining cycles, approving actions that violate policy. Without telemetry and drift baselines, no one notices the change until an audit months later.

#### Detection
Gaps or inconsistencies in audit trails
Unexplained spikes in API billing, latency, or resource consumption
Lack of log entries during active usage periods
Incident response teams reporting “no data available” during investigations
Sudden drop in telemetry ingestion volume

#### Immediate Remediation
Re-enable detailed logging at all MCP layers (agent, tool, and network).
Deploy forwarders to send logs to central SIEM/XDR with retention guarantees.
Implement masking and pseudonymization to balance privacy and audit needs.
Reconstruct minimal timeline from external system logs (firewalls, proxies).
Perform root-cause review and enforce mandatory logging for all MCP agents.


### References & Further Reading
- abc

#### [Make your suggestions on Github -](https://github.com/OWASP/www-project-mcp-top-10/blob/main/2025/MCP08-2025%E2%80%93Lack-of-Audit-and-Telemetry.md) 


