---
title: "100 Days of DevOps: Docker (Days 35–37)"
tags: [devops, docker, kodekloud, runbook]
---

# 100 Days of DevOps: Docker (Days 35–37)

## Contents

- [Day 35: Install Docker and initiate the service](#day-35-install-docker-and-initiate-the-service)
- [Day 36: Run a container](#day-36-run-a-container)
- [Day 37: Copy a file to a container](#day-37-copy-a-file-to-a-container)

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

### Verify

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

### Notes

- Don't decrypt or rename the `.gpg` file. The task only moves opaque bytes.
- `docker cp` works on stopped containers too, but it does not create missing parent directories.
- A trailing slash on the destination means "into this directory"; without one, the path may be treated as a new filename.
