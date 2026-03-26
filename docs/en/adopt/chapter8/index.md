---
prev:
  text: 'Chapter 7: Tools and Scheduled Tasks'
  link: '/en/adopt/chapter7'
next:
  text: 'Chapter 9: Remote Access and Networking'
  link: '/en/adopt/chapter9'
---

# Chapter 8: Gateway Operations

Gateway is the nerve center of the lobster. This chapter teaches you how to manage it — starting, monitoring, and troubleshooting.

> **Prerequisites**: You have completed [Chapter 2: OpenClaw Quick Installation](/en/adopt/chapter2/), and Gateway is installed and running.

## 0. What Is Gateway?

Gateway is the core background process of OpenClaw. It connects chat platforms, schedules AI models, and manages sessions and tools. A single machine typically runs one Gateway, which listens on `ws://127.0.0.1:18789` by default and is only accessible locally.

```
User message → [Feishu/QQ/Telegram] → Gateway → AI model → User
                                           ↕
                                   Dashboard / CLI
```

## 1. Starting and Managing

```bash
# Start
openclaw gateway --port 18789

# Check status
openclaw gateway status    # Shows "Runtime: running" + "RPC probe: ok" = healthy
openclaw status            # Overall status

# Live logs
openclaw logs --follow
```

### Running as a System Service (Recommended)

Register as a system service to auto-start on boot and auto-restart on crash:

```bash
openclaw gateway install   # Install the service
openclaw gateway restart   # Restart (apply config changes)
openclaw gateway stop      # Stop
```

- **macOS**: LaunchAgent (`ai.openclaw.gateway`)
- **Linux / WSL2**: systemd user service (`openclaw-gateway.service`)

<details>
<summary>Developer Mode (Dev Profile)</summary>

An isolated development environment that does not affect the main Gateway:

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Dev mode uses an isolated state/config directory and defaults to port 19001.

</details>

## 2. Configuration Management

The config file uses JSON5 format (supports comments and trailing commas). For the full structure, see [Appendix G](/en/appendix/appendix-g).

| Edit Method | Command / Action |
|-------------|-----------------|
| Config wizard | `openclaw onboard` |
| CLI | `openclaw config set <key> <value>` |
| Web dashboard | `openclaw dashboard` |
| Direct edit (not recommended) | Manually editing JSON is error-prone — see [Appendix G](/en/appendix/appendix-g) |

**What happens if the config is wrong**: Gateway refuses to start. Run `openclaw doctor` to see why, and `openclaw doctor --fix` to auto-repair.

### Hot Reload

