# Phase 3: Onboarding and tool policy

## Goal

Configure the model and the Gateway with no channel attached, then reduce the agent's tools to the minimum before anything can reach it.

## Options considered

| Onboarding path | Result | Assessment |
|---|---|---|
| `openclaw onboard`, Quick start | Full access, foreground gateway, opens a browser | Unsuitable for a headless, restricted setup |
| `--classic`, QuickStart | Loopback bind, generated token, tool profile `full` | Good network defaults, permissive tools |
| `--classic --flow manual` (chosen) | Every step is asked | You see and decide each setting |

Onboarding sets `tools.profile` to `full` when no profile is configured, so the tools are restricted immediately afterwards, before a channel is added.

**Code Mode**

| Option | For | Against |
|---|---|---|
| Off (chosen) | The model never writes code that is executed | Larger prompts. Negligible with a handful of tools. |
| QuickJS executor | Guest code runs in a WASM sandbox | The model still executes code. One more layer to trust. |
| Node executor (the default) | | `node:vm` is documented as not being a security boundary |

## Steps

### 1. Prepare an API key

Create a dedicated key for the agent in its own workspace at the model provider and set a spend limit on that workspace. Paste the key into the wizard prompt. Do not pass it as a command-line argument.

### 2. Onboard without channels

> [!WARNING]
> Writes configuration, stores the API key on disk and installs a systemd user service.

```bash
openclaw onboard --classic --flow manual
```

Answers used: local mode, default workspace, Anthropic with an API key, port 18789, bind loopback, token auth, Tailscale off, plaintext secret storage, **skip all channels**, skip web search, skip all skills, install the daemon.

If the wizard appears to hang at "Preparing Gateway service…", see [troubleshooting.md](troubleshooting.md).

### 3. Restrict tools

> [!WARNING]
> Changes configuration.

```bash
cat > ~/hardening.patch.json5 <<'EOF'
{
  tools: {
    profile: "messaging",
    deny: [
      "group:runtime",
      "group:fs",
      "group:web",
      "group:ui",
      "group:nodes",
      "group:automation",
      "sessions_spawn",
      "sessions_send",
    ],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
    sessions: { visibility: "agent" },
    agentToAgent: { enabled: false },
  },
  agents: {
    defaults: {
      heartbeat: { every: "0m" },
    },
  },
}
EOF

openclaw config patch --file ~/hardening.patch.json5 --dry-run
openclaw config patch --file ~/hardening.patch.json5
openclaw config validate
```

The profile selects a base set. The deny list removes tools by name or group and always wins over an allow. Heartbeat is disabled so the agent does not run turns on its own.

### 4. Disable features that are on by default

Three settings are absent from a fresh configuration and are active anyway.

> [!WARNING]
> Changes configuration and restarts the gateway.

```bash
openclaw config set browser.enabled false --strict-json
openclaw config set tools.codeMode false --strict-json
openclaw config set tools.toolSearch false --strict-json
openclaw gateway restart
```

- **Browser control** defaults to enabled.
- **Code Mode** behaves as `"auto"` when unset and engages for current Claude models. The model then sees an `exec` tool that runs JavaScript it writes inside the gateway process. Denying `group:runtime` does not remove it.
- **Tool Search** is used automatically when unset. It is harmless, but with it on the model's own account of its tools is harder to interpret.

### 5. Remove remaining tools

After a channel exists (phase 4), the agent reported three more tools. They are denied by name. A patch replaces the whole array, so the full list is repeated:

```bash
cat > ~/tools-deny.patch.json5 <<'EOF'
{
  tools: {
    deny: [
      "group:runtime",
      "group:fs",
      "group:web",
      "group:ui",
      "group:nodes",
      "group:automation",
      "sessions_spawn",
      "sessions_send",
      "intent",
      "presence",
      "subagents",
    ],
  },
}
EOF

openclaw config patch --file ~/tools-deny.patch.json5 --dry-run
openclaw config patch --file ~/tools-deny.patch.json5
openclaw gateway restart
```

The resulting policy is in [../examples/openclaw-policy.json5](../examples/openclaw-policy.json5).

## Verify

```bash
openclaw gateway status
ss -tlnp | grep 18789
stat -c '%a %U %n' ~/.openclaw/openclaw.json
openclaw config get tools
openclaw security audit
```

Expected: service enabled and running, listening on `127.0.0.1:18789` and `[::1]:18789` only, configuration file mode `600`, and the audit shows `tools.elevated: disabled` and `browser control: disabled` with 0 critical findings.

**Behavioural check (after phase 4).** Send `/new` to the bot, then ask it to list its tools exactly as defined and to attempt a call to `intent`. On the tested version the agent listed 11 tools: `ask_user`, `conversations_list`, `conversations_send`, `conversations_turn`, `message`, `session_status`, `sessions`, `sessions_history`, `sessions_list`, `sessions_search`, `sessions_yield`. It could not call `intent`.

Use a fresh session for this. In an existing session the model repeated tools from earlier answers that were no longer available.

## Rollback

OpenClaw keeps backups next to the configuration file (`openclaw.json.bak`). Individual settings can be removed with `openclaw config unset <path>`.

## Security notes

- The audit reports permissive defaults as information, not as findings. A clean audit does not mean a restricted agent.
- The model's description of its tools is a useful alarm and a poor measurement. Treat it as a prompt to investigate.
- With file tools denied, the agent cannot write its own identity files. They are written by hand. An agent that can edit its own system prompt can be persuaded to change it permanently.
- Skills that call an HTTP API through a shell need `exec`. Typed tools from an MCP server do not.

## References

- [OpenClaw: Onboarding (CLI)](https://docs.openclaw.ai/start/wizard)
- [OpenClaw: Hardened baselines](https://docs.openclaw.ai/gateway/security/hardened-baseline)
- [OpenClaw: Tool policy](https://docs.openclaw.ai/gateway/config-tools/tool-policy)
- [OpenClaw: Code Mode configuration](https://docs.openclaw.ai/tools/code-mode/configuration)
- [OpenClaw: Code Mode executors](https://docs.openclaw.ai/tools/code-mode/executors)
- [OpenClaw: Tool Search](https://docs.openclaw.ai/tools/tool-search)
- [OpenClaw: Heartbeat](https://docs.openclaw.ai/gateway/heartbeat)
- [OpenClaw: Browser configuration](https://docs.openclaw.ai/tools/browser/configuration)
- [OpenClaw: Config CLI](https://docs.openclaw.ai/cli/config)
