We are tasked to fix the issue with Dockerfile.

1) Locate the Dockerfile and attempt to run it and watch it fail

2) Open docker hub and look for base image. Notice that path is defined differently than in our Dockerfile:

Our file:
```dockerfile
RUN sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf.d/httpd.conf
```

Documentation:
```dockerfile
FROM httpd:2.4
COPY ./my-httpd.conf /usr/local/apache2/conf/httpd.conf
```

3) Apply changes to Dockerfile and attempt to build it

4) Startup the container and check whether httpd service is running

```bash
docker top 1f777
```

```bash
curl http://172.12.0.2:8080
This Dockerfile works!
```