After modifying config via CLI, **most settings take effect automatically without a restart**. Only `gateway.*` fields (port, bind, auth, TLS) and infrastructure fields like `discovery` and `plugins` require a restart.

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },  // default values
  },
}
```

<details>
<summary>Hot reload vs. requires restart: full reference table</summary>

| Category | Fields | Requires Restart? |
|----------|--------|-------------------|
| Channels | `channels.*`, `web` | No |
| Agents and models | `agent`, `agents`, `models`, `routing` | No |
| Automation | `hooks`, `cron`, `agent.heartbeat` | No |
| Sessions and messages | `session`, `messages` | No |
| Tools and media | `tools`, `browser`, `skills`, `audio`, `talk` | No |
| UI and misc | `ui`, `logging`, `identity`, `bindings` | No |
| Gateway server | `gateway.*` (port, bind, auth, TLS, HTTP) | **Yes** |
| Infrastructure | `discovery`, `canvasHost`, `plugins` | **Yes** |

> `gateway.reload` and `gateway.remote` are exceptions — modifying them does not trigger a restart.

</details>

<details>
<summary>Environment Variables</summary>

OpenClaw reads environment variables in this priority order:

1. Environment variables passed from the parent process
2. `.env` file in the current working directory
3. `~/.openclaw/.env` (global fallback)

`.env` files do not override existing environment variables. You can also set them inline in the config file:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

If Gateway runs under systemd/launchd, it is recommended to place API keys in `~/.openclaw/.env` to ensure the daemon can read them.

</details>

<details>
<summary>Config RPC (Programmatic Updates)</summary>

Dashboard write operations (`config.apply`, `config.patch`, `update.run`) are rate-limited: a maximum of 3 requests per 60 seconds (calculated per deviceId + clientIp). When exceeded, the response returns `UNAVAILABLE` and `retryAfterMs`.

</details>

<details>
<summary>Common Config Templates</summary>

**Recommended starter config**:

```json5
{
  identity: {
    name: "Clawd",
    theme: "helpful assistant",
    emoji: "🦞",
  },
  agent: {
    workspace: "~/.openclaw/workspace",
    model: { primary: "anthropic/claude-sonnet-4-5" },
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

**Multi-platform config**:

```json5
{
  agent: { workspace: "~/.openclaw/workspace" },
  channels: {
    whatsapp: { allowFrom: ["+15555550123"] },
    telegram: {
      enabled: true,
      botToken: "YOUR_TOKEN",
      allowFrom: ["123456789"],
    },
    discord: {
      enabled: true,
      token: "YOUR_TOKEN",
      dm: { allowFrom: ["123456789012345678"] },
    },
  },
}
```

**Local model only**:

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
    model: { primary: "lmstudio/minimax-m2.5-gs32" },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "minimax-m2.5-gs32",
            name: "MiniMax M2.7 GS32",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

For more config templates, see [Appendix G: Config File Reference](/en/appendix/appendix-g).

</details>

## 3. Authentication and Security

Gateway listens on localhost (`127.0.0.1`) by default. **If you need external access (LAN, Tailnet), you must configure authentication**, otherwise Gateway will refuse to start.

```json5
{
  gateway: {
    bind: "loopback",  // loopback | lan | tailnet | custom
    auth: { mode: "token", token: "your-gateway-password" },
  },
}
```

### Model API Keys

Recommended placement is `~/.openclaw/.env` (required for daemon scenarios):

```bash
cat >> ~/.openclaw/.env <<'EOF'
ANTHROPIC_API_KEY=sk-ant-...
OPENROUTER_API_KEY=sk-or-...
EOF

openclaw models status   # verify
```

<details>
<summary>Secret Management (SecretRef)</summary>

For production environments, OpenClaw supports **SecretRef** — no need to write secrets in plaintext in the config file. Three source types are supported:

**Environment variable reference**:
```json5
{ source: "env", provider: "default", id: "OPENAI_API_KEY" }
```

**File reference**:
```json5
{ source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
```

**External command reference** (supports 1Password, HashiCorp Vault, sops, etc.):
```json5
{ source: "exec", provider: "vault", id: "providers/openai/apiKey" }
```

Configure secret providers:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
  },
}
```

Secret management commands:

```bash
openclaw secrets audit --check    # check for plaintext secrets
openclaw secrets configure        # interactive SecretRef setup
openclaw secrets reload           # refresh secret snapshot
```

SecretRef resolution is **done once at startup** (not on every request). Resolution failures will prevent Gateway from starting. At runtime, an in-memory snapshot is used to avoid secret provider failures affecting the request path.

**1Password integration example**:

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true,
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

</details>

<details>
<summary>Reverse Proxy Authentication (Trusted Proxy)</summary>

If OpenClaw runs behind an identity-aware proxy (Pomerium, Caddy + OAuth, nginx + oauth2-proxy), you can use `trusted-proxy` mode:

```json5
{
  gateway: {
    bind: "loopback",
    trustedProxies: ["10.0.0.1"],
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["nick@example.com"],
      },
    },
  },
}
```

**nginx + oauth2-proxy example**:

```nginx
location / {
    auth_request /oauth2/auth;
    auth_request_set $user $upstream_http_x_auth_request_email;

    proxy_pass http://openclaw:18789;
    proxy_set_header X-Auth-Request-Email $user;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

> **Security Warning**: This mode delegates authentication entirely to the proxy. You must ensure: the proxy is the only entry point, `trustedProxies` is minimized, and the proxy overwrites (not appends) forwarded headers. When enabled, `openclaw security audit` will flag this as critical — this is an intentional reminder.

</details>

<details>
<summary>Security Audit and Security Model</summary>

Run security audits regularly (especially after modifying config or exposing to the network):

```bash
openclaw security audit           # basic audit
openclaw security audit --deep    # deep audit
openclaw security audit --fix     # auto-fix
```

It checks for: Gateway auth exposure, browser control exposure, privilege escalation allowlists, file permissions, and other common issues.

**Core security principle**: OpenClaw uses a **personal assistant trust model** — one trusted operator per Gateway. It is not a hostile multi-tenant security boundary. If you need multi-user isolation, run separate Gateway instances with separate credentials (ideally separate OS users/hosts).

</details>

## 4. Logs and Health Checks

### Viewing Logs

```bash
openclaw logs --follow          # live
openclaw logs --limit 100       # last 100 entries
openclaw logs --limit 100 --json  # JSON format
```

Log files default to `/tmp/openclaw/openclaw-YYYY-MM-DD.log`. `--verbose` only affects console output; to capture verbose info in file logs, set `logging.level: "debug"`.

<details>
<summary>Log Level Configuration</summary>

```json5
{
  logging: {
    level: "info",           // file log level
    consoleLevel: "info",    // console log level
    consoleStyle: "pretty",  // pretty | compact | json
    redactSensitive: "tools",
  },
}
```

</details>

### Health Checks

```bash
openclaw status          # overall status (Gateway, channels, sessions)
openclaw status --deep   # deep check (including channel probing)
openclaw health --json   # JSON snapshot
```

### Doctor Auto-Diagnostics

When you encounter any issue, run `openclaw doctor` first — it automatically detects and fixes most common problems.

```bash
openclaw doctor          # interactive diagnostics
openclaw doctor --fix    # auto-fix
openclaw doctor --deep   # scan for redundant Gateway installations
```

<details>
<summary>Full list of Doctor checks</summary>

1. **Optional updates** (optionally pull latest code for git installs)
2. **UI protocol refresh** (rebuild dashboard UI when Protocol Schema is updated)
3. **Health check + restart prompt**
4. **Skills status summary** (available / missing / blocked)
5. **Config normalization** (legacy value format migration)
6. **OpenCode provider override warning**
7. **Legacy disk layout migration** (sessions, agent directories, WhatsApp auth)
8. **Legacy cron storage migration**
9. **State integrity check** (missing directories, permissions, cloud sync warnings)
10. **Model auth health** (OAuth expiry, cooldown/disabled state)
11. **Hooks model validation**
12. **Sandbox image repair**
13. **Gateway service migration and cleanup**
14. **Security warnings**
15. **systemd linger check** (Linux)
16. **Skills status**
17. **Gateway auth check**
18. **Runtime and port diagnostics**
19. **Runtime best practices** (Node vs Bun check)

</details>

## 5. Heartbeat Mechanism

Heartbeat lets the lobster proactively check in on a schedule — instead of passively waiting for messages, it actively patrols. Each heartbeat is a full agent conversation turn, and **shorter intervals burn more tokens**.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",   // every 30 minutes
        target: "last", // send to the most recently contacted channel
      },
    },
  },
}
```

Create `~/.openclaw/workspace/HEARTBEAT.md` in the workspace to tell the lobster what to check. When there is nothing to report, the lobster replies `HEARTBEAT_OK` and stays silent; it only sends an alert when something important comes up.

```markdown
# Heartbeat Checklist

