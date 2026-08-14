---
title: "Block a Port Except From the LBR Host (iptables-services)"
tags: [linux, firewall, iptables, centos, runbook]
created: 2026-08-12
---

# Block a Port Except From the LBR Host

Restrict inbound TCP on a service port so that **only the load-balancer host
(`stlb01`) is allowed**, and everyone else is rejected — persisted across reboots
via `iptables-services`.


## 1. Install iptables-services on each app host

```bash
sudo dnf install -y iptables-services
```

> [!NOTE]
> The bare `iptables` package provides only the CLI. The **`iptables.service`**
> systemd unit — which saves and restores rules from `/etc/sysconfig/iptables` —
> ships in **`iptables-services`**. That package pulls in `iptables` as a
> dependency.

## 2. Enable and start the service

```bash
sudo systemctl enable iptables
sudo systemctl start iptables
```

`enable` wires the unit for boot; `start` brings it up now. Combine with
`sudo systemctl enable --now iptables` if preferred.

## 3. Edit the persistent rules file

```bash
sudo vim /etc/sysconfig/iptables
```

This file is in `iptables-save` format. Add the two rules **inside** the
`*filter` block, **before** `COMMIT`, with `ACCEPT` above `REJECT`:

```text
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -p tcp --dport 6300 -s stlb01 -j ACCEPT
-A INPUT -p tcp --dport 6300 -j REJECT
COMMIT
```

> [!IMPORTANT]
> **Order is the whole task.** iptables evaluates an INPUT chain top-to-bottom and
> stops at the first match:
>
> - `ACCEPT` from `stlb01` first → LBR traffic is allowed and stops matching.
> - `REJECT` second → all other sources fall through and are rejected.
>
> Reverse them and the `REJECT` matches everyone first — including `stlb01` — so
> the load balancer gets blocked too. If the file already contains a generic
> catch-all `-A INPUT ... -j REJECT`, both custom lines must sit **above** it.

> [!NOTE]
> `-s stlb01` is resolved to an IP address at rule-load time. If the name has
> multiple addresses or DNS changes, the rule can behave unexpectedly. Using the
> host's IP directly is more robust; the hostname is fine for the lab as long as it
> resolves (via `/etc/hosts` or DNS).

## 4. Apply the rules

```bash
sudo systemctl restart iptables
```

Restart flushes the live ruleset and reloads it from `/etc/sysconfig/iptables`.

> [!NOTE]
> A manual `sudo iptables -F` before editing is **redundant** here — the restart
> replaces the live ruleset from the file regardless. It's harmless, just
> unnecessary.

## 5. Verify

```bash
sudo iptables -L -n -v
```

| Flag | Meaning |
| ---- | ------- |
| `-L` | List all chains in the default `filter` table. |
| `-n` | Numeric output — no reverse-DNS on IPs, no port-name translation. Faster, avoids DNS hangs. |
| `-v` | Verbose — per-rule packet/byte counters and the `in`/`out` interface columns. |

You should see the `stlb01` ACCEPT listed **above** the REJECT for port 6300.

## 6. Persistence across reboot

No extra step needed. Because the rules live in `/etc/sysconfig/iptables` and
`iptables.service` is **enabled** (step 2), they are restored automatically on
every boot.

## Summary

| Step | Command | Purpose |
| ---- | ------- | ------- |
| 1 | `dnf install -y iptables-services` | Install tools **and** the service unit |
| 2 | `systemctl enable iptables` + `start` | Enable at boot, start now |
| 3 | `vim /etc/sysconfig/iptables` | Add ACCEPT (LBR) then REJECT, before `COMMIT` |
| 4 | `systemctl restart iptables` | Load the new ruleset |
| 5 | `iptables -L -n -v` | Confirm order and counters |
| 6 | *(none)* | Enabled service + config file = survives reboot |

## Key takeaways

- Correct file path is `/etc/sysconfig/iptables`.
- First-match-wins: LBR `ACCEPT` must precede the `REJECT`.
- Persistence is free once the config file holds the rules and the service is enabled.
- `iptables-services` is the package that provides the service unit.
- Reconcile the port number (6300 vs 6100) against the actual task requirement.
