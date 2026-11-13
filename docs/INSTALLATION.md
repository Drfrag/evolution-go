# Docker Installation Guide

This guide installs the Drfrag build of Evolution Go using its prebuilt GitHub Container Registry (GHCR) image and PostgreSQL. It is intended for users who want to run the application, not compile the source code.

## 1. Requirements

- Docker Engine 24 or newer
- Docker Compose plugin (Compose v2)
- At least 1 GB RAM available to the application and database
- A host port available for the API (default: 8080)

No Go installation or local Docker build is required.

## 2. Create the deployment folder

```bash
mkdir -p evolution-go
cd evolution-go
```

Create a file named `compose.yaml` with the following content:

```yaml
services:
  evolution-go:
    image: ghcr.io/drfrag/evolution-go:0.7.3
    container_name: evolution-go
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - evolution_data:/app/dbdata

  postgres:
    image: postgres:15-alpine
    container_name: evolution-go-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  evolution_data:
  postgres_data:
```

The service name `postgres` is the hostname used by the API container to reach the database. PostgreSQL is not published on a host port in this example; only the API port is exposed.

## 3. Create the environment file

Generate two different random secrets:

```bash
openssl rand -hex 32
openssl rand -hex 24
```

Create a file named `.env` in the same folder:

```env
SERVER_PORT=8080
CLIENT_NAME=evolution
GLOBAL_API_KEY=PASTE_THE_FIRST_RANDOM_SECRET_HERE
DATABASE_SAVE_MESSAGES=false
DEBUG_ENABLED=INFO
LOG_TYPE=console
CONNECT_ON_STARTUP=false
WEBHOOK_FILES=true
OS_NAME=Linux

POSTGRES_USER=evolution
POSTGRES_PASSWORD=PASTE_THE_SECOND_RANDOM_SECRET_HERE

POSTGRES_AUTH_DB=postgresql://evolution:PASTE_THE_SECOND_RANDOM_SECRET_HERE@postgres:5432/evogo_auth?sslmode=disable
POSTGRES_USERS_DB=postgresql://evolution:PASTE_THE_SECOND_RANDOM_SECRET_HERE@postgres:5432/evogo_users?sslmode=disable
```

Replace both placeholders consistently: the password in the two database URLs must exactly match `POSTGRES_PASSWORD`. The example uses hexadecimal secrets so they are safe to place in a PostgreSQL URL without URL-encoding special characters.

Protect the file:

```bash
chmod 600 .env
```

**Never publish or commit this file.** It contains the API key and database credentials.

## 4. Start Evolution Go

From the folder containing `compose.yaml` and `.env`:

```bash
docker compose pull
docker compose up -d
```

Check the startup logs:

```bash
docker compose logs -f evolution-go
```

The application connects to PostgreSQL and creates `evogo_auth` and `evogo_users` if they do not exist. It also runs its application migrations at startup. You do not need to run a separate SQL initialization script for this Compose setup.

## 5. Open the API and activate the license

- API: [http://localhost:8080](http://localhost:8080)
- Swagger: [http://localhost:8080/swagger/index.html](http://localhost:8080/swagger/index.html)
- Manager: [http://localhost:8080/manager/login](http://localhost:8080/manager/login)

Use the API URL and the value of `GLOBAL_API_KEY` when configuring the Manager. Complete the license registration/activation flow shown by the application. If the server is hosted remotely, replace `localhost` with the server's reachable hostname or IP address and configure HTTPS/reverse proxy as appropriate.

## 6. Check status and logs

```bash
docker compose ps
docker compose logs -f evolution-go
docker compose logs -f postgres
curl -i http://localhost:8080/server/ok
```

## 7. Updating Evolution Go

The Compose example pins version `0.7.3` so an update does not happen unexpectedly. To move to a newer published version:

1. Back up the PostgreSQL data.
2. Change the image tag in `compose.yaml` to the desired published version.
3. Pull and recreate only the application service:

```bash
docker compose pull evolution-go
docker compose up -d evolution-go
docker compose logs -f evolution-go
```

The `latest` tag is also published, but pinning a version is recommended for predictable production deployments.

## 8. Backups and persistence

The PostgreSQL data is stored in the named Docker volume `postgres_data`. The Compose file also declares `evolution_data` for the application's local data directory.

For a logical PostgreSQL backup:

```bash
docker compose exec -T postgres sh -c 'pg_dumpall -U "$POSTGRES_USER"' > evolution-go-postgres-backup.sql
```

Store the backup somewhere safe and verify that it is usable before relying on it.

**Do not use `docker compose down -v` for routine maintenance or updates.** The `-v` option deletes the named volumes and can permanently remove the database contents.

## 9. Using an existing PostgreSQL server

You can use an existing PostgreSQL instance instead of the PostgreSQL service in the example:

1. Create `evogo_auth` and `evogo_users`, or allow the configured database user to create them.
2. Ensure the account used in the connection URLs has permission to connect and create databases when automatic creation is needed.
3. Set `POSTGRES_AUTH_DB` and `POSTGRES_USERS_DB` to connection URLs reachable **from the Evolution Go container**.
4. Remove the `postgres` service and its `depends_on` entry from `compose.yaml`.

Inside a container, `localhost` refers to that same container, not the Docker host or another database server. Use the database server's DNS name, LAN address, or a Docker network alias.

Example URL:

```text
postgresql://DB_USER:DB_PASSWORD@DB_HOST:5432/evogo_users?sslmode=disable
```

If a password contains reserved URL characters (such as `@`, `/`, `:` or `#`), URL-encode the password before placing it in the connection URL.

## 10. Troubleshooting

### Database creation or permission error

The application attempts to create each configured database when it is missing. The PostgreSQL user must have `CREATEDB` permission. Check that the username/password are correct and that the database server is reachable from the API container.

### Connection refused or timeout

Check that the PostgreSQL container is healthy and that the database hostname in both connection URLs is `postgres` when using the Compose example. Do not use `localhost` for the database host from inside the API container.

### API port already in use

Change the host side of the port mapping, for example `8081:8080`, then open the API on port 8081. Keep `SERVER_PORT=8080` because that is the port inside the container.

### API returns an authorization error

Confirm that the client is sending the same API key configured in `GLOBAL_API_KEY`. Do not expose the key in screenshots, public logs, or support posts.

### View the complete error

```bash
docker compose logs --tail=200 evolution-go
docker compose logs --tail=200 postgres
```

---

For local development and source builds, see the [repository README](../README.md) and the project Makefile.
