---
domain: getting-started
topic: "Getting Started: Onboarding"
type: procedure
keywords:
  - onboarding
  - getting started
  - setup
  - quickstart
source: start/onboarding.md
---

This doc describes the **current** first-run setup flow. The goal is a
smooth "day 0" experience: pick where the Gateway runs, connect auth, run the
wizard, and let the agent bootstrap itself.
For a general overview of onboarding paths, see [Onboarding Overview](/start/onboarding-overview).

**Approve macOS warning**


<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />


**Approve find local networks**


<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />


**Welcome and security notice**


<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />


Security trust model:

- By default, OpenClaw is a personal agent: one trusted operator boundary.
- Shared/multi-user setups require lock-down (split trust boundaries, keep tool access minimal, and follow [Security](/gateway/security)).
- Local onboarding now defaults new configs to `tools.profile: "coding"` so fresh local setups keep filesystem/runtime tools without forcing the unrestricted `full` profile.
- If hooks/webhooks or other untrusted content feeds are enabled, use a strong modern model tier and keep strict tool policy/sandboxing.


**Local vs Remote**


<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />


Where does the **Gateway** run?

- **This Mac (Local only):** onboarding can configure auth and write credentials
  locally.
- **Remote (over SSH/Tailnet):** onboarding does **not** configure local auth;
  credentials must exist on the gateway host.
- **Configure later:** skip setup and leave the app unconfigured.

> **Note:** **Gateway auth tip:**

- The wizard now generates a **token** even for loopback, so local WS clients must authenticate.
- If you disable auth, any local process can connect; use that only on fully trusted machines.
- Use a **token** for multi-machine access or non-loopback binds.



**Permissions**


<img src="/assets/macos-onboarding/05-permissions.png" alt="" />


Onboarding requests TCC permissions needed for:

- Automation (AppleScript)
- Notifications
- Accessibility
- Screen Recording
- Microphone
- Speech Recognition
- Camera
- Location


**CLI**

> **Note:** This step is optional
  The app can install the global `openclaw` CLI via npm, pnpm, or bun.
  It prefers npm first, then pnpm, then bun if that is the only detected
  package manager. For the Gateway runtime, Node remains the recommended path.

**Onboarding Chat (dedicated session)**

After setup, the app opens a dedicated onboarding chat session so the agent can
  introduce itself and guide next steps. This keeps first-run guidance separate
  from your normal conversation. See [Bootstrapping](/start/bootstrapping) for
  what happens on the gateway host during the first agent run.

## Related

- [Onboarding overview](/start/onboarding-overview)
- [Getting started](/start/getting-started)
