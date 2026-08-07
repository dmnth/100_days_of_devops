---
title: "iptables: Listing Rules, a Catch-All Block, and an Exception"
description: "How to list current iptables rules, plus a breakdown of a default REJECT rule and an ACCEPT exception."
tags: [devops, iptables, firewall, linux, networking]
---

# iptables: Listing Rules, a Catch-All Block, and an Exception

A short reference covering three things: how to list the rules currently
loaded in the kernel, how to read the default catch-all rule that blocks
traffic, and how to add an exception that allows a specific port through.

## Listing the current rules

The core command lists every rule in the default `filter` table:

```bash
sudo iptables -L -n -v
```

| Flag | Meaning |
| --- | --- |
| `-L` | List the rules in the chain(s). |
| `-n` | Numeric output — skip DNS/port-name lookups (faster, no hangs). |
| `-v` | Verbose — show packet/byte counters and in/out interfaces. |

Useful variations:

```bash
sudo iptables -L -n -v --line-numbers   # add an index to each rule
sudo iptables -S                         # dump rules as the commands that created them
sudo iptables -t nat -L -n -v            # inspect the NAT table instead of filter
```

`iptables -S` is often the easiest to read because it prints each rule as the
exact spec used to create it.

> **Note:** on systems using nftables as the backend, an empty or sparse
> `iptables` listing is expected. Check the native ruleset instead with
> `sudo nft list ruleset`.

## The rule that blocks all traffic

A typical last line of an `INPUT` chain looks like this:

```text
6   360 REJECT   all  --  *  *  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
```

Reading it field by field:

| Field | Value | Meaning |
| --- | --- | --- |
| pkts | `6` | Packets matched since counters were last reset. |
| bytes | `360` | Total bytes of those matched packets. |
| target | `REJECT` | Reject matching packets (vs. `DROP`, which is silent). |
| prot | `all` | Matches every protocol (tcp, udp, icmp, ...). |
| opt | `--` | No option flags (e.g. not fragment-only). |
| in | `*` | Any input interface. |
| out | `*` | Any output interface. |
| source | `0.0.0.0/0` | Any source address. |
| destination | `0.0.0.0/0` | Any destination address. |
| — | `reject-with icmp-host-prohibited` | Reply with an ICMP "host prohibited" error. |

**In plain terms:** anything from anywhere to anywhere is rejected, and the
sender receives an ICMP host-prohibited message instead of a silent drop.

This is almost always the **final catch-all** in the chain. Traffic you want
to permit is accepted by earlier rules; whatever reaches this line matched none
of them and gets bounced. The `icmp-host-prohibited` reply is the conventional
default on RHEL/CentOS/Fedora — politer than a silent drop, since the sender
gets an immediate "no" rather than waiting for a timeout.

## Adding an exception

Because the `REJECT` is a catch-all at the **bottom** of the chain, an allow
rule has to be **inserted above it**. Appending with `-A` would place the new
rule *after* the REJECT, where it can never be reached.

```bash
# Insert at the top of INPUT (position 1), safely before the REJECT
sudo iptables -I INPUT -p tcp --dport 8082 -j ACCEPT
```

| Part | Meaning |
| --- | --- |
| `-I INPUT` | Insert into the `INPUT` chain at position 1. |
| `-p tcp` | Match the TCP protocol (HTTP runs over TCP). |
| `--dport 8082` | Match packets destined for port 8082. |
| `-j ACCEPT` | Allow the packet. |

Verify the new rule sits above the REJECT:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

The `ACCEPT ... tcp dpt:8082` rule should show a **lower** line number than the
`REJECT` line.

### Two things to remember

**Persistence.** Live changes are lost on reboot unless saved.

- CentOS/RHEL 7 with the iptables service:
  `sudo service iptables save` (writes to `/etc/sysconfig/iptables`).
- On NixOS, do not edit rules by hand — they are regenerated on rebuild. Open
  the port declaratively instead:

  ```nix
  networking.firewall.allowedTCPPorts = [ 8082 ];
  ```

  Then apply with `nixos-rebuild switch`.

**The firewall only permits traffic — the service must be listening.** Confirm
the app is bound to `0.0.0.0:8082` (all interfaces), not just
`127.0.0.1:8082`, or remote requests will not arrive even with the rule open:

```bash
sudo ss -tlnp | grep 8082
```
