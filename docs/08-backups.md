# Phase 8: Encrypted backups

## Goal

Keep a copy of the agent's state off the Pi, in a form that neither the Pi nor the storage provider can read.

`openclaw backup create` archives the state directory: configuration, databases, credentials and the workspace. On the tested system the archive was 2 MB and held 117 entries, including `openclaw.json` with its backups and the Telegram token file. It is not encrypted, so it is as sensitive as the installation itself.

System configuration under `/etc` is not in the archive. This guide is its backup: the host is rebuilt from the earlier phases, and only the agent's state is restored.

## Options considered

**Where the copy lives**

| Option | Protects against | Does not protect against |
|---|---|---|
| On the Pi only | A broken configuration or a failed update | Disk or hardware failure |
| Also in a synced cloud folder, encrypted (chosen) | Hardware failure as well | Loss of the decryption key |
| OpenClaw's own offsite location (`--to`) | Hardware failure, automatically | Not evaluated. Needs storage credentials on the Pi and another egress destination. |

**How it is encrypted**

| Option | For | Against |
|---|---|---|
| `age` with a key pair (chosen) | The Pi holds only the public key, so it can create backups and cannot open them | Less widely known than GPG |
| GPG | Ubiquitous | Keyrings and trust settings that are easy to get wrong |
| A password-based archive | No new tool | The password would have to be stored on the Pi |

The private key never touches the Pi. If the Pi is compromised later, earlier backups stay closed. The same holds if the agent is one day given read access to the cloud folder.

Scheduling was left out. The backup is run by hand, and always before an update.

## Steps

### 1. Create the key pair on your own computer

```bash
age-keygen -o ~/openclaw-backup-key.txt
```

The command prints the public key (`age1...`). Store the contents of the file in a password manager and delete the file. Without it the backups cannot be opened. Do not put it in the cloud folder.

### 2. Prepare the Pi

> [!WARNING]
> Installs a package and creates a directory.

As `<ADMIN_USER>`:

```bash
sudo apt install age
sudo install -d -o openclaw -g <ADMIN_USER> -m 2750 /srv/openclaw-backup-out
```

The directory is a hand-over point. The agent account writes ciphertext into it and the administrator's group can read it. The agent account's home directory stays closed, and collecting a backup needs no sudo.

### 3. Create and encrypt a backup

> [!WARNING]
> Writes an archive containing secrets to the agent account's home directory.

As `openclaw`:

```bash
mkdir -p ~/backups && chmod 700 ~/backups
cd ~
openclaw backup create --output ~/backups --verify

cd ~/backups
A=$(ls -t *.tar.gz | head -1)
age -r <AGE_PUBLIC_KEY> -o "/srv/openclaw-backup-out/$A.age" "$A"
chmod 640 "/srv/openclaw-backup-out/$A.age"
sha256sum "$A"
```

The output directory must be outside the state directory. The gateway does not need to be stopped.

### 4. Collect the ciphertext

On your own computer:

```bash
scp "<PI_HOST>:/srv/openclaw-backup-out/*.age" <SYNCED_FOLDER>/
```

## Verify

**The copy opens.** Decrypt into a temporary location outside the synced folder, compare the hash with the one printed on the Pi, then delete the plaintext and the key file:

```bash
age -d -i <KEY_FILE> -o /tmp/restore-test.tar.gz <SYNCED_FOLDER>/<ARCHIVE>.age
shasum -a 256 /tmp/restore-test.tar.gz
rm <KEY_FILE> /tmp/restore-test.tar.gz
```

**The archive restores.** On the Pi, as `openclaw`:

```bash
cd ~/backups
A=$(ls -t *.tar.gz | head -1)
openclaw backup restore "$A" --target ~/restore-test
R=$(find ~/restore-test -name openclaw.json | head -1)
cmp "$R" ~/.openclaw/openclaw.json && echo "config identical"
rm -rf ~/restore-test
```

`restore` extracts into an empty directory and never writes over the live state. Both checks passed on the tested environment.

TODO: not verified. Activating a restored state on a rebuilt host has not been tried.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
sudo rm -r /srv/openclaw-backup-out
sudo apt remove age
rm -r /home/openclaw/backups
```

## Security notes

- The plaintext archives in `~/backups` hold the same secrets as the live state on the same disk. Delete old ones.
- `age` encrypts and does not sign. Anyone with the public key can produce a file for it. The archive's integrity is checked by `openclaw backup verify`.
- Losing the private key makes every backup unreadable.
- A backup that has never been restored is an assumption. Repeat the checks after an OpenClaw upgrade.

## References

- [OpenClaw: Backup CLI](https://docs.openclaw.ai/cli/backup)
- [age](https://age-encryption.org/)
