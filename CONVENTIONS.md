# Localdev conventions for consuming repos

**Status: adopted 2026-10-24.** The app repos come into line one PR each
(NEU-1910 to NEU-1917); until a repo's PR lands, its setup may still differ.

Every app repo that runs on this stack follows the same contract for its
`Taskfile.yml`, `docker-compose.yml` and `.pre-commit-config.yaml`. The point is that
`task build`, `task test` or a commit means the same thing in every repo, and that
an agent asked to "fix the dev setup" in one repo produces the same shape as in the
others.

The contract covers **names and behaviour**, not byte-identical files. App-specific
tasks (ingest jobs, pipelines, `db:refresh` modes, release tooling) stay in their
own repo. Where a core task has to be implemented differently, keep the name and
behaviour and add a comment saying why. upmovies-backend's `deps:lock` in a
throwaway container is an example.

Applies to: sunday-spin-frontend, music-discovery-engine, tvbf-frontend,
tvbf-backend, upcoming-movies-frontend, upcoming-movies-backend,
boone-gifts-frontend, boone-gifts-backend.

## Taskfile

### Header

```yaml
version: '3'

# Traefik, Postgres, Mailpit and the external `proxy` network are the shared
# tbc-localdev-infra stack; `task infra:up` starts it. Optional, so a checkout
# without it (CI) still loads every task that doesn't need it.
includes:
  infra:
    taskfile: ~/projects/tbc-localdev-infra/Taskfile.yml
    dir: ~/projects/tbc-localdev-infra
    optional: true

vars:
  COMPOSE: docker compose
  SVC: <service name, see Compose>
  EXEC: "{{.COMPOSE}} exec {{.SVC}}"
```

- **One include path everywhere.** Use `~/projects/...`, not `../` or `../../`. The
  relative form only differs because of the umbrella folders (tvbingefriend/,
  upcoming-movies/, boone-gifts/). go-task expands `~` and still honours `optional`
  when the path is missing (verified on go-task 3.54).
- **Use one `EXEC`, without `-T`.** Compose detects a non-TTY stdin by itself, so
  `docker compose exec` works from pre-commit with no flag (verified on Compose
  v5.5.0). The `EXEC` / `EXEC_T` split, and the `-T` comments in the current
  hook configs, are leftovers from older Compose versions. If a machine runs a
  Compose version that does need the flag, add `-T` to `EXEC`; that is a one-line
  change.
- Build `EXEC` from `COMPOSE` and `SVC`, so the service name is written only once.

### Core tasks

Every repo has the tasks below with these names and this behaviour, except where a
row says otherwise. Use colon
namespaces (`test:watch`, `deps:add`, `db:init`), never hyphens.

| Task | Behaviour |
|---|---|
| `up` | `task: infra:up`, then `{{.COMPOSE}} up -d`. Does not rebuild. |
| `down` | `{{.COMPOSE}} down`. Leaves the shared infra running. |
| `restart` | `{{.COMPOSE}} restart {{.SVC}}` |
| `build` | Rebuild the dev image, sync dependencies into it, and recreate the container. Run it after any change to the dependencies or the Dockerfile. |
| `logs` | `{{.COMPOSE}} logs -f {{.SVC}}` |
| `shell` | `{{.EXEC}} bash` (`sh` on alpine images) |
| `ps` | `{{.COMPOSE}} ps` |
| `test` | Run the whole suite and pass `{{.CLI_ARGS}}` through, so `task test -- path/to/file` replaces a separate `test-file` task. |
| `lint` | Run the linter (oxlint / `ruff check`). |
| `format` | Run the formatter in write mode. Only in repos that have a formatter (sunday-spin-frontend and boone-gifts-frontend have none today). |
| `format:check` | Run the formatter in check mode. `{{.CLI_ARGS \| default "<project paths>"}}` lets a hook pass staged files. Same condition as `format`. |
| `typecheck` | `tsc -b` (plus typegen where the stack needs it) / `pyright`. |
| `deps:add` | `task deps:add -- <pkg>` |
| `deps:add-dev` | `task deps:add-dev -- <pkg>` |
| `deps:remove` | `task deps:remove -- <pkg>` |

Frontend repos also have:

