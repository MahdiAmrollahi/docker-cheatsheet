# Dockerfile Cheatsheet

A comprehensive reference for every Dockerfile instruction, BuildKit feature, and production best practice.

## 🔗 Navigation

- **[← Back to Main Docker Cheatsheet](README.md)** — essential Docker commands
- **[Docker Compose Cheatsheet](docker-compose_cheatsheet.md)** — multi-container orchestration

## Table of Contents

- [Introduction](#introduction)
- [Syntax & Parser Directives](#syntax--parser-directives)
- [Build Context & .dockerignore](#build-context--dockerignore)
- [Core Instructions](#core-instructions)
- [Advanced Instructions](#advanced-instructions)
- [ENTRYPOINT vs CMD vs RUN](#entrypoint-vs-cmd-vs-run)
- [Best Practices](#best-practices)
- [BuildKit Features](#buildkit-features)
- [Common Examples](#common-examples)
- [Instruction Quick Reference](#instruction-quick-reference)
- [Common Anti-Patterns](#common-anti-patterns)
- [What's New (2024–2026)](#whats-new-20242026)
- [References](#references)

## Introduction

A Dockerfile is a text file with a series of instructions that Docker executes to build an image. Each instruction produces a layer; layers are cached and reused, which is why **order matters** for build speed.

```
Dockerfile + build context ──▶ BuildKit ──▶ image (layers) ──▶ registry
```

### Basic Structure

```dockerfile
# Comment
INSTRUCTION arguments
```

### Syntax Rules

- Instructions are case-insensitive; convention is UPPERCASE.
- The first instruction must be `FROM` (parser directives and `ARG` may precede it).
- Each instruction generally creates a new layer — combine related commands.
- Comments start with `#`; a `#` elsewhere in a line is literal.
- Line continuation with `\`; arguments can be split across lines.
- Use `.dockerignore` to keep the build context small.
- Prefer `COPY` over `ADD`.

## Syntax & Parser Directives

Optional directives at the top of the file control the parser and linter:

```dockerfile
# syntax=docker/dockerfile:1
# escape=\
# check=error=true

FROM node:24-alpine
```

| Directive | Purpose |
|-----------|---------|
| `# syntax=docker/dockerfile:1` | Use the latest stable BuildKit frontend (needed for heredocs, `--mount`, `--link`, …) |
| `# syntax=docker/dockerfile:1.7` | Pin an exact frontend version for reproducible builds |
| `# escape=\` (or `` ` ``) | Change the line-continuation character (Windows paths) |
| `# check=error=true` | Fail the build on linter warnings (`docker build --check`) |

## Build Context & .dockerignore

The **build context** is everything sent to the builder (`docker build .` → the current directory). A large context slows builds and can leak files into images.

`.dockerignore` lives next to the Dockerfile:

```
**/.git
**/.gitignore
**/node_modules
**/.env
**/.env.*
**/*.log
**/__pycache__
**/.venv
**/dist
**/coverage
Dockerfile*
docker-compose*.yml
*.md
!README.md
```

- Patterns are evaluated in order; `!` negates.
- `**/` matches any directory depth, `*` matches within a path segment.
- Excluding secrets (`.env`), VCS data (`.git`), and dependencies (`node_modules`) is mandatory hygiene.

## Core Instructions

### FROM

Sets the base image; must be the first instruction of a stage.

```dockerfile
FROM node:24-alpine                 # official image, pinned major
FROM ubuntu:24.04                   # specific version
FROM scratch                        # empty base — static binaries only
FROM node:24 AS build               # named stage for multi-stage builds
FROM --platform=linux/amd64 ubuntu:24.04
FROM --platform=$BUILDPLATFORM golang:1.25-alpine AS build   # cross-compilation
FROM node:24-alpine@sha256:<digest> # fully reproducible pin
```

- `ARG` may be used in `FROM` only if declared before it:

```dockerfile
ARG NODE_VERSION=24
FROM node:${NODE_VERSION}-alpine
```

- Prefer slim/alpine/distroless/scratch bases; `latest` hurts reproducibility.

### RUN

Executes commands **at build time**, in a new layer.

```dockerfile
# Shell form — runs via /bin/sh -c (string interpolation works)
RUN apt-get update && apt-get install -y --no-install-recommends curl

# Exec form — no shell; use when you need no shell processing
RUN ["/bin/bash", "-c", "set -eux; echo hello"]

# Combine and clean up in the SAME layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Heredoc (BuildKit, frontend 1.4+)
RUN <<EOF
set -eux
echo "multiple lines"
EOF
```

### CMD

Default command for the container. Only the **last** `CMD` in a stage takes effect; overridden by any arguments passed to `docker run`.

```dockerfile
CMD ["node", "server.js"]            # exec form (recommended)
CMD node server.js                   # shell form — runs as /bin/sh -c "node server.js"
CMD ["--help"]                       # default args for ENTRYPOINT
```

### ENTRYPOINT

Makes the container behave like an executable. `CMD` supplies default arguments.

```dockerfile
ENTRYPOINT ["node", "server.js"]     # exec form (recommended)
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]   # args appended to ENTRYPOINT
```

Override at runtime with `docker run --entrypoint sh <image>`.

### COPY

Copies files from the build context (or another stage) into the image.

```dockerfile
COPY package.json package-lock.json ./      # relative to WORKDIR
COPY . /app/
COPY --chown=1000:1000 app /app             # set ownership without a chown layer
COPY --from=build /app/dist ./dist          # copy from a named stage
COPY --from=nginx:1.28-alpine /etc/nginx/nginx.conf /etc/nginx/
COPY --link --chown=1000:1000 . /app        # independent layer, cache friendly (BuildKit 1.4+)
COPY --exclude=*.log --exclude=node_modules . /app   # frontend 1.19+
```

- Destination ending in `/` (or `.`/`..`) = directory; otherwise treated as a file for a single source.
- `COPY` never extracts archives and never accesses the network.

### ADD

Like `COPY`, with two extra behaviors — and two good reasons to prefer `COPY`.

```dockerfile
ADD app.tar.gz /app/                 # AUTO-EXTRACTS local tar archives
ADD https://example.com/file.tar.gz /tmp/   # downloads BUT does not extract
```

- Remote `ADD` cannot be cached well and provides no checksum verification — use `RUN curl -fsSL … | tar …` instead.
- Use `ADD` deliberately (e.g. unpacking a trusted local tarball); otherwise `COPY`.

### WORKDIR

Sets the working directory for `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD`. Creates the directory if missing.

```dockerfile
WORKDIR /app
WORKDIR src            # relative → /app/src
WORKDIR /var/www/html  # absolute resets
```

Avoid `RUN cd /app` — it does not persist.

### ENV

Sets environment variables that persist in the image and in every container.

```dockerfile
ENV NODE_ENV=production
ENV APP_HOME=/app PORT=3000         # multiple in one layer
ENV PATH="$APP_HOME/bin:$PATH"      # expand previous variables
```

`ENV` values are visible in `docker inspect` — never put secrets here.

### ARG

Build-time variables, not persisted in the final image runtime environment.

```dockerfile
ARG NODE_VERSION=24                 # before FROM: usable in FROM line only
FROM node:${NODE_VERSION}-alpine
ARG NODE_VERSION                    # redeclare to use inside the stage
RUN echo "$NODE_VERSION"

ARG BUILD_DATE
LABEL org.opencontainers.image.created=$BUILD_DATE
```

```bash
docker build --build-arg NODE_VERSION=22 .
```

- `ARG` values **are** recorded in image history — never pass secrets as build args.
- `ARG` scope is per stage; redeclare after each `FROM`.
- A same-named `ENV` overrides `ARG`.

### EXPOSE

Documents the ports the container listens on (metadata only — it does not publish anything).

```dockerfile
EXPOSE 3000
EXPOSE 80/tcp 53/udp
```

Publish with `docker run -p 8080:80` or Compose `ports:`.

## Advanced Instructions

### USER

Sets the user (and optional group) for subsequent instructions and the container runtime.

```dockerfile
# Alpine
RUN addgroup -S -g 1001 app && adduser -S -u 1001 -G app app
USER app

# Debian / Ubuntu
RUN groupadd --system --gid 1001 app \
 && useradd --system --uid 1001 --gid app --create-home app
USER app

USER 1001:1001                      # numeric (works with any base image)
USER root                           # switch back when needed
```

`COPY --chown` should be used instead of `RUN chown -R` (avoids duplicating a whole tree in a layer).

### VOLUME

Declares a mount point backed by a Docker-managed volume.

```dockerfile
VOLUME ["/data"]
VOLUME /var/lib/mysql
```

Caveats:

- Documented behavior: changes made to a `VOLUME` path **after** the instruction are discarded at runtime — so `VOLUME /app` swallows later writes.
- Use it to declare a data boundary (databases), not to seed data; prefer mounting volumes via `docker run`/Compose.
- Anonymous volumes created by `VOLUME` live on after `docker rm` unless removed with `-v`.

### LABEL

Adds image metadata. Prefer the standard [OCI keys](https://github.com/opencontainers/image-spec/blob/main/annotations.md).

```dockerfile
LABEL org.opencontainers.image.title="myapp" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.revision=$GIT_SHA \
      org.opencontainers.image.licenses="MIT"
```

`MAINTAINER` is deprecated — use `org.opencontainers.image.authors`.

### HEALTHCHECK

Tells Docker how to test whether the container is still working.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --start-interval=2s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/health || exit 1

HEALTHCHECK CMD ["node", "healthcheck.js"]    # exec form, no shell
HEALTHCHECK NONE                              # disable inherited check
```

| Option | Meaning |
|--------|---------|
| `--interval` | Time between checks (default 30s) |
| `--timeout` | Max duration of a single check (default 30s) |
| `--retries` | Consecutive failures before `unhealthy` (default 3) |
| `--start-period` | Grace period during startup (default 0s) |
| `--start-interval` | Check interval during `--start-period` (Docker 25+) |

Use tools that exist in the image (`wget` in Alpine, Python `urllib` in slim images); avoid installing `curl` just for the check.

### SHELL

Changes the shell used by the shell form of `RUN`, `CMD`, and `ENTRYPOINT`.

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]   # fail on broken pipes
SHELL ["powershell", "-Command"]              # Windows containers
```

### STOPSIGNAL

Signal sent by `docker stop` (after which the container is killed).

```dockerfile
STOPSIGNAL SIGQUIT     # e.g. nginx graceful shutdown
STOPSIGNAL SIGTERM     # default
```

### ONBUILD

Registers a trigger that runs in **child** builds (images built `FROM` this one).

```dockerfile
ONBUILD COPY package.json ./
ONBUILD RUN npm install
ONBUILD COPY . ./
```

- Triggers run during the child's build and are not inherited by grandchildren.
- They run with the child's context/permissions — surprising and hard to debug. Modern practice: prefer explicit multi-stage builds or build args.

## ENTRYPOINT vs CMD vs RUN

| Instruction | When it runs | Override |
|-------------|--------------|----------|
| `RUN` | Build time (creates a layer) | Rebuild |
| `CMD` | Container start (default command/args) | `docker run <image> <args>` replaces it |
| `ENTRYPOINT` | Container start (fixed executable) | `docker run --entrypoint <cmd>` |

Common patterns:

```dockerfile
# Executable with default args
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# Script that forwards signals — use exec "$@"
ENTRYPOINT ["/entrypoint.sh"]
CMD ["postgres"]
```

```bash
#!/bin/sh
set -e
# ... init ...
exec "$@"        # replace the shell so PID 1 receives signals
```

Rules of thumb:

- Use **exec form** (`["cmd", "arg"]`) so your process is PID 1 and receives signals — shell form runs `/bin/sh -c` and may swallow `SIGTERM`.
- If PID 1 cannot reap zombies, run with `docker run --init` or Compose `init: true`.

## Best Practices

### 1. Multi-stage Builds

Build with the full toolchain; ship only the artifact.

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM nginx:1.28-alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

Also useful for running tests in CI without shipping test dependencies:

```dockerfile
FROM build AS test
RUN npm test

FROM nginx:1.28-alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
```

### 2. Layer Caching Order

Order instructions from **least** to **most** frequently changed:

```dockerfile
# Good: dependencies cached until lockfile changes
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# Bad: any source change re-installs dependencies
COPY . .
RUN npm ci
```

Also:

- `apt-get update` and `apt-get install` must be in the **same** `RUN`.
- Combine multiple shell commands to reduce layers (each `RUN` is a layer).
- Use `--mount=type=cache` (BuildKit) rather than relying on layer caching for package managers.

### 3. Image Size

```dockerfile
# Pick a minimal base that satisfies your needs
FROM node:24-alpine        # or node:24-slim / distroless / scratch

# Clean package manager data in the same layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

# Python
RUN pip install --no-cache-dir -r requirements.txt

# Node
RUN npm ci --omit=dev      # skip devDependencies

# Go: static binary, no libc needed
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app .
```

- `--no-install-recommends` avoids extra apt packages.
- Don't run `apt-get upgrade` in images (unpredictable, bloats layers).
- Strip binaries (`-s -w` in Go, `strip` for C/C++).

### 4. Security

```dockerfile
# Pin versions/digests, not latest
FROM node:24-alpine@sha256:<digest>

# Run as non-root
RUN addgroup -S -g 1001 app && adduser -S -u 1001 -G app app
USER app

# Never bake secrets — mount them for the build instead
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci

# Install only what you need; remove package managers if possible
```

- Secrets in `ENV`, `ARG`, or `COPY` remain in layer history — always rotatable, never private.
- Use `docker scout cves <image>` / `trivy image <image>` in CI.
- Add `--security-opt no-new-privileges` and `--read-only` at **runtime** where possible.
- Keep the base image updated; rebuild regularly (e.g. Renovate/Dependabot for base tags).

### 5. `.dockerignore` and Context

Small context = fast builds, no accidental secret copies:

```
**/.git
**/node_modules
**/.env*
**/*.log
**/dist
```

### 6. Reproducibility

- `# syntax=docker/dockerfile:1.7` (pinned frontend).
- `npm ci` / `pip install -r` / `go mod download` from lockfiles.
- Pin base images by digest for critical builds.
- `SOURCE_DATE_EPOCH` and `--provenance=true` for verifiable builds.

## BuildKit Features

BuildKit is the default builder (Docker 23+). These features require `# syntax=docker/dockerfile:1` (or newer). See [Building with BuildKit](https://docs.docker.com/build/buildkit/).

### Cache Mounts

Persist package-manager caches **outside** the image layer.

```dockerfile
RUN --mount=type=cache,target=/root/.npm npm ci
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/var/cache/apt apt-get update && apt-get install -y curl
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app .
```

- Cache contents are not part of the image.
- Add `sharing=locked` when parallel builds share a cache (`apt`).
- The cache survives rebuilds; clear it with `docker builder prune`.

### Secrets

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
RUN --mount=type=secret,id=aws,required=false test -f /run/secrets/aws
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
docker build --secret id=aws,src=$HOME/.aws/credentials -t app .
```

With Compose:

```yaml
services:
  app:
    build:
      context: .
      secrets:
        - npmrc
secrets:
  npmrc:
    file: $HOME/.npmrc
```

### SSH Agent Forwarding

```dockerfile
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git
```

```bash
docker build --ssh default -t app .
```

### Multi-Platform Builds

Automatic platform arguments: `BUILDPLATFORM`, `BUILDOS`, `BUILDARCH`, `TARGETPLATFORM`, `TARGETOS`, `TARGETARCH`, `TARGETVARIANT`.

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.25-alpine AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM scratch
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t user/app:1.0 --push .
```

### COPY --link and Named Contexts

```dockerfile
# --link: layer does not depend on previous layers → better cache + parallelism
COPY --link --chown=1000:1000 . /app

# Named build contexts
FROM base AS build
COPY --from=config /etc/app.conf /etc/app.conf
```

```bash
docker buildx build --build-context config=./config .
```

### Build Checks (Linter)

```dockerfile
# check=error=true
```

```bash
docker build --check .
```

Catches mistakes like undefined variables, invalid stages, and deprecated syntax, with suggestions. See [Build checks](https://docs.docker.com/build/checks/).

### Attestations

```bash
docker buildx build --sbom=true --provenance=true -t user/app:1.0 --push .
docker buildx imagetools inspect user/app:1.0
```

See [Build attestations](https://docs.docker.com/build/attestations/).

### Declarative Builds with `buildx bake`

```hcl
# docker-bake.hcl
target "app" {
  context = "."
  tags    = ["user/app:1.0"]
  platforms = ["linux/amd64", "linux/arm64"]
}
```

```bash
docker buildx bake
```

## Common Examples

### Node.js (production, multi-stage)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM node:24-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
RUN addgroup -S -g 1001 app && adduser -S -u 1001 -G app app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Python (production, virtualenv)

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.13-slim AS build
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1
WORKDIR /app
RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

FROM python:3.13-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH="/venv/bin:$PATH"
RUN groupadd --system --gid 1001 app \
 && useradd --system --uid 1001 --gid app app
WORKDIR /app
COPY --from=build /venv /venv
COPY --chown=app:app . .
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')" || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
```

### Go (scratch, multi-arch)

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.25-alpine AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
RUN apk add --no-cache ca-certificates
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/app /app
USER 65534:65534
EXPOSE 8080
ENTRYPOINT ["/app"]
```

### Java (Maven multi-stage)

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace
COPY .mvn .mvn
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B dependency:go-offline
COPY src src
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B package -DskipTests

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S -g 1001 app && adduser -S -u 1001 -G app app
WORKDIR /app
COPY --from=build --chown=app:app /workspace/target/*.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

### Static Site (build then nginx)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM nginx:1.28-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

### Development Dockerfile (with Compose override)

```dockerfile
FROM node:24-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]
```

## Instruction Quick Reference

| Instruction | Purpose | Notes |
|-------------|---------|-------|
| `FROM` | Base image / stage | Must be first (after `ARG`) |
| `RUN` | Execute at build time | Combine commands; one layer each |
| `CMD` | Default command/args | Overridden by `docker run` args |
| `ENTRYPOINT` | Fixed executable | Override with `--entrypoint` |
| `COPY` | Copy from context/stage | Preferred over `ADD` |
| `ADD` | Copy + tar extract + URL | Use only when needed |
| `WORKDIR` | Set working directory | Creates the directory |
| `ENV` | Persistent env var | Visible in `docker inspect` |
| `ARG` | Build-time variable | Visible in image history |
| `EXPOSE` | Document ports | Metadata only |
| `USER` | Runtime user | Prefer non-root |
| `VOLUME` | Declare a mount point | Changes after it are discarded |
| `LABEL` | Image metadata | Use OCI labels |
| `HEALTHCHECK` | Container health check | `NONE` to disable |
| `SHELL` | Change build shell | Affects shell-form `RUN`/`CMD` |
| `STOPSIGNAL` | Signal for `docker stop` | e.g. `SIGQUIT` |
| `ONBUILD` | Trigger in child builds | Prefer multi-stage |
| `MAINTAINER` | Deprecated | Use OCI labels |

## Common Anti-Patterns

| Anti-pattern | Better approach |
|--------------|-----------------|
| `FROM image:latest` | Pin a version or digest |
| `RUN apt-get upgrade` | Rebuild from an updated base instead |
| Separate `apt-get update` and `install` | Same `RUN` to avoid stale caches |
| `COPY . .` before installing deps | Copy lockfiles first, install, then copy source |
| Secrets via `ENV`/`ARG`/`COPY` | `RUN --mount=type=secret` |
| Running as root | Create a user and `USER app` |
| Installing build tools in the final image | Multi-stage builds |
| `ADD` for local copies / URL downloads | `COPY` or `RUN curl` with checksums |
| `VOLUME /app` in app images | Mount at runtime via Compose/`docker run` |
| Shell-form `CMD` for servers | Exec form (signals/PID 1) |
| No `.dockerignore` | Add one (`.git`, `node_modules`, `.env`) |
| Multiple `RUN` per package | Combine to reduce layers |
| Changing `WORKDIR` with `RUN cd` | Use `WORKDIR` |
| Huge images from cache files | Clean caches in the same layer or use cache mounts |

## What's New (2024–2026)

- **BuildKit is the default** — `DOCKER_BUILDKIT=1` is obsolete; `--mount`/heredocs work out of the box.
- **`COPY --link`** — layer-independent copies for better cache reuse and parallelism.
- **`COPY --exclude`** — exclude paths from `COPY`/`ADD` without `.dockerignore` adjustments (frontend 1.19+).
- **Heredocs** — multi-line `RUN`/`COPY` without shell escaping.
- **Build checks** — `docker build --check` lints Dockerfiles and suggests fixes; `# check=error=true` makes warnings fail the build.
- **Attestations** — `--sbom=true --provenance=true` attach verifiable metadata to pushed images.
- **Docker Scout** — `docker scout cves` replaced the retired `docker scan`.
- **Multi-platform everywhere** — `$BUILDPLATFORM`/`$TARGETARCH` cross-compilation patterns are standard.
- **Minimal bases** — Distroless, `chainguard` images, and `scratch` are common production choices.
- **Frontend versioning** — `# syntax=docker/dockerfile:1` tracks the latest stable 1.x; pin for reproducibility.

## References

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — official instruction grammar
- [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/) — official guide
- [Building with BuildKit](https://docs.docker.com/build/buildkit/)
- [Build checks](https://docs.docker.com/build/checks/) — Dockerfile linter and `--check`
- [Build secrets](https://docs.docker.com/build/building/secrets/)
- [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)
- [Build cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Build attestations](https://docs.docker.com/build/attestations/) — SBOM and provenance
- [docker buildx bake](https://docs.docker.com/build/bake/)
- [Frontend releases (docker/dockerfile)](https://github.com/docker/dockerfile/releases) — heredoc, `--link`, `--exclude`, checks
- [OCI image annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md) — standard `org.opencontainers.image.*` labels
- [Distroless images](https://github.com/GoogleContainerTools/distroless)
- [Hadolint](https://github.com/hadolint/hadolint) — alternative Dockerfile linter
- [dive](https://github.com/wagoodman/dive) — inspect layers and image size
- [docker/awesome-compose](https://github.com/docker/awesome-compose) — real-world Dockerfile/Compose examples

---

**Note**: Covers Dockerfile syntax and BuildKit features for Docker Engine 28+. For the full grammar see the [Dockerfile reference](https://docs.docker.com/reference/dockerfile/).
