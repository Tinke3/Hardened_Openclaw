# Threat model

## Starting assumption

A language model can be misled. Any text the agent reads may contain instructions that it follows. A safeguard that depends on the model obeying its prompt is not counted as a safeguard here.

## Components

| Component | Runs as | Holds |
|---|---|---|
| Raspberry Pi OS | root, `<ADMIN_USER>` | SSH host keys, system configuration |
| OpenClaw Gateway | `openclaw` (unprivileged, systemd user service) | Model API key, gateway token, Telegram bot token, session history |
| Model provider | External service | Every prompt and reply |
| Telegram | External service | Every message exchanged with the bot. Bot chats are not end-to-end encrypted. |

## Data flows

- Administrator to Pi: SSH from `<MGMT_SUBNET>`, public key only.
- Gateway to model provider: outbound HTTPS.
- Gateway to Telegram: outbound HTTPS long polling. No inbound connection.
- Control UI: loopback only. Reached through an SSH tunnel when needed.

## Trust boundaries

1. **Network.** The Pi is in its own segment. Its firewall accepts new inbound connections only for SSH and ping from `<MGMT_SUBNET>`. Outbound, processes of the agent account may use DNS and HTTPS to public addresses and nothing else.
2. **OS user.** The agent account has no password, no sudo, no SSH access and no membership in privileged groups. Home directories are mode 700.
3. **Gateway access control.** Only one numeric Telegram user id may message the agent. Groups are disabled.
4. **Gateway tool policy.** The agent can exchange messages and inspect its own sessions. It cannot read or write files, run commands, fetch web content, drive a browser, schedule work or spawn sub-agents. It has no skills, and no hook stores or runs anything on its behalf.

## What this defends against

- Password guessing and root login over SSH.
- A service accidentally bound to all interfaces becoming reachable.
- Strangers messaging the bot.
- A prompt injection turning into command execution or file access on the host, as far as gateway tool policy holds.
- A compromised agent process reaching the administrator's files or becoming root through sudo.

## What it does not defend against

- **Compromise of the owner's Telegram account.** The attacker becomes the owner as far as the agent can tell. Enable two-step verification on the account.
- **The agent account's own secrets.** Anything running as `openclaw` can read the API key and tokens. Tool policy keeps the model away from them. A code-execution bug in the gateway would not.
- **The external services.** The model provider and Telegram see the content that passes through them.
- **Kernel or gateway vulnerabilities**, beyond keeping both patched.
- **Data leaving over HTTPS or DNS.** The agent account can reach any public HTTPS server and can make DNS queries. Local networks and all other ports are blocked for it. A destination allowlist is not in place.

## Lesson from testing

Configuration files show what was requested, not what is in effect. Two defaults that widened the agent's capabilities were absent from the configuration and were found only by asking the agent, in a fresh session, to list its tools. See [03-onboarding-and-tool-policy.md](03-onboarding-and-tool-policy.md).
