---
domain: security
topic: "Security Audit Checks: Sandbox, Tools Exec, Skills, Plugins, Exposure, and Model checkIds"
type: reference
keywords:
  - security audit
  - checkId
  - tools.exec
  - sandbox audit
  - plugins.code_safety
  - security.exposure
  - open channels exec
  - models.legacy
  - tools.exec.security_full_configured
  - strictInlineEval
  - openclaw security audit
source: gateway/security/audit-checks.md
related:
  - security/audit-checks-fs-gateway
  - security/security-model
  - security/configuration-hardening
  - security/prompt-injection
  - gateway/sandboxing
  - tools/exec-approvals
---

`openclaw security audit` emits structured findings keyed by `checkId`. This page catalogs `sandbox.*`, `tools.exec.*`, `skills.*`, `plugins.*`, `security.exposure.*`, `security.trust_model.*`, and `models.*` check IDs. For filesystem, gateway, config, hooks, and browser checks, see [Security audit checks: filesystem and gateway](/security/audit-checks-fs-gateway).

## Sandbox (sandbox.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `sandbox.docker_config_mode_off` | warn | Sandbox Docker config present but inactive | `agents.*.sandbox.mode` | no |
| `sandbox.bind_mount_non_absolute` | warn | Relative bind mounts can resolve unpredictably | `agents.*.sandbox.docker.binds[]` | no |
| `sandbox.dangerous_bind_mount` | critical | Sandbox bind mount targets blocked system, credential, or Docker socket paths | `agents.*.sandbox.docker.binds[]` | no |
| `sandbox.dangerous_network_mode` | critical | Sandbox Docker network uses `host` or `container:*` namespace-join mode | `agents.*.sandbox.docker.network` | no |
| `sandbox.dangerous_seccomp_profile` | critical | Sandbox seccomp profile weakens container isolation | `agents.*.sandbox.docker.securityOpt` | no |
| `sandbox.dangerous_apparmor_profile` | critical | Sandbox AppArmor profile weakens container isolation | `agents.*.sandbox.docker.securityOpt` | no |
| `sandbox.browser_cdp_bridge_unrestricted` | warn | Sandbox browser bridge is exposed without source-range restriction | `sandbox.browser.cdpSourceRange` | no |
| `sandbox.browser_container.non_loopback_publish` | critical | Existing browser container publishes CDP on non-loopback interfaces | browser sandbox container publish config | no |
| `sandbox.browser_container.hash_label_missing` | warn | Existing browser container predates current config-hash labels | `openclaw sandbox recreate --browser --all` | no |
| `sandbox.browser_container.hash_epoch_stale` | warn | Existing browser container predates current browser config epoch | `openclaw sandbox recreate --browser --all` | no |

## Tools Exec (tools.exec.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `tools.exec.host_sandbox_no_sandbox_defaults` | warn | `exec host=sandbox` fails closed when sandbox is off | `tools.exec.host`, `agents.defaults.sandbox.mode` | no |
| `tools.exec.host_sandbox_no_sandbox_agents` | warn | Per-agent `exec host=sandbox` fails closed when sandbox is off | `agents.list[].tools.exec.host`, `agents.list[].sandbox.mode` | no |
| `tools.exec.security_full_configured` | warn/critical | Host exec is running with `security="full"` | `tools.exec.security`, `agents.list[].tools.exec.security` | no |
| `tools.exec.fs_tools_disabled_but_exec_enabled` | warn | Filesystem tool policy does not make shell execution read-only | `tools.deny`, `agents.list[].tools.deny`, `agents.*.sandbox.workspaceAccess` | no |
| `tools.exec.auto_allow_skills_enabled` | warn | Exec approvals trust skill bins implicitly | `~/.openclaw/exec-approvals.json` | no |
| `tools.exec.allowlist_interpreter_without_strict_inline_eval` | warn | Interpreter allowlists permit inline eval without forced reapproval | `tools.exec.strictInlineEval`, `agents.list[].tools.exec.strictInlineEval`, exec approvals allowlist | no |
| `tools.exec.safe_bins_interpreter_unprofiled` | warn | Interpreter/runtime bins in `safeBins` without explicit profiles broaden exec risk | `tools.exec.safeBins`, `tools.exec.safeBinProfiles`, `agents.list[].tools.exec.*` | no |
| `tools.exec.safe_bins_broad_behavior` | warn | Broad-behavior tools in `safeBins` weaken the low-risk stdin-filter trust model | `tools.exec.safeBins`, `agents.list[].tools.exec.safeBins` | no |
| `tools.exec.safe_bin_trusted_dirs_risky` | warn | `safeBinTrustedDirs` includes mutable or risky directories | `tools.exec.safeBinTrustedDirs`, `agents.list[].tools.exec.safeBinTrustedDirs` | no |

