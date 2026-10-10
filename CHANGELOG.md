# Changelog

## 2026-10-10

- Initial version.
- Added the threat model and phases 1 to 4: OS hardening, agent user and installation, onboarding and tool policy, Telegram channel.
- Added troubleshooting entries for the onboarding hang, Code Mode being active by default, and stale tool listings.
- Added example nftables ruleset and the resulting agent policy.
- Added MIT license and a gitleaks pre-commit hook.
- Marked the rollback sections as not tested and pointed the Raspberry Pi update reference at the exact section.
- Added phase 5 (agent prompt, hooks and skills) with example workspace files, and phase 6 (egress filtering for the agent user).
- Extended the example nftables ruleset with the agent egress chain.
- Added phase 7 (Home Assistant over MCP): outbound message guard, non-admin user, one firewall path, tool filter.
- Tested with OpenClaw 2026.9.9 on Raspberry Pi OS Lite (Debian 13.7) and Home Assistant Core 2026.10.0.
