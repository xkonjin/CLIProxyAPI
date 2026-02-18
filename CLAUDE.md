# CLIProxyAPI — CLAUDE.md

## Architecture

```
cmd/server/main.go          entry point, flag parsing, login flows
sdk/cliproxy/service.go     Service struct — lifecycle, hot-reload, token refresh
internal/api/server.go      Gin HTTP server, route registration
internal/translator/        Protocol cross-mapping (init.go registers all pairs)
internal/registry/          Global ModelRegistry (RWMutex, ref-counted)
internal/auth/              OAuth token storage and refresh per provider
internal/config/config.go   Config struct, YAML unmarshal
internal/runtime/executor/  Per-provider upstream HTTP execution
internal/watcher/           fs watcher for auth dir hot-reload
```

### Request Flow

```
HTTP request
  → Gin middleware (auth, CORS)
  → SDK handler (openai/claude/gemini)
  → Translator.Translate(srcProtocol, dstProtocol, payload)
  → ModelRegistry.SelectClient(modelID)   ← round-robin, quota-aware
  → executor.Execute(upstreamRequest)
  → SSE/JSON response back to client
```

### Translator Registration

All protocol translators register themselves via `init()` in `internal/translator/init.go`. Translators cover 6 source protocols × 5 destination protocols. Adding a new provider requires:

1. Creating `internal/translator/{src}/{dst}/` package with `init()` call
2. Importing it in `internal/translator/init.go`

### Model Registry

- `internal/registry/model_registry.go` — `ModelRegistry` (singleton via `GetGlobalRegistry()`)
- `internal/registry/model_definitions_static_data.go` — 86 static model entries
- `internal/registry/model_definitions.go` — `GetStaticModelDefinitionsByChannel()` dispatcher
- Ref-counting: `RegisterClient` / `UnregisterClient` track active credential count per model
- Quota: `SetModelQuotaExceeded` → 5-minute cooldown; `SuspendClientModel` for indefinite suspend
- Models hidden from `/v1/models` when `effectiveClients == 0`

## Key Commands

| Action         | Command                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| Build          | `go build -o ./cli-proxy-api ./cmd/server/`                             |
| Build (Docker) | `docker build -t cliproxyapi .`                                         |
| Run            | `./cli-proxy-api --config config.yaml`                                  |
| Run (Docker)   | `docker-compose up`                                                     |
| Test           | `go test ./...`                                                         |
| Test specific  | `go test ./internal/registry/... -v`                                    |
| Login Gemini   | `./cli-proxy-api --login`                                               |
| Login Claude   | `./cli-proxy-api --claude-login`                                        |
| Login Codex    | `./cli-proxy-api --codex-login`                                         |
| Login Kimi     | `./cli-proxy-api --kimi-login`                                          |
| Login Qwen     | `./cli-proxy-api --qwen-login`                                          |
| Health check   | `curl http://127.0.0.1:8317/health`                                     |
| List models    | `curl http://127.0.0.1:8317/v1/models -H "Authorization: Bearer <key>"` |

## Key File Locations

| File                                                 | Purpose                                            |
| ---------------------------------------------------- | -------------------------------------------------- |
| `config.yaml`                                        | Main config (port, api-keys, provider credentials) |
| `config.example.yaml`                                | Annotated reference config                         |
| `cmd/server/main.go`                                 | Entry point, CLI flags                             |
| `internal/config/config.go`                          | Config struct                                      |
| `internal/api/server.go`                             | HTTP server + route setup                          |
| `internal/translator/init.go`                        | All translator registrations                       |
| `internal/registry/model_registry.go`                | ModelRegistry singleton                            |
| `internal/registry/model_definitions_static_data.go` | 86 static model definitions                        |
| `sdk/cliproxy/service.go`                            | Service lifecycle (embed-friendly)                 |
| `sdk/cliproxy/auth/`                                 | Auth provider implementations                      |
| `internal/runtime/executor/`                         | Per-provider upstream executors                    |
| `~/.cli-proxy-api/`                                  | OAuth token files (default auth-dir)               |
| `logs/`                                              | Rotating log files (when logging-to-file: true)    |
| `FALLBACK_CHAIN.md`                                  | Live fallback chain documentation                  |

## Configuration Reference (key fields)

```yaml
port: 8317
auth-dir: "~/.cli-proxy-api"
api-keys: ["sk-proxy"]
routing:
  strategy: "round-robin" # or fill-first
request-retry: 3
quota-exceeded:
  switch-project: true
  switch-preview-model: true
```

Provider credential blocks: `gemini-api-key`, `codex-api-key`, `claude-api-key`, `openai-compatibility`, `vertex-api-key`, `ampcode`.

Each block supports: `api-key`, `prefix`, `base-url`, `headers`, `proxy-url`, `models[]` (name+alias), `excluded-models[]` (wildcards supported).

## Anti-Patterns and Gotchas

| Gotcha                     | Detail                                                                                                |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| Schema prefix              | Never query with `plasma.` prefix — not applicable here; keep model IDs clean                         |
| Translator side-effects    | Translators register via `init()` — missing import = silent missing route                             |
| Quota cooldown             | 5-minute window tracked in-memory; restart clears all quota state                                     |
| `config.yaml` hot-reload   | Auth dir is watched; `config.yaml` changes require restart                                            |
| OAuth token files          | Stored in `auth-dir` as JSON; deleting a file immediately unregisters that client                     |
| `force-model-prefix: true` | Requires callers to use `prefix/model-name` format; breaks bare model calls                           |
| `commercial-mode: true`    | Disables request logging middleware — no per-request log entries                                      |
| Model aliases              | `oauth-model-alias` only applies to OAuth channels, not `*-api-key` blocks                            |
| Multiple binary builds     | Repo contains many `cliproxyapi-*` binaries — use `./cli-proxy-api` built from source                 |
| WS relay                   | `/v1/ws` WebSocket relay is separate from REST — `ws-auth: true` adds Bearer check                    |
| Thinking budgets           | Provider-native units (tokens for Claude/Gemini, levels for Codex/iFlow); normalization in translator |

## Provider Channels Summary

| Channel                | OAuth flow                       | API key              | Notes                                         |
| ---------------------- | -------------------------------- | -------------------- | --------------------------------------------- |
| `claude`               | Yes (claude-login)               | Yes                  | Cloaking support for non-Claude-Code clients  |
| `gemini`               | No                               | Yes                  | Direct Gemini API key                         |
| `gemini-cli`           | Yes (login)                      | No                   | Google account OAuth                          |
| `aistudio`             | Yes (login)                      | No                   | Google AI Studio OAuth                        |
| `vertex`               | No                               | Yes (x-goog-api-key) | Vertex-compatible endpoints                   |
| `codex`                | Yes (codex-login)                | Yes                  | OpenAI OAuth or API key                       |
| `claude` OAuth         | Yes (claude-login)               | —                    | Anthropic Claude OAuth                        |
| `qwen`                 | Yes (qwen-login)                 | No                   | Alibaba Qwen                                  |
| `iflow`                | Yes (iflow-login / iflow-cookie) | No                   | Zhipu GLM / iFlow                             |
| `kimi`                 | Yes (kimi-login)                 | No                   | Moonshot Kimi                                 |
| `antigravity`          | Google OAuth                     | No                   | Experimental Google backend                   |
| `openai-compatibility` | No                               | Yes                  | Any OpenAI-compat provider (OpenRouter, etc.) |
| `ampcode`              | —                                | Upstream key         | Amp CLI integration with model-mapping        |
