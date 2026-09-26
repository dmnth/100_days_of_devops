https://easyengine.io/tutorials/php/directly-connect-php-fpm/
https://easyengine.io/tutorials/php/fpm-status-page/

https://www.php.net/manual/en/install.fpm.configuration.php



Check if dnf package installer has specific module listed in repo:

```bash
dnf --showduplicates list php-fpm
```

Installing specific version of php + php-fmp and adding a repo

https://php.watch/articles/php-8.3-install-upgrade-on-fedora-rhel-el

Unix permissions guide:

https://daily.dev/blog/linux-user-groups-and-permissions-guide/

How to configure php-fpm with nginx:

https://www.php.net/manual/en/install.unix.nginx.php

Properly install the php-fpm with config - no need

a. Install `nginx` on `app server 1` , configure it to use port `8091` and its document root should be `/var/www/html`.

```bash
```
sudo dnf install nginx
/etc/nginx/nginx.con
start, enable, check status with systemctl

b. Install `php-fpm` version `8.3` on `app server 1`, it must use the unix socket `/var/run/php-fpm/default.sock` (create the parent directories if don't exist).


```bash
# Save existing php package list to packages.txt file
sudo dnf list installed | grep php | tee packages.txt

# Add Remi's repo
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(cut -d ' ' -f 4 /etc/redhat-release | cut -d '.' -f 1).noarch.rpm -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-$(cut -d ' ' -f 4 /etc/redhat-release | cut -d '.' -f 1).rpm -y

# Install new PHP 8.3 packages
sudo dnf install php83 php83-php-fpm

# Remove old packages
sudo dnf remove php82*

# Create symlinks from `php` to actual PHP binary
sudo dnf install php83-syspaths -y
```

Note on unix sockets: 

In Linux, a **socket** is ==a software abstraction that acts as an endpoint for bidirectional communication between two processes==. [](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/7/html/reference_guide/chap-sockets)

True to the core Unix philosophy that **"everything is a file,"** Linux treats a socket as a special type of file.

**FPM creates it.** In the pool config (RHEL: `/etc/php-fpm.d/www.conf`), the `listen` directive is what makes the socket:

Fpm configuration file search, because it's not in default location

```bash
find / -type f -name www.conf 2>/dev/null
/etc/opt/remi/php83/php-fpm.d/www.conf
```

**`user` / `group` — who the PHP processes run as**

```ini
user = nginx
group = nginx
```

**`listen.owner` / `listen.group` — who owns the socket**

In the config file we will need to modify permissions so that nginx user will be able to use thes socket. Don't forget to uncomment, as i had to manually set permissions later using chmod and chown:

```ini
listen.owner = nginx
listen.group = nginx
listen.mode  = 0660
```

This directive creates the actual socket, subfolders are expected to be created beforehand:
```ini
listen = /run/php-fpm/www.sock
```

Nginx config /etc/nginx/

```nginx
location ~ \.php$ {
    include        fastcgi_params;
    fastcgi_pass   unix:/run/php-fpm/www.sock;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

- **`location ~ \.php$ {`** — regex match (`~` = case-sensitive) for any URI ending in `.php`; those requests go to PHP instead of being served as static files.
- **`include fastcgi_params;`** — pulls in nginx's default list of FastCGI env vars (`REQUEST_METHOD`, `QUERY_STRING`, headers, etc.) so you don't retype them.
- **`fastcgi_pass unix:/run/php-fpm/www.sock;`** — hands the request to PHP-FPM over that Unix socket. Must match the `listen =` path in your pool. (The hyperlink in your paste is just editor auto-linking — it's a plain path.)
- **`fastcgi_index index.php;`** — default file when the request maps to a directory.
- **`fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;`** — the important one: builds the absolute on-disk path (docroot + script name) that FPM actually executes. Wrong value → PHP's "File not found."

Apply@verify:

```bash
sudo systemctl restart php-fpm
ls -l /run/php-fpm/www.sock          # expect: srw-rw---- nginx nginx
sudo nginx -t && sudo systemctl reload nginx
curl http://stapp01:8085/index.php
Welcome to xFusionCorp Industries!
```
