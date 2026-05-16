---
domain: security
topic: "Security Audit Checks: Filesystem, Gateway, Discovery, Config, and Hooks checkIds"
type: reference
keywords:
  - security audit
  - checkId
  - audit findings
  - fs.state_dir
  - gateway.bind_no_auth
  - gateway.tailscale_funnel
  - hooks.token_reuse_gateway_token
  - audit --fix
  - openclaw security audit
  - sandbox audit
source: gateway/security/audit-checks.md
related:
  - security/audit-checks-tools-plugins
  - security/security-model
  - security/network-hardening
  - security/configuration-hardening
  - gateway/trusted-proxy-auth
---

`openclaw security audit` emits structured findings keyed by `checkId`. This page catalogs `fs.*`, `gateway.*`, `discovery.*`, `config.*`, `hooks.*`, `logging.*`, and `browser.*` check IDs. For the high-level threat model and hardening guidance, see [Security](/gateway/security). For tools, plugins, sandbox, and exposure checks, see [Security audit checks: tools and plugins](/security/audit-checks-tools-plugins).

Run the audit with:
```bash
openclaw security audit
openclaw security audit --deep    # adds live Gateway probe
openclaw security audit --fix     # auto-fix supported findings
openclaw security audit --json
```

## Filesystem (fs.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `fs.state_dir.perms_world_writable` | critical | Other users/processes can modify full OpenClaw state | filesystem perms on `~/.openclaw` | yes |
| `fs.state_dir.perms_group_writable` | warn | Group users can modify full OpenClaw state | filesystem perms on `~/.openclaw` | yes |
| `fs.state_dir.perms_readable` | warn | State dir is readable by others | filesystem perms on `~/.openclaw` | yes |
| `fs.state_dir.symlink` | warn | State dir target becomes another trust boundary | state dir filesystem layout | no |
| `fs.config.perms_writable` | critical | Others can change auth/tool policy/config | filesystem perms on `~/.openclaw/openclaw.json` | yes |
| `fs.config.symlink` | warn | Symlinked config files are unsupported for writes and add another trust boundary | replace with a regular config file or point `OPENCLAW_CONFIG_PATH` at the real file | no |
| `fs.config.perms_group_readable` | warn | Group users can read config tokens/settings | filesystem perms on config file | yes |
| `fs.config.perms_world_readable` | critical | Config can expose tokens/settings | filesystem perms on config file | yes |
| `fs.config_include.perms_writable` | critical | Config include file can be modified by others | include-file perms referenced from `openclaw.json` | yes |
| `fs.config_include.perms_group_readable` | warn | Group users can read included secrets/settings | include-file perms referenced from `openclaw.json` | yes |
| `fs.config_include.perms_world_readable` | critical | Included secrets/settings are world-readable | include-file perms referenced from `openclaw.json` | yes |
| `fs.auth_profiles.perms_writable` | critical | Others can inject or replace stored model credentials | `agents/<agentId>/agent/auth-profiles.json` perms | yes |
| `fs.auth_profiles.perms_readable` | warn | Others can read API keys and OAuth tokens | `agents/<agentId>/agent/auth-profiles.json` perms | yes |
| `fs.credentials_dir.perms_writable` | critical | Others can modify channel pairing/credential state | filesystem perms on `~/.openclaw/credentials` | yes |
| `fs.credentials_dir.perms_readable` | warn | Others can read channel credential state | filesystem perms on `~/.openclaw/credentials` | yes |
| `fs.sessions_store.perms_readable` | warn | Others can read session transcripts/metadata | session store perms | yes |
| `fs.log_file.perms_readable` | warn | Others can read redacted-but-still-sensitive logs | gateway log file perms | yes |
| `fs.synced_dir` | warn | State/config in iCloud/Dropbox/Drive broadens token/transcript exposure | move config/state off synced folders | no |

