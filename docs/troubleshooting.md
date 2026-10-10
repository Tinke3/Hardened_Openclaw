# Troubleshooting

Problems that occurred on the tested environment (OpenClaw 2026.9.9, Debian 13.7, Raspberry Pi 5).

## Onboarding hangs at "Preparing Gateway service…"

**Symptom.** `openclaw onboard --classic --flow manual` showed a spinner at "Preparing Gateway service…" and did not continue. It was left for more than two hours.

**What was observed.** From a second session:

```bash
ps -u openclaw -o pid,ppid,etime,stat,args --forest
sudo ss -tnp | grep -E 'pid=(<PID1>|<PID2>)'
sudo ls -la /home/openclaw/.config/systemd/user/
```

The wizard process was sleeping, had no child processes and no open network connections. The service unit had not been written. `openclaw.json` had already been saved.

**Fix.** Press Enter in the wizard's terminal. A prompt was hidden behind the spinner, and the wizard continued and finished normally.

TODO: not verified which prompt was hidden or which default Enter accepted. Run `openclaw security audit` afterwards and review the result.

## The model sees an `exec` tool although `group:runtime` is denied

**Symptom.** After applying the tool deny list, the agent listed `exec`, `wait` and `sessions_yield` as its direct tools and described `exec` as running JavaScript.

**Cause.** Code Mode. With `tools.codeMode` unset the gateway behaves as `"auto"` and engages Code Mode for current Claude models. Its `exec` is separate from the shell `exec` in `group:runtime`. The default executor is Node's `node:vm`, which the OpenClaw documentation describes as not a security boundary.

The tool policy page states that Code Mode is disabled by default. The Code Mode configuration page states that an absent setting means `"auto"`. Observed behaviour on 2026.9.9 matched the second.

**Fix.**

```bash
openclaw config set tools.codeMode false --strict-json
openclaw gateway restart
```

## The agent still lists tools that were just denied

**Symptom.** After adding `intent` and `subagents` to `tools.deny`, the agent's next answer still listed both.

**What resolved it.** Disabling Tool Search and starting a fresh session:

```bash
openclaw config set tools.toolSearch false --strict-json
openclaw gateway restart
```

Then `/new` in the chat and the same question. Both tools were gone and a call to `intent` was not possible.

TODO: not verified whether the earlier answer came from conversation history or from a stale search catalog.

## Installer leaves group-writable directories

**Symptom.** After `install-cli.sh`, `~/.openclaw` and its subdirectories were mode 775.

**Fix.** `chmod 700 ~/.openclaw` and `umask 077` in the agent account's `~/.profile`. Files created later by onboarding were 600 and 700.

## `systemctl --user` fails after `sudo -iu openclaw`

**Cause.** `sudo -i` does not create a login session, so `XDG_RUNTIME_DIR` is unset.

**Fix.** Enable lingering for the account and export `XDG_RUNTIME_DIR="/run/user/$(id -u)"` in its `~/.profile`. See [02-agent-user-and-install.md](02-agent-user-and-install.md).
