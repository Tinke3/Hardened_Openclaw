# Phase 1: OS hardening

## Goal

Lock down the host before the agent exists on it. The agent will read untrusted input and hold API credentials, so the operating system is the outermost boundary.

## Options considered

**Firewall tool**

| Option | For | Against |
|---|---|---|
| nftables directly (chosen) | Native backend, one readable file, supports per-user egress rules later | Rules are written by hand |
| ufw | Quick | Hides the real ruleset. Docker bypasses it with its own chains. |
| firewalld | Zone model | Heavy for a single-interface device |

**Automatic updates**

| Option | For | Against |
|---|---|---|
| Debian security only | Least chance of unattended breakage | Kernel and firmware come from the Raspberry Pi archive and would not be patched |
| Debian plus Raspberry Pi archive (chosen) | Kernel is patched too | Kernel changes unattended. Takes effect only after a reboot. |

Reboots are left manual until the services have been seen to come back reliably.

## Steps

### 1. Record the starting state

```bash
grep PRETTY_NAME /etc/os-release; uname -rm
findmnt -no SOURCE,FSTYPE /
lsusb -t
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin) '
ss -tlnp
sudo ss -ulnp
sudo ls -l /etc/sudoers.d/
```

These commands only read. `sshd -T` prints the effective configuration after all drop-in files are merged. `lsusb -t` shows whether the SSD is on a 5000M (USB 3) bus.

### 2. SSH: keys only

Confirm that key login works before changing anything:

```bash
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no <ADMIN_USER>@<PI_HOST> 'echo ok'
ssh-keygen -lf ~/.ssh/authorized_keys
```

> [!WARNING]
> The next command disables password login. Keep your current session open and test from a second terminal.

```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
AuthenticationMethods publickey
PermitRootLogin no
AllowUsers <ADMIN_USER>
EOF

sudo sshd -t && sudo systemctl reload ssh
```

sshd uses the first occurrence of each setting, so the `00-` prefix makes this file win over other drop-ins. `AllowUsers` means the agent account created later cannot log in over SSH even if a key appears in its home directory.

### 3. Update

> [!WARNING]
> Upgrades packages and may replace the kernel.

```bash
sudo apt update
apt list --upgradable
sudo apt full-upgrade
sudo rpi-eeprom-update
```

`full-upgrade` is used because Raspberry Pi kernel and firmware packages sometimes change dependencies. `rpi-eeprom-update` without options only reports whether the bootloader is current.

### 4. Unattended upgrades

Read the archive origin from the machine instead of assuming it:

```bash
apt-cache policy | grep -E '^\s+release' | sort -u
```

> [!WARNING]
> Installs a package and enables daily automatic upgrades.

```bash
sudo apt install unattended-upgrades

sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local >/dev/null <<'EOF'
Unattended-Upgrade::Origins-Pattern {
    "origin=Raspberry Pi Foundation,codename=${distro_codename},label=Raspberry Pi Foundation";
};
Unattended-Upgrade::Automatic-Reboot "false";
EOF
```

`Origins-Pattern` is a list, so this adds the Raspberry Pi archive next to the Debian defaults.

### 5. Disable mDNS

Skip this if you reach the Pi by a `.local` name.

> [!WARNING]
> Stops and disables avahi.

```bash
sudo systemctl disable --now avahi-daemon.socket avahi-daemon.service
```

### 6. Firewall

Check which source address the Pi sees for your session. It must be inside `<MGMT_SUBNET>`. If your router translates addresses between segments, the SSH rule below would lock you out.

```bash
echo $SSH_CONNECTION
```

> [!WARNING]
> Replaces `/etc/nftables.conf`.

```bash
sudo cp -a /etc/nftables.conf /etc/nftables.conf.orig
```

Write the ruleset from [../examples/nftables.conf](../examples/nftables.conf) to `/etc/nftables.conf`, replacing the example subnet with `<MGMT_SUBNET>`. The example file also contains the `agent_out` chain added in [phase 6](06-egress-filtering.md). At this point, leave out that chain and the `jump` rule in `output`. The Home Assistant rule inside it belongs to [phase 7](07-home-assistant-mcp.md). Then check and load it with an automatic rollback:

```bash
sudo nft -c -f /etc/nftables.conf && echo "syntax ok"
sudo systemd-run --on-active=180 --unit=nft-rollback /usr/sbin/nft flush ruleset
sudo nft -f /etc/nftables.conf
```

Open a new SSH session. If it works, cancel the rollback and make the rules persistent:

```bash
sudo systemctl stop nft-rollback.timer
sudo systemctl enable nftables
```

Rules attach to hooks, not to interfaces, so the default-drop policy also covers interfaces that appear later.

## Verify

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|authenticationmethods|permitrootlogin|allowusers) '
apt-config dump | grep -i 'APT::Periodic'
sudo unattended-upgrade --dry-run --debug 2>&1 | grep -i 'allowed origins'
sudo nft list ruleset
systemctl is-enabled nftables
systemctl is-active nft-rollback.timer
```

Expected: `passwordauthentication no`, both `APT::Periodic` values `"1"`, the allowed origins include Debian, Debian-Security and Raspberry Pi Foundation, the ruleset is loaded, nftables is `enabled`, and the rollback timer is `inactive`.

From another machine, `ssh -o PubkeyAuthentication=no <ADMIN_USER>@<PI_HOST>` must fail with `Permission denied (publickey)`.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
sudo rm /etc/ssh/sshd_config.d/00-hardening.conf && sudo systemctl reload ssh
sudo rm /etc/apt/apt.conf.d/52unattended-upgrades-local
sudo cp -a /etc/nftables.conf.orig /etc/nftables.conf && sudo nft flush ruleset
sudo systemctl enable --now avahi-daemon.socket avahi-daemon.service
```

## Security notes

- Without the firewall, a later mistake such as binding the gateway to all interfaces exposes it to the network.
- Without `AllowUsers`, SSH access for the agent account depends only on the absence of a key file.
- Skipping the Raspberry Pi origin leaves the kernel unpatched until you upgrade by hand.
- fail2ban and a non-default SSH port were left out on purpose. With passwords disabled and SSH limited to one subnet they add little.

## References

- [OpenClaw: Raspberry Pi](https://docs.openclaw.ai/install/raspberry-pi)
- [Raspberry Pi documentation: Update your current Raspberry Pi software](https://www.raspberrypi.com/documentation/computers/os.html#update-software)
