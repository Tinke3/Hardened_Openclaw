# Phase 7: Home Assistant over MCP

## Goal

Let the agent read a few sensors and switch a few lights in Home Assistant, with every limit enforced outside the model.

Home Assistant runs in a different, trusted network segment. This is the first time the agent is allowed to reach across that boundary.

## Assessment before the change

| Question | Answer |
|---|---|
| Private data reachable | Yes. The state of every entity exposed to the agent. |
| Untrusted content | Yes. Device names and state text reach the model, as does text forwarded in the chat. |
| Outbound effect | Yes. Physical devices, and messages in the owner's chat. |
| Worst credible case | The owner's Telegram account is taken over, or text planted in a device state steers the agent, and it switches something. How bad that is depends entirely on what is exposed. |
| Technical limits | Home Assistant's exposure list. A non-admin Home Assistant user limited to the Assist API. A per-server tool filter in OpenClaw. One destination address and port in the Pi's firewall and in the router. The Telegram sender allowlist. |
| Prompt-level limits | Rules in `AGENTS.md`. Not relied upon. |

All three risk factors are present at once. The consequence is kept minor and reversible by exposing only lights and temperature sensors. Locks, alarm panels, doors, cameras, switches, scripts, scenes and automations are not exposed.

## Options considered

| Option | For | Against |
|---|---|---|
| Home Assistant's own MCP Server integration (chosen) | Typed tools, no shell. Limits sit in Home Assistant, in OpenClaw and in the network. | Plain HTTP unless Home Assistant is given TLS |
| A self-written REST skill | Full control over every call | Needs the `exec` tool, which stays denied |
| A third-party skill | Quick | Third-party code next to a long-lived token, and `exec` |

## Steps

### 1. Limit outbound messages to the current conversation

The sender allowlist limits who can write to the agent, not whom the agent can write to. The key names come from `openclaw config schema` on the tested version.

> [!WARNING]
> Changes configuration.

```bash
cat > ~/message-guard.patch.json5 <<'EOF'
{
  tools: {
    message: {
      crossContext: {
        allowWithinProvider: false,
        allowAcrossProviders: false,
      },
      broadcast: { enabled: false },
    },
  },
}
EOF

openclaw config patch --file ~/message-guard.patch.json5 --dry-run
openclaw config patch --file ~/message-guard.patch.json5
openclaw config get tools.message
```

### 2. Prepare Home Assistant

In the Home Assistant UI, as an administrator:

1. **Create a user for the agent** under Settings > People > Users. Leave Administrator off.
2. **Add the Model Context Protocol Server integration.** Select only the Assist API and turn off "Require an administrator account", so that the non-admin user can use it.
3. **Reduce the exposure list** under Settings > Voice assistants > Expose to the lights and temperature sensors the agent should see. Turn off automatic exposure of new entities. The list applies to every Assist client, including voice assistants.
4. **Create a long-lived access token** while logged in as the agent's user. Store it in a password manager.

### 3. Open one path in the router

Allow TCP from `<PI_IP>` to `<HA_IP>` port 8123, as single addresses, above the rule that isolates the agent's segment. Return traffic must be allowed.

### 4. Open the same path in the Pi's firewall

> [!WARNING]
> Changes the firewall ruleset.

```bash
sudo cp -a /etc/nftables.conf /etc/nftables.conf.bak2
sudo sed -i '/agent to private/i\        ip daddr <HA_IP> tcp dport 8123 accept comment "home assistant"' /etc/nftables.conf
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
sudo nft list chain inet filter agent_out
```

The rule sits above the private-range rule in `agent_out`. No rollback timer is needed, because the change affects only the agent account's outbound traffic.

Test without a token:

```bash
sudo -iu openclaw curl -sS -m 5 -o /dev/null -w 'code: %{http_code}\n' http://<HA_IP>:8123/api/mcp/assist
sudo -iu openclaw curl -sS -m 5 -o /dev/null http://<HA_IP>:80; echo "exit $?"
```

