# Phase 4: Telegram channel

## Goal

Let one person talk to the agent through a Telegram bot, and nobody else.

This is the first path by which input from outside the machine reaches the model.

## Assessment before the change

| Question | Answer |
|---|---|
| Private data reachable | Conversation history only. The agent has no file, command or web tools. |
| Untrusted content | Limited. Messages come from the owner, but forwarded or pasted text can carry instructions. |
| Outbound effect | Messages to the owner. Nothing else. |
| Worst credible case | The owner's Telegram account is taken over. The attacker is the owner as far as the agent can tell. |
| Technical limits | Numeric sender allowlist checked by the gateway before the model, groups disabled, tool deny list, long polling with no inbound port, provider spend limit |
| Prompt-level limits | None relied upon |

Telegram's default transport in OpenClaw is long polling, so the firewall needs no inbound rule.

## Options considered

| Option | For | Against |
|---|---|---|
| Telegram, allowlist of one (chosen) | Simple, works from a phone | Content passes through Telegram unencrypted end to end |
| Signal | End-to-end encrypted | Separate number, more to maintain |
| Control UI over an SSH tunnel only | No third party | Not usable from a phone |

## Steps

### 1. Create the bot

In Telegram, talk to `@BotFather` (check the handle exactly), run `/newbot`, and store the token in a password manager. Run `/setjoingroups` and choose Disable.

Enable two-step verification on your own Telegram account.

### 2. Store the token in a file

The quick-start command takes the token as an argument, which leaves it in shell history. A token file avoids that and keeps the configuration free of secrets.

```bash
install -m 600 /dev/null ~/.openclaw/telegram.token
read -rs -p "Bot token: " T && printf '%s' "$T" > ~/.openclaw/telegram.token; unset T; echo
```

### 3. Enable the channel in pairing mode

Pairing mode is used once, to learn your numeric user id without a third-party bot.

> [!WARNING]
> Changes configuration and opens the first channel.

```bash
cat > ~/telegram.patch.json5 <<'EOF'
{
  channels: {
    telegram: {
      enabled: true,
      tokenFile: "/home/openclaw/.openclaw/telegram.token",
      dmPolicy: "pairing",
      groupPolicy: "disabled",
    },
  },
}
EOF

openclaw config patch --file ~/telegram.patch.json5 --dry-run
openclaw config patch --file ~/telegram.patch.json5
openclaw channels status --probe
```

### 4. Pair

Send any message to the bot. It replies with a pairing code and your numeric user id.

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

The first approved pairing also makes you the command owner (`commands.ownerAllowFrom`).

### 5. Lock to an allowlist

> [!WARNING]
> Changes configuration.

```bash
cat > ~/telegram-lock.patch.json5 <<'EOF'
{
  channels: {
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["<TELEGRAM_USER_ID>"],
    },
  },
}
EOF

openclaw config patch --file ~/telegram-lock.patch.json5 --dry-run
openclaw config patch --file ~/telegram-lock.patch.json5
```

Both keys go in one change because an allowlist with no entries is rejected. Access no longer depends on a stored pairing approval, and other senders no longer receive a pairing code.

## Verify

```bash
openclaw config get channels.telegram
openclaw config get commands.ownerAllowFrom
openclaw channels status --probe
openclaw security audit | grep -E 'Summary|CRIT|WARN|groups'
```

Expected: `dmPolicy` is `allowlist`, `groupPolicy` is `disabled`, the status line shows `mode:polling` and `works`, and the audit shows 0 critical findings.

Then run the behavioural check described in [03-onboarding-and-tool-policy.md](03-onboarding-and-tool-policy.md#verify).

## Rollback

```bash
openclaw config set channels.telegram.enabled false --strict-json
```

Revoke the token in BotFather with `/revoke` and delete `~/.openclaw/telegram.token`.

## Security notes

- Without the allowlist, anyone who finds the bot's username can request pairing.
- Two-step verification on the owner's account is the only technical limit on the worst case. It matters more once the agent controls devices.
- The bot username and your numeric id are not secrets. The token is.

## References

- [OpenClaw: Telegram setup](https://docs.openclaw.ai/channels/telegram/setup)
- [OpenClaw: Telegram access control](https://docs.openclaw.ai/channels/telegram/access-control)
- [OpenClaw: Telegram transports](https://docs.openclaw.ai/channels/telegram/transports)
