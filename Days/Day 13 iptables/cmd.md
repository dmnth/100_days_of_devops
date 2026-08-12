---
title: "iptables-services Setup on CentOS Stream 9"
tags: [linux, firewall, iptables, centos, runbook]
created: 2026-08-12
---

# iptables-services Setup on CentOS Stream 9

Walkthrough of an `iptables` service setup session, command by command. Two lines
in the original history don't do what they appear to — those are flagged inline.

> [!NOTE]
> On CentOS Stream 9 / RHEL 8+, `firewalld` is the default firewall front-end and
> `iptables-services` is the legacy alternative. Running both at once causes
> conflicts. The usual pattern is to disable firewalld first:
>
> ```bash
> sudo systemctl disable --now firewalld
> ```

## 1. Install the package

```bash
sudo dnf install iptables
```

Installs the iptables userspace tools (`iptables` / `ip6tables`) via `dnf`.

> [!WARNING]
> This package provides the **commands** but **not** the systemd service unit that
> steps 2–4 rely on. The `iptables.service` unit (which saves/restores rules) ships
> in a separate package. This line should have been:
>
> ```bash
> sudo dnf install iptables-services
> ```
>
> which pulls in `iptables` as a dependency. Without it, the `systemctl ... iptables`
> calls below have no unit to act on.

## 2. Enable the service (BROKEN)

```bash
sudo systemctl iptables enable
```

> [!CAUTION]
> **Malformed — this fails.** `systemctl` syntax is `systemctl <verb> <unit>`; the
> verb comes first. Here the unit name sits where the verb belongs, so systemd
> rejects it with `Unknown command verb iptables`. Effectively a no-op typo. The
> intended command is step 3.

## 3. Enable the service (correct)

```bash
sudo systemctl enable iptables
```

Creates the symlinks so `iptables.service` starts automatically at boot. Does
**not** start it now — only wires it up for future boots. Requires
`iptables-services` to be installed (see step 1).

## 4. Start the service

```bash
sudo systemctl start iptables
```

Starts the service immediately for the current session. On start,
`iptables.service` restores rules from `/etc/sysconfig/iptables`.

`enable` (step 3) and `start` (step 4) are independent actions. To do both in one
shot:

```bash
sudo systemctl enable --now iptables
```

## 5. Edit the persistent rules file

```bash
sudo vi /etc/sysconfig/iptables
```

Opens the persistent rules file in `vi`. This file is in `iptables-save` format
(raw table/chain/rule syntax) and is the source of truth the service reads on
start/restart. Editing it here is how rule changes survive reboots under
`iptables-services` — as opposed to live `iptables` commands, which are lost on
flush or reboot unless saved.

## 6. Apply the edits

```bash
sudo systemctl restart iptables
```

Restarts the service: flushes the currently loaded rules and re-applies them from
`/etc/sysconfig/iptables`. This is the step that makes the edits from step 5 take
effect.

## 7. Verify the active ruleset

```bash
sudo iptables -L -n -v
```

Lists the active ruleset to confirm the restart worked.

| Flag | Meaning |
| ---- | ------- |
| `-L` | List rules in all chains of the default `filter` table. |
| `-n` | Numeric output — skip reverse-DNS on IPs and don't translate port numbers to service names. Faster, and avoids hangs on DNS lookups. |
| `-v` | Verbose — adds per-rule packet/byte counters plus the `in`/`out` interface columns. This is what you want when checking whether traffic is actually hitting a rule. |

## 8. Restart again

```bash
sudo systemctl restart iptables
```

Identical to step 6 — a second restart, typically after another edit to the rules
file, or just to re-confirm state.

## Summary

| Step | Command | Effect |
| ---- | ------- | ------ |
| 1 | `dnf install iptables` | Installs CLI tools (should be `iptables-services`) |
| 2 | `systemctl iptables enable` | **Fails** — malformed syntax |
| 3 | `systemctl enable iptables` | Enable at boot |
| 4 | `systemctl start iptables` | Start now |
| 5 | `vi /etc/sysconfig/iptables` | Edit persistent rules |
| 6 | `systemctl restart iptables` | Apply edits |
| 7 | `iptables -L -n -v` | Verify active ruleset |
| 8 | `systemctl restart iptables` | Re-apply / re-confirm |

## Key takeaways

- `iptables-services` is the real dependency for steps 2–8, not bare `iptables`.
- `systemctl` syntax is verb-first: `systemctl <verb> <unit>`.
- `enable` and `start` are independent; combine with `enable --now`.
- Rule edits belong in `/etc/sysconfig/iptables` to survive reboots.
- Disable `firewalld` before using `iptables-services` to avoid conflicts.
