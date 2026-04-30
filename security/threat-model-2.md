---
domain: security
topic: "Threat Model Part 2: Detailed Threat Categories and Countermeasures"
type: reference
keywords:
  - threat categories
  - threat mitigations
  - security countermeasures
  - threat analysis
related:
  - security/threat-model
  - gateway/security-overview
source: security/THREAT-MODEL-ATLAS.md
---

Threat model continued: detailed threat categories and mitigations.

      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - AI Model Inference API Access                                                                     |
| **Description**         | Attacker intercepts pairing code during pairing grace period (1h for DM channel pairing, 5m for node pairing) |
| **Attack Vector**       | Shoulder surfing, network sniffing, social engineering                                                        |
| **Affected Components** | Device pairing system                                                                                         |
| **Current Mitigations** | 1h expiry (DM pairing) / 5m expiry (node pairing), codes sent via existing channel                            |
| **Residual Risk**       | Medium - Grace period exploitable                                                                             |
| **Recommendations**     | Reduce grace period, add confirmation step                                                                    |

#### T-ACCESS-002: AllowFrom Spoofing

| Attribute               | Value                                                                          |
| ----------------------- | ------------------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0040 - AI Model Inference API Access                                      |
| **Description**         | Attacker spoofs allowed sender identity in channel                             |
| **Attack Vector**       | Depends on channel - phone number spoofing, username impersonation             |
| **Affected Components** | AllowFrom validation per channel                                               |
| **Current Mitigations** | Channel-specific identity verification                                         |
| **Residual Risk**       | Medium - Some channels vulnerable to spoofing                                  |
| **Recommendations**     | Document channel-specific risks, add cryptographic verification where possible |

#### T-ACCESS-003: Token Theft

| Attribute               | Value                                                       |
| ----------------------- | ----------------------------------------------------------- |
| **ATLAS ID**            | AML.T0040 - AI Model Inference API Access                   |
| **Description**         | Attacker steals authentication tokens from config files     |
| **Attack Vector**       | Malware, unauthorized device access, config backup exposure |
| **Affected Components** | ~/.openclaw/credentials/, config storage                    |
| **Current Mitigations** | File permissions                                            |
| **Residual Risk**       | High - Tokens stored in plaintext                           |
| **Recommendations**     | Implement token encryption at rest, add token rotation      |

---

### 3.3 Execution (AML.TA0005)

#### T-EXEC-001: Direct Prompt Injection

| Attribute               | Value                                                                                     |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0051.000 - LLM Prompt Injection: Direct                                              |
| **Description**         | Attacker sends crafted prompts to manipulate agent behavior                               |
| **Attack Vector**       | Channel messages containing adversarial instructions                                      |
| **Affected Components** | Agent LLM, all input surfaces                                                             |
| **Current Mitigations** | Pattern detection, external content wrapping                                              |
| **Residual Risk**       | Critical - Detection only, no blocking; sophisticated attacks bypass                      |
| **Recommendations**     | Implement multi-layer defense, output validation, user confirmation for sensitive actions |

#### T-EXEC-002: Indirect Prompt Injection

| Attribute               | Value                                                       |
| ----------------------- | ----------------------------------------------------------- |
| **ATLAS ID**            | AML.T0051.001 - LLM Prompt Injection: Indirect              |
| **Description**         | Attacker embeds malicious instructions in fetched content   |
| **Attack Vector**       | Malicious URLs, poisoned emails, compromised webhooks       |
| **Affected Components** | web_fetch, email ingestion, external data sources           |
| **Current Mitigations** | Content wrapping with XML tags and security notice          |
| **Residual Risk**       | High - LLM may ignore wrapper instructions                  |
| **Recommendations**     | Implement content sanitization, separate execution contexts |

#### T-EXEC-003: Tool Argument Injection

| Attribute               | Value                                                        |
| ----------------------- | ------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0051.000 - LLM Prompt Injection: Direct                 |
| **Description**         | Attacker manipulates tool arguments through prompt injection |
| **Attack Vector**       | Crafted prompts that influence tool parameter values         |
| **Affected Components** | All tool invocations                                         |
| **Current Mitigations** | Exec approvals for dangerous commands                        |
| **Residual Risk**       | High - Relies on user judgment                               |
| **Recommendations**     | Implement argument validation, parameterized tool calls      |

#### T-EXEC-004: Exec Approval Bypass

| Attribute               | Value                                                      |
| ----------------------- | ---------------------------------------------------------- |
| **ATLAS ID**            | AML.T0043 - Craft Adversarial Data                         |
| **Description**         | Attacker crafts commands that bypass approval allowlist    |
| **Attack Vector**       | Command obfuscation, alias exploitation, path manipulation |
| **Affected Components** | exec-approvals.ts, command allowlist                       |
| **Current Mitigations** | Allowlist + ask mode                                       |
| **Residual Risk**       | High - No command sanitization                             |
| **Recommendations**     | Implement command normalization, expand blocklist          |

---

### 3.4 Persistence (AML.TA0006)

#### T-PERSIST-001: Malicious Skill Installation

| Attribute               | Value                                                                    |
| ----------------------- | ------------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0010.001 - Supply Chain Compromise: AI Software                     |
| **Description**         | Attacker publishes malicious skill to ClawHub                            |
| **Attack Vector**       | Create account, publish skill with hidden malicious code                 |
| **Affected Components** | ClawHub, skill loading, agent execution                                  |
| **Current Mitigations** | GitHub account age verification, pattern-based moderation flags          |
| **Residual Risk**       | Critical - No sandboxing, limited review                                 |
| **Recommendations**     | VirusTotal integration (in progress), skill sandboxing, community review |

#### T-PERSIST-002: Skill Update Poisoning

| Attribute               | Value                                                          |
| ----------------------- | -------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0010.001
