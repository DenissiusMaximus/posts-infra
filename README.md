# Posts infrastructure

This  folder is the VM deployment bundle. Application containers are pulled from GHCR; the VM does not build them.

## First deployment

Requirements: Docker Engine with the Compose plugin and access to the GHCR packages.

```bash
cp .env.example .env
for file in env/*.env.example; do cp "$file" "${file%.example}"; done
nano .env

docker login ghcr.io

docker compose pull
docker compose up -d

docker compose ps
```

On Windows PowerShell, copy the environment file with:

```powershell
Copy-Item .env.example .env
Get-ChildItem env\*.env.example | ForEach-Object {
    Copy-Item $_ ($_.FullName -replace '\.example$', '')
}
```

The root `.env` contains only Docker Compose settings such as image names,
versions, and host ports. Runtime variables are configured per container in
separate files:

| Container | Environment file |
| --- | --- |
| `posts-auth-db` | `env/auth-db.env` |
| `posts-auth-api` | `env/auth-api.env` |
| `posts-gateway-api` | `env/gateway-api.env` |
| `posts-loki-read` | `env/loki-read.env` |
| `posts-loki-write` | `env/loki-write.env` |
| `posts-loki-backend` | `env/loki-backend.env` |
| `posts-minio` | `env/minio.env` |

Keep the real files out of source control. They are created from the tracked
`env/*.env.example` templates and may contain secrets.

The auth API accepts the `DB__HOST`, `DB__PORT`, `DB__USER`, `DB__PASSWORD`, and
`DB__NAME` names used by its deployment template, as well as the legacy
`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, and `DB_NAME` names used by the
local development compose file. Keep the API database values aligned with
`env/auth-db.env` (the template defaults to database and user `auth`).

## Webhook deployment

The GitHub Actions workflow only sends a POST request to the webhook. The
webhook's deploy script must pull and restart the Compose services explicitly:

```bash
#!/usr/bin/env bash
set -euo pipefail

cd /srv/app/posts-infra
docker compose pull posts-auth-api posts-gateway-api
docker compose up -d posts-auth-api posts-gateway-api
```

The service name is `posts-auth-api`; `posts-auth` does not exist in this
Compose project. After updating the script, run it manually once to verify the
deployment:

```bash
sudo /srv/app/posts-webhook/deploy.sh
sudo docker compose ps
```

## Image list

Edit only these variables in `.env` when a new image or tag is published:

```dotenv
AUTH_API_IMAGE=ghcr.io/your-github-user/mytwitter-auth:latest
GATEWAY_API_IMAGE=ghcr.io/your-github-user/mytwitter-gateway:latest
```

Use immutable tags, such as a release number or Git commit, for predictable VM deployments. After changing a tag:

```bash
docker compose pull posts-auth-api posts-gateway-api
docker compose up -d posts-auth-api posts-gateway-api
```

## Adding a container

1. Add its image to the list in `.env`:

```dotenv
NOTIFICATIONS_API_IMAGE=ghcr.io/grid-mint/notifications/api:latest
```

2. Add a service to `docker-compose.yml`:

```yaml
	posts-notifications-api:
		image: ${NOTIFICATIONS_API_IMAGE}
		container_name: posts-notifications-api
		env_file:
			- ./env/notifications-api.env
		depends_on:
			posts-auth-db:
				condition: service_healthy
		restart: unless-stopped
```

Create `env/notifications-api.env.example` beside the Compose service, copy it
to `env/notifications-api.env`, and put only this container's variables in it.

The service automatically joins the default Compose network and can reach other services by their Compose names, for example `http://posts-auth-api:8080`. If it needs Loki access, add `networks: [loki]` as well.

3. If the service must be public, add a host to `Caddyfile`:

```caddyfile
notifications.lacarte.dev {
		reverse_proxy posts-notifications-api:8080
}
```

4. Pull and start only the new service:

```bash
docker compose config
docker compose pull posts-notifications-api
docker compose up -d posts-notifications-api
docker compose logs -f posts-notifications-api
```

If Watchtower should update it automatically, add `posts-notifications-api` to its `command` list in `docker-compose.yml`. Otherwise update it manually with the commands above.

Watchtower is enabled for these two application services and checks for new images every 60 seconds. It requires the Docker socket and a readable Docker GHCR login configuration. For a manual and predictable update, use the commands above instead.

## GHCR authentication

For public packages, `docker login ghcr.io` is optional. For private packages, create a GitHub token with `read:packages` and run the login command on the VM. Do not commit `.env` or the token.

## Useful commands

```bash
# Show rendered configuration without starting containers
docker compose config

# Follow application logs
docker compose logs -f posts-auth-api posts-gateway-api

# Update all images and restart changed services
docker compose pull
docker compose up -d

# Stop the stack without deleting data
docker compose down
```

The Compose file keeps database, Caddy, and MinIO data in named or local volumes. `docker compose down` does not remove that data; do not use `down -v` unless you intentionally want to delete it.
