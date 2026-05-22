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
/home/poryadok/projects/agent/.env
```

The currently running Docker Compose project is labelled as `db-infra` with
`/data/compose/16/docker-compose.yml` as its config file. If that file is
missing, restore it from this repository before recreating the stack:

```bash
sudo install -m 0644 docker-compose.yml /data/compose/16/docker-compose.yml
sudo install -m 0644 redis-users.acl /data/compose/16/redis-users.acl
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