- Quick scan: any urgent emails in the inbox?
- If a task is blocked, remind me on the next conversation
```

> Save money: keep `HEARTBEAT.md` short, or set `target: "none"` to run internally without sending messages.

<details>
<summary>Advanced Heartbeat Configuration</summary>

**Active hours** (avoid late-night interruptions):

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last",
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "Asia/Shanghai",
        },
      },
    },
  },
}
```

**Multiple agents with independent heartbeats**:

```json5
{
  agents: {
    defaults: {
      heartbeat: { every: "30m", target: "last" },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678",
          prompt: "Check server status. If everything is normal, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

**Visibility controls** (customized per channel):

- `showOk: false` (default): silently swallow `HEARTBEAT_OK`
- `showAlerts: true` (default): send alert content
- `useIndicator: true` (default): emit an indicator event

**Manually trigger a heartbeat**:

```bash
openclaw system event --text "Check urgent follow-ups" --mode now
```

**Lightweight context mode**:

Set `lightContext: true` to make the heartbeat only inject `HEARTBEAT.md`, reducing context size and token consumption.

**Reasoning passthrough**:

Set `includeReasoning: true` to additionally send a message prefixed with `Reasoning:` containing the reasoning process, which helps you understand why the lobster decided to notify you. It is recommended to disable this in group chats.

</details>

## 6. Sandbox and Tool Policies

Two independent security control layers:

| Control Layer | Purpose | Configuration |
|---------------|---------|---------------|
| **Sandbox** | **Where** tools run (Docker container vs. host) | `agents.defaults.sandbox.mode` |
| **Tool policy** | **Which** tools are available | `tools.allow` / `tools.deny` |

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "non-main" },  // off | non-main | all
    },
  },
  tools: {
    deny: ["browser", "canvas"],  // deny always takes precedence
  },
}
```