| Task | Behaviour |
|---|---|
| `build:app` | Production bundle (`vite build`). Never call this `build`. |
| `test:watch` | Watch mode. |
| `install` | Install from the lockfile into the running container (`npm ci` / `pnpm install`). |

Python backends also have:

| Task | Behaviour |
|---|---|
| `deps:lock` | Re-resolve `uv.lock`. |
| `migrate` | `alembic upgrade head` |
| `makemigration` | `task makemigration -- "message"` (autogenerate) |
| `db:init` | Postgres repos only. Creates the app and `_test` databases (and schemas) in `tbc_postgresql_db`, idempotently. |

Optional, only where the suite is split: `test:unit`, `test:integration`, `coverage`.

**Remove** the duplicates: `rebuild` (use `build`), `stop` (use `down`), `test-file`
(use `test -- <path>`), and rename `test-watch`, `migration`, `add`, `remove` and
`lock` to `test:watch`, `makemigration` and `deps:*`. Don't add aliases.

### Why `build` syncs dependencies

The frontend containers mount a named `node_modules` volume over `/app/node_modules`.
Docker fills a named volume from the image **only when the volume is first
created**. After that the volume hides the image's copy. `up -d --build` therefore
rebuilds the image with new dependencies, and the running container still uses the
old ones. Frontend `build` is:

```yaml
build:
  desc: Rebuild the image, sync deps into the node_modules volume, recreate the container
  cmds:
    - "{{.COMPOSE}} build"
    - "{{.COMPOSE}} run --rm --no-deps {{.SVC}} <npm ci | pnpm install --frozen-lockfile>"
    - "{{.COMPOSE}} up -d"
```

Backends have no dependency volume (the venv is in the image), so their `build` is
`{{.COMPOSE}} build` followed by `{{.COMPOSE}} up -d`.

Rejected: `up` = `up -d --build`, which is what sunday-spin, music-discovery-engine
and boone do today. It hides the volume problem rather than fixing it, and it makes
every `up` pay for a build check.

### Tasks that run on the host

Git, git-cliff, gh, ssh and `docker exec` into infra containers run on the host,
because the container has no remote, credentials or Docker socket. Every such task
gets a comment saying why, as boone-gifts-frontend's release tasks and
music-discovery-engine's `db:init` already do.

## Compose

```yaml
services:
  <app>-<frontend|backend>:
    build:
      context: .
      target: dev
    volumes:
      - .:/app
      - node_modules:/app/node_modules   # frontend
      # - /app/.venv                     # backend: hide a host IDE venv
    environment: ...
    # No published port: Traefik is the only route in.
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.<service>.rule=Host(`<host>`)"
      - "traefik.http.routers.<service>.entrypoints=websecure"
      - "traefik.http.routers.<service>.tls=true"
      - "traefik.http.services.<service>.loadbalancer.server.port=<5173|8000>"
    healthcheck: ...
    networks:
      - proxy

volumes:
  node_modules:      # frontend only

networks:
  proxy:
    external: true
```

- **Service name `<app>-frontend` / `<app>-backend`, with `<app>` the short
  product prefix, not the repo name: `tvbf`, `upmovies`, `boone-gifts`,
  `sundayspin`.** Compose registers the service name as
  a DNS alias on every network the service joins, including the shared `proxy`
  network. Two repos with a service called `app` would both answer `http://app:8000`,
  and requests would be split between them. The service name is also the `SVC`
  var. Traefik router names only have to be unique across the stack, so existing
  ones (`boone-gifts-web`, `music-discovery-api`, ...) stay; new services name
  their router after the service.
- **No `container_name`.** It gets in the way of running two checkouts at once
  (worktrees), and nothing but docs refers to the current ones. Use
  `docker compose exec`, never `docker exec <name>`, for app containers. Infra
  containers (`tbc_postgresql_db`, `tbc_mailpit`) keep their fixed names.
- **Join only `proxy`**, plus `traefik.docker.network=proxy` on every service so the
  label is right if a second network is ever added.
- **Mount the whole repo** (`.:/app`), so `uv add`, `uv lock` and `npm install`
  write `pyproject.toml`, the lockfiles and `package.json` back to the host.
