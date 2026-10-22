`For web service:`  
  
a. Container name must be `php_apache`.  
b. Use image `php` with any `apache` tag. Check [here](https://hub.docker.com/_/php?tab=tags/) for more details.  
c. Map `php_apache` container's port `80` with host port `8084`  
d. Map `php_apache` container's `/var/www/html` volume with host volume `/var/www/html`.  
  
`For DB service:`  
  
a. Container name must be `mysql_apache`.  
b. Use image `mariadb` with any tag (preferably `latest`). Check [here](https://hub.docker.com/_/mariadb?tab=tags/) for more details.  
c. Map `mysql_apache` container's port `3306` with host port `3306`  
d. Map `mysql_apache` container's `/var/lib/mysql` volume with host volume `/var/lib/mysql`.  
e. Set MYSQL_DATABASE=`database_apache` and use any custom user ( except root ) with some complex password for DB connections.  
3. After running docker-compose up you can access the app with curl command `curl <server-ip or hostname>:8084/`

```yml
services:
 web-service:
    image: php:8.2-apache-bookworm
    container_name: php_apache
    ports:
      - "8084:80"
    volumes:
      - "/var/www/html:/var/www/html"
 db-service:
    image: mariadb:latest
    container_name: mysql_apache
    ports:
      - "3306:3306"
    volumes:
      - "/var/lib/mysql:/var/lib/mysql"
    environment:
      - MYSQL_DATABASE=database_apache
      - MARIADB_ROOT_PASSWORD=pr0Ff3sS10n@lN0l1f3
      - MARIADB_USER=boris
      - MARIADB_PASSWORD=n0futur3n3v3rm0@r!
~                                            
```

### Sidequest - debuggin user creation on Maria DB initialization

---

title: Debugging MariaDB Container User Creation tags:

- docker
- docker-compose
- mariadb
- troubleshooting
- runbook aliases:
- MariaDB access denied in Docker
- MariaDB user not created created: 2026-10-22

---

# Debugging MariaDB Container User Creation

Use this runbook when a user defined in the compose file can't log in to a MariaDB container:

```
ERROR 1045 (28000): Access denied for user 'boris'@'localhost' (using password: YES)
```

## Context

|Item|Value|
|---|---|
|Host|`stapp01` (App Server 1, Stratos DC)|
|DB container|`mysql_apache` (`mariadb:latest`)|
|Data directory|`/var/lib/mysql` (host) → `/var/lib/mysql` (container)|
|Database|`database_apache`|
|App user|`boris`|

Relevant part of the compose file:

```yaml
 db-service:
    image: mariadb:latest
    container_name: mysql_apache
    ports:
      - "3306:3306"
    volumes:
      - "/var/lib/mysql:/var/lib/mysql"
    environment:
      - MYSQL_DATABASE=database_apache
      - MARIADB_ROOT_PASSWORD=<root_password>
      - MARIADB_USER=boris
      - MARIADB_PASSWORD=<user_password>
```

## Root cause (summary)

The MariaDB image applies `MARIADB_USER`, `MARIADB_PASSWORD`, `MARIADB_ROOT_PASSWORD` and `MYSQL_DATABASE` **only on the first start, when the data directory is empty**. An earlier run had already initialized `/var/lib/mysql` without valid user variables. Later edits to the compose file were ignored, so `boris` was never created.

---

## 1. Reproduce the error

```bash
docker exec -it mysql_apache mariadb -u boris -p -e 'SHOW DATABASES;'
```

`ERROR 1045 Access denied` means either the user doesn't exist or the password doesn't match. The next steps find out which.

## 2. Check the container is running

```bash
docker ps -a --filter name=mysql_apache --format '{{.Names}}\t{{.Status}}'
```

The status should be `Up`. If it's `Restarting` or `Exited`, read `docker logs mysql_apache` before going further.

## 3. Check the variables the container received

```bash
docker exec mysql_apache env | grep -E 'MYSQL|MARIADB'
```

The values must match the compose file exactly. A shortened or changed password points to YAML or Compose parsing problems:

- `$` is interpolated by Compose, so write it as `$$`.
- `#` (space then hash) starts a YAML comment and cuts off the value.

## 4. Check whether initialization ran

```bash
docker logs mysql_apache 2>&1 | head -25
docker logs mysql_apache 2>&1 | grep -iE 'initializ|creating user|creating database'
```

If there's no "Creating user" line, the current container skipped setup and reused existing data.

> [!NOTE] `docker logs` only covers the **current** container. If the container was recreated, the logs from the first initialization are gone.

## 5. Inspect the data directory

```bash
ls -la /var/lib/mysql
```

If files already exist, including a directory named after the database, an earlier run initialized it. Any later changes to the environment variables were ignored.

## 6. List the existing accounts as root

```bash
docker exec -it mysql_apache mariadb -u root -p -e "SELECT user, host FROM mysql.user;"
```

|Result|Cause|
|---|---|
|User missing|Created by an earlier run without valid user variables|
|User present|Password differs from the current file|
|Root denied|Earlier run used a different root password|

Observed in this case (`boris` missing):

```
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| root        | %         |
| healthcheck | 127.0.0.1 |
| healthcheck | ::1       |
| healthcheck | localhost |
| mariadb.sys | localhost |
| root        | localhost |
+-------------+-----------+
```

## 7. Rule out a paste problem

```bash
docker exec -it mysql_apache mariadb -u boris -p'<user_password>' -e 'SELECT 1;'
```

Single quotes keep bash from interpreting characters like `!` or `$`, and passing the password inline avoids hidden characters from pasting at the prompt.

---

## 8a. Resolve by re-initializing (recommended)

```bash
cd <compose file directory>
docker compose down
sudo rm -rf /var/lib/mysql/*
docker compose up -d
```

> [!WARNING] This permanently deletes every database in `/var/lib/mysql`. Only do it when the data is disposable and the directory isn't used by a MariaDB service installed on the host (`systemctl status mariadb`).

The current file's values are applied from scratch, so the running state matches the file.

## 8b. Resolve in place (keeps the data)

```bash
docker exec -it mysql_apache mariadb -u root -p
```

```sql
CREATE USER 'boris'@'%' IDENTIFIED BY '<user_password>';
GRANT ALL PRIVILEGES ON database_apache.* TO 'boris'@'%';

-- if the user exists but the password is wrong:
ALTER USER 'boris'@'%' IDENTIFIED BY '<user_password>';
```

> [!CAUTION] This fix isn't recorded in the compose file. Recreating the setup from the file alone would bring the problem back.

---

## 9. Verify

```bash
# after 8a: both lines should appear
docker logs mysql_apache 2>&1 | grep -iE 'creating user|creating database'

# boris | % should be listed
docker exec -it mysql_apache mariadb -u root -p -e "SELECT user, host FROM mysql.user;"

# database_apache should be listed
docker exec -it mysql_apache mariadb -u boris -p -e 'SHOW DATABASES;'

# web app still reachable
curl http://stapp01:8084/
```

## 10. Prevent it next time

- Database environment variables apply **only on the first start with an empty data directory**.
- After changing any of them, clear the data directory (bind mount) or run `docker compose down -v` (named volume).
- Run `docker compose config` before the first `up`, so a bad variable doesn't get baked in.
- Keep real passwords out of committed files. Use placeholders, or an `.env` file listed in `.gitignore`.

## Quick reference

|Symptom|Likely cause|Fix|
|---|---|---|
|`Access denied` for app user|User never created (data dir pre-initialized)|Re-initialize (8a) or create the user (8b)|
|`Access denied` for root too|Data dir initialized with a different root password|Re-initialize (8a)|
|Password in `env` differs from file|`$` or `#` mangled by Compose/YAML|Escape `$` as `$$`; quote the value|
|Container keeps restarting|Missing root password, or `MARIADB_USER=root`|Fix the variables, then re-initialize|
|No "Creating user" in logs|Initialization skipped|Check data dir (step 5)|
