
https://easyengine.io/tutorials/php/directly-connect-php-fpm/
https://easyengine.io/tutorials/php/fpm-status-page/

https://www.php.net/manual/en/install.fpm.configuration.php

Configure fpm with nginx:

https://www.php.net/manual/en/install.unix.nginx.php

Check if dnf package installer has specific module listed in repo:

```bash
dnf --showduplicates list php-fpm
```

Installing specific version of php + php-fmp and adding a repo

https://php.watch/articles/php-8.3-install-upgrade-on-fedora-rhel-el

Will need to install nginx under a dedicated user

Properly install the php-fpm with config - no need

```bash
find / -type f -name www.conf 2>/dev/null
/etc/opt/remi/php82/php-fpm.d
```