- `non-main`: group chat / channel sessions run in a Docker sandbox; the main session runs directly on the host
- `all`: all sessions are sandboxed

```bash
openclaw sandbox explain   # view the currently active policy
```

<details>
<summary>Tool Groups (Shorthand)</summary>

Tool policies support `group:*` shorthand:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Available groups:

| Group | Included Tools |
|-------|---------------|
| `group:runtime` | exec, bash, process |
| `group:fs` | read, write, edit, apply_patch |
| `group:sessions` | sessions_list, sessions_history, etc. |
| `group:memory` | memory_search, memory_get |
| `group:ui` | browser, canvas |
| `group:automation` | cron, gateway |
| `group:messaging` | message |
| `group:openclaw` | all built-in tools (excluding plugin tools) |

</details>

<details>
<summary>Sandbox Docker Image and Setup</summary>

Default image: `openclaw-sandbox:bookworm-slim`

```bash
# Build the base sandbox image
scripts/sandbox-setup.sh

# Build a more fully-featured image (includes curl, jq, nodejs, python3, git)
scripts/sandbox-common-setup.sh

# Build the sandbox browser image
scripts/sandbox-browser-setup.sh
```

The default sandbox container has **no network** access. To enable networking, set `agents.defaults.sandbox.docker.network`.

**Custom bind mounts**:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
  },
}
```

> **Security note**: Bind mounts penetrate the sandbox filesystem. OpenClaw blocks dangerous bind sources (such as `docker.sock`, `/etc`, `/proc`, `/sys`). Use `:ro` for sensitive mounts.

**One-time in-container initialization** (`setupCommand`):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          setupCommand: "apt-get update && apt-get install -y curl jq",
          network: "bridge",        // network needed for package installation
          readOnlyRoot: false,      // write access needed for package installation
          user: "0:0",             // root needed for package installation
        },
      },
    },
  },
}
```

</details>

## 7. Remote Access and Networking

### Network Model

Gateway listens on the local loopback address (`127.0.0.1:18789`) by default. There are three ways to enable remote access:

| Method | Use Case | Recommendation |
|--------|----------|----------------|
| **Tailscale / VPN** | Secure access across networks | Recommended |
| **SSH tunnel** | Any environment with SSH | Universal fallback |
| **LAN binding** | Local network access | Requires authentication |

### SSH Tunnel (Simplest)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@remote-host
```

Then connect the local client to `ws://127.0.0.1:18789`. Even over an SSH tunnel, a Gateway configured with authentication still requires the client to send a token/password.

### Tailscale Binding

```json5
{
  gateway: {
    bind: "tailnet",
    auth: {
      mode: "token",
      token: "your-gateway-password",
    },
    tailscale: { mode: "serve" },
  },
}
```

<details>
<summary>LAN Discovery (Bonjour / mDNS)</summary>

Gateway can broadcast its WebSocket endpoint on the local network via Bonjour, making it easy for clients to discover automatically:

- Service type: `_openclaw-gw._tcp`
- TXT records include port, TLS status, Tailnet DNS, and other hints

**Bonjour limitations**: LAN only — does not cross networks. Use Tailscale or SSH for cross-network access.

Debug commands (macOS):

```bash
dns-sd -B _openclaw-gw._tcp local.    # browse instances
dns-sd -L "<instance-name>" _openclaw-gw._tcp local.  # resolve details
```

**Cross-network Bonjour (Wide-Area DNS-SD over Tailscale)**:

If the node and Gateway are on different networks, you can use Tailscale with unicast DNS-SD for cross-network discovery:

```json5
{
  gateway: { bind: "tailnet" },
  discovery: { wideArea: { enabled: true } },
}
```

One-time DNS server setup: `openclaw dns setup --apply`

Disable broadcasting: set the environment variable `OPENCLAW_DISABLE_BONJOUR=1`.

</details>

<details>
<summary>Running Multiple Gateways (Advanced)</summary>

Most scenarios only require one Gateway. If you need strict isolation or redundancy (e.g., a rescue bot), you can run multiple Gateways.

**Configuration that must be unique per instance**:

- `gateway.port` (or `--port`)
- `OPENCLAW_CONFIG_PATH`
- `OPENCLAW_STATE_DIR`
- `agents.defaults.workspace`

**Recommended: use Profile isolation**:

