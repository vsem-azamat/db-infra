# db-infra

Personal database infrastructure with Docker Compose.

## Services

- **PostgreSQL 18** — primary database
- **Databasus** — database backup management service
- **Redis** — shared cache/message store
- **MinIO** — S3-compatible object storage

## Runtime Source Of Truth

This repository contains the canonical `db-infra` Docker Compose definition.

Production credentials are not committed here. Shared infrastructure and
per-service database credentials are stored in:

```bash
/home/poryadok/projects/AGENTS.md/.env
```

The currently running Docker Compose project is `db-infra` with
`/data/compose/16` as its project directory. The canonical Compose file is the
repository copy. Portainer's legacy copy at `/data/compose/16/docker-compose.yml`
must be kept synchronized if Portainer is used to recreate the stack:

```bash
sudo install -m 0644 docker-compose.yml /data/compose/16/docker-compose.yml
sudo install -m 0644 redis-users.acl /data/compose/16/redis-users.acl
```

## Maintenance Updates

This stack is stateful and is not currently compatible with zero-downtime
rolling replacement. The services use fixed `container_name` values, fixed host
ports, and single-writer persistent volumes, so tools such as `docker-rollout`
cannot safely start a second replacement container beside the old one.

Use a low-downtime update flow instead: pull images first, then recreate one
service at a time with `--no-deps`. Existing client connections to the recreated
service will be dropped during the restart.

```bash
docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  pull postgres

docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  up -d --no-deps postgres
```

To update every service after image tags have been reviewed:

```bash
docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  pull

docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  up -d --no-deps postgres

docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  up -d --no-deps redis

docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  up -d --no-deps minio

docker compose --env-file /home/poryadok/projects/AGENTS.md/.env \
  -p db-infra \
  --project-directory /data/compose/16 \
  -f /home/poryadok/projects/db-infra/docker-compose.yml \
  up -d --no-deps databasus
```

## Quick Start

```bash
cp .env.example .env
docker compose up -d
```

## Ports

| Service    | Default Port |
|------------|--------------|
| PostgreSQL | 5432         |
| Databasus  | 4005         |
| Redis      | 6379         |
| MinIO API  | 9000         |
| MinIO UI   | 9001         |
