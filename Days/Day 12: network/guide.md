---
title: "Runbook: Web Server on stapp01:8082 Unreachable"
description: "Diagnosing a failed httpd service and a firewall rule blocking port 8082."
tags: [devops, troubleshooting, httpd, iptables, firewall, linux]
---

# Runbook: Web Server on `stapp01:8082` Unreachable

A step-by-step walkthrough for diagnosing why a web server was unreachable —
first a stopped `httpd` service, then a catch-all firewall rule blocking the
port.

## 1. Confirm the failure

Reproduce the problem against the target server:

```bash
curl http://stapp01:8082
```

The server fails to respond, confirming there is an issue to chase down.

## 2. Install the networking tools

`netstat` ships in the `net-tools` package (there is no package named
`netstat` itself):

```bash
sudo dnf install net-tools
```

## 3. Check what is (or isn't) listening

Look at active listeners and the processes behind them:

```bash
sudo netstat -tulpn
```

| Flag | Meaning |
| --- | --- |
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-p` | Show the owning process/PID |
| `-n` | Numeric addresses and ports |

Use this to see whether `httpd` is bound to `8082`, or whether another process
has taken the port.

## 4. Restart httpd

Restart the service (clearing any conflicting process first if one held the
port):

```bash
sudo systemctl restart httpd
```

## 5. Verify httpd is running

```bash
sudo systemctl status httpd
```

Confirm the unit shows `active (running)` and is listening on `8082`.

## 6. Re-test from the remote host

From the client machine, the web server is **still** unreachable — so the
service is up locally, but something on the network path is blocking it. That
points at the firewall.

## 7. Inspect the firewall and find the blocker

Back on `stapp01`, list the rules and spot the catch-all reject:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

```text
6   360 REJECT   all  --  *  *  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
```

This rule rejects **all** traffic from any source to any destination, replying
with an ICMP host-prohibited error. Sitting at the end of the chain, it bounces
anything that earlier rules did not explicitly allow — including our port 8082
traffic.

## 8. Add an exception for port 8082

Because the `REJECT` is a catch-all at the bottom, the allow rule must be
**inserted above it** with `-I`. Appending (`-A`) would place it after the
REJECT, where it would never be reached:

```bash
sudo iptables -I INPUT -p tcp --dport 8082 -j ACCEPT
```

## 9. Confirm the rule landed above REJECT

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

The `ACCEPT ... tcp dpt:8082` rule should show a **lower** line number than the
`REJECT` line.

## 10. Save the change so it survives a reboot

`iptables-save` prints to stdout — redirect it to the restore file (or use the
service's save verb). On CentOS/RHEL 7:

```bash
sudo iptables-save | sudo tee /etc/sysconfig/iptables >/dev/null
# or, equivalently:
sudo service iptables save
```

## 11. Verify from the remote host

Re-run the original test from the client; the web server is now reachable:

```bash
curl http://stapp01:8082
```

A successful response confirms both fixes — the service is running and the
firewall permits the traffic.