```bash
# Main Gateway
openclaw --profile main gateway --port 18789

# Rescue Gateway
openclaw --profile rescue gateway --port 19001

# Install each as a system service separately
openclaw --profile main gateway install
openclaw --profile rescue gateway install

# Check status separately
openclaw --profile main status
openclaw --profile rescue status
```

Port spacing recommendation: leave at least 20 ports between instances to avoid conflicts with derived ports (browser CDP, etc.).

</details>

<details>
<summary>Gateway Lock Mechanism</summary>

Gateway uses TCP port binding as a lock — only one instance can run on the same port.

- If the port is in use, a `GatewayLockError` is thrown at startup
- When the process exits (including crashes and SIGKILL), the OS automatically releases the port
- No extra lock files or cleanup steps are needed

If the port is occupied by another process, use `openclaw gateway --port <other-port>` or `--force` to forcibly release it.

</details>

<details>
<summary>HTTP API Endpoints (Developer / Integration Scenarios)</summary>

## 8. HTTP API Endpoints

Gateway can expose HTTP API endpoints, which are **disabled by default** and must be manually enabled.

> **Security Warning**: HTTP endpoints = full operator permissions. **Do not expose to the public internet** — restrict to loopback / Tailnet / private networks only.

### OpenAI Chat Completions Compatible Endpoint

```json5
{
  gateway: {
    http: { endpoints: { chatCompletions: { enabled: true } } },
  },
}
```

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer your-gateway-password' \
  -H 'Content-Type: application/json' \
  -d '{"model": "openclaw:main", "messages": [{"role":"user","content":"Hello"}]}'
```

Streaming output is supported: add `"stream": true`. The response format is SSE and ends with `data: [DONE]`.

### OpenResponses Compatible Endpoint

```json5
{
  gateway: {
    http: { endpoints: { responses: { enabled: true } } },
  },
}
```

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer your-gateway-password' \
  -H 'Content-Type: application/json' \
  -d '{"model": "openclaw:main", "input": "Hello"}'
```

### Models and Embeddings Endpoints

For broader client and RAG pipeline compatibility, the Gateway provides model listing and embedding endpoints:

```bash
# List available models
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer your-gateway-password'

# Embeddings
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer your-gateway-password' \
  -H 'Content-Type: application/json' \
  -d '{"model": "text-embedding-3-small", "input": "Hello world"}'
```

Explicit model override forwarding is supported through the `model` field in `/v1/chat/completions` and `/v1/responses` endpoints — the Gateway passes the override to the upstream provider.

### Tools Invoke Endpoint (Enabled by Default)

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer your-gateway-password' \
  -H 'Content-Type: application/json' \
  -d '{"tool": "sessions_list", "action": "json", "args": {}}'
```

### Agent Selection

Select an agent via the `model` field: `"openclaw:main"`, `"openclaw:beta"`, `"agent:ops"`.
Or via header: `x-openclaw-agent-id: main`.

### CLI Backends (Local AI CLI Fallback Channel)

```bash
openclaw agent --message "hi" --model claude-cli/opus-4.6
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Configure as fallback:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/opus-4.6"],
      },
    },
  },
}
```

Limitations: CLI Backend mode has no OpenClaw tool calls and does not support streaming output.

</details>

<details>
<summary>Background Command Execution (exec / process tools)</summary>

Gateway runs shell commands via the `exec` tool and manages background tasks via the `process` tool.

### exec Tool Parameters

| Parameter | Description |
|-----------|-------------|
| `command` | The command to execute (required) |
| `background` | Execute immediately in the background |
| `timeout` | Timeout in seconds, default 1800 (30 minutes) |
| `yieldMs` | Auto-background delay, default 10000ms |
| `elevated` | Run on the host machine (in sandbox mode) |

Foreground commands return output directly; background commands return a `sessionId` that can be queried later with the `process` tool.

### process Tool Operations

| Operation | Description |
|-----------|-------------|
| `list` | List running and completed background tasks |
| `poll` | Read new output (including exit status) |
| `log` | Read the full output log (supports offset + limit) |
| `write` | Send stdin input |
| `kill` | Terminate a background task |
| `clear` | Clear in-memory records of completed tasks |

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,   // auto-background delay (milliseconds)
      timeoutSec: 1800,      // timeout (seconds)
      cleanupMs: 1800000,    // cleanup delay after completion (milliseconds)
      notifyOnExit: true,    // trigger heartbeat notification when background task exits
    },
  },
}
```

