# adapter-zeroclaw

Adapter that translates Reflection agent config into a [ZeroClaw](https://github.com/zeroclaw-labs/zeroclaw) runtime with native Telegram transport.

## What it does

adapter-zeroclaw wraps agent.nix to produce Docker images running agents on ZeroClaw — an open-source agent runtime written in Rust. ZeroClaw is a single 8 MB binary with 60+ built-in tools, native Telegram channel, SQLite memory, and session persistence. Everything adapter-claude does in a bash script, ZeroClaw ships as compiled code.

The adapter reads the capsule's agent config, generates ZeroClaw's TOML config and identity files, packages them into a Docker image, and injects runtime secrets at container startup.

## Usage

A capsule imports adapter-zeroclaw instead of agent-nix:

```nix
{
  inputs.adapter-zeroclaw.url = "github:reflection-network/adapter-zeroclaw";

  outputs = { self, adapter-zeroclaw }:
    adapter-zeroclaw.lib.mkAgent {
      agent = {
        name = "Ada";
        system-prompt = "You are Ada, a helpful assistant.";
        provider = "claude-code";
        model = "claude-sonnet-4-5-20250929";
        transports.telegram.enable = true;
      };
    };
}
```

Build and run:

```bash
nix build .#docker
docker load < result
docker run -d --memory 4g \
  -e TELEGRAM_BOT_TOKEN=<token> \
  -e AGENT_HOSTNAME=https://your-agent.example.com \
  -p 8080:8080 \
  -v agent-home:/home/agent \
  -v ~/.claude/.credentials.json:/home/agent/.claude/.credentials.json \
  ada:latest
```

The container exposes port 8080 — an nginx web server proxying all requests to ZeroClaw's gateway. Specific paths defined in agent.nix (like `/hello`) take priority over the catch-all proxy.

## Agent config fields

### Required

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent's display name (also used as Docker image name) |
| `system-prompt` | string | Behavioral instructions — written to `IDENTITY.md` in ZeroClaw's workspace |

### Optional

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `provider` | string | `"anthropic"` | LLM provider: `"claude-code"`, `"anthropic"`, or any ZeroClaw-supported provider |
| `model` | string | `"claude-sonnet-4-5-20250929"` | Model identifier passed to the provider |
| `transports.telegram.enable` | bool | `false` | Enable the Telegram channel |
| `transports.telegram.allowed-users` | list of strings | `[]` | Telegram usernames allowed to interact with the bot |
| `transports.telegram.mention-only` | bool | `false` | Only respond when @mentioned in groups |

## Schema support

Which agent.nix fields this adapter uses:

| Field | Supported | Notes |
|-------|-----------|-------|
| `name` | Yes | Used as Docker image name and in IDENTITY.md header |
| `system-prompt` | Yes | Written to IDENTITY.md, injected into LLM system prompt |
| `provider` | Yes | Selects LLM backend; defaults to `"anthropic"` |
| `model` | Yes | Passed to provider; defaults to `"claude-sonnet-4-5-20250929"` |
| `transports.telegram.enable` | Yes | Enables native Telegram channel |
| `transports.telegram.allowed-users` | Yes | Restricts bot access to listed usernames |
| `transports.telegram.mention-only` | Yes | Bot only responds when @mentioned in groups |

## Provider options

### `provider = "claude-code"`

Uses Claude Code CLI as a subprocess for LLM calls. Reuses existing Claude Code OAuth credentials — no API key needed. The CLI is bundled into the Docker image automatically.

Requires mounting credentials into the container:

```bash
-v ~/.claude/.credentials.json:/home/agent/.claude/.credentials.json
```

### `provider = "anthropic"`

Calls the Anthropic API directly. Requires an API key passed at runtime:

```bash
-e API_KEY=sk-ant-...
```

No credential mount needed. Smaller Docker image (no Claude Code CLI bundled).

### Other providers

ZeroClaw supports 15+ LLM providers. Set `provider` to the provider name and pass credentials via `API_KEY`. See [ZeroClaw documentation](https://github.com/zeroclaw-labs/zeroclaw) for the full list.

## Runtime configuration

### Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | When Telegram enabled | Bot token from [@BotFather](https://t.me/BotFather) |
| `API_KEY` | When `provider != "claude-code"` | LLM provider API key |
| `AGENT_HOSTNAME` | For public URL | Public base URL of the agent (e.g. `https://agent.example.com`). Used by ZeroClaw to generate correct links in Telegram messages. Defaults to `http://localhost:8080`. |

### Volume mounts

| Mount | Required | Description |
|-------|----------|-------------|
| `~/.claude/.credentials.json:/home/agent/.claude/.credentials.json` | When `provider = "claude-code"` | Claude Code OAuth credentials |

### Memory limit

Always run with `--memory` to limit container RAM:

```bash
docker run -d --memory 4g ...
```

Without a memory limit, a runaway allocation on a server without swap can make the entire machine unresponsive before the OOM killer acts.

## Architecture

```
Capsule flake.nix
  └─ adapter-zeroclaw.lib.mkAgent { agent config }
       ├─ agent-nix.lib.mkAgent (validation + devShell + web)
       └─ Docker image
            ├─ zeroclaw (Rust binary, 8 MB)
            ├─ nginx + static page (from agent.nix web output)
            ├─ bash, coreutils, cacert
            ├─ claude-code, curl, jq (only when provider = "claude-code")
            ├─ /etc/zeroclaw/config.toml (generated from agent config)
            ├─ /etc/zeroclaw/IDENTITY.md (agent's system prompt)
            ├─ /etc/zeroclaw/tunnel-url.sh (public URL script)
            ├─ /etc/nginx/conf.d/gateway.conf (reverse proxy config)
            └─ entrypoint
                 ├─ Copies config to writable ~/.zeroclaw/
                 ├─ Injects TELEGRAM_BOT_TOKEN and API_KEY into config
                 ├─ Starts nginx (from agent.nix web.startCmd)
                 └─ Runs: zeroclaw daemon
```

### Web server and reverse proxy

nginx (from agent.nix) listens on port 8080. The adapter adds a catch-all `location /` proxy in `/etc/nginx/conf.d/gateway.conf` that forwards all requests to ZeroClaw's gateway (port 42617), including WebSocket upgrades.

More specific locations defined in agent.nix (e.g. `/hello`) take priority over the catch-all proxy. This allows agent.nix to claim paths for agent-level services while the adapter handles everything else.

ZeroClaw endpoints accessible through nginx: health, webhook, REST API, WebSocket chat (`/ws/chat`), web dashboard, metrics.

### Public URL

ZeroClaw needs to know its public URL to generate correct links in Telegram messages. The adapter uses a custom tunnel script that reads `AGENT_HOSTNAME` from the environment and outputs it as the public URL. No real tunnel is needed — nginx handles the reverse proxy and the hostname is known at deploy time.

### Config generation

The adapter generates two files from the capsule's agent attributes:

- **`config.toml`** — ZeroClaw's main config: provider, model, channel settings, memory backend, autonomy level, gateway settings (including `trust_forwarded_headers = true`), and custom tunnel config. Baked at build time, secrets injected at runtime via sed.
- **`IDENTITY.md`** — The agent's system prompt. ZeroClaw reads workspace identity files at startup and injects them into the LLM system prompt.

### Telegram pairing

On first startup, ZeroClaw requires pairing with your Telegram account. The container logs will show:

```
🔐 Telegram pairing required. One-time bind code: 805295
   Send `/bind <code>` from your Telegram account.
```

Send `/bind <code>` to your bot on Telegram to complete pairing. This only happens once — the pairing is persisted in SQLite.

## ZeroClaw binary

The adapter fetches ZeroClaw v0.6.0 prebuilt binaries from GitHub releases. Supported platforms:

| Platform | Architecture |
|----------|-------------|
| Linux | x86_64, aarch64 |
| macOS | aarch64 (Apple Silicon) |

The binary is dynamically linked against glibc. The Nix derivation uses `autoPatchelfHook` to rewrite the ELF interpreter and library paths to point at the Nix store's glibc.

## Comparison with adapter-claude

| | adapter-claude | adapter-zeroclaw |
|---|---|---|
| Runtime | Bash + Claude Code CLI | ZeroClaw (Rust binary) |
| LLM backend | Claude Code only | 15+ providers |
| Telegram | Long-poll bash loop | Native channel |
| Memory | Claude's `--resume` | Built-in SQLite |
| Tools | Claude Code's tools | 60+ built-in |
| Image size | ~120 MB | ~136 MB (with claude-code) |

## Known issues

- **"database is locked"** — SQLite contention between ZeroClaw's daemon components on startup. Does not prevent operation; investigating.
- **Memory limit required** — first test run without `--memory` crashed the host server. Always use `--memory` on `docker run`.

## Documentation

- [Adapters](https://docs.reflection.network/adapters) — the adapter pattern and available adapters
- [Getting started](https://docs.reflection.network/getting-started) — create a capsule to use with this adapter
