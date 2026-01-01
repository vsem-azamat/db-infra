# db-infra

Personal database infrastructure with Docker Compose.

## Services

- **PostgreSQL 18** — primary database
- **Databasus** — database backup management service

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