## Skills (skills.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `skills.workspace.symlink_escape` | warn | Workspace `skills/**/SKILL.md` resolves outside workspace root (symlink-chain drift) | workspace `skills/**` filesystem state | no |
| `skills.code_safety` | warn/critical | Skill installer metadata/code contains suspicious or dangerous patterns | skill install source | no |
| `skills.code_safety.scan_failed` | warn | Skill code scan could not complete | skill scan environment | no |

## Plugins (plugins.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `plugins.extensions_no_allowlist` | warn | Plugins are installed without an explicit plugin allowlist | `plugins.allowlist` | no |
| `plugins.installs_unpinned_npm_specs` | warn | Plugin index records are not pinned to immutable npm specs | plugin install metadata | no |
| `plugins.installs_missing_integrity` | warn | Plugin index records lack integrity metadata | plugin install metadata | no |
| `plugins.installs_version_drift` | warn | Plugin index records drift from installed packages | plugin install metadata | no |
| `plugins.code_safety` | warn/critical | Plugin code scan found suspicious or dangerous patterns | plugin code / install source | no |
| `plugins.code_safety.entry_path` | warn | Plugin entry path points into hidden or `node_modules` locations | plugin manifest `entry` | no |
| `plugins.code_safety.entry_escape` | critical | Plugin entry escapes the plugin directory | plugin manifest `entry` | no |
| `plugins.code_safety.scan_failed` | warn | Plugin code scan could not complete | plugin path / scan environment | no |

## Security Exposure (security.exposure.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `security.exposure.open_channels_with_exec` | warn/critical | Shared/public rooms can reach exec-enabled agents | `channels.*.dmPolicy`, `channels.*.groupPolicy`, `tools.exec.*`, `agents.list[].tools.exec.*` | no |
| `security.exposure.open_groups_with_elevated` | critical | Open groups + elevated tools create high-impact prompt-injection paths | `channels.*.groupPolicy`, `tools.elevated.*` | no |
| `security.exposure.open_groups_with_runtime_or_fs` | critical/warn | Open groups can reach command/file tools without sandbox/workspace guards | `channels.*.groupPolicy`, `tools.profile/deny`, `tools.fs.workspaceOnly`, `agents.*.sandbox.mode` | no |

## Trust Model (security.trust_model.*) and Tools Profile Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `security.trust_model.multi_user_heuristic` | warn | Config looks multi-user while gateway trust model is personal-assistant | split trust boundaries, or shared-user hardening (`sandbox.mode`, tool deny/workspace scoping) | no |
| `tools.profile_minimal_overridden` | warn | Agent overrides bypass global minimal profile | `agents.list[].tools.profile` | no |
| `plugins.tools_reachable_permissive_policy` | warn | Extension tools reachable in permissive contexts | `tools.profile` + tool allow/deny | no |

## Model (models.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `models.legacy` | warn | Legacy model families are still configured | model selection | no |
| `models.weak_tier` | warn | Configured models are below current recommended tiers | model selection | no |
| `models.small_params` | critical/info | Small models + unsafe tool surfaces raise injection risk | model choice + sandbox/tool policy | no |

## Summary Check

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `summary.attack_surface` | info | Roll-up summary of auth, channel, tool, and exposure posture | multiple keys (see finding detail) | no |

## How to Read Audit Findings

Each finding includes:
- `checkId` — the structured identifier (e.g., `tools.exec.security_full_configured`)
- `severity` — `critical`, `warn`, or `info`
- `why` — a short explanation of the risk
- `fix` — the config key or action to resolve it
- `autoFix` — whether `openclaw security audit --fix` can automatically repair it

`critical` findings should be addressed before exposing the gateway to any non-loopback network or any untrusted users. `warn` findings are hardening opportunities appropriate for your threat model.
