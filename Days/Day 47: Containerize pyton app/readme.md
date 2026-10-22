---

title: Containerizing a Python App with a Dockerfile tags:

- docker
- dockerfile
- python
- runbook aliases:
- nautilus python-app
- pythonapp_nautilus created: 2026-10-22

---

# Containerizing a Python App with a Dockerfile

This runbook builds a Docker image for a Python app, runs it as a container, and tests it on **App Server 2** (`stapp02`) in Stratos DC.

## Task

Create `/python_app/Dockerfile` that:

- uses any `python` image as the base;
- installs dependencies from `requirements.txt`;
- exposes port `5000`;
- runs `server.py` using `CMD`.

Then:

1. Build an image named `nautilus/python-app`.
2. Run a container named `pythonapp_nautilus`, mapping container port `5000` to host port `8098`.
3. Test the app with `curl` on App Server 2.

## Project layout

```
/python_app
├── Dockerfile
└── src
    ├── requirements.txt
    └── server.py
```

Confirm it before building:

```bash
ls -R /python_app
```

---

## Final Dockerfile

```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /usr/src/app

COPY src/requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY src/server.py ./

EXPOSE 5000

CMD ["python3", "./server.py"]
```

### What each line does

|Instruction|Purpose|
|---|---|
|`FROM python:3.11-slim`|Debian-based Python image. Small, and most packages install as prebuilt wheels.|
|`ENV PYTHONDONTWRITEBYTECODE=1`|Stops Python from writing `.pyc` files into the image|
|`ENV PYTHONUNBUFFERED=1`|Sends `print`/log output straight to `docker logs` instead of buffering it|
|`WORKDIR /usr/src/app`|Sets (and creates) the working directory for the following instructions and at runtime|
|`COPY src/requirements.txt ./`|Copies only the dependency list first, for layer caching|
|`RUN pip install --no-cache-dir ...`|Installs dependencies; `--no-cache-dir` keeps pip's download cache out of the image|
|`COPY src/server.py ./`|Copies the application code|
|`EXPOSE 5000`|Documents the port the app listens on. It does **not** publish the port.|
|`CMD ["python3", "./server.py"]`|Default command. The exec (JSON) form runs Python as PID 1, so it receives stop signals.|

---

## Before / after

### Original

```dockerfile
FROM python:3.11-alpine3.23

WORKDIR /usr/src/app

COPY /src/requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY /src/server.py ./
EXPOSE 5000
CMD ["python3", "./server.py"]
```

The original already met every requirement. The changes below improve reliability and clarity.

|Change|Before|After|Why|
|---|---|---|---|
|Base image|`python:3.11-alpine3.23`|`python:3.11-slim`|Very specific Alpine tags only exist for some Python/Alpine combinations. Alpine also uses musl, so packages without musl wheels compile from source and can fail without `gcc`. `slim` (Debian) avoids both problems.|
|COPY source paths|`/src/requirements.txt`|`src/requirements.txt`|COPY sources are always relative to the **build context**. A leading `/` still means `<context>/src/...`, but it misleadingly looks like a host absolute path.|
|Python environment|none|`PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`|Cleaner image; app output shows up in `docker logs` straight away, which makes the verification step reliable.|
|Kept as is|Layer order|Layer order|Installing requirements before copying code means code changes don't re-run `pip install`.|

> [!NOTE] Alpine is still a valid choice when image size matters most. If you use it, check the tag exists first (`docker pull python:3.11-alpine`) and expect build tools to be needed for packages with C extensions.

---

## Pre-check: what the app listens on

```bash
grep -nE 'run\(|host|port' /python_app/src/server.py
```

|Finding|Result|
|---|---|
|`host="0.0.0.0", port=5000`|Works through the port mapping|
|No host, or `127.0.0.1`|App only listens on the container's loopback. `curl` gets `Connection reset` / `Empty reply` even though the container is `Up`.|
|A port other than 5000|The `8098:5000` mapping points at nothing|

---

## Steps

### 1. Create the Dockerfile

```bash
sudo vi /python_app/Dockerfile
```

The filename must be exactly `Dockerfile` (capital D, no extension). `/python_app` is usually root-owned, so `sudo` is needed.

### 2. Build the image

```bash
cd /python_app
docker build -t nautilus/python-app .
```

The `.` makes `/python_app` the build context, which is what `COPY src/...` is relative to.

### 3. Confirm the image exists

```bash
docker images nautilus/python-app
```

### 4. Run the container

```bash
docker container run -d \
  --name pythonapp_nautilus \
  -p 8098:5000 \
  nautilus/python-app
```

`-p 8098:5000` follows `HOST:CONTAINER` order: host port 8098 forwards to container port 5000.

### 5. Verify

```bash
docker ps --filter name=pythonapp_nautilus
# STATUS: Up    PORTS: 0.0.0.0:8098->5000/tcp

docker logs pythonapp_nautilus
# e.g. "Running on http://0.0.0.0:5000"

curl http://localhost:8098/
```

> [!IMPORTANT] Run the build, run and test on **App Server 2** (`stapp02`). Building on another server leaves the image and container in the wrong place.

---

## Troubleshooting

|Symptom|Likely cause|Fix|
|---|---|---|
|`manifest unknown` on build|Base image tag doesn't exist|Use a standard tag such as `python:3.11-slim`|
|`COPY failed: file not found`|Wrong build context or file location|Build from `/python_app`; check `ls /python_app/src`|
|`permission denied` creating the Dockerfile|`/python_app` is root-owned|Use `sudo`|
|pip fails compiling a package (Alpine)|No musl wheel, no compiler|Switch to `slim`, or install build tools|
|`Conflict. The container name ... is already in use`|Leftover container from an earlier attempt|`docker rm -f pythonapp_nautilus`|
|Container exits immediately|Missing module or Python error|`docker logs pythonapp_nautilus`|
|`Connection reset` / `Empty reply`|App bound to `127.0.0.1`|App must listen on `0.0.0.0`|
|`Connection refused` on 8098|Container not running, or wrong port mapping|`docker ps -a`, `docker port pythonapp_nautilus`|

## Rebuilding after changes

```bash
docker rm -f pythonapp_nautilus
docker build -t nautilus/python-app /python_app
docker container run -d --name pythonapp_nautilus -p 8098:5000 nautilus/python-app
```

A container keeps using the image it was created from. After a rebuild, remove the container and create it again.

## Key takeaways

1. COPY paths are relative to the **build context**, not the host's root.
2. Copy dependency files first and code last, for faster rebuilds.
3. `EXPOSE` documents the port; `-p HOST:CONTAINER` publishes it.
4. Prefer the exec form of `CMD` (`["python3", "server.py"]`).
5. The app must listen on `0.0.0.0` inside the container to be reachable through a port mapping.
6. A `slim` base is the safe default for Python; Alpine trades compatibility for size.
