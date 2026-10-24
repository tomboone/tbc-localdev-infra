# tbc-localdev-infra

Unified localdev infrastructure — Traefik, MySQL, PostgreSQL, and Mailpit in a single Docker Compose project.

## Project structure

- `docker-compose.yml` — all four services
- `traefik/dynamic.yml` — TLS certificate configuration for Traefik
- `traefik/certs/` — TLS certificate and key (gitignored)
- `Taskfile.yml` — orchestration tasks (up, down, restart, logs, status)
- `CONVENTIONS.md` — the dev-setup contract every consuming app repo follows

## Key details

- **Traefik** reverse proxy for HTTP (:80) and HTTPS (:443)
- **MySQL** (mysql:9.2) — direct port publishing (:3306), volume `mysql_mysql-data` (created by Compose)
- **PostgreSQL** (postgres:17) — direct port publishing (:5432), volume `postgresql_postgres-data` (created by Compose), data at `/var/lib/postgresql` (non-standard mount, do not change)
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
    optional: true
```

The rest of a consuming repo's dev setup (Taskfile core tasks, compose service
rules, pre-commit hooks) is defined in [CONVENTIONS.md](../CONVENTIONS.md). When the contract changes,
change it there first, then bring the app repos in line.
