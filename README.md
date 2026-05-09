# Crowdsec scenario — `Guezli/mailcow-f2b-feed` (Mailcow / netfilter)

> Propagate **Mailcow's internal F2B bans** into the local Crowdsec LAPI,
> so the host-side nftables-bouncer (and any other Crowdsec consumers)
> see and act on bans that Mailcow detects but Crowdsec wouldn't.
> Specifically built for and tested with **[Mailcow](https://mailcow.email/)**.

## Why this exists

Mailcow ships its own Fail2Ban-like component, `netfilter-mailcow`, which subscribes to a Redis channel `F2B_CHANNEL` populated by SOGo, dovecot, postfix, the Mailcow UI, and other internal services. It implements iptables/nftables-level bans inside Mailcow's Docker network.

The catch: those bans live **only** in Mailcow's container. They never reach Crowdsec, so:

- The **host-side nftables-bouncer** (acting on Crowdsec decisions) doesn't know about them.
- They aren't **logged as Crowdsec alerts**, so cross-instance sharing or unified alerting (CAPI, Gotify, Loki) misses them entirely.
- A SOGo-Webmail bruteforce (which Crowdsec out of the box doesn't parse) gets banned by Mailcow, but the bruteforcer can keep hitting other services on the same host that Crowdsec *does* protect.

This scenario closes that gap by tailing the `netfilter-mailcow` container's stdout, extracting each finalized ban (`CRIT: Banning <IP>/32 for <N> minutes`), and feeding it into Crowdsec as a trigger-style decision.

## What it does

```yaml
type: trigger
groupby: evt.Meta.source_ip
blackhole: 1h
```

→ Every line `CRIT: Banning <IP>/32 for <N> minutes` from `netfilter-mailcow` produces one Crowdsec ban for that IP. The 1-hour `blackhole` prevents re-triggering when Mailcow renews a ban for the same IP shortly after.

The matching parser (`Guezli/mailcow-f2b-bans`, included in this repo) extracts the source IP and the requested ban duration. The trigger scenario then emits a decision the local LAPI registers as a normal Crowdsec ban.

**Bans stay LOCAL.** This scenario does not push to CAPI or any remote LAPI. If you want cross-host sharing, configure that separately via Crowdsec's standard CAPI/multi-LAPI mechanisms.

## Installation

This repo is **not in the official Crowdsec Hub**, so `cscli scenarios install` won't find it. Use the bundled installer instead.

### One command, done

```bash
curl -fsSL https://raw.githubusercontent.com/Guezli/crowdsec-mailcow-f2b-feed/main/install.sh | sudo bash
```

That's it. The script handles everything end-to-end and exits when it's done — there's nothing else to configure. Re-run anytime to upgrade (idempotent).

### How [`install.sh`](install.sh) works

It's a ~120-line bash script that performs six steps and exits.

| Step | What it does | If already done |
|---|---|---|
| 1 | Checks `cscli` is on PATH | aborts if Crowdsec isn't installed |
| 2 | Auto-detects the `netfilter-mailcow` container by name pattern | aborts if no Mailcow container found |
| 3 | Writes acquisition `/etc/crowdsec/acquis.d/mailcow-f2b-feed.yaml` (docker source, `type: mailcow-f2b`) | skipped if file already exists |
| 4 | Downloads `parsers/s01-parse/mailcow-f2b-bans.yaml` to `/etc/crowdsec/parsers/s01-parse/` | skipped if byte-identical |
| 5 | Downloads `scenarios/mailcow-f2b-feed.yaml` to `/etc/crowdsec/scenarios/` | skipped if byte-identical |
| 6 | `systemctl reload crowdsec` and verifies parser + scenario are loaded | warns if Crowdsec isn't active |

The script makes **no other changes** — no collections touched, no profiles, no bouncers, no other acquisitions.

### Verify it worked

```bash
sudo cscli scenarios list | grep mailcow-f2b-feed
sudo cscli parsers list | grep mailcow-f2b-bans
sudo cscli alerts list --scenario Guezli/mailcow-f2b-feed --since 24h
```

### Manual install (if you'd rather inspect every change)

```bash
# 1. Acquisition (replace container name if non-default)
sudo tee /etc/crowdsec/acquis.d/mailcow-f2b-feed.yaml >/dev/null <<'EOF'
source: docker
container_name:
  - mailcowdockerized-netfilter-mailcow-1
labels:
  type: mailcow-f2b
EOF

# 2. Parser
sudo curl -fsSL -o /etc/crowdsec/parsers/s01-parse/mailcow-f2b-bans.yaml \
  https://raw.githubusercontent.com/Guezli/crowdsec-mailcow-f2b-feed/main/parsers/s01-parse/mailcow-f2b-bans.yaml

# 3. Scenario
sudo curl -fsSL -o /etc/crowdsec/scenarios/mailcow-f2b-feed.yaml \
  https://raw.githubusercontent.com/Guezli/crowdsec-mailcow-f2b-feed/main/scenarios/mailcow-f2b-feed.yaml

# 4. Reload
sudo systemctl reload crowdsec
```

## Architecture

```
┌──────────────────────────────────────────────┐
│ Mailcow stack (Docker network)               │
│                                              │
│  SOGo Web ──┐                                │
│  Dovecot ───┤                                │
│  Postfix ───┼─→ Redis F2B_CHANNEL            │
│  Mailcow UI ┘            │                   │
│                          ▼                   │
│      netfilter-mailcow container             │
│      stdout: CRIT: Banning 1.2.3.4/32 ...    │
└──────────────────────────┼───────────────────┘
                           │ docker logs
                           ▼
   Crowdsec docker source (acquis.d/mailcow-f2b-feed.yaml)
                           │ type: mailcow-f2b
                           ▼
   crowdsecurity/non-syslog (s00-raw, hub default)
                           │ program=mailcow-f2b, message=raw line
                           ▼
   Guezli/mailcow-f2b-bans (s01-parse, this repo)
                           │ grok → meta.source_ip + meta.log_type
                           ▼
   Guezli/mailcow-f2b-feed (scenario, type: trigger)
                           │ blackhole: 1h
                           ▼
                    local Crowdsec LAPI
                           │ decision
                           ▼
                  nftables-bouncer (host-side)
```

## Requirements

| Component | Source |
|---|---|
| Crowdsec agent | `cscli` must be on PATH |
| Local LAPI | default at `127.0.0.1:8080` |
| Docker | `docker ps` must be readable as root |
| Mailcow stack running | `netfilter-mailcow` container must exist and be running |

## Mailcow-specific notes

- **Crowdsec runs on the Mailcow host**, not inside a Mailcow container.
- **Both detection systems run in parallel.** Mailcow's F2B bans an IP at the Mailcow Docker network edge. The Crowdsec-feed adds a second nftables ban at the host edge — defense in depth, no conflict.
- After Mailcow updates the `netfilter-mailcow` container is rebuilt; the default name (`mailcowdockerized-netfilter-mailcow-1`) survives that. If you renamed the Mailcow compose project, edit `acquis.d/mailcow-f2b-feed.yaml` accordingly.
- Mailcow's allowlist (own IPs, internal subnets) lives independently in Mailcow config — Crowdsec doesn't see it. Whitelist critical IPs in **both** systems.

## Tuning

Defaults are deliberately conservative:

- `blackhole: 1h` — prevents re-trigger if Mailcow renews a ban for the same IP within the hour. The Crowdsec ban duration itself comes from Crowdsec's profile/decision-defaults (typically 4h).

If you want shorter Crowdsec-bans (so the host-side nftables ban expires before Mailcow's):

- adjust your Crowdsec **profile** (`/etc/crowdsec/profiles.yaml`) decision-duration for this scenario, *not* this scenario file

## False-positive considerations

This scenario is **passive** — it only fires on lines Mailcow has already classified as ban-worthy. False positives here mirror Mailcow's own F2B false positives:

- Mailcow F2B is generally well-tuned out of the box; if you trust its bans, you can trust the propagated ones.
- If Mailcow F2B has historically false-banned legitimate clients (e.g. internal mail relays, monitoring probes), they'll get the same treatment in Crowdsec. **Whitelist them in Crowdsec** via `crowdsecurity/whitelists` or a custom whitelist.

## Pairs well with

- **[`Guezli/postfix-sasl-bf`](https://github.com/Guezli/postfix-sasl-bf)** — slow-distributed SASL-LOGIN bruteforce on postfix (host-level Crowdsec).
- **[`Guezli/postfix-honeypot-users`](https://github.com/Guezli/postfix-honeypot-users)** — instant-bans IPs hitting honeypot SASL usernames (`postmaster@`, `admin@`, …).
- **[`melite/dovecot-slow-bf`](https://hub.crowdsec.net/author/melite/scenarios/dovecot-slow-bf)** — slow-distributed IMAP-bruteforce on dovecot.

Together with this feed, you get host-level Crowdsec coverage on the postfix/dovecot/SASL layers AND Mailcow's broader detection set (SOGo Web UI, Mailcow admin UI, rspamd UI) propagated as Crowdsec decisions.

## License

MIT — see [LICENSE](LICENSE).
