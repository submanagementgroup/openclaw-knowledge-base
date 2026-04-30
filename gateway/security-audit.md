---
domain: gateway
topic: "Security Audit: openclaw security audit Checks and Remediation"
type: reference
keywords:
  - security audit
  - openclaw security audit
  - security checks
  - audit remediation
  - security hardening
related:
  - gateway/security-overview
  - security/security-overview
source: gateway/security/audit-checks.md
---

The `openclaw security audit` command checks your gateway configuration against security best practices. This reference describes what each check validates.

## Running the Security Audit

```bash
openclaw security audit           # full audit
openclaw security audit --json    # machine-readable output
openclaw security audit --fix     # auto-fix where possible
```

## Audit Check Categories

`openclaw security audit` emits structured findings keyed by `checkId`. This
page is the reference catalog for those IDs. For the high-level threat model
and hardening guidance, see [Security](/gateway/security).

High-signal `checkId` values you will most likely see in real deployments (not
exhaustive):

| `checkId`                                                     | Severity      | Why it matters                                                                       | Primary fix key/path                                                                                 | Auto-fix |
| ------------------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- | -------- |
| `fs.state_dir.perms_world_writable`                           | critical      | Other users/processes can modify full OpenClaw state                                 | filesystem perms on `~/.openclaw`                                                                    | yes      |
| `fs.state_dir.perms_group_writable`                           | warn          | Group users can modify full OpenClaw state                                           | filesystem perms on `~/.openclaw`                                                                    | yes      |
| `fs.state_dir.perms_readable`                                 | warn          | State dir is readable by others                                                      | filesystem perms on `~/.openclaw`                                                                    | yes      |
| `fs.state_dir.symlink`                                        | warn          | State dir target becomes another trust boundary                                      | state dir filesystem layout                                                                          | no       |
| `fs.config.perms_writable`                                    | critical      | Others can change auth/tool policy/config                                            | filesystem perms on `~/.openclaw/openclaw.json`                                                      | yes      |
| `fs.config.symlink`                                           | warn          | Symlinked config files are unsupported for writes and add another trust boundary     | replace with a regular config file or point `OPENCLAW_CONFIG_PATH` at the real file                  | no       |
| `fs.config.perms_group_readable`                              | warn          | Group users can read config tokens/settings                                          | filesystem perms on config file                                                                      | yes      |
| `fs.config.perms_world_readable`                              | critical      | Config can expose tokens/settings                                                    | filesystem perms on config file                                                                      | yes      |
| `fs.config_include.perms_writable`                            | critical      | Config include file can be modified by others                                        | include-file perms referenced from `openclaw.json`                                                   | yes      |
| `fs.config_include.perms_group_readable`                      | warn          | Group users can read included secrets/settings                                       | include-file perms referenced from `openclaw.json`                                                   | yes      |
| `fs.config_include.perms_world_readable`                      | critical      | Included secrets/settings are world-readable                                         | include-file perms referenced from `openclaw.json`                                                   | yes      |
| `fs.auth_profiles.perms_writable`                             | critical      | Others can inject or replace stored model credentials                                | `agents/<agentId>/agent/auth-profiles.json` perms                                                    | yes      |
| `fs.auth_profiles.perms_readable`                             | warn          | Others can read API keys and OAuth tokens                                            | `agents/<agentId>/agent/auth-profiles.json` perms                                                    | yes      |
| `fs.credentials_dir.perms_writable`                           | critical      | Others can modify channel pairing/credential state                                   | filesystem perms on `~/.openclaw/credentials`                                                        | yes      |
| `fs.credentials_dir.perms_readable`                           | warn          | Others can read channel credential state                                             | filesystem perms on `~/.openclaw/credentials`                                                        | yes      |
| `fs.sessions_store.perms_readable`                            | warn          | Others can read session transcripts/metadata                                         | session store perms                                                                                  | yes      |
| `fs.log_file.perms_readable`                                  | warn          | Others can read redacted-but-still-sensitive logs                                    | gateway log file perms                                                                               | yes      |
| `fs.synced_dir`                                               | warn          | State/config in iCloud/Dropbox/Drive broadens token/transcript exposure              | move config/state off synced folders                                                                 | no       |
| `gateway.bind_no_auth`                                        | critical      | Remote bind without shared secret                                                    | `gateway.bind`, `gateway.auth.*`                                                                     | no       |
| `gateway.loopback_no_auth`                                    | critical      | Reverse-proxied loopback may become unauthenticated                                  | `gateway.auth.*`, proxy setup                                                                        | no       |
| `gateway.trusted_proxies_missing`                             | warn          | Reverse-proxy headers are present but not trusted                                    | `gateway.trustedProxies`                                                                             | no       |
| `gateway.http.no_auth`                                        | warn/critical | Gateway HTTP APIs reachable with `auth.mode="none"`                                  | `gateway.auth.m
