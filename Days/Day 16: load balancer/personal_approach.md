We are tasked to configre nginx to forward and split incoming http traffic between 3 web-servers - stapp01, stapp02 and stapp03.

1.  Check health of httpd process on remote hosts
```
ssh steve@stapp02
sudo systemctl status httpd
```
On all three instances httpd server is running and serving web content on port 6000
```bash
Aug 30 17:23:59 stapp02 httpd[24641]: Server configured, listening on: port 6000
```

2.  Switch to lbr server and check whether nginx is installed and running
```bash
ssh loki@stlb01
```
```nginx
**○ nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: inactive (dead)**
```
It's not, that's ok - we will configre the load balancer and start it up

3.  Looking at the docs we should add following lines to nginx.conf under http context:
``` nginx
http {
    upstream backend {
        server stapp01:6000;
        server stapp02:6000;
        server stapp03:6000;
    }
}
```
This groups servers under the `backend` variable

Than let nginx know where to route requests using this variable:

```nginx
server {
    location / {
        proxy_pass http://backend;
    }
}
```

4. Save changes and validate config file:
```bash
sudo nginx -t
```
```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

5. Enable and start and check the process status
```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```
```nginx
 nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Sun 2026-08-30 17:37:45 UTC; 6s ago
    Process: 28396 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 28397 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 28404 ExecStart=/us
    Aug 30 17:37:45 stlb01 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 30 17:37:45 stlb01 nginx[28397]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 30 17:37:45 stlb01 nginx[28397]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 30 17:37:45 stlb01 systemd[1]: Started The nginx HTTP and reverse proxy server.
```

6. Test the proxy from jump host:

```bash
curl http://stlb01:80
Welcome to xFusionCorp Industries!
