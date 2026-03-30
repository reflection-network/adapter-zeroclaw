# adapter-zeroclaw

Adapter wrapping agent.nix for the ZeroClaw runtime. Full support for all agent.nix schema fields.

## Stack

Nix, ZeroClaw (prebuilt Rust binary from GitHub releases).

## How it works

Generates `config.toml` and `IDENTITY.md` from agent config at build time. At runtime, copies config to writable `~/.zeroclaw/`, injects secrets via `sed`, starts nginx, then runs `zeroclaw daemon`.

## Build-time config generation

- **config.toml** — provider, model, temperature, agent settings, channels, memory (sqlite), autonomy level, gateway config, tunnel
- **IDENTITY.md** — agent name header + system-prompt body
- **tunnel-url.sh** — outputs `AGENT_HOSTNAME` env var

Telegram section conditionally included when `transports.telegram.enable = true`. `allowed_users` and `mention_only` mapped to TOML types.

## Runtime secret injection

Secrets (`TELEGRAM_BOT_TOKEN`, `API_KEY`) are placeholder strings in the build-time config, replaced via `sed` at container start from environment variables.

## Multi-provider support

- `claude-code` — uses Claude Code CLI subprocess, requires credentials mount
- `anthropic` — direct Anthropic API calls, requires `API_KEY` env var
- Others — 15+ ZeroClaw-supported providers via `API_KEY`

When provider is `claude-code`, Docker image conditionally adds claude-code, curl, and jq.

## Nginx reverse proxy

Adds `/etc/nginx/conf.d/gateway.conf` — catch-all reverse proxy to ZeroClaw gateway on port 42617. Agent.nix-specific locations (like `/hello`) take priority.

## Telegram

Native ZeroClaw telegram support with `/bind` pairing command. Configured via `allowed_users` and `mention_only` from agent.nix schema.
