# Phase 2: Agent user and installation

## Goal

Create an unprivileged account for the agent and install OpenClaw under it without running third-party code as root.

OpenClaw installs the Gateway as a systemd user service of whichever account runs the installer, and stores state and credentials in that account's `~/.openclaw/`. The agent is therefore exactly as privileged as that account.

## Options considered

**Which account**

| Option | For | Against |
|---|---|---|
| Your own account | Simplest, matches the official Pi guide | The agent can read your SSH keys and attempt sudo |
| Dedicated account without sudo (chosen) | Kernel-enforced boundary, enables per-user firewall rules | Maintenance means switching users |
| Container | Also isolates filesystem and processes | More moving parts, interacts with the firewall |

**Which installer**

| | `install.sh` | `install-cli.sh` (chosen) |
|---|---|---|
| Node.js | NodeSource apt repository, installed as root | Pinned tarball, SHA-256 verified |
| Root needed | Yes, and a third-party apt source remains | No |
| Location | Node system-wide | Everything under `~/.openclaw` |
| Node updates | Through apt | Not through apt. TODO: not verified how Node is updated in this layout. |

With either installer the OpenClaw code itself is writable by the agent account. Tool policy in phase 3 is what keeps the model away from it.

## Steps

### 1. Create the account

> [!WARNING]
> Creates a user and enables lingering for it.

```bash
sudo adduser --disabled-password --comment "" openclaw
sudo loginctl enable-linger openclaw
```

The account has no password, so the only way in is `sudo -iu openclaw` from the administrative account. Lingering makes systemd start the user's services at boot and keep them running without a login session.

### 2. Prerequisites

> [!WARNING]
> Installs packages.

```bash
sudo apt install git curl ca-certificates
```

The installer would try to install Git with apt, which the agent account cannot do. `build-essential` was not needed.

### 3. Download and read the installer

```bash
sudo -iu openclaw
curl -fsSL --proto '=https' --tlsv1.2 -o install-cli.sh https://openclaw.ai/install-cli.sh
sha256sum install-cli.sh
less install-cli.sh
```

The script is saved instead of piped into a shell so that it can be read and its hash recorded. The hash on the test date was:

```
ff843ff6c1cb4bdb672710b6a4ca091aad6d6468183aca9b1bba080760ab9e31
```

No official reference hash was found, so this identifies the script version used here and does not prove authenticity. Your hash will differ if the script has changed.

### 4. Install

> [!WARNING]
> Writes to the agent account's home directory.

```bash
bash install-cli.sh --version 2026.9.9 --no-onboard
```

### 5. Shell environment and permissions

The installer left its directories group-writable.

> [!WARNING]
> Changes permissions and the account's `~/.profile`.

```bash
chmod 700 ~/.openclaw

cat >> ~/.profile <<'EOF'

umask 077
export PATH="$HOME/.openclaw/bin:$PATH"
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
EOF
exit
```

`sudo -iu` does not create a full login session. Without `XDG_RUNTIME_DIR`, `systemctl --user` cannot find the user's systemd instance.

## Verify

As `<ADMIN_USER>`:

```bash
id openclaw
sudo passwd -S openclaw
stat -c '%a %U %n' /home/<ADMIN_USER> /home/openclaw
loginctl show-user openclaw -p Linger
sudo -iu openclaw sudo -n true
```

Expected: no privileged groups, password status `L`, both home directories `700`, `Linger=yes`, and the last command fails with `sudo: a password is required`.

As `openclaw` (after `sudo -iu openclaw`):

```bash
umask
command -v openclaw
openclaw --version
systemctl --user is-system-running
```

Expected: `0077`, a path under `~/.openclaw/bin`, `OpenClaw 2026.9.9`, and `running`.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
sudo loginctl disable-linger openclaw
sudo deluser --remove-home openclaw
```

## Security notes

- Running the agent under your own account puts a prompt injection one sudo password away from root.
- Do not run `openclaw onboard` yet. Onboarding stores credentials and sets security-relevant defaults. It is handled in phase 3.

## References

- [OpenClaw: Installer internals](https://docs.openclaw.ai/install/installer)
- [OpenClaw: Install](https://docs.openclaw.ai/install)
- [OpenClaw: Raspberry Pi](https://docs.openclaw.ai/install/raspberry-pi)
