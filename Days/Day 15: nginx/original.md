
We are tasked to install and configre nginx webserver on app server 2.

1. SSH to stapp02
```bash
ssh steve@stapp02
```
2. Install nginx 
```bash
sudo dnf install nginx
```
3.  Modify the default /etc/nginx.conf removing settings unrelated to secure ssl setup 

```bash 
# Settings for a TLS enabled server.

    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/nautilus.crt";
        ssl_certificate_key "/etc/pki/nginx/nautilus.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
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

}
```

and move our certificate and key to a dedicated directory
```bash
sudo mkdir /etc/pki/nginx
sudo cp /tmp/nautilus.* /etc/pki/nginx
```

4. Stream nginx logs, for debugging purposes:
```bash
sudo tail -f /var/log/nginx/error.log
```

5.  Modify index.html in root directory

6. Restart the nginx process
```bash 
sudo systemctl restart nginx
```

7. Verify that process has started
```bash
sudo systemctl status nginx
```

```bash

nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Sat 2026-08-29 17:05:46 UTC; 8s ago
    Process: 34609 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 34610 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 34617 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 34624 (nginx)
      Tasks: 17 (limit: 404712)
     Memory: 16.6M
        CPU: 56ms
     CGroup: /system.slice/nginx.service
             ├─34624 "nginx: master process /usr/sbin/nginx"
             ├─34625 "nginx: worker process"
             ├─34626 "nginx: worker process"
             ├─34627 "nginx: worker process"
             ├─34628 "nginx: worker process"
             ├─34629 "nginx: worker process"
             ├─34630 "nginx: worker process"
             ├─34631 "nginx: worker process"
             ├─34632 "nginx: worker process"
             ├─34633 "nginx: worker process"
             ├─34634 "nginx: worker process"
             ├─34635 "nginx: worker process"
             ├─34636 "nginx: worker process"
             ├─34637 "nginx: worker process"
             ├─34638 "nginx: worker process"
             ├─34639 "nginx: worker process"
             └─34640 "nginx: worker process"

Aug 29 17:05:46 stapp02 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 29 17:05:46 stapp02 nginx[34610]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 29 17:05:46 stapp02 nginx[34610]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 29 17:05:46 stapp02 systemd[1]: Started The nginx HTTP and reverse proxy server.
```

8. Test from remote host
```bash
curl -Ik https://stapp02/
```
```bash 
HTTP/2 200 
server: nginx/1.20.1
date: Sat, 29 Aug 2026 17:07:49 GMT
content-type: text/html
content-length: 2713881
last-modified: Fri, 12 Dec 2025 14:14:29 GMT
etag: "693c2345-296919"
accept-ranges: bytes

```
