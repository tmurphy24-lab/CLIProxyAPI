# FORK-INFO — Local CPA system inventory + fork rationale

> Fork of `router-for-me/CLIProxyAPI` (upstream remote). Purpose: unified
> per-provider **OAuth account pools** with quota tracking and automatic
> account switching, usable by every local agent (Codex, OpenCode,
> Antigravity, AMP, Devin, Warp, ACP adapters) through one endpoint.
>
> Companion plan lives in the local (non-git) agent hub:
> `C:\Users\trevo\Desktop\.agents\CPA-FORK-PLAN.md` — not copied here
> because this repo is public.
>
> **Sanitization:** this file contains paths, port numbers, provider names,
> and file-name *patterns* only. No emails, tokens, keys, or org IDs.

## 1. Live install (what we're forking from)

| Piece | Where | Version |
|---|---|---|
| GUI driver | `C:\Users\trevo\Desktop\Apps\EasyCLIProxyAPI\EasyCLIProxyAPI.exe` | v0.3.27 |
| Core binary | `…\EasyCLIProxyAPI\cpa-core\cli-proxy-api.exe` | v8.0.21 (commit `54946fa3`) |
| Core config | `…\cpa-core\config.yaml` | `config-version: 8` |
| GUI config | `…\EasyCLIProxyAPI\config.toml` | port 8317, `auth-dir: ..\oauth` |
| OAuth creds | `…\EasyCLIProxyAPI\oauth\<type>-<account>.json` | hot-loaded |
| Usage DB | `…\EasyCLIProxyAPI\usage-records\usage.db` | SQLite |
| Plugins | `…\cpa-core\plugins\` + `credential-tier-router\state.json` | native ABI |

- Core listens on `127.0.0.1:8317` (`allow-lan: false`).
- Management API: `/v0/management/*` + `/v8/management/*`, bcrypt key.
- `/v1/models` currently merges **1,073** model entries across all pools.
- GUI has an "agents" integration store (`agents/opencode`, `agents/omp`
  hold `integration-state.json`).

## 2. Current account pools (live `GET /v0/management/auth-files`)

OAuth pools (file-backed, `oauth/*.json`):

| Pool (`type`) | Count | Notes |
|---|---|---|
| `antigravity` | 6 | Google OAuth; fields: access/refresh/id tokens, `expired`, `project_id` |
| `codex` | 2 | both `plan_type: plus`; has `account_id`, `id_token`, `last_refresh` |
| `devin` | 1 | `auth_kind: oauth`, has `org_id`, `plan`, `session_token` |
| claude / kimi / xai / meta / vertex | 0 | supported, not signed in |

API-key pools (`api-keys:` upstream groups in config.yaml):

| Group | Keys | base-url |
|---|---|---|
| `gemini` (google) | 1 | default Google |
| `claude` (anthropic) | 1 | api.anthropic.com |
| `openai-compatibility.poolside` | 7 | inference.poolside.ai/v1 |
| openrouter, groq, inception, opencode, ollama-cloud, vercel, opencode-go, requesty, crusoe, abliteration-ai, siliconflow, the-grid-ai, nvidia, fireworks-ai, fastrouter | 1+ each | various |

**Separation rule:** `antigravity` OAuth accounts ≠ Gemini/AI-Studio API
keys — different pools, endpoints, and quota rules. Never merge.

## 3. CLI auth surface (binary flags, `--help` verified)

`-codex-login`, `-codex-device-login`, `-claude-login`,
`-antigravity-login`, `-kimi-login`, `-kimi-ai-login`, `-xai-login`,
`-devin-login`, `-meta-login`, `-vertex-import <file>`,
`-vertex-import-prefix <p>`, `-no-browser` (URL-only mode),
`-oauth-callback-port <n>`, `-tui`, `-standalone`,
`-management-base-url`, `-local-model`, `-discover*`, `-home-jwt`.

Source layout that owns these:

```
internal/auth/{antigravity,claude,codex,devin,kimi,meta,vertex,xai}/
    codex/:  oauth_server.go, pkce.go, jwt_parser.go, openai_auth.go, token.go
    devin/:  devin_auth.go, pkce.go, record.go, user_status.go
internal/registry/   model catalogs + per-provider updaters
internal/translator/{antigravity,claude,codex,gemini,openai,interactions}/
internal/access/, internal/store/, internal/watcher/, internal/pluginhost/
cmd/server/          entry point; cmd/fetch_*_models/ helpers
auths/               example auth-file tree
```

## 4. What upstream already does (do not rebuild)

- Per-provider OAuth login → writes `oauth/<type>-<email>.json` → watcher
  hot-loads into the credential pool; `auth-auto-refresh-workers` pool
  refreshes tokens.
- Routing: `round-robin | weighted-round-robin | fill-first`,
  `session-affinity` (+ subagent inheritance + TTL), per-auth `priority`
  + `weight`.
- Quota failover: `quota-exceeded.switch-project` / `switch-preview-model`.
- Cooling: `routing.cooldown` (transient-error secs, optional `.cds`
  persistence via `save-cooldown-status`), `routing.retry`
  (`request-retry`, `max-retry-credentials`, `max-retry-interval`).
- Per-channel model aliases (`oauth.model-alias`) for vertex, aistudio,
  antigravity, claude, codex, kimi, xai, meta + per-auth `model_aliases`.
- `auth-files` management payload already carries `status`, `disabled`,
  `unavailable`, `cooldowns[]`, `quota.signals{}`, `recent_requests[]`,
  `success`/`failed`, `priority`, `project_id`.
- API-key upstream groups already pool keys (weight, request-retry,
  request-scoped-errors, proxy-url).
- `credentials.concurrency` + `credentials.in-flight` coordination
  primitives (leases, reclaim grace, snapshots).
- Client polish: `client.codex.optimize-multi-agent-v2`, Claude fingerprint
  (`upstream.claude.header-defaults`), `upstream.codex.disable-codex-cloaking`.

## 5. Headless OAuth — already exists (verified live 2026-10-08)

Management routes (working right now on `127.0.0.1:8317`):

| Endpoint | Result |
|---|---|
| `GET /v0/management/codex-auth-url` | 200 `{state, url: auth.openai.com/…}` |
| `GET /v0/management/anthropic-auth-url` | 200 `{state, url: claude.ai/…}` |
| `GET /v0/management/antigravity-auth-url` | 200 `{state, url: accounts.google.com/…}` |
| `GET /v0/management/kimi-auth-url` | 200 `{flow:"device", expires_in:1800, …}` |
| `GET /v0/management/kimi-ai-auth-url` | 200 device flow |
| `GET /v0/management/xai-auth-url` | 200 device flow (accounts.x.ai) |
| `GET /v0/management/devin-auth-url` | 200 `{url: app.devin.ai/auth/cli/continue?…}` |
| `GET /v0/management/meta-auth-url` | 200 device flow (auth.meta.com, 600s) |
| `GET /v8/management/oauth/auth-url?provider=X` | unified v8 variant |
| `GET/POST /v*/management/oauth/callback` | shared callback sink |
| `GET /v8/management/oauth/status` | poll session status |
| `POST /v8/management/oauth/import` | import creds |
| `DELETE /v*/management/oauth/session` | cancel in-flight flow |

Note: the provider key for Claude is **`anthropic`**, not `claude`, on the
v0 route. kimi/kimi-ai/xai/meta use **device flow**, not loopback.

**Callback mechanics** (`auth_files_oauth_callback.go`):
per-provider loopback `callbackForwarder` binds `0.0.0.0:<provider port>`
(codex `1455`, anthropic `54545`, others per-provider) and 302-forwards the
incoming OAuth redirect query to the management callback URL
(`http://127.0.0.1:8317/v*/management/oauth/callback`), which exchanges the
code and writes the auth file. `RegisterOAuthSession(state, provider)`
pins state→provider so callbacks can't cross pools.