> Background tasks are lost on process restart (not persisted to disk).

</details>

## 10. Container Execution

If OpenClaw runs inside a Docker or Podman container, use the `--container` flag to execute commands directly inside the container without manual `docker exec`:

```bash
openclaw --container status
openclaw --container gateway status
openclaw --container config set agent.model "anthropic/claude-opus-4-6"
```

You can also set this via environment variable: `OPENCLAW_CONTAINER=1`.

## 11. Troubleshooting

When problems occur, run the following in order:

```bash
openclaw status                    # overall status
openclaw gateway status            # Gateway status
openclaw logs --follow             # live logs
openclaw doctor                    # auto-diagnostics (most issues are resolved here)
openclaw channels status --probe   # channel probing
```

### Common Issues Quick Reference

**Gateway refuses to start**:

| Error | Cause | Solution |
|-------|-------|----------|
| `refusing to bind gateway ... without auth` | Non-loopback binding without configured auth | Configure `gateway.auth.token` |
| `another gateway instance is already listening` / `EADDRINUSE` | Port in use | Change port or use `--force` |
| `Gateway start blocked: set gateway.mode=local` | Remote mode configured | Set `gateway.mode="local"` |

**No response**:

| Symptom | What to Check |
|---------|---------------|
| Channel connected but no response | DM policy (pairing / allowlist), group mention rules (`requireMention`) |
| Pairing pending approval | `openclaw pairing list --channel <channel>` |
| Message filtered by policy | Search logs for `blocked`, `pairing request`, `mention required` |

**Dashboard cannot connect**:

| Error | Solution |
|-------|----------|
| `device identity required` | Not a secure context or missing device authentication |
| `unauthorized` | Auth token mismatch — check `gateway.auth.token` |
| `gateway connect failed` | Check whether the URL/port is correct |

**Heartbeat / Cron not working**:

```bash
openclaw cron list                        # view cron jobs
openclaw cron runs --id <jobId> --limit 20  # view execution history
openclaw system heartbeat last            # last heartbeat info
```

Common causes: cron not enabled, skipped outside active hours, DM policy blocking delivery (`directPolicy: "block"`).

<details>
<summary>Issues After Upgrading</summary>

Most post-upgrade issues are config drift or stricter defaults being enforced:

1. **Auth and URL behavior changes**
   - With `gateway.mode=remote`, the CLI may point to a remote instance
   - Check `openclaw config get gateway.mode` and `gateway.auth.mode`

2. **Stricter binding and auth rules**
   - Non-loopback bindings now require authentication to be configured
   - `gateway.token` (old) is not the same as `gateway.auth.token` (new)

3. **Pairing and device identity state changes**
   - Check `openclaw devices list` and `openclaw pairing list`

If the service config and runtime are inconsistent, reinstall the service metadata:

```bash
openclaw gateway install --force
openclaw gateway restart
```

</details>

<details>
<summary>Browser Tool Issues</summary>

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw doctor
```

Common issues:
- `Failed to start Chrome CDP on port` — browser process failed to start
- `browser.executablePath not found` — configured path is invalid
- `userDataDir not accessible` — browser user data directory permission issue

</details>

## 12. FAQ

**Q: The config is broken and Gateway won't start. What do I do?**

A: Run `openclaw doctor` — it will tell you what's wrong, and `--fix` will auto-repair most issues.

**Q: What is the priority order for port and binding settings?**

A: Port: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`. Binding: CLI override > `gateway.bind` > `loopback`.

**Q: Will frequent heartbeats be expensive?**

A: Yes. Recommendations: use an interval of `1h` or longer, keep `HEARTBEAT.md` short, or set `target: "none"` for internal-only checks.

**Q: What is the relationship between sandbox and elevated (privilege escalation)?**

A: The sandbox determines where tools run. Elevated is the sandbox's "escape hatch" — used when you are inside the sandbox but need to execute a command on the host machine. Use `openclaw sandbox explain` to view the current policy.

**Q: Is it safe for multiple people to share one Gateway?**

A: OpenClaw is a **single-user personal assistant model**, not a multi-tenant system. Sharing a Gateway = sharing tool authorization. When isolation is needed, run separate Gateway instances.

**Q: How do I switch between local and cloud models?**

A: See [Chapter 5: Model Management](/en/adopt/chapter5/). Short version: use `models.mode: "merge"` + `model.fallbacks` — if the local model goes down, it automatically falls back to a cloud model.
