---
domain: security
topic: "OpenClaw Threat Model: Attack Vectors, Trust Boundaries, and Security Mitigations"
type: reference
keywords:
  - threat model
  - security model
  - attack vectors
  - trust boundaries
  - ATLAS
  - threat analysis
related:
  - gateway/security-overview
  - gateway/authentication
  - security/network-proxy
source: security/THREAT-MODEL-ATLAS.md
---

OpenClaw threat model: comprehensive security analysis covering attack vectors, trust boundaries, and mitigations for self-hosted AI agent deployments.

# OpenClaw Threat Model v1.0

## MITRE ATLAS Framework

**Version:** 1.0-draft
**Last Updated:** 2026-02-04
**Methodology:** MITRE ATLAS + Data Flow Diagrams
**Framework:** [MITRE ATLAS](https://atlas.mitre.org/) (Adversarial Threat Landscape for AI Systems)

### Framework attribution

This threat model is built on [MITRE ATLAS](https://atlas.mitre.org/), the industry-standard framework for documenting adversarial threats to AI/ML systems. ATLAS is maintained by [MITRE](https://www.mitre.org/) in collaboration with the AI security community.

**Key ATLAS Resources:**

- [ATLAS Techniques](https://atlas.mitre.org/techniques/)
- [ATLAS Tactics](https://atlas.mitre.org/tactics/)
- [ATLAS Case Studies](https://atlas.mitre.org/studies/)
- [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data)
- [Contributing to ATLAS](https://atlas.mitre.org/resources/contribute)

### Contributing to This Threat Model

This is a living document maintained by the OpenClaw community. See [CONTRIBUTING-THREAT-MODEL.md](/security/CONTRIBUTING-THREAT-MODEL) for guidelines on contributing:

- Reporting new threats
- Updating existing threats
- Proposing attack chains
- Suggesting mitigations

---

## 1. Introduction

### 1.1 Purpose

This threat model documents adversarial threats to the OpenClaw AI agent platform and ClawHub skill marketplace, using the MITRE ATLAS framework designed specifically for AI/ML systems.

### 1.2 Scope

| Component              | Included | Notes                                            |
| ---------------------- | -------- | ------------------------------------------------ |
| OpenClaw Agent Runtime | Yes      | Core agent execution, tool calls, sessions       |
| Gateway                | Yes      | Authentication, routing, channel integration     |
| Channel Integrations   | Yes      | WhatsApp, Telegram, Discord, Signal, Slack, etc. |
| ClawHub Marketplace    | Yes      | Skill publishing, moderation, distribution       |
| MCP Servers            | Yes      | External tool providers                          |
| User Devices           | Partial  | Mobile apps, desktop clients                     |

### 1.3 Out of Scope

Nothing is explicitly out of scope for this threat model.

---

## 2. System Architecture

### 2.1 Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                    UNTRUSTED ZONE                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  WhatsApp   │  │  Telegram   │  │   Discord   │  ...         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
└─────────┼────────────────┼────────────────┼──────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 1: Channel Access                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      GATEWAY                              │   │
│  │  • Device Pairing (1h DM / 5m node grace period)           │   │
│  │  • AllowFrom / AllowList validation                       │   │
│  │  • Token/Password/Tailscale auth                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 2: Session Isolation              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   AGENT SESSIONS                          │   │
│  │  • Session key = agent:channel:peer                       │   │
│  │  • Tool policies per agent                                │   │
│  │  • Transcript logging                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 3: Tool Execution                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  EXECUTION SANDBOX                        │   │
│  │  • Docker sandbox OR Host (exec-approvals)                │   │
│  │  • Node remote execution                                  │   │
│  │  • SSRF protection (DNS pinning + IP blocking)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 4: External Content               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              FETCHED URLs / EMAILS / WEBHOOKS             │   │
│  │  • External content wrapping (XML tags)                   │   │
│  │  • Security notice injection                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 5: Supply Chain                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLAWHUB                              │   │
│  │  • Skill publishing (semver, SKILL.md required)           │   │
│  │  • Pattern-based moderation flags                         │   │
│  │  • VirusTotal scanning (coming soon)                      │   │
│  │  • GitHub account age verification                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flows

| Flow | Source  | Destination | Data               | Protection           |
| ---- | ------- | ----------- | ------------------ | -------------------- |
| F1   | Channel | Gateway     | User messages      | TLS, AllowFrom       |
| F2   | Gateway | Agent       | Routed messages    | Session isolation    |
| F3   | Agent   | Tools       | Tool invocations   | Policy enforcement   |
| F4   | Agent   | External    | web_fetch requests | SSRF blocking        |
| F5   | ClawHub | Agent       | Skill code         | Moderation, scanning |
| F6   | Agent   | Channel     | Responses          | Output filtering     |

---

## 3. Threat Analysis by ATLAS Tactic

### 3.1 Reconnaissance (AML.TA0002)

#### T-RECON-001: Agent Endpoint Discovery

| Attribute               | Value                                                                |
| ----------------------- | -------------------------------------------------------------------- |
| **ATLAS ID**            | AML.T0006 - Active Scanning                                          |
| **Description**         | Attacker scans for exposed OpenClaw gateway endpoints                |
| **Attack Vector**       | Network scanning, shodan queries, DNS enumeration                    |
| **Affected Components** | Gateway, exposed API endpoints                                       |
| **Current Mitigations** | Tailscale auth option, bind to loopback by default                   |
| **Residual Risk**       | Medium - Public gateways discoverable                                |
| **Recommendations**     | Document secure deployment, add rate limiting on discovery endpoints |

#### T-RECON-002: Channel Integration Probing

| Attribute               | Value                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| **ATLAS ID**            | AML.T0006 - Active Scanning                                        |
| **Description**         | Attacker probes messaging channels to identify AI-managed accounts |
| **Attack Vector**       | Sending test messages, observing response patterns                 |
| **Affected Components** | All channel integrations                                           |
| **Current Mitigations** | None specific                                                      |
| **Residual Risk**       | Low - Limited value from discovery alone                           |
| **Recommendations**     | Consider response timing randomization                             |

---

### 3.2 Initial Access (AML.TA0004)

#### T-ACCESS-001: Pairing Code Interception

| Attribute               | Value