## 6. Verified gaps (what the fork adds)

1. **No `amp` or `warp` provider** — zero hits in flags, config schema, or
   `internal/auth/`. (Legacy `ampcode`/`amp-upstream-*` config keys are
   explicitly stripped in `config_yaml.go`; repo AGENTS.md mentions an
   `internal/api/modules/amp/` module that isn't in this tree — check
   upstream history. `warp` has no hits at all.)
2. **Quota windows are thin** — `quota.signals{}` mostly empty on live
   accounts; need normalized per-account `QuotaState` (5h/7d/monthly
   windows + reset_at) persisted to the auth JSON.
3. **No agent-embedded lifecycle** — CPA runs standalone; add supervisor /
   OpenCode plugin path (single-instance via port check).
4. **Secrets in plaintext config** — client + upstream keys live in
   `config.yaml`/`config.toml`; fork should offer a secrets sidecar or OS
   credential store.
5. **`/v0/management` is deprecated** upstream — all new routes go under
   `/v8/management`.

## 7. Fork work items (summary of hub plan §9)

- **P0** clone + build parity (this commit).
- **P1** OAuth surface is mostly built — remaining work: per-account
  block view in GUI data path, devin manual-paste docs, provider-flow
  config-driven (`provider.oauth-flows` so new providers are data not
  code). All new mgmt routes go on `/v8/management` only.
- **P2** quota engine: normalize headers/errors → `QuotaState`, persist to
  auth JSON, cooldown-until-reset, structured all-exhausted error.
- **P3** scheduler: model-capability filter, anti-stampede lease reuse,
  manual pin endpoint.
- **P4** provider coverage: amp (was a removed legacy integration — check
  `internal/api/modules/amp` upstream history), warp investigation;
  poolside key health.
- **P5** embedded lifecycle: `cpa-ensure` launcher + OpenCode plugin.
- **P6** observability: per-account quota board, switch audit log.

## 7. Safety rules for this fork

- Public repo — never commit emails, tokens, key material, org IDs, or
  the local `oauth/` files (gitignored). Local-only details stay in the
  hub plan doc.
- Upstream merges via `upstream` remote; keep `main` clean, work on
  `hub/*` branches.