## Gateway (gateway.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `gateway.bind_no_auth` | critical | Remote bind without shared secret | `gateway.bind`, `gateway.auth.*` | no |
| `gateway.loopback_no_auth` | critical | Reverse-proxied loopback may become unauthenticated | `gateway.auth.*`, proxy setup | no |
| `gateway.trusted_proxies_missing` | warn | Reverse-proxy headers are present but not trusted | `gateway.trustedProxies` | no |
| `gateway.http.no_auth` | warn/critical | Gateway HTTP APIs reachable with `auth.mode="none"` | `gateway.auth.mode`, `gateway.http.endpoints.*` | no |
| `gateway.http.session_key_override_enabled` | info | HTTP API callers can override `sessionKey` | `gateway.http.allowSessionKeyOverride` | no |
| `gateway.tools_invoke_http.dangerous_allow` | warn/critical | Re-enables dangerous tools over HTTP API | `gateway.tools.allow` | no |
| `gateway.nodes.allow_commands_dangerous` | warn/critical | Enables high-impact node commands (camera/screen/contacts/calendar/SMS) | `gateway.nodes.allowCommands` | no |
| `gateway.nodes.deny_commands_ineffective` | warn | Pattern-like deny entries do not match shell text or groups | `gateway.nodes.denyCommands` | no |
| `gateway.tailscale_funnel` | critical | Public internet exposure | `gateway.tailscale.mode` | no |
| `gateway.tailscale_serve` | info | Tailnet exposure is enabled via Serve | `gateway.tailscale.mode` | no |
| `gateway.control_ui.allowed_origins_required` | critical | Non-loopback Control UI without explicit browser-origin allowlist | `gateway.controlUi.allowedOrigins` | no |
| `gateway.control_ui.allowed_origins_wildcard` | warn/critical | `allowedOrigins=["*"]` disables browser-origin allowlisting | `gateway.controlUi.allowedOrigins` | no |
| `gateway.control_ui.host_header_origin_fallback` | warn/critical | Enables Host-header origin fallback (DNS rebinding hardening downgrade) | `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback` | no |
| `gateway.control_ui.insecure_auth` | warn | Insecure-auth compatibility toggle enabled | `gateway.controlUi.allowInsecureAuth` | no |
| `gateway.control_ui.device_auth_disabled` | critical | Disables device identity check | `gateway.controlUi.dangerouslyDisableDeviceAuth` | no |
| `gateway.real_ip_fallback_enabled` | warn/critical | Trusting `X-Real-IP` fallback can enable source-IP spoofing via proxy misconfig | `gateway.allowRealIpFallback`, `gateway.trustedProxies` | no |
| `gateway.token_too_short` | warn | Short shared token is easier to brute force | `gateway.auth.token` | no |
| `gateway.auth_no_rate_limit` | warn | Exposed auth without rate limiting increases brute-force risk | `gateway.auth.rateLimit` | no |
| `gateway.trusted_proxy_auth` | critical | Proxy identity now becomes the auth boundary | `gateway.auth.mode="trusted-proxy"` | no |
| `gateway.trusted_proxy_no_proxies` | critical | Trusted-proxy auth without trusted proxy IPs is unsafe | `gateway.trustedProxies` | no |
| `gateway.trusted_proxy_no_user_header` | critical | Trusted-proxy auth cannot resolve user identity safely | `gateway.auth.trustedProxy.userHeader` | no |
| `gateway.trusted_proxy_no_allowlist` | warn | Trusted-proxy auth accepts any authenticated upstream user | `gateway.auth.trustedProxy.allowUsers` | no |
| `gateway.trusted_proxy_allow_loopback` | warn | Trusted-proxy auth accepts explicitly allowed loopback proxy sources | `gateway.auth.trustedProxy.allowLoopback` | no |
| `gateway.probe_auth_secretref_unavailable` | warn | Deep probe could not resolve auth SecretRefs in this command path | deep-probe auth source / SecretRef availability | no |
| `gateway.probe_failed` | warn/critical | Live Gateway probe failed | gateway reachability/auth | no |

## Discovery (discovery.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `discovery.mdns_full_mode` | warn/critical | mDNS full mode advertises `cliPath`/`sshPort` metadata on local network | `discovery.mdns.mode`, `gateway.bind` | no |

## Config (config.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `config.insecure_or_dangerous_flags` | warn | Any insecure/dangerous debug flags enabled | multiple keys (see finding detail) | no |
| `config.secrets.gateway_password_in_config` | warn | Gateway password is stored directly in config | `gateway.auth.password` | no |
| `config.secrets.hooks_token_in_config` | warn | Hook bearer token is stored directly in config | `hooks.token` | no |

## Hooks (hooks.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `hooks.token_reuse_gateway_token` | critical | Hook ingress token also unlocks Gateway auth | `hooks.token`, `gateway.auth.token` | no |
| `hooks.token_too_short` | warn | Easier brute force on hook ingress | `hooks.token` | no |
| `hooks.default_session_key_unset` | warn | Hook agent runs fan out into generated per-request sessions | `hooks.defaultSessionKey` | no |
| `hooks.allowed_agent_ids_unrestricted` | warn/critical | Authenticated hook callers may route to any configured agent | `hooks.allowedAgentIds` | no |
| `hooks.request_session_key_enabled` | warn/critical | External caller can choose sessionKey | `hooks.allowRequestSessionKey` | no |
| `hooks.request_session_key_prefixes_missing` | warn/critical | No bound on external session key shapes | `hooks.allowedSessionKeyPrefixes` | no |
| `hooks.path_root` | critical | Hook path is `/`, making ingress easier to collide or misroute | `hooks.path` | no |
| `hooks.installs_unpinned_npm_specs` | warn | Hook install records are not pinned to immutable npm specs | hook install metadata | no |
| `hooks.installs_missing_integrity` | warn | Hook install records lack integrity metadata | hook install metadata | no |
| `hooks.installs_version_drift` | warn | Hook install records drift from installed packages | hook install metadata | no |

## Logging (logging.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `logging.redact_off` | warn | Sensitive values leak to logs/status | `logging.redactSensitive` | yes |

## Browser (browser.*) Checks

| `checkId` | Severity | Why it matters | Primary fix key/path | Auto-fix |
|---|---|---|---|---|
| `browser.control_invalid_config` | warn | Browser control config is invalid before runtime | `browser.*` | no |
| `browser.control_no_auth` | critical | Browser control exposed without token/password auth | `gateway.auth.*` | no |
| `browser.remote_cdp_http` | warn | Remote CDP over plain HTTP lacks transport encryption | browser profile `cdpUrl` | no |
| `browser.remote_cdp_private_host` | warn | Remote CDP targets a private/internal host | browser profile `cdpUrl`, `browser.ssrfPolicy.*` | no |
