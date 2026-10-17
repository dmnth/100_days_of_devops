---
title: "100 Days of DevOps: Docker (Days 41–43)"
tags: [devops, docker, kodekloud, runbook]
---

# 100 Days of DevOps: Docker (Days 41–43)

## Contents

- [Day 41: Write a Dockerfile](#day-41-write-a-dockerfile)
- [Day 42: Create a Docker network](#day-42-create-a-docker-network)
- [Day 43: Docker ports mapping](#day-43-docker-ports-mapping)

> The values below (hosts, ports, names, subnets) are examples. Each lab instance gets its own, so replace them with the exact strings from your task. The checker compares them literally.

---

## Day 41: Write a Dockerfile

Task: create `/opt/docker/Dockerfile` on the app server. Use `ubuntu:24.04` as the base image, install `apache2`, and make it listen on port `5003`. Don't change any other Apache settings.

| Value | Example |
|---|---|
| Host | `stapp03` |
| Path | `/opt/docker/Dockerfile` |
| Base image | `ubuntu:24.04` |
| Apache port | `5003` |

### Create the file

The filename must be exactly `Dockerfile`, with a capital D and no extension. `/opt/docker` may need `sudo`:

```bash
sudo mkdir -p /opt/docker
sudo vi /opt/docker/Dockerfile
```

```dockerfile
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y apache2 && \
    sed -i 's/^Listen 80$/Listen 5003/' /etc/apache2/ports.conf && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 5003

CMD ["apache2ctl", "-D", "FOREGROUND"]
```

### How each line works

- `FROM` pins the exact base image tag from the task.
- `ARG DEBIAN_FRONTEND=noninteractive` stops apt from waiting on prompts, such as the timezone question, which would hang the build. `ARG` exists only at build time.
- One `RUN` chains update, install, config change and cache cleanup into a single layer.
- `sed` changes only the `Listen` line in `ports.conf`, the same technique as [Day 40](docker-days-35-40.md#day-40-install-and-configure-apache-in-an-ubuntu-container). The task says not to touch other settings, so the vhost and document root stay as they are.
- `EXPOSE` is documentation only. It doesn't publish anything.
- `CMD` must run Apache in the **foreground**. A container lives only as long as its main process, and `service apache2 start` exits immediately after starting the daemon in the background.

### Build and test

The build context is the directory holding the Dockerfile:

```bash
cd /opt/docker
sudo docker build -t apache-test:5003 .
sudo docker run -d --name apache-test -p 5003:5003 apache-test:5003
curl -I http://localhost:5003      # HTTP/1.1 200 OK
```

Check that the port change actually landed in the image:

```bash
sudo docker exec apache-test grep '^Listen' /etc/apache2/ports.conf   # Listen 5003
```

Clean up the test container afterwards if the task doesn't ask for one:

```bash
sudo docker rm -f apache-test
```

### Notes

- `RUN` executes once, at build time. `CMD` runs every time a container starts.
- The checker usually reads the Dockerfile at the exact path, builds it, and curls the port. A typo in the path or filename fails it even when the content is right.
- The build fails if apt asks a question. Use `-y` and `DEBIAN_FRONTEND`.

[Back to contents](#contents)

---

## Day 42: Create a Docker network

Task: create a Docker network on the app server with a specific driver, subnet and IP range.

| Value | Example |
|---|---|
| Host | `stapp01` |
| Network name | `official` |
| Driver | `macvlan` |
| Subnet | `172.28.0.0/24` |
| IP range | `172.28.0.0/24` |

### Create the network

```bash
sudo docker network create \
  --driver macvlan \
  --subnet 172.28.0.0/24 \
  --ip-range 172.28.0.0/24 \
  official
```

The network name goes last, as a positional argument.

### Verify

```bash
sudo docker network ls
sudo docker network inspect -f '{{.Driver}} {{range .IPAM.Config}}{{.Subnet}} {{.IPRange}}{{end}}' official
```

```text
macvlan 172.28.0.0/24 172.28.0.0/24
```

### Subnet, IP range and gateway

| Option | Meaning |
|---|---|
| `--subnet` | The whole address space of the network |
| `--ip-range` | The part of the subnet Docker hands out to containers. It must sit inside the subnet |
| `--gateway` | The gateway address. If omitted, Docker picks the first usable address |

When subnet and IP range are the same, containers can get any address in the subnet.

### Network drivers

| Driver | What it does |
|---|---|
| `bridge` | Default. A private network on one host; containers reach outside through NAT |
| `host` | No isolation; the container uses the host's network stack directly |
| `macvlan` | Each container gets its own MAC address and appears as a separate device on the physical network |
| `overlay` | Spans several Docker hosts (Swarm) |
| `none` | No networking |

### Notes

- Mixing up `--subnet` and `--ip-range` is the most common mistake. The checker reads both fields.
- macvlan normally takes a parent interface (`-o parent=eth0`). Without one, Docker attaches it to a dummy internal interface, which is fine for the lab.
- A network can't be edited after creation. To fix a wrong value, `docker network rm official` and create it again (it must have no containers attached).

[Back to contents](#contents)

---

## Day 43: Docker ports mapping

Task: pull an image, run a named container from it, map a host port to the container's port, and keep it running.

| Value | Example |
|---|---|
| Host | `stapp03` |
| Image | `nginx:alpine-perl` |
| Container name | `demo` |
| Host port | `3002` |
| Container port | `80` |

### Pull and run

```bash
sudo docker pull nginx:alpine-perl
sudo docker run -d --name demo -p 3002:80 nginx:alpine-perl
```

- Use the image **name and tag**, not the ID. The checker reads `.Config.Image` (see [Day 36](docker-days-35-40.md#day-36-run-a-container)).
- `-d` runs the container in the background, so it keeps running after the command returns.
- `-p 3002:80` is always **host:container**. Swapping them is the classic mistake.

### Verify

```bash
sudo docker ps --filter name=demo
sudo docker port demo
```

```text
80/tcp -> 0.0.0.0:3002
```

Check that the page answers through the host port:

```bash
curl -I http://localhost:3002      # HTTP/1.1 200 OK
```

### Port mapping forms

| Flag | Result |
|---|---|
| `-p 3002:80` | Host port 3002, all host interfaces, to container port 80 |
| `-p 127.0.0.1:3002:80` | Same, but reachable only from the host itself |
| `-p 80` | Random free host port to container port 80 |
| `-P` | Every `EXPOSE`d port to a random host port |
| `-p 3002:80/udp` | UDP instead of the default TCP |

### Notes

- Port mappings can't be added to a running container. To change one, remove the container and run it again.
- If the host port is taken, `docker run` fails with `port is already allocated`. Find the owner with `ss -tlnp | grep 3002`.
- The container port must match what the app listens on inside the container. nginx listens on 80 by default, so `-p 3002:8080` would map to nothing.

[Back to contents](#contents)
