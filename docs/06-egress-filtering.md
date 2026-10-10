# Phase 6: Egress filtering for the agent user

## Goal

Limit where processes of the `openclaw` user can connect. If the gateway is ever compromised, this decides what the attacker can reach from it.

After phase 1 the firewall filters inbound traffic only. The agent account can open connections to the internet and to every device on the local networks. The gateway needs three things: DNS, HTTPS to the model provider and HTTPS to Telegram.

nftables can match the user that sent a packet (`meta skuid`), so the limits apply to the agent account only. Your own account, system updates and NTP are unaffected.

## Options considered

| Option | For | Against |
|---|---|---|
| Ports plus a block on private ranges (chosen) | Simple and stable. HTTPS to the public internet and DNS, nothing else. No access to local devices. | Any HTTPS server on the internet is still reachable, so data can still be sent out |
| Destination allowlist through a proxy | Only named hosts are reachable | Another service to run. Not evaluated yet. |
| IP address allowlist | No proxy | Cloud provider addresses change constantly |

## Steps

### 1. Find the resolver and the gateway's connections

```bash
cat /etc/resolv.conf
systemctl is-active systemd-resolved
id -u openclaw
sudo ss -tnpe | grep "uid:$(id -u openclaw)"
```

If the resolver is a local stub (`127.0.0.53`), DNS goes over loopback and needs no rule. On the tested system NetworkManager pointed `resolv.conf` directly at the router, so port 53 is allowed to that one address.

### 2. Write the ruleset

> [!WARNING]
> Replaces `/etc/nftables.conf`.

```bash
sudo cp -a /etc/nftables.conf /etc/nftables.conf.bak
```

Write [../examples/nftables.conf](../examples/nftables.conf) to `/etc/nftables.conf`. Replace the management subnet, the uid and the resolver address with your own.

- The uid is numeric on purpose. A user name is resolved when the ruleset loads. If the account is removed, loading fails at boot and the host comes up with no firewall.
- `reject` is used instead of `drop`, so a blocked connection fails at once and shows up as an error.
- A destination on a local network, such as a home automation server, gets its own `accept` rule above the private-range rule.

```bash
sudo nft -c -f /etc/nftables.conf && echo "syntax ok"
```

### 3. Apply with an automatic rollback

```bash
sudo systemd-run --on-active=180 --unit=nft-rollback /usr/sbin/nft -f /etc/nftables.conf.bak
sudo nft -f /etc/nftables.conf
```

Open a new SSH session. If it works:

```bash
sudo systemctl stop nft-rollback.timer
```

## Verify

As `<ADMIN_USER>`:

```bash
curl -sS -m 5 -o /dev/null -w 'admin http: %{http_code}\n' http://example.com
sudo -iu openclaw curl -sS -m 5 -o /dev/null -w 'agent https: %{http_code}\n' https://api.telegram.org
sudo -iu openclaw curl -sS -m 5 -o /dev/null http://example.com; echo "agent http, exit $?"
sudo -iu openclaw curl -sS -m 5 -o /dev/null http://<DNS_RESOLVER>; echo "agent router, exit $?"
sudo nft list chain inet filter agent_out
```

Expected: `admin http: 200`, `agent https: 302`, both remaining commands fail immediately with exit code 7, and the `agent to private` and `agent other` counters have increased.

Then send the bot a message. A reply shows that Telegram and the model provider are still reachable.

## Rollback

> [!NOTE]
> Not tested. These commands were not run on the tested environment.

```bash
sudo cp -a /etc/nftables.conf.bak /etc/nftables.conf && sudo nft -f /etc/nftables.conf
```

## Security notes

- This does not stop data leaving over HTTPS to an arbitrary host, or through DNS queries. A destination allowlist would.
- If the resolver address changes, the agent stops working until the rule is updated.
- The agent account cannot use git over SSH or plain HTTP. TODO: not verified that `openclaw update` works under these rules.
- IPv6 is rejected for the agent account. The tested host had link-local addresses only.

## References

- [nftables wiki: Matching packet metainformation](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation)
