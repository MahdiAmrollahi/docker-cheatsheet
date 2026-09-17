# Docker Cheatsheet

A comprehensive, practical Docker reference: core concepts, everyday commands, networking, storage, security, and troubleshooting — kept up to date with Docker Engine 28+ and Compose v2.

![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Compose](https://img.shields.io/badge/Compose-v2-2496ED?logo=docker&logoColor=white)

## 📚 Related Cheatsheets

- **[Dockerfile Cheatsheet](dockerfile_cheatsheet.md)** — every Dockerfile instruction, BuildKit features, multi-stage builds, best practices
- **[Docker Compose Cheatsheet](docker-compose_cheatsheet.md)** — services, profiles, watch mode, real-world stacks

## Table of Contents

- [Core Concepts](#core-concepts)
- [Installation & Setup](#installation--setup)
- [Docker CLI Essentials](#docker-cli-essentials)
- [Image Commands](#image-commands)
- [Building Images](#building-images)
- [Container Commands](#container-commands)
- [Docker Compose](#docker-compose)
- [Networking](#networking)
- [Volumes & Storage](#volumes--storage)
- [Registries & Image Distribution](#registries--image-distribution)
- [Contexts & Remote Daemons](#contexts--remote-daemons)
- [Resources & Limits](#resources--limits)
- [Health Checks](#health-checks)
- [Logging](#logging)
- [Security Best Practices](#security-best-practices)
- [Cleanup & Maintenance](#cleanup--maintenance)
- [Troubleshooting](#troubleshooting)
- [Common Flags Reference](#common-flags-reference)
- [Quick Reference](#quick-reference)
- [References & Further Reading](#references--further-reading)

## Core Concepts

| Concept | What it is |
|---------|------------|
| **Image** | Read-only template containing your app + dependencies (built from a Dockerfile) |
| **Container** | Runnable instance of an image; adds a thin writable layer on top |
| **Dockerfile** | Text recipe describing how to build an image |
| **Registry** | Stores and distributes images (Docker Hub, GHCR, ECR, GCR, self-hosted) |
| **Volume** | Docker-managed persistent storage, independent of container lifecycle |
| **Network** | Isolated virtual network; containers on the same user-defined network resolve each other by name |
| **Compose** | YAML-based tool for defining and running multi-container apps |
| **BuildKit** | Modern build engine (default since Docker 23) with caching, secrets, and multi-platform support |
| **Daemon / CLI** | `dockerd` does the work; the `docker` CLI talks to it over a socket or TCP/SSH |

Architecture in one line:

```
docker CLI  ──HTTP/socket──▶  dockerd (Engine)  ──▶  containerd  ──▶  runc  ──▶  container
```

> **v1 vs v2 CLI:** `docker-compose` (Python, v1) reached end of life in July 2023. Use `docker compose` (Go plugin, v2) — all commands are subcommands of `docker`. This cheatsheet uses v2 only.

## Installation & Setup

- **Windows / macOS:** [Docker Desktop](https://docs.docker.com/desktop/) (bundles Engine, CLI, Compose, Buildx, Kubernetes).
- **Linux:** [Docker Engine](https://docs.docker.com/engine/install/) + the `docker-compose-plugin` package.
- **Rootless mode (Linux):** run the daemon without root:

```bash
dockerd-rootless-setuptool.sh install
```

### Verify Installation

```bash
docker version            # client + server versions
docker info               # daemon, storage driver, cgroups, plugins
docker compose version    # Compose v2 plugin
docker run hello-world    # smoke test
```

### Daemon Configuration (optional, Linux)

`/etc/docker/daemon.json` (restart with `systemctl restart docker`):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "live-restore": true,
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ]
}
```

`live-restore` keeps containers running while the daemon restarts (not supported on Swarm).

## Docker CLI Essentials

```bash
docker --help                          # list all commands
docker <command> --help                # help for a specific command
docker version --format '{{.Server.Version}}'
docker info --format '{{.Driver}} | cgroup {{.CgroupVersion}}'
docker system events                   # real-time daemon event stream
docker system df -v                    # disk usage per image/container/volume
docker context ls                      # which daemon am I talking to?
```

## Image Commands

```bash
# Pull / list / remove
docker pull nginx:1.28-alpine
docker images                                  # all local images
docker image ls --filter dangling=true         # untagged (<none>) images
docker rmi nginx:1.28-alpine                   # remove by tag or ID
docker image prune                             # remove dangling images only
docker image prune -a --filter "until=168h"    # unused images older than 7 days

# Inspect
docker image inspect nginx:1.28-alpine          # full JSON metadata
docker image history nginx:1.28-alpine          # layer-by-layer history
docker image inspect -f '{{.Architecture}} {{.Os}}' nginx:1.28-alpine

# Tag & push
docker tag nginx:1.28-alpine registry.example.com/team/nginx:1.28
docker push registry.example.com/team/nginx:1.28

# Save / load (keeps layers and history)
docker save nginx:1.28-alpine -o nginx.tar
docker load -i nginx.tar

# Export / import a container filesystem (flattened, loses history — prefer save/load)
docker export web > web.tar
docker import web.tar myimage:flat
```

`docker search` exists but is limited; browse images at [hub.docker.com](https://hub.docker.com).

## Building Images

```bash
docker build -t myapp:1.0 .                          # build from ./Dockerfile
docker build -f docker/Dockerfile.prod -t myapp:prod .
docker build --build-arg NODE_ENV=production -t myapp:1.0 .
docker build --target builder -t myapp:debug .       # stop at a named stage
docker build --no-cache --pull -t myapp:1.0 .        # ignore cache, refresh base image
docker build --progress=plain -t myapp:1.0 .         # verbose BuildKit output
docker build --secret id=npmrc,src="$HOME/.npmrc" -t myapp:1.0 .
docker build --check .                               # lint the Dockerfile (Docker 26+)
```

See the **[Dockerfile Cheatsheet](dockerfile_cheatsheet.md)** for every instruction and pattern.

### Multi-platform & Cache (Buildx)

```bash
# Create and use a dedicated builder (needed for cross-platform builds / QEMU emulation)
docker buildx create --use --name multiarch
docker buildx inspect --bootstrap

# Multi-arch build — must push or load, the result cannot live only in the local image store
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t user/myapp:1.0 --push .

# Same image for local use
docker buildx build --platform linux/amd64 -t user/myapp:1.0 --load .

# Registry cache (great for CI)
docker buildx build \
  --cache-from type=registry,ref=user/myapp:buildcache \
  --cache-to   type=registry,ref=user/myapp:buildcache,mode=max \
  -t user/myapp:1.0 --push .

# SBOM + provenance attestations
docker buildx build --sbom=true --provenance=true -t user/myapp:1.0 --push .

# Inspect a published multi-arch image (manifests, platforms, digests)
docker buildx imagetools inspect user/myapp:1.0

# Declarative multi-image builds
docker buildx bake -f docker-bake.hcl
```

> `DOCKER_BUILDKIT=1` is no longer needed — BuildKit has been the default builder since Docker 23. Legacy inline cache (`--build-arg BUILDKIT_INLINE_CACHE=1`) is only needed for old registries; registry cache is preferred.

## Container Commands

### `docker run` Anatomy

```bash
docker run -d --name web \
  -p 8080:80 \
  -e TZ=UTC \
  -v webdata:/usr/share/nginx/html \
  --network appnet \
  --restart unless-stopped \
  --memory 512m --cpus 1.5 \
  --health-cmd "wget -qO- http://localhost/ || exit 1" \
  --health-interval 30s --health-retries 3 \
  --read-only --tmpfs /tmp \
  --security-opt no-new-privileges:true \
  nginx:1.28-alpine
```

### Lifecycle

```bash
docker create --name web nginx:1.28-alpine   # prepare without starting
docker start web
docker stop -t 30 web                        # SIGTERM, then SIGKILL after 30s (default 10)
docker restart web
docker kill -s SIGUSR1 web                   # send a custom signal
docker pause web       # freeze processes (cgroup freezer)
docker unpause web
docker wait web                              # block until it exits, print the exit code
docker rename web web-old
docker rm web                                # remove a stopped container
docker rm -f web                             # force remove a running one
docker container prune                       # remove all stopped containers
docker update --restart=always --memory 1g web
```

### Listing & Filtering

```bash
docker ps
docker ps -a
docker ps -q -f status=exited
docker ps -f "label=com.docker.compose.project=myapp"
docker ps --filter publish=8080 --filter health=unhealthy
docker ps --size                                   # include writable-layer size
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

### Interacting

```bash
docker exec -it web sh                       # shell inside a running container
docker exec -u 1000 -w /app -e FOO=bar web env
docker attach web                            # attach to PID 1 (detach: Ctrl-P Ctrl-Q)
docker logs -f --tail 100 web
docker logs --since 10m --timestamps web
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./index.html web:/usr/share/nginx/html/
docker diff web                              # files changed in the writable layer
docker top web                               # host-side process list
docker stats --no-stream
docker inspect web
docker inspect -f '{{.State.Status}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
docker events --filter container=web --since 1h
docker commit web myimage:snapshot           # avoid — reproduce the change in a Dockerfile instead
```

### `docker run` Common Flags

| Flag | Description |
|------|-------------|
| `-d` | Detached (background) |
| `-it` | Interactive + TTY |
| `--rm` | Remove container when it exits |
| `--name` | Container name |
| `-p 8080:80` | Publish port (host:container) |
| `-P` | Publish all `EXPOSE`d ports to random host ports |
| `-e`, `--env-file` | Environment variables |
| `-v`, `--mount` | Volume / bind mount |
| `--network` | Attach to a network |
| `-w` | Working directory |
| `-u` | User (`uid:gid` or name) |
| `--entrypoint` | Override ENTRYPOINT |
| `--restart` | `no`, `on-failure[:n]`, `always`, `unless-stopped` |
| `--init` | Run `tini` as PID 1 (zombie reaping) |
| `--platform` | Force an architecture (`linux/amd64`, `linux/arm64`) |
| `--memory`, `--cpus` | Resource limits |
| `--read-only`, `--tmpfs` | Immutable root filesystem |
| `--health-cmd` etc. | Health check |
| `--log-driver`, `--log-opt` | Logging configuration |
| `--gpus all` | GPU access (with NVIDIA Container Toolkit) |

## Docker Compose

Full reference: **[Docker Compose Cheatsheet](docker-compose_cheatsheet.md)**

```bash
docker compose up -d --build           # build + start in background
docker compose ps
docker compose logs -f --tail=100
docker compose exec web sh
docker compose run --rm web npm test   # one-off command in a new container
docker compose down -v --remove-orphans
docker compose config                  # validate + print the resolved file
docker compose up --watch              # sync/rebuild on source changes (v2.22+)
```

> Compose v2 does not use the `version:` top-level key anymore — it is obsolete and ignored.

## Networking

### Drivers

| Driver | Use case |
|--------|----------|
| `bridge` | Default. Containers on the same user-defined bridge can resolve each other by name |
| `host` | No isolation — container shares the host network (Linux only) |
| `none` | No networking |
| `overlay` | Multi-host networking (Swarm); `--attachable` allows standalone containers |
| `macvlan` / `ipvlan` | Give containers their own MAC/IP on the physical LAN |

### Commands

```bash
docker network ls
docker network create appnet
docker network create --internal isolated                      # no external connectivity
docker network create --subnet 172.28.0.0/16 --gateway 172.28.0.1 appnet
docker network inspect appnet
docker network connect appnet web                              # add a running container
docker network disconnect appnet web
docker network prune
```

### Container Networking

```bash
docker run -d --name api --network appnet -p 127.0.0.1:3000:3000 myapi
docker run --network host nginx                       # host networking
docker run --network none alpine ping 127.0.0.1       # isolated
docker run --network container:api curlimages/curl http://localhost:3000   # share another container's netns
```

### Port Publishing Cheatsheet

```bash
-p 8080:80              # host:container
-p 127.0.0.1:8080:80    # bind to loopback only (safer than 0.0.0.0)
-p 8080:80/udp
-p 8000-8010:8000-8010  # port range
-P                      # publish all EXPOSEd ports to random host ports
```

> DNS: containers on the **default** bridge network can only reach each other by IP (legacy `--link` aside). On a **user-defined** network, the container name and network aliases resolve automatically via Docker's embedded DNS (127.0.0.11).

## Volumes & Storage

| Type | Managed by | Use case |
|------|-----------|----------|
| **Named volume** | Docker (`/var/lib/docker/volumes`) | Persistent app/db data; portable, backed up easily |
| **Bind mount** | You (host path) | Source code, config files during development |
| **tmpfs** | Kernel (RAM) | Scratch data that must never hit disk |

```bash
# Volume management
docker volume create appdata
docker volume ls
docker volume inspect appdata
docker volume rm appdata
docker volume prune              # anonymous unused volumes
docker volume prune -a           # include unused named volumes (careful!)

# Mounting
docker run -v appdata:/var/lib/data nginx                # named volume
docker run -v "$PWD/app:/app" nginx                      # bind mount
docker run -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" nginx
docker run --mount type=volume,source=appdata,target=/data nginx
docker run --mount type=bind,source="$PWD/app",target=/app,readonly nginx
docker run --tmpfs /tmp:rw,size=64m nginx                # ephemeral RAM disk
```

Notes:

- Prefer `--mount` for scripts: it is explicit and fails loudly if the source does not exist.
- Relative paths in `-v` are not allowed; use `$PWD` (Linux/macOS) or `${PWD}` (PowerShell).
- On SELinux hosts add `:z` (shared) or `:Z` (private) to bind mounts.

### Backup & Restore a Volume

```bash
# Backup
docker run --rm -v appdata:/data -v "$PWD":/backup alpine \
  tar czf /backup/appdata.tar.gz -C /data .

# Restore
docker run --rm -v appdata:/data -v "$PWD":/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/appdata.tar.gz -C /data"
```

## Registries & Image Distribution

```bash
docker login registry.example.com
cat token.txt | docker login -u "$USER" --password-stdin registry.example.com
docker logout registry.example.com
```

Image names follow `[registry/]namespace/name[:tag][@digest]`:

```bash
docker pull ghcr.io/org/app:1.0
docker tag app:1.0 ghcr.io/org/app:1.0
docker push ghcr.io/org/app:1.0
docker pull --platform linux/arm64 ghcr.io/org/app:1.0
docker buildx imagetools inspect ghcr.io/org/app:1.0
```

Run a local registry (useful for offline/CI):

```bash
docker run -d -p 5000:5000 --name registry registry:2
docker tag app:1.0 localhost:5000/app:1.0
docker push localhost:5000/app:1.0
```

> Pull policy: if a tag exists locally, `docker run` uses it. Use `docker pull` or `docker compose up --pull always` to force a refresh — tags are mutable, digests are not.

## Contexts & Remote Daemons

```bash
docker context ls
docker context create remote --docker "host=ssh://user@server"
docker context use remote
docker context use default
docker context rm remote

DOCKER_HOST=ssh://user@server docker ps        # per-command override
DOCKER_HOST=tcp://192.168.1.10:2376 docker ps  # remote TCP (use TLS in production)
```

## Resources & Limits

```bash
docker run --memory 512m --memory-swap 1g --cpus 1.5 --cpuset-cpus 0,1 --pids-limit 200 nginx
docker update --cpus 2 --memory 1g web         # adjust a running container
docker stats                                   # live usage
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
docker inspect -f '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' web
```

- Memory limits are enforced by the kernel (cgroup). Without a limit a container can exhaust host memory (OOM).
- `--memory-swap` controls swap; equal to `--memory` means no swap.
- `NanoCpus` = CPU cores × 1e9 (e.g. `1500000000` = 1.5 CPUs).
- Compose equivalent: `deploy.resources.limits` (applied by `docker compose up` on modern versions).

## Health Checks

```bash
docker run -d --name web \
  --health-cmd "wget -qO- http://localhost/ || exit 1" \
  --health-interval 30s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-start-period 10s \
  nginx:1.28-alpine

docker inspect -f '{{.State.Health.Status}}' web    # starting | healthy | unhealthy
docker ps --filter health=unhealthy
docker events --filter event=health_status
```

- `HEALTHCHECK` in a Dockerfile or `healthcheck:` in Compose sets this declaratively.
- `depends_on: condition: service_healthy` lets Compose wait for readiness.
- Prefer exec form inside the check and keep the image's toolset in mind (`wget` exists in Alpine, `curl` may not).

## Logging

```bash
docker logs -f web                          # follow
docker logs --tail 50 --since 30m web
docker logs --until 2026-01-01T00:00:00 web
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 nginx
docker run --log-driver local nginx         # efficient rotating default
docker run --log-driver none nginx
```

Set defaults in `daemon.json` (see [Installation](#installation--setup)). In Compose use the `logging:` key per service. `docker logs` only works with the `json-file` and `local` drivers — with others (e.g. `syslog`, `journald`, `fluentd`) use the external system.

## Security Best Practices

```bash
# Run as a non-root user (image must have it, or use an arbitrary UID)
docker run --user 1000:1000 nginx:1.28-alpine

# Immutable root filesystem
docker run --read-only --tmpfs /tmp nginx:1.28-alpine

# Drop all capabilities, add back only what is needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx:1.28-alpine

# Prevent privilege escalation
docker run --security-opt no-new-privileges:true nginx:1.28-alpine

# Seccomp / AppArmor profiles
docker run --security-opt seccomp=profile.json nginx:1.28-alpine   # avoid seccomp=unconfined
docker run --security-opt apparmor=docker-default nginx:1.28-alpine

# Misc hardening
docker run --pids-limit 100 --memory 512m --ulimit nofile=1024:2048 nginx:1.28-alpine
```

### Image Security

```bash
# Scan for vulnerabilities (Docker Scout)
docker scout quickview nginx:1.28-alpine
docker scout cves nginx:1.28-alpine
docker scout recommendations nginx:1.28-alpine

# Alternatives: trivy image nginx:1.28-alpine ; grype nginx:1.28-alpine
```

- Pin versions and digests (`nginx:1.28-alpine@sha256:...`) — `latest` is a moving target.
- Keep images minimal (Alpine/Slim/Distroless/`scratch`); every package is attack surface.
- Never bake secrets into layers — use BuildKit `--mount=type=secret` (see Dockerfile cheatsheet).
- Sign and attest images with [Sigstore cosign](https://docs.sigstore.dev/) or Notation; the legacy Docker Content Trust (Notary v1) is deprecated.

### Host & Daemon Hardening

```bash
# User namespaces: map container root to an unprivileged host user
# /etc/docker/daemon.json: { "userns-remap": "default" }

# Rootless mode (Linux)
dockerd-rootless-setuptool.sh install
```

- Keep the Engine patched; restrict who can access the Docker socket (it is effectively root).
- Prefer a rootless or dedicated CI runner over mounting `/var/run/docker.sock` into containers.
- Isolate workloads with user-defined, `--internal` networks and drop unused ports.

## Cleanup & Maintenance

| Command | Removes |
|---------|---------|
| `docker container prune` | All stopped containers |
| `docker image prune` | Dangling (untagged) images |
| `docker image prune -a` | All images not used by a container |
| `docker volume prune` | Unused anonymous volumes (`-a` includes named) |
| `docker network prune` | Unused networks |
| `docker builder prune` | Build cache (`--all` removes everything) |
| `docker buildx prune` | BuildKit cache for a specific builder |
| `docker system prune` | Containers + networks + dangling images + build cache |
| `docker system prune -a --volumes` | Everything unused, including images and volumes |

```bash
docker system df                 # summary
docker system df -v              # per-object breakdown
docker system prune -a --filter "until=720h" -f   # nothing used in the last 30 days
```

> `docker system prune -a --volumes` deletes **data**. Never run it on a machine with stateful volumes you care about.

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| `port is already allocated` | Another process/container uses the port: `docker ps -f publish=8080`, then stop it or use another port |
| Container exits immediately | Wrong `CMD`/entrypoint: `docker logs <c>`, then `docker run -it --entrypoint sh <image>` |
| `Cannot connect to the Docker daemon` | Daemon not running, wrong context (`docker context ls`), or bad `DOCKER_HOST` |
| `permission denied` on bind mount | UID mismatch: run with `--user $(id -u):$(id -g)`, fix host permissions, or use a named volume |
| `no space left on device` | `docker system df`, then `docker system prune -a` / `docker builder prune -a` |
| `exec format error` | Image built for another CPU arch: rebuild with `--platform` or pull the matching one |
| Can't reach a container by name | Containers are not on the same user-defined network |
| DNS resolution fails inside container | Add `--dns 8.8.8.8`, check host firewall/iptables, avoid default-bridge name resolution |
| Container becomes `unhealthy` | `docker inspect <c>`, run the health command manually via `docker exec` |
| Build fails on `COPY` with "not found" | File excluded by `.dockerignore` or wrong build context |
| Windows: entrypoint.sh fails with `\r` | CRLF line endings: convert scripts to LF |
| Daemon won't start (Linux) | `journalctl -u docker --no-pager -n 100`; invalid `daemon.json` is a common cause |

Debug toolbox:

```bash
docker run --rm -it --network container:<container> nicolaka/netshoot   # dig, curl, tcpdump, …
docker exec -it <container> sh
docker inspect <container> | jq '.[0].State'
docker events --since 1h
```

## Common Flags Reference

### `docker ps`

| Flag | Description |
|------|-------------|
| `-a` | Include stopped containers |
| `-q` | IDs only |
| `-s` | Show writable-layer size |
| `-f`, `--filter` | Filter by `status`, `health`, `name`, `label`, `ancestor`, `publish`, `network`, `volume`, … |
| `--format` | Go template / `table …` output |

### `docker logs`

| Flag | Description |
|------|-------------|
| `-f` | Follow output |
| `--tail N` | Last N lines |
| `--since` / `--until` | Time window (`10m`, `2026-01-01T00:00:00`) |
| `-t`, `--timestamps` | Prefix timestamps |

### `docker exec`

| Flag | Description |
|------|-------------|
| `-i`, `-t` | Keep stdin open / allocate TTY |
| `-u` | Run as user |
| `-w` | Working directory |
| `-e` | Environment variable |
| `--privileged` | Full host access (avoid) |

### `docker build` / `buildx build`

| Flag | Description |
|------|-------------|
| `-t`, `--tag` | Name and tag |
| `-f`, `--file` | Dockerfile path |
| `--build-arg` | Build-time variable |
| `--target` | Stop at a stage |
| `--no-cache` | Ignore build cache |
| `--pull` | Always refresh base images |
| `--platform` | Target platform(s) |
| `--secret`, `--ssh` | Build secrets / agent forwarding |
| `--cache-from` / `--cache-to` | Cache import/export |
| `--output` | Export the result (`type=local,dest=out`) |
| `--check` | Run Dockerfile linter |
| `--progress` | `auto`, `tty`, `plain` |

## Quick Reference

```bash
# The essentials
docker ps -a                                    # all containers
docker images -a                                # all images
docker run --rm -it alpine sh                   # throwaway shell
docker exec -it <c> sh                          # shell into running container
docker logs -f <c>                              # follow logs
docker inspect <c>                              # full metadata
docker stats                                    # live resources
docker system df                                # disk usage

# Compose
docker compose up -d --build
docker compose logs -f
docker compose down -v

# Cleanup
docker system prune -a --volumes                # destructive: removes unused data

# Backup image / volume
docker save app:1.0 | gzip > app-1.0.tar.gz
docker run --rm -v data:/data -v "$PWD":/bk alpine tar czf /bk/data.tar.gz -C /data .
```

## References & Further Reading

Official documentation:

- [Docker documentation](https://docs.docker.com/) — the source of truth
- [Get started workshop](https://docs.docker.com/get-started/workshop/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Compose CLI reference](https://docs.docker.com/reference/cli/docker/compose/)
- [Building with BuildKit](https://docs.docker.com/build/buildkit/)
- [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)
- [Build cache](https://docs.docker.com/build/cache/)
- [Build attestations (SBOM/provenance)](https://docs.docker.com/build/attestations/)
- [Build checks (Dockerfile linter)](https://docs.docker.com/build/checks/)
- [Docker Scout (vulnerability scanning)](https://docs.docker.com/scout/)
- [Engine networking](https://docs.docker.com/engine/network/)
- [Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Container resource constraints](https://docs.docker.com/engine/containers/resource_constraints/)
- [Engine security](https://docs.docker.com/engine/security/)
- [Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [Logging drivers](https://docs.docker.com/engine/logging/)

Images, samples & tools:

- [Docker Hub](https://hub.docker.com/) — official and community images
- [Docker Official Images](https://hub.docker.com/search?image_filter=official) — curated, maintained base images
- [docker/awesome-compose](https://github.com/docker/awesome-compose) — production-ready Compose samples
- [dockersamples](https://github.com/dockersamples) — official demo projects
- [compose-spec](https://github.com/compose-spec/compose-spec) — the Compose Specification
- [Hadolint](https://github.com/hadolint/hadolint) — Dockerfile linter
- [dive](https://github.com/wagoodman/dive) — explore image layers and wasted space
- [Trivy](https://github.com/aquasecurity/trivy) / [Grype](https://github.com/anchore/grype) — vulnerability scanners
- [Distroless images](https://github.com/GoogleContainerTools/distroless) — minimal runtime images
- [netshoot](https://github.com/nicolaka/netshoot) — network troubleshooting container
- [Play with Docker](https://labs.play-with-docker.com/) — free browser sandbox

---

**Note**: Covers the most commonly used Docker features for Docker Engine 28+ and Compose v2. For advanced usage refer to the [official Docker documentation](https://docs.docker.com/) and [Docker CLI reference](https://docs.docker.com/reference/cli/docker/).
