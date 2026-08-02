---
title: Deploy Tomcat 11 on Application Server
tags: [devops, tomcat, centos, java]
---

# Deploy Tomcat 11 on Application Server

## 1. Switch to Application server

```bash
ssh steve@stapp02
```

## 2. Check Java version

```bash
java -version
```

## 3. Tomcat requires Java 17 at lease, installing latest version

```bash
sudo dnf install -y java-21-openjdk-devel
```

## 4. Switch Java version

```bash
sudo alternatives -config java
```

## 5. Create tomcat user with shell access(for convenience and testing)

```bash
sudo useradd -r -U -d /opt/tomcat -s /bin/bash tomcat
```

## 6. Create directory /opt/tomcat and change it's permissions

```bash
sudo mkdir /opt/tomcat
sudo chown tomcat:tomcat /opt/tomcat
su tomcat
cd /opt/tomcat
```

## 7. Download latest tomcat(11) archive and have it extracted

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.24/bin/apache-tomcat-11.0.24.tar.gz
tar -xvfi apache-tomcat-11.0.24.tar.gz
rm -rf *.tar.gz
mv apache-tomcat-11.0.24 apache-tomcat-11
cd apcahe-tomcat-11
```

## 8. Configure apache to startup on port 8085

```xml
vi conf/server.xml
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8085" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
:wq
```

## 9. Copy web archive from jump host

```bash
scp thor@jump-host:/tmp/ROOT.war /opt/tomcat/apache-tomcat-11/webapps
```

## 10. Startup the webserver

```bash
/bin/startup.sh
```

## 11. Check if app is running:

```bash
curl http://stapp02:8085
```

```html
bash-5.1$ curl http://localhost:8085
<!DOCTYPE html>
<!--
To change this license header, choose License Headers in Project Properties.
To change this template file, choose Tools | Templates
and open the template in the editor.
-->
<html>
    <head>
        <title>SampleWebApp</title>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
    </head>
    <body>
        <h2>Welcome to xFusionCorp Industries!</h2>
        <br>

    </body>
</html>
```

## 12. Revoke shell from tomcat user

```bash
sudo usermod -s /sbin/nologin tomcat
```
