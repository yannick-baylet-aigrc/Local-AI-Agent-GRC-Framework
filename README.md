# Local-AI-Agent-GRC-Framework
AI GRC LAB on local installation project

Context:
Initially I wanted to install a local AI, hosted at home on a dedicated server. Quite quickly, GRC considerations came up and led me to view this personal project as an interesting use case for a governance framework, ultimately convincing me that it should be zero-trust by design.

Zero-Trust AI Agents Governance Framework (ZTA-GRF)
🛡️ Overview
This repository aims at referencing an architecture and policy engine for deploying autonomous AI agents locally, in high-security framework.
Designed to align with ISO 27001, NIS2, and the EU AI Act, this framework treats Large Language Models (LLMs) and execution agents as inherently untrusted entities by default.
Rather than granting broad system access, the architecture enforces Capability-Based Security, Strict Network Segmentation, and Human-in-the-Loop (HITL) Hardware-Bound Cryptographic Approvals for critical operations.

🏗️ Architecture
The architecture separates agent execution from the administrative control using a dedicated, direct physical link.
```text
[Hardened Management Node (laptop)] ─── (Direct Ethernet / Private Link) ─── [Compute Node (AI X3)]
(Console / Policy UI / NAT)                                                (Isolated Agent Runtime)
          │                                                                         │
          ▼                                                                         ▼
   Approval & Audit                                                        Untrusted AI Agent
   Strict Egress Control                                                   Tool Broker & Policy Engine
                                                                           Hardware Auth (FIDO2)
```
```yaml
# [ANALYSIS & GRC MAPPING - ISO 27001 / NIS2 / EU AI ACT]
compliance_alignment:
  standard references: "ISO/IEC 27001:2022 A.9 & NIS2 Article 21"
  actor_model: "Untrusted LLM -> Sandboxed Runtime -> Policy Engine"
  risk_mitigation: "Eliminates lateral movement and ensures non-repudiation."
```
Core Security Principles:
*Least Privilege by Default: The AI agent executes under a dedicated, unprivileged service account with zero administrative capabilities.
*Action Segregation (The Tool Broker): The LLM never touches the operating system directly. It outputs structured intent JSON; a localized Tool Broker evaluates this compared to an AppArmor-enforced Policy Engine.
*Hardware Cryptographic Approvals: Critical operations require physical human verification using an isolated FIDO2 hardware token (such as an Ubikey). By separating the authorization device -and workflow- from the raw compute node to better simulate a production-grade enterprise environment, the policy engine issues single-use, cryptographically signed execution tokens.
*Network Segmentation: The compute node has no default route to internet. Outbound traffic is blocked by default and permitted exclusively through audited proxies on the hardened Management Node.
```yaml
# [POLICY CONFIGURATION BLOCK - ACCESS CONTROL]
policy_engine:
  default_action: "DENY"
  enforcement_layer: "AppArmor + systemd sandboxing"
  audit_logging: true
```

⚙️ Authorization Tiers
The framework categorizes each agent intents into four strict execution tiers:
* **Tier 0 (Observation): Read-only access to localized datasets, model querying, and internal logs. No approval required.
* **Tier 1 (Reversible Actions): Sandboxed data transformation, temporary file creation, and local benchmarking. No approval required; fully logged.
* **Tier 2 (Sensitive Actions): Requesting restricted outbound network access (e.g., API calls) or modifying non-critical configurations. Requires remote software approval via the Management Node.
* **Tier 3 (Critical Actions):** Destructive file operations, system state changes, security policy modifications, or financial resource allocation (such as automated API credit top-ups, strictly bounded by a hard-coded $10 USD ceiling via temporary virtual tokens). *Requires physical FIDO2 hardware touch authorization directly on the compute node.*

📜 Audit Trail & Non-Repudiation
Every transaction processed by the Tool Broker generates a structured, immutable log entry for compliant auditing:
```json
{
  "event_id": "evt_8f3b2a1c",
  "timestamp": "2026-09-23T17:00:00Z",
  "actor_agent": "qwen-agent-v3",
  "requested_tool": "filesystem.delete",
  "target_resource": "/data/models/legacy.gguf",
  "risk_classification": "CRITICAL",
  "policy_reference": "POL-SYS-004",
  "auth_mechanism": "hardware_fido2_challenge",
  "nonce": "n_992184a",
  "human_approver": "operator_verified",
  "execution_status": "SUCCESS"
}
```
```bash
# [AUDIT VERIFICATION CLI COMMAND]
jq '. | select(.risk_classification == "CRITICAL")' /var/log/zta-grf/audit.log
```

🔒 System Hardening Implementation
USBGuard: Strict kernel-level whitelisting of approved hardware tokens (e.g., specific YubiKey serial numbers). All unapproved USB insertion events are blocked.
Systemd Sandboxing: The agent service runs with directives including `ProtectSystem=strict`, `PrivateNetwork=true`, and `NoNewPrivileges=true`.
AppArmor Profiles: Restricts the file system paths and system calls accessible to the agent runtime environment.
```bash
# [HARDENING CHECK SCRIPT]
systemctl show zta-agent-runner.service --property=ProtectSystem,PrivateNetwork,NoNewPrivileges
usbguard list-devices
```



⚠️ Disclaimer
This architecture is provided as a conceptual reference implementation for AI governance and security research. Proper deployment requires thorough penetration testing, environment-specific threat modeling, and adherence to local regulatory requirements.
