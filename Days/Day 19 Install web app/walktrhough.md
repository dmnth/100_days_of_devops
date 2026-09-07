We are tasked to startup a web server and spin up two web apps.

1. Check whether httpd is installed
```bash
sudo systemctl status httpd
Unit httpd.service could not be found.
```

2. Install the service 
```bash
sudo dnf install httpd
```

3. Change the listen port
```bash
sudo vim /etc/httpd/conf/httpd.conf
Listen 6100
```

4. Create conf file in, due to this directory is in Include path
```bash
sudo touch tony.conf
```

5. Create two virtual hosts in tony.conf pointing to two different directories with html content
```php
<VirtualHost ::6100>
    DocumentRoot "/var/www/html/news"
</VirtualHost>
<VirtualHost ::6100>
    DocumentRoot "/var/www/html/cluster"
</VirtualHost>****
```

6. Copy files from remote host into /var/www/html

```bash 
sudo scp -r thor@jump-host:/home/thor/cluster .
sudo scp -r thor@jump-host:/home/thor/news .
```

7.  Check config
```bash
apachectl configtest
[Mon Sep 07 19:11:45.606834 2026] [core:error] [pid 36754:tid 36754] (EAI 2)Name or service not known: AH00547: Could not resolve host name : -- ignoring!
[Mon Sep 07 19:11:45.606915 2026] [core:error] [pid 36754:tid 36754] (EAI 2)Name or service not known: AH00547: Could not resolve host name : -- ignoring!
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.29.225. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

8. Restart the service
```bash
sudo systemctl restart httpd
```

9. Verify with curl

```
curl http://localhost:6100/news
```

```http
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
</body></html>
[tony@stapp01 ~]$ curl http://localhost:6100/news/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
</body></html>
[tony@stapp01 ~]$ curl http://localhost:6100/news/
<!DOCTYPE html>
<html>
<body>

<h1>KodeKloud</h1>

<p>This is a sample page for our news website</p>

</body>
</html>[tony@stapp01 ~]$ 
```
