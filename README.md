# tbc-localdev-infra

Unified localdev infrastructure — Traefik, MySQL, PostgreSQL, and Mailpit in a single Docker Compose project.

## Prerequisites

- Docker
- [Task](https://taskfile.dev/) (go-task)

## Setup

1. Copy your TLS certificates into `traefik/certs/`:

   ```
   traefik/certs/certificate.pem
   traefik/certs/key.pem
   ```

2. Start all services:

   ```
   task up
   ```

## Services

| Service    | Image                  | Host access                      | Container access (proxy network) |
|------------|------------------------|----------------------------------|----------------------------------|
| Traefik    | traefik:v3.6           | https://localhost (reverse proxy) |                                  |
| MySQL      | mysql:9.2              | localhost:3306                    | mysql_db:3306                    |
| PostgreSQL | postgres:17            | localhost:5432                    | postgresql_db:5432               |
| Mailpit    | axllent/mailpit:latest | https://mailpit.localhost         | mailpit:1025 (SMTP)              |

### Default credentials

| Service    | User | Password |
|------------|------|----------|
| MySQL      | root | root     |
| PostgreSQL | root | root     |

## Commands

```
task up        # create proxy network + start all services
task down      # stop all services
task restart   # down then up
task logs      # follow all logs
task status    # docker compose ps
```

## Using from other projects

Add this to your project's `Taskfile.yml`:

```yaml
includes:
  infra:
    taskfile: ~/projects/tbc-localdev-infra/Taskfile.yml
    dir: ~/projects/tbc-localdev-infra
```

Then run `task infra:up`, `task infra:down`, etc.
