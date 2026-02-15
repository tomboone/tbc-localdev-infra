# tbc-localdev-infra

Unified localdev infrastructure — Traefik, MySQL, PostgreSQL, and Mailpit in a single Docker Compose project.

## Project structure

- `docker-compose.yml` — all four services
- `traefik/dynamic.yml` — TLS certificate configuration for Traefik
- `traefik/certs/` — TLS certificate and key (gitignored)
- `Taskfile.yml` — orchestration tasks (up, down, restart, logs, status)

## Key details

- **Traefik** reverse proxy with TCP entrypoints for MySQL (:3306) and PostgreSQL (:5432), plus HTTP (:80) and HTTPS (:443)
- **MySQL** (mysql:9.2) — routed through Traefik TCP, volume `mysql_mysql-data` (external)
- **PostgreSQL** (postgres:17) — routed through Traefik TCP, volume `postgresql_postgres-data` (external), data at `/var/lib/postgresql` (non-standard mount, do not change)
- **Mailpit** — HTTP only via Traefik at `https://mailpit.localhost`
- **Network**: `proxy` (external, shared with other localdev projects)

## Commands

Use `task` (go-task) to manage services:

```
task up       # create proxy network + start all services
task down     # stop all services
task restart  # down then up
task logs     # follow all logs
task status   # docker compose ps
```

Other projects include this Taskfile under the `infra:` namespace via:

```yaml
includes:
  infra:
    taskfile: ~/projects/tbc-localdev-infra/Taskfile.yml
    dir: ~/projects/tbc-localdev-infra
```
