---
title: "Install & Configure nginx with SSL/TLS on App Server 2"
description: "A runbook for deploying an HTTPS-only nginx web server on RHEL/CentOS, with verification and troubleshooting steps."
layout: default
tags: [nginx, ssl, tls, rhel, devops]
---

# Install & Configure nginx with SSL/TLS (stapp02)

> **Goal:** Install nginx on `stapp02`, serve content over HTTPS using the provided
> `nautilus` certificate and key, and verify the TLS endpoint end to end.

**Target host:** `stapp02` &nbsp;•&nbsp; **OS:** RHEL/CentOS 8 &nbsp;•&nbsp; **nginx:** 1.20.x

---

## Prerequisites

- SSH access to `stapp02` as a sudo-capable user.
- Certificate and key staged at `/tmp/nautilus.crt` and `/tmp/nautilus.key`.
- Ports 80/443 reachable from wherever you test.

---

## 1. Connect to the app server

```bash
ssh steve@stapp02
```

## 2. Install nginx

```bash
sudo dnf install -y nginx
```

## 3. Stage the certificate and key

Move the cert/key into a dedicated directory owned by root. The private key is read
by the nginx **master** process (running as root) at startup, so it never needs to be
readable by the unprivileged worker user — keep it locked down.

```bash
sudo mkdir -p /etc/pki/nginx
sudo cp /tmp/nautilus.crt /etc/pki/nginx/
sudo cp /tmp/nautilus.key /etc/pki/nginx/

sudo chown root:root /etc/pki/nginx/nautilus.*
sudo chmod 644 /etc/pki/nginx/nautilus.crt   # public cert
sudo chmod 600 /etc/pki/nginx/nautilus.key   # private key — root only
```

## 4. Configure the TLS server block

Edit `/etc/nginx/nginx.conf` and replace the default `server { ... }` block with the
TLS-enabled block below.

> ⚠️ **Cert paths must match step 3.** The `ssl_certificate` and `ssl_certificate_key`
> directives reference `nautilus.crt` / `nautilus.key` — the exact files copied above.
> A mismatch here makes `nginx -t` fail with `No such file or directory`.

```nginx
# Settings for a TLS enabled server.
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  _;
    root         /usr/share/nginx/html;

    ssl_certificate     "/etc/pki/nginx/nautilus.crt";
    ssl_certificate_key "/etc/pki/nginx/nautilus.key";
    ssl_session_cache   shared:SSL:1m;
    ssl_session_timeout 10m;
    ssl_ciphers         PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    error_page 404 /404.html;
    location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

> **Note on `ssl_protocols`:** it is intentionally omitted. On RHEL 8,
> `ssl_ciphers PROFILE=SYSTEM` defers to the system-wide crypto policy, which is the
> recommended pattern — no need to pin protocols manually.

## 5. (Optional) Set the landing page

```bash
echo 'Welcome to stapp02 — served over HTTPS' | sudo tee /usr/share/nginx/html/index.html
```

## 6. Validate the config *before* restarting

```bash
sudo nginx -t
```

Expected:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 7. Stream logs while you work (optional, separate terminal)

```bash
sudo tail -f /var/log/nginx/error.log
```

## 8. Restart and enable nginx

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx     # start on boot
```

## 9. Verify the service is running

```bash
sudo systemctl status nginx
```

Look for `Active: active (running)` and a master + worker process tree.

---

## Verify the TLS endpoint

A quick reachability check (the `-k` skips cert validation — fine for a self-signed
cert, but it does **not** confirm *which* certificate is served):

```bash
curl -Ik https://stapp02/
```

Expected first line: `HTTP/2 200`.

To confirm the **correct certificate** is actually being presented — subject, issuer,
and validity dates — inspect the served cert directly:

```bash
openssl s_client -connect stapp02:443 -servername stapp02 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Confirm the paths in effect match the files on disk:

```bash
sudo nginx -T | grep -i ssl_certificate
ls -l /etc/pki/nginx/
```

Check what's actually listening (spot a stray plain-HTTP :80 block):

```bash
sudo ss -tlnp | grep nginx
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `nginx -t`: `cannot load certificate ... No such file or directory` | Cert path in config ≠ file on disk | Align `ssl_certificate*` paths with `ls /etc/pki/nginx/` |
| `403 Forbidden` / `(13: Permission denied)` in error log | Worker can't traverse to docroot, or SELinux label wrong | See below |
| `curl` connection refused on 443 | nginx not running, or firewall blocking | `systemctl status nginx`; open the port (below) |
| Cert served but browser warns | Self-signed / CN mismatch | Expected for `nautilus` self-signed cert; verify with `openssl s_client` |

### Permission denied (13) on static files

The worker process (`nginx` user) needs **execute** on every directory in the path and
**read** on the file. Find the broken link in the chain:

```bash
namei -om /usr/share/nginx/html/index.html
```

### SELinux (RHEL/CentOS)

SELinux logs `errno 13` too, so correct Unix permissions aren't enough. Content must
carry the `httpd_sys_content_t` type:

```bash
getenforce
ls -Z /usr/share/nginx/html/index.html
sudo ausearch -m avc -ts recent | grep nginx     # recent denials

# Relabel a custom docroot if needed:
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/www/project(/.*)?"
sudo restorecon -Rv /srv/www/project
```

### Open the firewall (if using firewalld)

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## Result

`stapp02` serves content over HTTP/2 + TLS using the `nautilus` certificate, validated
by both a live request (`curl`) and direct certificate inspection (`openssl s_client`).
