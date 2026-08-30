---
title: "Configure nginx as a Load Balancer for 3 Web Servers"
description: "A runbook for setting up nginx on the LBR host to split incoming HTTP traffic across stapp01, stapp02, and stapp03."
layout: default
tags: [nginx, load-balancer, reverse-proxy, rhel, devops]
---

# Configure nginx as a Load Balancer (stlb01)

> **Goal:** Configure nginx on the load-balancer host `stlb01` to forward and split
> incoming HTTP traffic across three backend web servers — `stapp01`, `stapp02`,
> and `stapp03` — each running `httpd` on port `6000`.

**LBR host:** `stlb01` &nbsp;•&nbsp; **Backends:** `stapp01`, `stapp02`, `stapp03` (httpd :6000) &nbsp;•&nbsp; **OS:** RHEL/CentOS

---

## Architecture

```text
                          ┌─────────────────┐
                          │  stapp01:6000   │
                          ├─────────────────┤
 client ──HTTP:80──▶ stlb01 (nginx) ──proxy─▶│  stapp02:6000   │
                          ├─────────────────┤
                          │  stapp03:6000   │
                          └─────────────────┘
```

---

## 1. Verify the backends are healthy

On each app server, confirm `httpd` is running and listening on port `6000`.

```bash
ssh steve@stapp01
sudo systemctl status httpd
```

Expected — the log line confirms the listening port:

```text
httpd[24641]: Server configured, listening on: port 6000
```

Repeat for `stapp02` and `stapp03`. All three must be `active (running)` before the
load balancer is useful.

## 2. Check nginx on the load-balancer host

```bash
ssh loki@stlb01
sudo systemctl status nginx
```

Expected at this stage — installed but not yet running:

```text
nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
   Active: inactive (dead)
```

> If nginx is **not installed**, install it first: `sudo dnf install -y nginx`.

## 3. Configure the upstream and proxy

Edit `/etc/nginx/nginx.conf`. Two pieces go inside the existing `http { }` context.

**a) Define the backend pool** — the `upstream` block groups the three servers under
a single name (`backend`):

```nginx
http {
    upstream backend {
        server stapp01:6000;
        server stapp02:6000;
        server stapp03:6000;
    }

    # ... existing http settings ...
}
```

**b) Route incoming requests to the pool** — inside a `server` block (also within
`http { }`), proxy all requests to the named upstream:

```nginx
    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://backend;
        }
    }
```

> **Load-balancing method:** with no directive specified, nginx uses **round-robin** —
> each request goes to the next server in turn. Other options include `least_conn`
> (fewest active connections) and `ip_hash` (sticky by client IP). Round-robin is the
> right default here.

> **Tip — preserve client headers.** For many backends it helps to pass the original
> host and client IP. Not required for this task, but good practice:
> ```nginx
> proxy_set_header Host              $host;
> proxy_set_header X-Real-IP         $remote_addr;
> proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
> ```

## 4. Validate the configuration

Always test syntax before restarting the service:

```bash
sudo nginx -t
```

Expected:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 5. Enable, start, and check nginx

```bash
sudo systemctl enable nginx      # start on boot
sudo systemctl start nginx
sudo systemctl status nginx
```

Expected — now `enabled` and `active (running)`:

```text
nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
   Active: active (running) since Sun 2026-08-30 17:37:45 UTC
```

## 6. Test the proxy from the jump host

```bash
curl http://stlb01:80
```

Expected:

```text
Welcome to xFusionCorp Industries!
```

---

## Verify traffic is actually being distributed

A single `curl` only proves the proxy works — it doesn't show that all three backends
are in rotation. To watch round-robin in action, hit the LB repeatedly:

```bash
for i in $(seq 1 6); do curl -s http://stlb01/ -o /dev/null -w "%{http_code}\n"; done
```

To see *which* backend served each request, tail the access logs on the app servers
while you run the loop:

```bash
# on each app server, in separate terminals:
sudo tail -f /var/log/httpd/access_log
```

You should see requests landing on `stapp01`, `stapp02`, and `stapp03` in turn. If one
backend never appears, check its `httpd` status and reachability from `stlb01`
(next section).

Confirm nginx is listening on 80 and the upstream is loaded:

```bash
sudo ss -tlnp | grep nginx
sudo nginx -T | grep -A5 'upstream backend'
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `502 Bad Gateway` | nginx can't reach a backend | Verify `httpd` is up on :6000; test `curl http://stapp0X:6000` from `stlb01` |
| `502` on RHEL with SELinux enforcing | SELinux blocks nginx outbound connections | Set the network boolean (below) |
| One backend never gets traffic | That `httpd` is down, or firewall blocks :6000 | Check its status; open the port |
| `nginx -t` fails after edit | `upstream`/`server` placed outside `http { }` | Both blocks must live inside the `http` context |

### SELinux — the classic load-balancer gotcha

On RHEL/CentOS with SELinux **enforcing**, nginx is not permitted to make outbound
network connections by default, so proxying to the backends fails with `502` even
though the config is perfect. Confirm and fix:

```bash
getenforce
sudo ausearch -m avc -ts recent | grep nginx     # look for a 'name_connect' denial

# Allow nginx/httpd to initiate network connections:
sudo setsebool -P httpd_can_network_connect on
```

### Firewall on the backends

Each app server must allow inbound `6000` from the load balancer:

```bash
# on each app server (firewalld):
sudo firewall-cmd --permanent --add-port=6000/tcp
sudo firewall-cmd --reload
```

### Reachability check from the LB host

```bash
# from stlb01 — each should return the backend's page:
for h in stapp01 stapp02 stapp03; do
  echo "== $h =="; curl -s http://$h:6000/ | head -1
done
```

---

## Result

`stlb01` accepts HTTP on port 80 and round-robins each request across `stapp01`,
`stapp02`, and `stapp03` on port 6000, verified by a live request through the proxy
and by observing distributed hits in the backend access logs.
