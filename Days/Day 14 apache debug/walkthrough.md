---
title: "Fix Faulty Apache Host (Port Bind Conflict)"
date: 2026-08-12
tags: [apache, httpd, troubleshooting, centos, netstat, devops]
host: stapp01
---

# Fix Faulty Apache Host (Port Bind Conflict)

## Objective

Identify the unresponsive app host and restore the Apache (`httpd`) service,
ensuring it listens on port `5003` across all app servers.

## 1. Identify the faulty host

Build a list of hosts to probe:

```bash
printf '%s\n' host1 host2 host3 > hosts.txt
```

Check each host's HTTP responsiveness:

```bash
while read -r h; do
  printf '%s: ' "$h"
  curl -s -o /dev/null -w '%{http_code}\n' --max-time 5 "$h" || echo FAIL
done < hosts.txt
```

The failing host is **`stapp01`**. Connect to it:

```bash
ssh stapp01
```

## 2. Check the Apache service

```bash
sudo systemctl status httpd
```

The service fails to start with a socket bind error:

```text
Aug 12 13:00:35 stapp01 httpd[10335]: (98)Address already in use: AH00072: make_sock: could not bind to address ...
```

`Address already in use` means another process already holds the port Apache
wants (`5003`).

## 3. Find the conflicting process

`netstat` ships in `net-tools`, which may not be installed:

```bash
sudo dnf install net-tools
```

List listening sockets with their owning PIDs:

```bash
sudo netstat -tulpn
```

```text
tcp  0  0  127.0.0.1:5003  0.0.0.0:*  LISTEN  9631/sendmail: acce
```

`sendmail` (PID `9631`) has claimed `127.0.0.1:5003` — the port Apache needs.

## 4. Free the port and restart Apache

Kill the conflicting process:

```bash
sudo kill 9631
```

Restart `httpd`:

```bash
sudo systemctl restart httpd
```

> **Note:** If port `5003` gets re-grabbed after the restart, the offending
> process is a managed service that respawns. In that case stop and disable it
> instead of killing the PID:
>
> ```bash
> sudo systemctl stop sendmail
> sudo systemctl disable sendmail
> ```

## 5. Verify

Confirm the service is active:

```bash
sudo systemctl status httpd
```

Confirm Apache answers on port `5003`:

```bash
curl http://stapp01:5003
```

The remaining app hosts were already confirmed healthy by the responsiveness
sweep in **step 1**, so no further per-host `curl http://stappNN:5003` check is
needed.
