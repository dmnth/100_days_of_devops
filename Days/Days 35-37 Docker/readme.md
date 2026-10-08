---
title: "100 Days of DevOps: Docker (Days 35–40)"
tags: [devops, docker, kodekloud, runbook]
---

# 100 Days of DevOps: Docker (Days 35–40)

## Contents

- [Day 35: Install Docker and initiate the service](#day-35-install-docker-and-initiate-the-service)
- [Day 36: Run a container](#day-36-run-a-container)
- [Day 37: Copy a file to a container](#day-37-copy-a-file-to-a-container)
- [Day 38: Pull and re-tag an image](#day-38-pull-and-re-tag-an-image)
- [Day 39: Back up a container](#day-39-back-up-a-container)
- [Day 40: Install and configure Apache in an Ubuntu container](#day-40-install-and-configure-apache-in-an-ubuntu-container)

---

## Day 35: Install Docker and initiate the service

> Run this on the app server named in the task. The jump host has no systemd, so `systemctl` fails there with `System has not been booted with systemd as init system (PID 1)`.

### Remove old packages

If Docker was installed before:

```bash
sudo dnf remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine
```

### Set up the repository

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### Install Docker CE and the Compose plugin

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Initiate the service

"Initiate" in lab tasks means start it now **and** enable it at boot:

```bash
sudo systemctl enable --now docker
```

Check that the service is up:

```bash
systemctl is-active docker    # active
systemctl is-enabled docker   # enabled
```

### Check the installation

```bash
sudo docker run hello-world
```

Reference: [Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

[Back to contents](#contents)

---

## Day 36: Run a container

### Find and pull the image

Find the image and tag on Docker Hub: [nginx tags](https://hub.docker.com/_/nginx?tag=stable-alpine)

```bash
docker pull nginx:stable-alpine
```

List the local images to see the ID:

```bash
docker image ls
```

```text
REPOSITORY   TAG             IMAGE ID       CREATED       SIZE
nginx        stable-alpine   43d9d8c1f896   12 days ago   62.4MB
```

### First attempt: run by image ID (failed the check)

```bash
docker run --name nginx_2 43d9
```

```bash
docker ps
```

```text
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
b25923fdc200   43d9      "/docker-entrypoint.…"   6 minutes ago   Up 5 minutes   80/tcp    nginx_2
```

The container ran, but the checker rejected it:

```text
- container is not using image nginx:alpine
```

Why: the checker reads `.Config.Image`, which stores the exact string passed to `docker run`. Here that string was `43d9`, not `nginx:alpine`.

```bash
docker inspect -f '{{.Config.Image}}' nginx_2   # 43d9
```

### Ways to check that a container is running

```bash
# List / filter
docker ps                                          # running only
docker ps -a                                       # all, incl. exited
docker ps --filter name=nginx_2 --filter status=running
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Image}}'

# State fields (checker-style)
docker inspect -f '{{.State.Status}}'  nginx_2     # running / exited / created / restarting
docker inspect -f '{{.State.Running}}' nginx_2     # true / false
docker inspect -f '{{.State.StartedAt}} {{.State.ExitCode}} {{.State.Restarting}}' nginx_2
docker inspect -f '{{.Config.Image}}'  nginx_2     # image as given at create time

# Liveness proof
docker top nginx_2                                 # processes inside
docker logs --tail 20 nginx_2                      # startup errors
docker exec nginx_2 true && echo alive             # fails if not running
docker stats --no-stream nginx_2                   # live CPU/mem

# Functional proof (if ports published)
docker port nginx_2
curl -I http://localhost:<host_port>
```

### Image ID vs image name

Use the ID for local debugging and the name for everything else.

**Why names win by default**

- People and tools can read them. `nginx:1.27-alpine` says what's running, while `43d9` says nothing.
- They're portable. Any host can pull a name, but an ID only exists where the image was built or pulled.
- Compose files, Kubernetes manifests, CI pipelines and lab checkers all expect names.

**The real weakness of names is mutable tags, not names themselves**

- `nginx:latest`, or even `nginx:alpine`, can point to different images next week.
- **The fix is a specific tag (`nginx:1.27.2-alpine`) or, stricter still, a digest: `nginx:1.27.2-alpine@sha256:...`. That keeps the readable name and pins the exact image.**

**When an ID makes sense**

- Quick local debugging: re-running something you just built, or an untagged `<none>` image.
- Inspecting or cleaning up dangling images.

**Summary:** use a specific tag, plus a digest when reproducibility matters (production, CI). Avoid `:latest` and bare IDs in anything shared or checked.

[Back to contents](#contents)

---

## Day 37: Copy a file to a container

### Copy the file

`docker cp` copies files between the host and a container. `docker exec` runs commands inside one and is only used here for verification.

```bash
docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/tmp
```

```text
Successfully copied 2.05kB to ubuntu_latest:/tmp
```

### Verify the copy

Check that the file is there:

```bash
docker exec ubuntu_latest ls -l /tmp
```

```text
total 4
-rw-r--r-- 1 root root 105 Oct  5 17:29 nautilus.txt.gpg
```

Check that the file was not modified. The hashes must match:

```bash
md5sum /tmp/nautilus.txt.gpg
docker exec ubuntu_latest md5sum /tmp/nautilus.txt.gpg
```

### Copy notes

- Don't decrypt or rename the `.gpg` file. The task only moves opaque bytes.
- `docker cp` works on stopped containers too, but it does not create missing parent directories.
- A trailing slash on the destination means "into this directory"; without one, the path may be treated as a new filename.

[Back to contents](#contents)

---

## Day 38: Pull and re-tag an image

### Pull and tag

Pull the `busybox` image and give it a custom tag:

```bash
docker pull busybox:musl
docker tag busybox:musl busybox:local
```

A tag is just another name for the same image ID. Both tags show the same ID in `docker images`.

### Remove the original tag

```bash
docker rmi busybox:musl
```

- `docker rmi` (or `docker image rm`) removes images. `docker rm` removes containers and would fail here.
- Because `busybox:local` still points to the image, this only removes the `musl` tag. The image itself stays.

[Back to contents](#contents)

---

## Day 39: Back up a container

### Commit the container to an image

Save the changes made inside a container as a new image:

```bash
docker container commit ubuntu_latest demo:datacenter
docker images
```

```text
REPOSITORY   TAG          IMAGE ID       CREATED         SIZE
demo         datacenter   270b3242cc4d   5 seconds ago   144MB
ubuntu       latest       6e92c4fd8dfd   11 days ago     101MB
```

### What docker commit captures

- The container's writable layer (every file change made inside it) on top of the original image's layers.
- The result is a new image you can tag, run, push or save.
- Image config like CMD, ENV and EXPOSE carries over, and you can change it with `-c`.

### What it misses

- **Volumes and bind mounts.** Data in them isn't part of the container's filesystem, so it's skipped. For databases this is usually the data that matters most.
- **Runtime settings** from `docker run`: published ports, the container name, networks, restart policy, mounted paths. The image has no record of them.
- **Running processes and memory state.** It's a filesystem snapshot, not a live one. By default it pauses the container while committing.

[Back to contents](#contents)

---

## Day 40: Install and configure Apache in an Ubuntu container

Task: install `apache2` in the `kkloud` container, make it listen on port 5004 on the container IP and localhost, and keep both Apache and the container running.

### Open a shell in the container

The container name comes before the command:

```bash
docker container exec --interactive --tty kkloud /bin/bash
```

### Install apache2

Inside the container:

```bash
service apache2 status   # not installed yet
apt update
apt install -y apache2
```

### Change the listen port

Look around `/etc/apache2` for config files. `ports.conf` holds the `Listen` directive:

```apache
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>
```

There's no text editor in the container, so use `sed` to replace the line:

```bash
sed -i 's/^Listen 80$/Listen 0.0.0.0:5004/' /etc/apache2/ports.conf
```

- `^...$` anchors the match to a line that is exactly `Listen 80`. It won't touch `#Listen 80`, `Listen 8080` or `Listen 443`.
- `-i` edits the file in place. Use `-i.bak` instead if you want a backup copy.
- `0.0.0.0` binds all IPv4 interfaces, which covers both `127.0.0.1` and the container IP.

### Update the virtual host

Change the default vhost to the same port:

```bash
sed -i 's/^<VirtualHost \*:80>$/<VirtualHost *:5004>/' /etc/apache2/sites-enabled/000-default.conf
```

- `\*` escapes the asterisk in the search pattern. In the replacement it's literal.
- `sites-enabled/000-default.conf` is a symlink. `sed -i` replaces it with a regular file; add `--follow-symlinks` to edit the target in `sites-available/` instead.

Check the syntax:

```bash
apachectl configtest   # Syntax OK
```

### Start apache2

There's no systemd in the container, so use `service`:

```bash
service apache2 start
```

```text
 * Starting Apache httpd web server apache2
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.12.0.2. Set the 'ServerName' directive globally to suppress this message
```

AH00558 is a warning, not an error. Apache started fine, and the IP it shows is the container's IP.

```bash
service apache2 status
```

```text
 * apache2 is running
```

### Find the container IP

From the host:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kkloud
```

Or look in the full `docker container inspect kkloud` output:

```json
"Networks": {
    "bridge": {
        "IPAddress": "172.12.0.2",
        "IPPrefixLen": 24
    }
}
```

### Verify the page is served

Inside the container, on localhost:

```bash
ss -tlnp | grep 5004          # or: netstat -tlnp
curl -I http://localhost:5004
```

From the host, on the container IP:

```bash
curl http://172.12.0.2:5004
```

```html
<div class="main_page">
  <div class="page_header floating_element">
    <img src="/icons/ubuntu-logo.png" alt="Ubuntu Logo" class="floating_element"/>
    <span class="floating_element">
      Apache2 Ubuntu Default Page
    </span>
  </div>
```

Confirm the container is still running:

```bash
docker ps --filter name=kkloud
```

[Back to contents](#contents)