- **Always set `target: dev`**, even with a single-stage Dockerfile.
- **Add a healthcheck** against the app's existing route (`/healthz`, `/health`,
  `/startup`). Don't rename routes just to match, because prod monitors hit them.
- **Environment:** infra URLs (`tbc_postgresql_db`, `tbc_mailpit`) are literal in
  `environment`. Secrets interpolate from `.env` with a placeholder fallback
  (`${VAR:-dev-placeholder}`). Whether to also use `env_file` is up to the repo,
  but write the reason in a comment, as upmovies-backend does.

Hostnames (`app.tvbf.localhost`, `boone-gifts.localhost`, ...) stay as they are.
Changing them touches CORS, cookie domains, `.env` files and Vite configs for no
practical gain.

## Pre-commit

```yaml
minimum_pre_commit_version: "3.5.0"

# Every hook calls a Taskfile target, and the target runs inside the dev container
# (tbc-localdev-infra/CONVENTIONS.md). The Taskfile is the only place the toolchain
# is invoked, so the hooks and `task <x>` can't drift apart. The container must be
# up (`task up`) and rebuilt after a dependency change (`task build`).
repos:
  - repo: local
    hooks:
      - id: lint
        name: lint (container)
        entry: task lint
        language: system
        pass_filenames: false
        files: <stack regex>

      - id: format
        name: format --check (container)
        entry: task format:check
        language: system
        pass_filenames: false
        files: <stack regex>

      - id: typecheck
        name: typecheck (container)
        entry: task typecheck
        language: system
        pass_filenames: false
        files: <stack regex>

      - id: test
        name: test (container)
        entry: task test
        language: system
        pass_filenames: false
        files: <stack regex>
```

Stack regex:

- frontend: `\.(ts|tsx|js|mjs|json)$`. Covers `package.json` and `tsconfig*.json`.
- backend: `\.py$|^pyproject\.toml$|^uv\.lock$`

Rules:

- **Hooks check and never fix.** Use `format:check`, not `format`, and no `ruff --fix`.
  You run `task format` yourself before committing. A hook that rewrites files
  leaves the commit missing its own fixes.
- **No `always_run`.** A docs-only commit doesn't run the suite. The `files` regex
  already includes the manifests and lockfiles.
- **Each hook config states its exceptions in a comment.** The ones known today:
  - sunday-spin-frontend, boone-gifts-frontend: no format hook, because neither
    repo has a formatter.
  - tvbf-backend: `entry: task test:unit`, because the integration suite is too
    slow for every commit.
  - boone-gifts-backend: no lint, format or typecheck hooks until ruff and pyright
    are clean on main.
  - upcoming-movies-frontend: `entry: task format:check --` with
    `pass_filenames: true`, so only staged files are checked, because of known
    prettier drift on main. tvbf-frontend does the same if `task format:check`
    fails on its main when its hooks are added.

## Adoption

One PR per repo. Each ticket carries that repo's full checklist (renames, files that
mention the old names, verification), so this table is only the index. Once all
eight have merged, this section can go.

| Repo | Ticket | Notes |
|---|---|---|
| music-discovery-engine | NEU-1910 | Service `app` → `sundayspin-backend`; ships with NEU-1911 |
| sunday-spin-frontend | NEU-1911 | Service `frontend` → `sundayspin-frontend`; proxy target follows NEU-1910 |
| tvbf-backend | NEU-1912 | Whole-repo mount so `deps:*` write through |
| tvbf-frontend | NEU-1913 | First `.pre-commit-config.yaml` |
| upcoming-movies-backend | NEU-1914 | |
| upcoming-movies-frontend | NEU-1915 | |
| boone-gifts-backend | NEU-1916 | Service `api` → `boone-gifts-backend` |
| boone-gifts-frontend | NEU-1917 | Service `web` → `boone-gifts-frontend`; `build` → `build:app` |

## Out of scope

- Release tooling (`changelog`, `release-notes`, `release-branch`). There are four
  different implementations, which is worth a cleanup of its own.
- Hostnames, prod compose files, CI workflows. CI doesn't call `task`. The one
  exception: music-discovery-engine's `perf-nightly.yml` (currently disabled by its
  `PERF_FULL_MIRROR_ENABLED` guard) runs `docker compose exec -T app` against the
  dev compose file, so it has to follow the service rename.