| Result of the first command | Meaning |
|---|---|
| `code: 405` | The path works and the endpoint exists. It accepts POST only. |
| `code: 404` | The integration is missing or the Assist API is not selected |
| Timeout | The router rule is missing or below the isolating rule |
| Immediate failure, exit 7 | The Pi's own rule did not load |

The second command must fail immediately with exit code 7.

### 5. Add the MCP server

No documented way was found to reference the token from a file, so it is stored as a literal header in `openclaw.json` (mode 600). The commands below keep it out of shell history and out of the process list.

> [!WARNING]
> Stores the Home Assistant token on disk and gives the agent new tools. Do not message the bot until step 6 is done.

```bash
read -rs -p "HA token: " T; echo
printf '{"mcp":{"servers":{"homeassistant":{"url":"http://<HA_IP>:8123/api/mcp/assist","transport":"streamable-http","headers":{"Authorization":"Bearer %s"}}}}}' "$T" | openclaw config patch --stdin --dry-run
printf '{"mcp":{"servers":{"homeassistant":{"url":"http://<HA_IP>:8123/api/mcp/assist","transport":"streamable-http","headers":{"Authorization":"Bearer %s"}}}}}' "$T" | openclaw config patch --stdin
unset T
openclaw mcp probe homeassistant --json
```

On Home Assistant Core 2026.10.0 with only lights and temperature sensors exposed, the probe listed seven tools: `GetLiveContext`, `GetDateTime`, `HassTurnOn`, `HassTurnOff`, `HassLightSet`, `HassBroadcast` and `HassCancelAllTimers`. The agent sees them with a `homeassistant__<domain>__` prefix.

### 6. Filter the tools

`HassBroadcast` announces text on voice assistant devices. It and the timer tool are left out.

> [!WARNING]
> Changes configuration.

```bash
openclaw mcp tools homeassistant --include '*GetLiveContext,*GetDateTime,*HassTurnOn,*HassTurnOff,*HassLightSet'
openclaw mcp reload
openclaw mcp probe homeassistant --json
```

Expected: `"tools": 5` and `"filteredTools": 2`.

### 7. Tell the agent what it can now do

`AGENTS.md` states the agent's capabilities, so it changes with them. The updated file is in [../examples/workspace/AGENTS.md](../examples/workspace/AGENTS.md). Commit the change in the workspace repository.

## Verify

In a fresh session (`/new`), with your own device names:

1. Ask for a room temperature. The agent reads it.
2. Ask it to turn a light on, then off. The light switches.
3. Ask it to announce something on a speaker. It says it cannot.

All three behaved as expected on the tested environment.

TODO: not verified that Home Assistant's Logbook attributes the changes to the agent's user.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
openclaw mcp unset homeassistant
sudo cp -a /etc/nftables.conf.bak2 /etc/nftables.conf && sudo nft -f /etc/nftables.conf
```

Delete the token in Home Assistant under the agent user's profile, and remove the router rule.

## Security notes

- **The exposure list is the boundary.** `HassTurnOn` and `HassTurnOff` work on any exposed entity, not only lights. If a lock is exposed later, the same tools operate it.
- **The connection is plain HTTP.** The token crosses the router unencrypted. A compromised device in the agent's segment could capture it. The damage is limited to what the non-admin token and the exposure list allow.
- **The token is a literal in the configuration file.** `openclaw mcp doctor` reports this.
- Exposing a script, scene or automation hands the agent whatever that script does.

## References

- [Home Assistant: Model Context Protocol Server](https://www.home-assistant.io/integrations/mcp_server/)
- [OpenClaw: Connect MCP servers](https://docs.openclaw.ai/tools/mcp)
- [OpenClaw: Manage saved MCP servers](https://docs.openclaw.ai/cli/mcp/registry)
- [OpenClaw: MCP transports and OAuth](https://docs.openclaw.ai/cli/mcp/transports)
