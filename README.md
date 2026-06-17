# db-infra

Personal shared infrastructure with Docker Compose.

## Services

- **PostgreSQL 18** - primary database
- **Databasus** - database backup management service exposed through host-level Caddy at `databasus.azamat.io`
- **Redis** - shared cache/message store
- **MinIO** - S3-compatible object storage
- **Garage** - S3-compatible object storage candidate for migration testing
- **Prometheus** - metrics storage and query UI exposed through host-level Caddy at `prometheus.azamat.io`
- **Grafana** - metrics and logs dashboard UI exposed through host-level Caddy at `grafana.azamat.io`
- **Loki** - log storage for Docker container logs
- **Alloy** - Docker log collector that forwards logs to Loki

## Runtime Source Of Truth

This repository is the runtime source of truth. Run Docker Compose from the repo
root so all relative bind mounts resolve to files in this checkout:

```bash
cd /home/poryadok/projects/db-infra
docker compose --env-file .env up -d
```

The Compose project name is fixed in `docker-compose.yml` as `db-infra`, so the
commands do not need `-p db-infra`. Do not use `/data/compose/16` as the Compose
project directory for new changes; that old Portainer path can leave duplicated
configs and secrets outside the repo.

Production credentials are not committed here. Shared infrastructure runtime
credentials live in ignored files in this checkout:

```bash
.env
prometheus/secrets/*
```

Application and per-service credentials are stored outside this git repository
under `/home/poryadok/projects/AGENTS.md/credentials/`, one file per
project/environment:

```bash
/home/poryadok/projects/AGENTS.md/credentials/circuit.dev.env
/home/poryadok/projects/AGENTS.md/credentials/circuit.prod.env
/home/poryadok/projects/AGENTS.md/credentials/moderator.prod.env
```

Keep `.env` scoped to values needed by `docker compose --env-file .env ...`.

## Maintenance Updates

This stack is stateful and is not currently compatible with zero-downtime
rolling replacement. The services use fixed `container_name` values, fixed host
ports, and single-writer persistent volumes, so tools such as `docker-rollout`
cannot safely start a second replacement container beside the old one.

Use a low-downtime update flow instead: pull images first, then recreate one
service at a time with `--no-deps`. Existing client connections to the recreated
service will be dropped during the restart.

```bash
cd /home/poryadok/projects/db-infra

docker compose --env-file .env pull postgres
docker compose --env-file .env up -d --no-deps postgres
```

To update every service after image tags have been reviewed:

```bash
cd /home/poryadok/projects/db-infra

docker compose --env-file .env pull

docker compose --env-file .env up -d --no-deps postgres
docker compose --env-file .env up -d --no-deps redis
docker compose --env-file .env up -d --no-deps minio
docker compose --env-file .env up -d --no-deps garage
docker compose --env-file .env up -d --no-deps databasus
docker compose --env-file .env up -d --no-deps prometheus
docker compose --env-file .env up -d --no-deps loki
docker compose --env-file .env up -d --no-deps alloy
docker compose --env-file .env up -d --no-deps grafana
```

## Quick Start

```bash
cp .env.example .env
docker compose --env-file .env up -d
```

## Ports

| Service    | Default Port |
|------------|--------------|
| PostgreSQL | 5432         |
| Databasus  | `databasus.azamat.io` via Caddy |
| Redis      | 6379         |
| MinIO API  | 9000         |
| MinIO UI   | 9001         |
| Garage S3  | `garage.azamat.io` via Caddy; local 3900 |
| Garage Admin | 3903 on `127.0.0.1` |
| Prometheus | `prometheus.azamat.io` via Caddy |
| Grafana    | `grafana.azamat.io` via Caddy |
| Loki       | 3100 on `127.0.0.1` |
| Alloy      | 12345 on `127.0.0.1` |

Databasus binds its host port only on `127.0.0.1`, and the host-level Caddy
container runs in host network mode so it can proxy HTTPS traffic to
`127.0.0.1:4005`.

Prometheus and Grafana also bind their host ports only on `127.0.0.1`.
Prometheus is protected by Caddy Basic Auth. Grafana uses its own login screen
with admin credentials from `.env`; anonymous access is explicitly disabled.

Garage is added side-by-side with MinIO for migration testing. The current
configuration is a single-node deployment with `replication_factor = 1`, so it
does not provide object redundancy by itself. Use it for compatibility testing
until a multi-node layout is planned. The public S3 endpoint is intended to be
`https://garage.azamat.io` through the host-level Caddy reverse proxy, using
path-style S3 requests.

## Service Metrics

Prometheus scrapes application metrics through public HTTPS endpoints protected
with bearer tokens. Keep scrape tokens out of git by storing them under
`prometheus/secrets/` and mounting that directory read-only into Prometheus.

CircuitNinja has separate scrape jobs and token files for each environment:

```bash
prometheus/secrets/circuitninja-dev-api-token
prometheus/secrets/circuitninja-prod-api-token
```

Each token value must match the corresponding `web-services` deployment
`OBSERVABILITY_METRICS_SECRET`. Queue validation gauges should stay disabled for
the first rollout; omit `OBSERVABILITY_VALIDATION_QUEUE_METRICS_ENABLED` or set
it to `false` until the base scrape is verified.

Use Prometheus labels for project/environment grouping instead of duplicating
Prometheus instances:

```promql
up{project="circuitninja", env="dev"}
up{project="circuitninja", env="prod"}
```

Grafana keeps project-level dashboards in folders. The first folder is
`CircuitNinja`.

## Service Logs

Loki stores Docker container logs. Alloy reads the Docker socket and forwards
logs to Loki. Grafana provisions the `Loki` datasource beside `Prometheus`. Alloy
keeps only containers whose Compose project matches `db-infra`, `circuitninja.*`,
or `web-services.*`; this avoids ingesting unrelated host logs by default.

Use low-cardinality Loki labels such as `container`, `service`,
`compose_project`, and `level`. Do not promote `request_id`, `order_id`, or
`validation_id` to labels; query them from JSON log fields instead.

Examples:

```logql
{compose_project="db-infra"}
{container=~".*gewen.*"} |= "error"
{container=~".*api.*"} | json req_id="req.id" | req_id="<request-id>"
```
