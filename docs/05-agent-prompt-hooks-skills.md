# Phase 5: Agent prompt, hooks and skills

## Goal

Make what the agent is told agree with what it is allowed to do, and switch off the remaining defaults that act or store data on their own.

The workspace files (`SOUL.md`, `IDENTITY.md`, `USER.md`, `AGENTS.md`) are the agent's system prompt. They are sent with every message. The defaults are written for an agent with full access. They are not a security boundary, but on a restricted agent they cause two problems: the agent keeps asking for tools it does not have, and they contain standing instructions that take effect the moment a tool is granted.

Passages in the 2026.9.9 defaults that conflict with this setup:

| File | Default instruction | Problem |
|---|---|---|
| `AGENTS.md` | "Do freely: anything the user asked for, including sending messages, emails, or posts on their behalf" | What "the user asked" is exactly what a prompt injection forges |
| `AGENTS.md` | Check email and calendar several times a day, reach out after 8 hours of silence, commit and push without asking | Unprompted activity and cost |
| `AGENTS.md`, `SOUL.md` | Update `AGENTS.md` when a lesson is learned. "This file is yours to evolve." | The agent edits its own instructions |
| `IDENTITY.md` | "Fill this in during your first conversation" | The agent asks for its name in every session |

## Options considered

| Option | For | Against |
|---|---|---|
| Rewrite the four files, short (chosen) | Prompt matches the tool policy, every line is known, about a third of the size | Upstream template changes are not picked up |
| Trim only the conflicting passages | Less work | Leaves text whose effect nobody has thought about |
| Keep the defaults | No work | The conflicts activate when tools are added |

## Steps

Run these as the `openclaw` user.

### 1. Put the defaults under version control

The workspace is already a git repository with no commits. Committing the defaults first makes every later change visible, including changes you did not make.

> [!WARNING]
> Changes the workspace's local git history.

```bash
cd ~/.openclaw/workspace
git config user.name "operator"
git config user.email "operator@localhost"
printf 'memory/\n' > .gitignore
git add .gitignore AGENTS.md IDENTITY.md SOUL.md USER.md
git commit -m "OpenClaw default workspace files"
```

Never push this repository to a public remote.

### 2. Replace the four files

> [!WARNING]
> Replaces the agent's system prompt.

Write the files from [../examples/workspace/](../examples/workspace/) over the defaults, filling in the placeholders. Then:

```bash
wc -c *.md
git diff --stat
git add -A && git commit -m "Replace defaults with a minimal prompt that matches the tool policy"
```

`USER.md` is sent to the model provider with every message. Put nothing in it that you would not send.

When a tool is granted later, update `AGENTS.md` in the same change. It states what the agent can and cannot do.

### 3. Disable hooks

In `openclaw hooks list`, `ready` means a hook's requirements are met, not that it is enabled. After onboarding, `openclaw config get hooks` showed only `session-memory` enabled. It saves an excerpt of each ended session to `memory/` in the workspace. The agent has no memory tools, so nothing reads those files.

> [!WARNING]
> Changes configuration.

```bash
openclaw hooks disable session-memory
openclaw hooks disable boot-md
openclaw hooks disable bootstrap-extra-files
```

The last two were not enabled. The explicit `false` keeps them off if a default changes. `boot-md` would run a workspace `BOOT.md` as a model call on every gateway start.

### 4. Give the agent no skills

`openclaw skills list` showed 19 of 55 bundled skills as ready. They include skills for installing other skills from ClawHub and for changing gateway configuration. None are usable without `exec`, web or browser tools.

> [!WARNING]
> Changes configuration.

```bash
openclaw config set agents.entries.<AGENT_ID>.skills '[]' --strict-json
```

The per-agent list is the agent's final skill set. An empty list means no skills. Add your own skills to it by name later.

### 5. Set the service umask

The gateway runs as a systemd user service and does not read `~/.profile`, so the `umask 077` set in phase 2 does not apply to it. Directories it created were mode 775.

> [!WARNING]
> Adds a systemd drop-in and restarts the gateway.

```bash
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d
printf '[Service]\nUMask=0077\n' > ~/.config/systemd/user/openclaw-gateway.service.d/umask.conf
systemctl --user daemon-reload
openclaw gateway restart
```

## Verify

```bash
openclaw config get hooks
openclaw config get agents.entries.<AGENT_ID>.skills
grep Umask /proc/$(systemctl --user show -p MainPID --value openclaw-gateway.service)/status
```

Expected: three hooks with `enabled: false`, skills `[]`, and `Umask: 0077`.

Behaviour, in a fresh session (`/new`):

- Ask who it is and what it can and cannot do. It gives its name, lists its limits once and does not ask for tools.
- Ask it to remember something. It says it cannot store anything and points to `USER.md`.
- Ask which skills it has. It has none.
- `ls ~/.openclaw/workspace/` shows no new `memory` directory after `/new`.

On the tested version the new workspace files took effect in the next session without a gateway restart.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
cd ~/.openclaw/workspace && git revert HEAD
openclaw hooks enable session-memory
openclaw config unset agents.entries.<AGENT_ID>.skills
rm ~/.config/systemd/user/openclaw-gateway.service.d/umask.conf && systemctl --user daemon-reload
```

## Security notes

- The prompt is not a control. It reduces how often the model attempts something that the gateway would refuse anyway.
- The agent still has message tools. The sender allowlist limits who can write to the agent, not whom the agent can write to. TODO: not verified whether outbound messages can be limited to the owner.
- The `session-memory` setting was written by onboarding although no hook question was shown. Review `openclaw config get hooks` after onboarding.

## References

- [OpenClaw: Agent workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [OpenClaw: Bundled hooks](https://docs.openclaw.ai/automation/hooks/bundled-hooks)
- [OpenClaw: Hooks CLI](https://docs.openclaw.ai/cli/hooks)
- [OpenClaw: Skills config](https://docs.openclaw.ai/tools/skills-config)
