# Docker Compose Cheatsheet

A comprehensive reference for Docker Compose v2: the Compose Specification, service configuration, profiles, Watch mode, CLI commands, and production-ready stacks.

## 🔗 Navigation

- **[← Back to Main Docker Cheatsheet](README.md)** — essential Docker commands
- **[Dockerfile Cheatsheet](dockerfile_cheatsheet.md)** — image build instructions and best practices

## Table of Contents

- [Introduction](#introduction)
- [File Structure & Top-Level Keys](#file-structure--top-level-keys)
- [Service Configuration](#service-configuration)
  - [image](#image)
  - [build](#build)
  - [ports](#ports)
  - [volumes](#volumes)
  - [environment & env_file](#environment--env_file)
  - [command & entrypoint](#command--entrypoint)
  - [depends_on](#depends_on)
  - [healthcheck](#healthcheck)
  - [restart](#restart)
  - [deploy & resources](#deploy--resources)
  - [networks](#networks)
  - [secrets & configs](#secrets--configs)
  - [logging](#logging)
  - [security & hardening](#security--hardening)
  - [Other useful keys](#other-useful-keys)
  - [profiles](#profiles)
  - [develop (Watch mode)](#develop-watch-mode)
- [Variables & Interpolation](#variables--interpolation)
- [Multiple Files & Overrides](#multiple-files--overrides)
- [YAML Anchors & Extensions](#yaml-anchors--extensions)
- [CLI Commands](#cli-commands)
- [Common Patterns](#common-patterns)
- [Real-world Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [What's New (2024–2026)](#whats-new-20242026)
- [References](#references)

## Introduction

Docker Compose defines and runs multi-container applications from a single YAML file (`compose.yaml` / `docker-compose.yml`). It creates the networks, volumes, and containers described in the file, in dependency order.

> **v1 vs v2:** `docker-compose` (Python, v1) is end-of-life (July 2023). Use the Go plugin `docker compose` — same file format, better performance, active development. All examples here use v2.

> **The `version:` key is obsolete.** It was required by v1 (e.g. `version: "3.8"`). Compose v2 follows the [Compose Specification](https://docs.docker.com/reference/compose-file/) and ignores it — don't add it.

### Minimal example

```yaml
services:
  web:
    image: nginx:1.28-alpine
    ports:
      - "8080:80"
```

```bash
docker compose up -d
```

## File Structure & Top-Level Keys

```yaml
name: myproject          # optional project name (defaults to the directory name)

services:                # required: the containers
  web:
    image: nginx:1.28-alpine

networks:                # optional: custom networks
  frontend:

volumes:                 # optional: named volumes
  data:

secrets:                 # optional: sensitive values
  db_password:
    file: ./secrets/db_password.txt

configs:                 # optional: non-sensitive configuration files
  app_config:
    file: ./config/app.conf

include:                 # optional: compose other files (v2.20+)
  - ./shared/monitoring.yml

x-shared: &shared        # optional: extension fields for YAML anchors
  restart: unless-stopped
```

| Key | Purpose |
|-----|---------|
| `name` | Project name; prefixes containers, networks, volumes (`myproject-web-1`) |
| `services` | Container definitions (the only required key) |
| `networks` | Named networks; a default network is created if omitted |
| `volumes` | Named volumes referenced by services |
| `secrets` | Files or env vars mounted read-only at `/run/secrets/<name>` |
| `configs` | Config files mounted read-only at `/<name>` |
| `include` | Compose other files into this project |
| `x-*` | Extension fields, typically used with YAML anchors |

Project name precedence: `-p` flag → `COMPOSE_PROJECT_NAME` env var → top-level `name` → directory name.

## Service Configuration

### image

```yaml
services:
  db:
    image: postgres:18-alpine
    # or a specific digest for full reproducibility
    # image: postgres@sha256:<digest>
```

- If both `image` and `build` are set, `image` names the built result (and is pulled if the build is skipped).
- Pin versions in production; `latest` is a moving target.

### build

```yaml
services:
  api:
    build:
      context: ./api                # build context directory
      dockerfile: Dockerfile.prod   # alternative Dockerfile
      target: runtime               # stop at a multi-stage target
      args:                         # build args
        NODE_ENV: production
      cache_from:
        - type=registry,ref=org/api:buildcache
      cache_to:
        - type=registry,ref=org/api:buildcache,mode=max
      secrets:
        - npm_token                 # BuildKit secrets
      ssh:
        - default
      platforms:
        - linux/amd64
        - linux/arm64
      labels:
        com.example.version: "1.0"
      no_cache: false
      pull: true                    # always refresh base images

    # shorthand forms
    # build: ./api
    # build: .
```

Declare build secrets at the top level:

```yaml
secrets:
  npm_token:
    file: $HOME/.npmrc
```

### ports

```yaml
services:
  web:
    ports:
      - "3000:3000"                # host:container (quotes required in YAML!)
      - "8080:80"
      - "127.0.0.1:5432:5432"      # bind to a specific host interface
      - "8000-8010:8000-8010"      # range
      - "6060:6060/udp"
      - "3000"                     # random host port (only container port given)

      # long syntax
      - target: 80
        published: "8080"
        protocol: tcp
        mode: host                 # host | ingress (ingress is Swarm)
```

- **Always quote** `"8080:80"` — YAML would otherwise interpret `80:80` as a sexagesimal number.
- Compose creates the container port based on `EXPOSE`/`target`; publishing is what matters.
- Use `expose:` to make a port reachable only to linked/same-network containers.

### volumes

```yaml
services:
  web:
    volumes:
      - ./app:/app                            # bind mount (host path)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # read-only
      - data:/var/lib/data                    # named volume
      - /var/lib/mysql                        # anonymous volume (not recommended)
      - ~/logs:/logs:cached                   # Desktop-only consistency hint

      # long syntax
      - type: volume
        source: data
        target: /var/lib/data
        read_only: true
      - type: bind
        source: ./config
        target: /etc/app
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 67108864                       # 64 MiB

volumes:
  data:
```

- Named volumes are managed by Docker and survive `docker compose down` (removed with `down -v`).
- Anonymous volume trick for `node_modules`: `- /app/node_modules` hides the host's copy.

Top-level volume options:

```yaml
volumes:
  data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/data
      o: bind
  shared:
    external: true        # already exists, created outside Compose
    name: actual-volume-name
```

### environment & env_file

```yaml
services:
  app:
    environment:
      NODE_ENV: production            # map form (clearer)
      API_URL: http://api:5000
    # or list form
    # environment:
    #   - NODE_ENV=production
    #   - API_URL=http://api:5000
    # pass through from the host shell (no value = take host value)
    #   - GITHUB_TOKEN
    env_file:
      - .env
      - path: .env.local
        required: false               # Compose 2.24+
```

Precedence inside the container: `environment` values override `env_file` values. References like `$VAR` in `environment` are interpolated from the shell/`.env` **before** the container starts.

### command & entrypoint

```yaml
services:
  app:
    image: myapp:1.0
    command: ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]   # overrides CMD
    entrypoint: ["/entrypoint.sh"]                               # overrides ENTRYPOINT
    working_dir: /app
```

- To keep the image's entrypoint and just add args, set only `command`.
- `command:` as a string runs via the shell (`/bin/sh -c`); YAML list = exec form.

### depends_on

Short syntax only controls **start order**, not readiness.

```yaml
services:
  api:
    depends_on:
      - db
      - redis
```

Long syntax waits for conditions:

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy               # wait for HEALTHCHECK to pass
        restart: true                            # restart api when db is updated (2.17+)
      migrate:
        condition: service_completed_successfully # one-shot jobs
      redis:
        condition: service_started
        required: false                           # don't fail if redis is absent (2.20+)
```

| Condition | Meaning |
|-----------|---------|
| `service_started` | Container is running (default) |
| `service_healthy` | Health check reports healthy |
| `service_completed_successfully` | Container exited with code 0 |

### healthcheck

```yaml
services:
  web:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]   # exec form
      # test: ["CMD-SHELL", "curl -fsS http://localhost:3000/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
      start_interval: 5s        # interval during start_period
      # disable an inherited health check:
      # test: ["NONE"]
```

`CMD` = exec form, `CMD-SHELL` = run through the container shell. Keep checks small and dependency-free.

### restart

```yaml
services:
  api:
    restart: unless-stopped
```

| Value | Behavior |
|-------|----------|
| `no` | Never restart (default) |
| `always` | Always restart, including after daemon restart |
| `on-failure[:max]` | Restart only on non-zero exit (optionally capped) |
| `unless-stopped` | Like `always`, but stays stopped if manually stopped |

### deploy & resources

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
      # Swarm-only keys (ignored by docker compose up):
      # replicas: 3
      # restart_policy:
      #   condition: on-failure
      #placement:
      #   constraints: [node.role == worker]
```

- `docker compose up` applies `deploy.resources.limits` to local containers on current Compose versions.
- For local scaling use `docker compose up --scale api=3` (or `deploy.replicas`, honored by Compose on recent versions — `--scale` is the safe, portable choice). Remove `container_name` before scaling.
- On Swarm, `deploy` is the native configuration format.

### networks

```yaml
services:
  web:
    networks:
      frontend:
        aliases:
          - www                   # extra DNS names
      backend:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true                   # no external connectivity
  external_net:
    external: true
    name: existing-network-name
```

- A default network is created per project; service names are DNS names on it.
- Use separate networks (and `internal: true`) to isolate tiers, e.g. keep the database off the frontend network.

### secrets & configs

`secrets` are for sensitive data, mounted at `/run/secrets/<name>`:

```yaml
services:
  db:
    image: postgres:18-alpine
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  # or from the environment of the Compose process (recent Compose versions)
  # session_key:
  #   environment: SESSION_KEY
  # external secrets (Swarm)
  # global_secret:
  #   external: true
```

`configs` are for non-sensitive files, mounted at `/<config_name>`:

```yaml
services:
  web:
    configs:
      - source: app_config
        target: /etc/app/app.conf
        mode: 0440

configs:
  app_config:
    file: ./config/app.conf
  # inline content (Compose 2.23+)
  # nginx_config:
  #   content: |
  #     server { listen 80; }
```

> In non-Swarm mode, `secrets`/`configs` support is provided by Compose itself; `configs` requires a recent Compose release. `secrets` never appear in `docker inspect` output as plain values.

### logging

```yaml
services:
  web:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    # driver: local      # efficient rotating default
    # driver: none       # disable
```

Always cap log size in production, or a chatty container can fill the disk.

### security & hardening

```yaml
services:
  web:
    image: nginx:1.28-alpine
    user: "1000:1000"                 # run as non-root
    read_only: true                   # immutable root filesystem
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE              # only what the process needs
    security_opt:
      - no-new-privileges:true
    init: true                        # tini as PID 1 (zombie reaping)
    pids_limit: 200
    ulimits:
      nofile:
        soft: 1024
        hard: 2048
```

### Other useful keys

```yaml
services:
  app:
    hostname: app
    extra_hosts:
      - "host.docker.internal:host-gateway"   # reach the host from Linux containers
    dns:
      - 8.8.8.8
    platform: linux/amd64                     # force architecture (Apple Silicon)
    pull_policy: missing                      # always | never | missing | build | daily | weekly
    stop_signal: SIGQUIT
    stop_grace_period: 30s
    tty: true
    stdin_open: true
    shm_size: "256mb"                         # /dev/shm size (browsers, Postgres)
    ipc: host
    pid: host
    sysctls:
      net.core.somaxconn: 1024
    group_add:
      - "1001"
    labels:
      com.example.tier: frontend
    profiles:
      - debug
```

| Key | Notes |
|-----|-------|
| `platform` | Use when an image has no arm64 variant |
| `pull_policy` | `build` rebuilds buildable images; `always` forces a pull |
| `stop_grace_period` | Time between `SIGTERM` and `SIGKILL` on stop |
| `shm_size` | Increase for Chrome/puppeteer, Postgres parallel workers |
| `extra_hosts` | `host-gateway` maps the host on Linux |
| `ipc` / `pid` | Share namespaces with the host — use knowingly |

### profiles

Services with profiles start **only** when the profile is enabled. Services without a profile always start. Docs: [Profiles](https://docs.docker.com/compose/how-tos/profiles/).

```yaml
services:
  app:
    image: myapp:1.0

  adminer:
    image: adminer:5
    profiles: ["tools"]

  debug:
    image: nicolaka/netshoot
    command: sleep infinity
    profiles: ["debug", "tools"]
```

```bash
docker compose up -d                        # app only
docker compose --profile tools up -d        # app + adminer + debug
docker compose --profile "*" up -d          # everything
COMPOSE_PROFILES=tools docker compose up -d
docker compose config --profiles
```

### develop (Watch mode)

Replaces bind-mount-based dev loops with automatic sync/rebuild. Docs: [Compose Watch](https://docs.docker.com/compose/how-tos/watch/).

```yaml
services:
  web:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
        # action: sync+restart | restart
```

```bash
docker compose watch              # start watching (also starts services)
docker compose up --watch         # same, as a flag
```

Actions: `sync` (copy changes into the container), `sync+restart`, `rebuild` (rebuild image + recreate), `restart`.

## Variables & Interpolation

Compose interpolates `${...}` in the YAML **before** creating resources. Docs: [Environment variables](https://docs.docker.com/compose/how-tos/environment-variables/).

```yaml
services:
  web:
    image: myapp:${APP_VERSION:-latest}
    ports:
      - "${PORT:?PORT must be set}:80"
    environment:
      LITERAL: $$not-interpolated      # $$ → literal $
```

| Syntax | Meaning |
|--------|---------|
| `${VAR}` | Value of `VAR` (empty if unset) |
| `${VAR:-default}` | Default when unset or empty |
| `${VAR-default}` | Default only when unset |
| `${VAR:?error}` | Fail with `error` if unset or empty |
| `${VAR:+replacement}` | `replacement` when set |
| `$$` | Escaped `$` |

Sources and precedence for interpolation:

1. Shell environment (`NODE_ENV=prod docker compose up`)
2. `--env-file <file>` if given, otherwise `.env` in the project directory
3. Unset → empty/default

`.env` example (commit a `.env.example`, never the real `.env`):

```bash
COMPOSE_PROJECT_NAME=myapp
APP_VERSION=1.0.0
PORT=8080
POSTGRES_PASSWORD=change-me
```

Useful Compose environment variables:

| Variable | Effect |
|----------|--------|
| `COMPOSE_FILE` | Default file list (`compose.yml:compose.prod.yml`; `;` on Windows) |
| `COMPOSE_PROFILES` | Enabled profiles |
| `COMPOSE_PROJECT_NAME` | Project name |
| `COMPOSE_PATH_SEPARATOR` | Custom separator for `COMPOSE_FILE` |
| `DOCKER_DEFAULT_PLATFORM` | Default `platform` for images without one |

## Multiple Files & Overrides

Compose merges files left to right. By default it also loads `compose.override.yml` automatically. Docs: [Merge](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/).

```bash
docker compose -f compose.yml -f compose.prod.yml up -d
COMPOSE_FILE=compose.yml:compose.prod.yml docker compose up -d
docker compose config                       # inspect the merged result
```

Merge rules:

- Mappings (`environment`, `labels`) merge by key.
- Sequences (`ports`, `volumes`, `command` args) are **appended**.
- Scalars (`image`, `command`) are replaced by the later file.
- Use `!reset` to remove a value and `!override` to replace a whole sequence (Compose 2.24+):

```yaml
# compose.override.yml
services:
  web:
    ports: !override
      - "127.0.0.1:8080:80"     # replaces base ports instead of appending
    environment:
      DEBUG: !reset null        # removes DEBUG from the merged environment
```

Typical layout:

```
compose.yml              # base (shared)
compose.override.yml     # local dev defaults (auto-loaded, gitignored or committed)
compose.prod.yml         # production deltas
compose.test.yml         # CI overrides
```

## YAML Anchors & Extensions

Compose supports standard YAML anchors and `x-` extension fields.

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

x-healthcheck: &default-healthcheck
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 15s

services:
  api:
    image: myapi:1.0
    logging: *default-logging
    healthcheck:
      <<: *default-healthcheck
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8000/health"]

  worker:
    image: myworker:1.0
    logging: *default-logging
```

Merge key `<<` merges the anchor's mapping into the current one; explicit keys win.

## CLI Commands

### Lifecycle

```bash
docker compose up -d                          # create + start (pulls if needed)
docker compose up -d --build                  # rebuild images first
docker compose up web db                      # specific services (+ their deps)
docker compose up --wait                      # block until healthy/running
docker compose up --scale web=3 worker=2
docker compose up --force-recreate --remove-orphans
docker compose up --abort-on-container-exit --exit-code-from web   # CI pipelines
docker compose watch                          # Watch mode

docker compose down                           # stop + remove containers/networks
docker compose down -v                        # also remove named volumes
docker compose down --rmi local               # also remove local images
docker compose down --remove-orphans --timeout 5

docker compose start | stop | restart [service]
docker compose kill | pause | unpause [service]
docker compose rm -sf [service]               # remove stopped service containers
```

### Build & Registry

```bash
docker compose build [service]                # build images
docker compose build --no-cache --pull --progress plain
docker compose build --with-dependencies web
docker compose pull [--ignore-buildable]      # pull images
docker compose push                           # push built images
docker compose create                         # create without starting
docker compose images                         # images used by services
```

### Inspect & Debug

```bash
docker compose ps [-a] [--format json]
docker compose logs [-f] [--tail 100] [--since 10m] [--no-log-prefix] [service]
docker compose top [service]
docker compose stats [service]
docker compose port web 80                    # resolved host port mapping
docker compose cp web:/etc/nginx/nginx.conf ./nginx.conf
docker compose exec web sh                    # exec in a RUNNING container
docker compose exec -T -u root web sh         # no TTY, root user
docker compose run --rm web npm test          # one-off in a NEW container
docker compose run --rm --service-ports web   # publish ports for the one-off
docker compose run --rm --no-deps --entrypoint sh web
docker compose wait web                       # wait for a container to exit
docker compose attach web                     # attach to the main process
```

`exec` vs `run`: `exec` targets a running service container (does not create one); `run` creates a new container with the service config, runs the command, and (with `--rm`) removes it. `run` does not publish ports unless `--service-ports` is passed.

### Configuration

```bash
docker compose config                          # validate + print resolved config
docker compose config -q                       # validate only
docker compose config --services
docker compose config --profiles
docker compose config --images
docker compose config --volumes
docker compose config --format json
docker compose config --hash="*"               # config hash per service (CI cache keys)

docker compose ls [-a]                         # list Compose projects
docker compose version
```

Global flags:

```bash
docker compose -f compose.prod.yml -p myapp --env-file .env.prod --profile tools up -d
```

## Common Patterns

### Web Stack with Readiness Gating

```yaml
services:
  web:
    image: nginx:1.28-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_healthy

  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://app:secret@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s

  db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:8-alpine
    volumes:
      - cache_data:/data

volumes:
  db_data:
  cache_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Development with Override + Watch

`compose.yml`:

```yaml
services:
  app:
    build: .
    environment:
      NODE_ENV: production
    ports:
      - "3000:3000"
```

`compose.override.yml` (auto-loaded by default):

```yaml
services:
  app:
    environment:
      NODE_ENV: development
    command: npm run dev
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: package.json
```

```bash
docker compose up --watch
```

### Admin Tools Behind a Profile

```yaml
services:
  auth:
    image: postgres:18-alpine
    environment:
      POSTGRES_PASSWORD: dev

  pgadmin:
    image: dpage/pgadmin4:8
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@example.com
      PGADMIN_DEFAULT_PASSWORD: dev
    profiles: ["tools"]
```

```bash
docker compose up -d                    # production-like
docker compose --profile tools up -d    # with pgadmin
```

## Real-world Examples

### LAMP Stack

```yaml
services:
  web:
    image: php:8.4-apache
    ports:
      - "80:80"
    volumes:
      - ./src:/var/www/html
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: secret
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: rootsecret
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
      interval: 10s
      timeout: 5s
      retries: 5

  phpmyadmin:
    image: phpmyadmin:5
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
    depends_on:
      - db
    profiles: ["tools"]

volumes:
  mysql_data:
```

### MERN Stack

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:5000
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      NODE_ENV: production
      MONGODB_URI: mongodb://mongo:27017/app
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET is required}
    depends_on:
      mongo:
        condition: service_healthy

  mongo:
    image: mongo:8.0
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express:1
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_URL: mongodb://mongo:27017/
    depends_on:
      - mongo
    profiles: ["tools"]

volumes:
  mongo_data:
```

### Django + PostgreSQL

```yaml
services:
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Microservices with an API Gateway

```yaml
services:
  gateway:
    image: nginx:1.28-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - users
      - orders

  users:
    build: ./users
    environment:
      DATABASE_URL: postgresql://app:secret@users-db:5432/users
    depends_on:
      users-db:
        condition: service_healthy

  orders:
    build: ./orders
    environment:
      DATABASE_URL: postgresql://app:secret@orders-db:5432/orders
    depends_on:
      orders-db:
        condition: service_healthy

  users-db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - users_db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

  orders-db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - orders_db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

volumes:
  users_db:
  orders_db:
```

### Reverse Proxy with Traefik

```yaml
services:
  traefik:
    image: traefik:v3
    command:
      - --providers.docker
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  whoami:
    image: traefik/whoami:v1.10
    labels:
      traefik.http.routers.whoami.rule: Host(`whoami.localhost`)
      traefik.http.services.whoami.loadbalancer.server.port: "80"
```

## Best Practices

1. **No `version:` key** — obsolete in Compose v2.
2. **Pin image versions** (`postgres:18-alpine`, not `postgres:latest`).
3. **Avoid `container_name`** unless required — it breaks scaling and can collide across projects.
4. **Use named volumes** for persistent data, bind mounts for source code.
5. **Health checks + `depends_on: condition: service_healthy`** — the *only* reliable way to sequence startup.
6. **Never hardcode secrets** — use `secrets:` (files) or environment variables from the shell; keep `.env` out of Git.
7. **Cap logs** with the `logging` key.
8. **Set resource limits** (`deploy.resources.limits`) for predictable behavior.
9. **Use profiles** for optional tooling (admin UIs, debug sidecars).
10. **Validate in CI** with `docker compose config -q` (and `--hash` for cache keys).
11. **`restart: unless-stopped`** for long-running services; `no` for one-shot jobs.
12. **Keep the base file environment-agnostic**; put deltas in overrides (`compose.override.yml`, `compose.prod.yml`).
13. **Prefer `pull_policy: build`** for images you build locally, so `up` rebuilds when needed.
14. **Don't mount `/var/run/docker.sock` into untrusted containers** — it is root-equivalent on the host.
15. **`--remove-orphans`** in scripts to clean up containers from removed services.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `port is already allocated` | Another container/process uses the port: `docker compose ps`, `docker ps -f publish=8080` |
| Service can't resolve another service by name | Not on a shared network, or the service has a `container_name`/alias mismatch |
| Changes to code not visible | Bind mount path wrong, or Watch not running (`docker compose watch`) |
| Volume permission denied | Container UID ≠ host UID: set `user:`, or `chown` the host directory |
| `no configuration file provided` | Run from the project directory or pass `-f` |
| YAML error (`did not find expected key`, tabs) | YAML forbids tabs; run `docker compose config` to locate the issue |
| Services start too early / crash-loop | Add `healthcheck` + `depends_on` conditions |
| `.env` values not picked up | `.env` must be in the project directory; check precedence and `--env-file` |
| `container name is already in use` | Remove `container_name` or `docker compose down --remove-orphans` |
| Orphan containers warnings | `docker compose down --remove-orphans` |
| Changes to Dockerfile not applied | Use `docker compose up --build` (or `pull_policy: build`) |
| Windows entrypoint script fails (`\r`) | Convert scripts to LF line endings |
| Health check always unhealthy | Run the command manually: `docker compose exec <svc> sh -c '<test cmd>'` |

Debug toolkit:

```bash
docker compose config               # what will actually be created
docker compose ps -a
docker compose logs --tail=200 <svc>
docker compose exec <svc> sh
docker compose top
docker compose events 2>/dev/null     # if supported, else: docker events
```

## Quick Reference

```bash
# Daily loop
docker compose up -d                  # start
docker compose up -d --build          # rebuild + start
docker compose ps                     # status
docker compose logs -f --tail=100     # logs
docker compose exec app sh            # shell
docker compose down                   # stop + clean

# CI
docker compose config -q                                   # validate config
docker compose up -d --wait                                # start, wait until healthy
docker compose up --abort-on-container-exit --exit-code-from tests   # run tests, propagate exit code
docker compose down -v --remove-orphans                    # clean up

# Compose file skeleton
# services:
#   app:
#     image: app:1.0
#     build: .
#     ports: ["8080:80"]
#     environment: { KEY: value }
#     env_file: [.env]
#     volumes: [data:/data, ./src:/app:ro]
#     networks: [frontend]
#     depends_on: { db: { condition: service_healthy } }
#     healthcheck: { test: ["CMD", "true"], interval: 30s }
#     restart: unless-stopped
#     deploy: { resources: { limits: { cpus: "1.0", memory: 512M } } }
#     logging: { driver: json-file, options: { max-size: 10m, max-file: "3" } }
#     profiles: [tools]
# networks: { frontend: {} }
# volumes: { data: {} }
```

## What's New (2024–2026)

- **`version:` is gone** — files are validated against the Compose Specification directly.
- **Compose Watch** (`develop.watch`, `docker compose watch`, `up --watch`) — sync/rebuild without bind mounts.
- **`!reset` / `!override`** — precise control when merging override files.
- **`include`** — compose files from other projects/directories.
- **`env_file.required`**, **`secrets.environment`**, **`configs.content`** — richer config handling.
- **`depends_on.restart`** and **`required`** — restart dependents when a dependency is updated, tolerate optional services.
- **`docker compose up --wait`** — block until services are healthy; ideal for CI and scripts.
- **`docker compose config --hash`** — stable per-service config hashes for cache invalidation.
- **`build.cache_from` / `cache_to`** — registry/local cache for faster CI builds (BuildKit).
- **Docker Compose v2 is actively developed**; v1 (`docker-compose`) is end-of-life and should be removed from scripts and CI images.

## References

Official documentation:

- [Compose overview](https://docs.docker.com/compose/)
- [Compose file reference](https://docs.docker.com/reference/compose-file/) — every top-level and service key
- [Services reference](https://docs.docker.com/reference/compose-file/services/)
- [Volumes reference](https://docs.docker.com/reference/compose-file/volumes/)
- [Networks reference](https://docs.docker.com/reference/compose-file/networks/)
- [Secrets reference](https://docs.docker.com/reference/compose-file/secrets/)
- [Deploy reference](https://docs.docker.com/reference/compose-file/deploy/)
- [Compose CLI reference](https://docs.docker.com/reference/cli/docker/compose/)
- [Environment variables & interpolation](https://docs.docker.com/compose/how-tos/environment-variables/)
- [Profiles](https://docs.docker.com/compose/how-tos/profiles/)
- [Compose Watch](https://docs.docker.com/compose/how-tos/watch/)
- [Merge & override files](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)
- [Include files](https://docs.docker.com/compose/how-tos/multiple-compose-files/include/)

Samples & community:

- [compose-spec on GitHub](https://github.com/compose-spec/compose-spec) — the specification itself
- [docker/awesome-compose](https://github.com/docker/awesome-compose) — production-ready stacks (LAMP, MERN, Django, …)
- [dockersamples](https://github.com/dockersamples) — official demo projects
- [Docker Hub](https://hub.docker.com/) — official images used in these examples
- [Trivy](https://github.com/aquasecurity/trivy) / [Grype](https://github.com/anchore/grype) — scan images built by Compose

---

**Note**: Covers the Compose Specification as implemented by Docker Compose v2 (2026). For the full schema see the [Compose file reference](https://docs.docker.com/reference/compose-file/) and [CLI reference](https://docs.docker.com/reference/cli/docker/compose/).
