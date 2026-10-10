# OpenClaw on a Raspberry Pi 5, hardened

A step-by-step record of installing an [OpenClaw](https://docs.openclaw.ai/) agent on a Raspberry Pi 5, with the host and the agent locked down before the agent is given anything to do.

This guide describes one person's home setup. It is not a general hardening standard. Every command in it was run on the tested environment below, and each phase ends with a verification step.

## Scope

Covered so far:

- Hardening Raspberry Pi OS Lite (SSH, updates, firewall)
- A dedicated, unprivileged OS user for the agent
- Installing OpenClaw without running any third-party code as root
- Onboarding with no channels, then restricting the agent's tools
- A Telegram channel restricted to a single owner
- A short, hand-written agent prompt, with bundled hooks and skills switched off
- Per-user egress filtering for the agent account

Not covered yet: Home Assistant integration, sandboxing, secret references, a destination allowlist for egress, backups and updates.

## Tested environment

| Item | Value |
|---|---|
| Hardware | Raspberry Pi 5, 16 GB RAM, 500 GB USB 3 SSD (UAS) |
| OS | Raspberry Pi OS Lite 64-bit, Debian 13.7 (trixie) |
| Kernel | 6.18.50+rpt-rpi-2712, aarch64 |
| nftables | 1.1.3 |
| OpenClaw | 2026.9.9 (bcfc888) |
| Node.js | 24.21.0, installed by `install-cli.sh` |
| Model provider | Anthropic API |
| Network | Wi-Fi for phases 1 to 5, Ethernet for phase 6. IPv4 by DHCP, IPv6 link-local only. |
| Test date | 2026-10-10 |

## Architecture in one paragraph

The Pi sits in an isolated network segment. It is administered over SSH from a separate management subnet. The OpenClaw Gateway runs as a systemd user service under an unprivileged account, listens on loopback only, and reaches the model provider and Telegram with outbound connections. No inbound port other than SSH is open, and the agent account can open outbound connections only for DNS and HTTPS to public addresses. See [docs/00-threat-model.md](docs/00-threat-model.md).

## Phases

1. [OS hardening](docs/01-os-hardening.md)
2. [Agent user and installation](docs/02-agent-user-and-install.md)
3. [Onboarding and tool policy](docs/03-onboarding-and-tool-policy.md)
4. [Telegram channel](docs/04-telegram-channel.md)
5. [Agent prompt, hooks and skills](docs/05-agent-prompt-hooks-skills.md)
6. [Egress filtering for the agent user](docs/06-egress-filtering.md)

Problems met along the way are in [docs/troubleshooting.md](docs/troubleshooting.md).

## Placeholders

| Placeholder | Meaning |
|---|---|
| `<ADMIN_USER>` | Your administrative account on the Pi (has sudo) |
| `<PI_HOST>` | Hostname or address of the Pi |
| `<MGMT_SUBNET>` | Subnet you administer the Pi from. Examples use `198.51.100.0/24`. |
| `<AGENT_UID>` | Numeric uid of the `openclaw` user |
| `<AGENT_ID>` | Agent id chosen during onboarding |
| `<TELEGRAM_USER_ID>` | Your numeric Telegram user id |
| `<TELEGRAM_BOT_TOKEN>` | Token issued by BotFather. Never commit it. |
| `<DNS_RESOLVER>` | Address of the DNS resolver in `/etc/resolv.conf`. The example ruleset uses `192.0.2.53`. |
| `<AGENT_NAME>`, `<OWNER_NAME>`, `<LANGUAGE>`, `<TIMEZONE>` | Values in the example workspace files |

The agent account is called `openclaw` throughout.

## A note on trust

Model-level instructions are not treated as a security control anywhere in this guide. Every limit that matters is enforced outside the model: OS user, file permissions, firewall, gateway allowlists and gateway tool policy.
