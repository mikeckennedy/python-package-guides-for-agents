# Docker Comprehensive Reference

A detailed reference for AI agents and developers working with Docker.
Distilled from the official Docker documentation source at
[docs.docker.com](https://docs.docker.com/).

---

## Table of Contents

- [1. Docker CLI Commands](#1-docker-cli-commands)
  - [Container Management](#container-management)
  - [Image Management](#image-management)
  - [Volume Management](#volume-management)
  - [Network Management](#network-management)
  - [System and Info](#system-and-info)
- [2. Dockerfile Reference](#2-dockerfile-reference)
  - [Parser Directives](#parser-directives)
  - [Instructions](#instructions)
  - [Shell vs Exec Form](#shell-vs-exec-form)
  - [Multi-Stage Builds](#multi-stage-builds)
  - [RUN Mount Options](#run-mount-options)
  - [COPY/ADD Options](#copyadd-options)
  - [Environment Variable Substitution](#environment-variable-substitution)
  - [.dockerignore](#dockerignore)
  - [CMD vs ENTRYPOINT Interaction](#cmd-vs-entrypoint-interaction)
- [3. Docker Compose Reference](#3-docker-compose-reference)
  - [File Structure](#file-structure)
  - [Service Configuration](#service-configuration)
  - [Network Configuration](#network-configuration-compose)
  - [Volume Configuration](#volume-configuration-compose)
  - [Configs and Secrets](#configs-and-secrets)
  - [Deploy Specification](#deploy-specification)
  - [Develop / Watch Mode](#develop--watch-mode)
  - [Environment Variable Handling](#environment-variable-handling)
  - [Profiles](#profiles)
  - [Compose CLI Commands](#compose-cli-commands)
- [4. Networking](#4-networking)
  - [Network Drivers](#network-drivers)
  - [Port Publishing](#port-publishing)
  - [DNS and Service Discovery](#dns-and-service-discovery)
- [5. Storage and Volumes](#5-storage-and-volumes)
  - [Volume Mounts](#volume-mounts)
  - [Bind Mounts](#bind-mounts)
  - [tmpfs Mounts](#tmpfs-mounts)
  - [Storage Drivers](#storage-drivers)
- [6. Build System (BuildKit / Buildx)](#6-build-system-buildkit--buildx)
  - [docker buildx build Options](#docker-buildx-build-options)
  - [Build Cache Strategies](#build-cache-strategies)
  - [Multi-Platform Builds](#multi-platform-builds)
  - [Build Secrets and SSH](#build-secrets-and-ssh)
  - [Bake Reference](#bake-reference)
  - [Builder Drivers](#builder-drivers)
  - [Build Exporters](#build-exporters)
- [7. Docker Engine Configuration](#7-docker-engine-configuration)
  - [Daemon Configuration](#daemon-configuration)
  - [Engine API](#engine-api)
  - [Security Features](#security-features)
  - [Logging Drivers](#logging-drivers)
  - [Container Runtimes](#container-runtimes)
  - [Docker Contexts](#docker-contexts)

---

## 1. Docker CLI Commands

### Container Management

#### `docker run`

Create and run a new container from an image.

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

| Flag | Short | Type | Description |
|------|-------|------|-------------|
| `--detach` | `-d` | bool | Run in background |
| `--interactive` | `-i` | bool | Keep STDIN open |
| `--tty` | `-t` | bool | Allocate pseudo-TTY |
| `--publish` | `-p` | list | Publish port(s): `HOST:CONTAINER` |
| `--volume` | `-v` | list | Bind mount a volume |
| `--mount` | | mount | Attach a filesystem mount |
| `--env` | `-e` | list | Set environment variables |
| `--env-file` | | list | Read env vars from file |
| `--name` | | string | Assign container name |
| `--network` | | network | Connect to network |
| `--restart` | | string | Restart policy: `no`, `always`, `on-failure[:max]`, `unless-stopped` |
| `--rm` | | bool | Auto-remove container on exit (works with `-d` too) |
| `--privileged` | | bool | Give extended privileges |
| `--user` | `-u` | string | Username or UID |
| `--workdir` | `-w` | string | Working directory |
| `--entrypoint` | | string | Override entrypoint |
| `--memory` | `-m` | bytes | Memory limit |
| `--cpus` | | decimal | Number of CPUs |
| `--cpu-shares` | `-c` | int64 | Relative CPU weight |
| `--gpus` | | gpu-request | GPU devices to add |
| `--platform` | | string | Target platform |
| `--pull` | | string | Pull policy: `always`, `missing`, `never` |
| `--read-only` | | bool | Read-only root filesystem |
| `--cap-add` | | list | Add Linux capabilities |
| `--cap-drop` | | list | Drop Linux capabilities |
| `--security-opt` | | list | Security options |
| `--label` | `-l` | list | Set metadata labels |
| `--log-driver` | | string | Logging driver |
| `--log-opt` | | list | Logging driver options |
| `--health-cmd` | | string | Health check command |
| `--health-interval` | | duration | Check interval |
| `--health-retries` | | int | Retries before unhealthy |
| `--health-timeout` | | duration | Check timeout |
| `--dns` | | list | Custom DNS servers |
| `--hostname` | `-h` | string | Container hostname |
| `--add-host` | | list | Add host-to-IP mapping |
| `--tmpfs` | | list | Mount tmpfs directory |
| `--device` | | list | Add host device |
| `--init` | | bool | Run init as PID 1 |
| `--pid` | | string | PID namespace: `host` |
| `--ipc` | | string | IPC mode |
| `--ulimit` | | ulimit | Ulimit options |
| `--sysctl` | | map | Sysctl options |
| `--stop-signal` | | string | Signal to stop container |
| `--stop-timeout` | | int | Timeout (seconds) to stop |
| `--shm-size` | | bytes | Size of `/dev/shm` |
| `--storage-opt` | | list | Storage driver options |

#### `docker container ls` / `docker ps`

List containers.

```
docker container ls [OPTIONS]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--all` | `-a` | Show all containers (default: running only) |
| `--filter` | `-f` | Filter by condition |
| `--format` | | Go template output |
| `--quiet` | `-q` | Only display container IDs |
| `--size` | `-s` | Display total file sizes |
| `--last` | `-n` | Show n last created containers |
| `--latest` | `-l` | Show the latest created container |
| `--no-trunc` | | Don't truncate output |

#### `docker container stop`

```
docker container stop [OPTIONS] CONTAINER [CONTAINER...]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--time` | `-t` | Seconds to wait before SIGKILL (default: 10) |
| `--signal` | `-s` | Signal to send |

#### `docker container rm`

```
docker container rm [OPTIONS] CONTAINER [CONTAINER...]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--force` | `-f` | Force remove (SIGKILL running container) |
| `--volumes` | `-v` | Remove anonymous volumes |
| `--link` | `-l` | Remove specified link |

#### `docker container exec`

Execute a command in a running container.

```
docker container exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--detach` | `-d` | Run in background |
| `--interactive` | `-i` | Keep STDIN open |
| `--tty` | `-t` | Allocate pseudo-TTY |
| `--user` | `-u` | Username or UID |
| `--workdir` | `-w` | Working directory |
| `--env` | `-e` | Set environment variables |
| `--privileged` | | Extended privileges |

#### `docker container logs` / `docker logs`

```
docker logs [OPTIONS] CONTAINER
```

| Flag | Short | Description |
|------|-------|-------------|
| `--follow` | `-f` | Follow log output |
| `--tail` | `-n` | Number of lines from end |
| `--timestamps` | `-t` | Show timestamps |
| `--since` | | Show logs since timestamp or relative |
| `--until` | | Show logs before timestamp or relative |
| `--details` | | Show extra details |

#### `docker container inspect` / `docker inspect`

```
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--format` | `-f` | Format output using Go template |
| `--size` | `-s` | Display total file sizes (containers) |
| `--type` | | Return JSON for specified type |

#### Other Container Commands

```
docker container start CONTAINER [CONTAINER...]
docker container restart [--time|-t SECONDS] CONTAINER [CONTAINER...]
docker container pause CONTAINER [CONTAINER...]
docker container unpause CONTAINER [CONTAINER...]
docker container kill [--signal|-s SIGNAL] CONTAINER [CONTAINER...]
docker container rename CONTAINER NEW_NAME
docker container cp [OPTIONS] SRC DEST              # Copy files between container and host
docker container diff CONTAINER                      # Inspect filesystem changes
docker container top CONTAINER [ps OPTIONS]          # Display running processes
docker container stats [OPTIONS] [CONTAINER...]      # Live resource usage
docker container wait CONTAINER [CONTAINER...]       # Block until container stops
docker container attach [OPTIONS] CONTAINER          # Attach to running container
docker container commit [OPTIONS] CONTAINER [REPO[:TAG]]  # Create image from container
docker container export [OPTIONS] CONTAINER          # Export filesystem as tar
docker container update [OPTIONS] CONTAINER          # Update resource limits
docker container prune [OPTIONS]                     # Remove stopped containers
```

### Image Management

#### `docker build` / `docker buildx build`

```
docker build [OPTIONS] PATH | URL | -
```

| Flag | Short | Description |
|------|-------|-------------|
| `--tag` | `-t` | Name and optional tag: `name:tag` |
| `--file` | `-f` | Dockerfile path |
| `--build-arg` | | Set build-time variables |
| `--target` | | Set target build stage |
| `--no-cache` | | Do not use cache |
| `--pull` | | Always pull base images |
| `--load` | | Load result to Docker (shorthand) |
| `--push` | | Push result to registry (shorthand) |
| `--platform` | | Target platform(s) |
| `--cache-from` | | External cache source |
| `--cache-to` | | Cache export destination |
| `--secret` | | Secret to expose to build |
| `--ssh` | | SSH agent socket or keys |
| `--output` | `-o` | Output destination |
| `--progress` | | Progress type: `auto`, `plain`, `tty`, `quiet` |
| `--network` | | Network mode for RUN instructions |
| `--build-context` | | Additional named build contexts |
| `--annotation` | | Add image annotations |
| `--attest` | | Attestation parameters |
| `--provenance` | | Provenance attestation shorthand |
| `--check` | | Validate Dockerfile without building |
| `--metadata-file` | | Write build metadata to file |

#### `docker image ls` / `docker images`

```
docker image ls [OPTIONS] [REPOSITORY[:TAG]]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--all` | `-a` | Show all images (including intermediate) |
| `--filter` | `-f` | Filter output |
| `--format` | | Go template output |
| `--quiet` | `-q` | Only show image IDs |
| `--digests` | | Show digests |
| `--no-trunc` | | Don't truncate output |

#### `docker pull`

```
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--all-tags` | `-a` | Download all tagged images |
| `--platform` | | Set platform |
| `--quiet` | `-q` | Suppress verbose output |

#### `docker push`

```
docker push [OPTIONS] NAME[:TAG]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--all-tags` | `-a` | Push all tagged images |
| `--quiet` | `-q` | Suppress verbose output |

#### `docker tag`

```
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

#### Other Image Commands

```
docker image rm IMAGE [IMAGE...]          # Remove images (alias: docker rmi)
docker image prune [OPTIONS]              # Remove unused images
docker image history IMAGE                # Show image layer history
docker image save [OPTIONS] IMAGE         # Save to tar archive
docker image load [OPTIONS]               # Load from tar archive
docker image import [OPTIONS] file|URL    # Import from tarball
```

### Volume Management

#### `docker volume create`

```
docker volume create [OPTIONS] [VOLUME]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--driver` | `-d` | Volume driver (default: `local`) |
| `--opt` | `-o` | Driver-specific options |
| `--label` | | Set metadata labels |

#### `docker volume ls`

```
docker volume ls [OPTIONS]
```

| Flag | Short | Description |
|------|-------|-------------|
| `--filter` | `-f` | Filter: `dangling`, `driver`, `label`, `name` |
| `--format` | | Go template output |
| `--quiet` | `-q` | Only display volume names |

#### Other Volume Commands

```
docker volume inspect VOLUME [VOLUME...]
docker volume rm VOLUME [VOLUME...]
docker volume prune [OPTIONS]
```

### Network Management

#### `docker network create`

```
docker network create [OPTIONS] NETWORK
```

| Flag | Short | Description |
|------|-------|-------------|
| `--driver` | `-d` | Driver: `bridge`, `overlay`, `macvlan`, `ipvlan`, `none` |
| `--subnet` | | Subnet in CIDR format |
| `--gateway` | | Gateway for the subnet |
| `--ip-range` | | Allocate IPs from a range |
| `--internal` | | Restrict external access |
| `--attachable` | | Allow manual container attachment (overlay) |
| `--opt` | `-o` | Driver-specific options |
| `--label` | | Set metadata labels |
| `--ipv6` | | Enable IPv6 |

#### `docker network connect`

```
docker network connect [OPTIONS] NETWORK CONTAINER
```

| Flag | Description |
|------|-------------|
| `--ip` | IPv4 address |
| `--ip6` | IPv6 address |
| `--alias` | Network-scoped alias |
| `--link` | Add link to another container |

#### Other Network Commands

```
docker network disconnect [--force] NETWORK CONTAINER
docker network ls [OPTIONS]
docker network inspect NETWORK [NETWORK...]
docker network rm NETWORK [NETWORK...]
docker network prune [OPTIONS]
```

### System and Info

```
docker version                           # Show Docker version
docker info                              # Display system-wide information
docker system df                         # Show disk usage
docker system prune [OPTIONS]            # Remove unused data
docker system events [OPTIONS]           # Get real-time events
docker login [OPTIONS] [SERVER]          # Log in to registry
docker logout [SERVER]                   # Log out from registry
docker search [OPTIONS] TERM             # Search Docker Hub
```

---

## 2. Dockerfile Reference

### Parser Directives

Must appear at the very top of the Dockerfile, before any instruction or comment.

```dockerfile
# syntax=docker/dockerfile:1
# escape=\
# check=skip=JSONArgsRecommended
```

| Directive | Description |
|-----------|-------------|
| `# syntax=<image>` | Dockerfile frontend image (recommended: `docker/dockerfile:1`) |
| `# escape=<char>` | Escape character for line continuation (default: `\`) |
| `# check=skip=<rules>` | Skip specific build checks |
| `# check=error=true` | Fail build on warnings |

### Instructions

#### FROM

Initialize a new build stage from a base image.

```dockerfile
FROM [--platform=<platform>] <image>[:<tag>|@<digest>] [AS <name>]
```

- Must be the first instruction (except ARG and parser directives)
- `AS <name>` creates a named stage for multi-stage builds
- `--platform` forces a specific platform (e.g., `linux/amd64`)

#### RUN

Execute commands in a new layer.

```dockerfile
# Shell form
RUN <command>

# Exec form
RUN ["executable", "param1", "param2"]

# With options
RUN [OPTIONS] <command>
```

Options: `--mount`, `--network`, `--security`, `--device`

#### CMD

Default command for container execution.

```dockerfile
CMD ["executable", "param1", "param2"]     # Exec form (preferred)
CMD ["param1", "param2"]                   # Default parameters for ENTRYPOINT
CMD command param1 param2                  # Shell form
```

Only the last CMD takes effect. Overridden by `docker run` arguments.

#### ENTRYPOINT

Configure the container as an executable.

```dockerfile
ENTRYPOINT ["executable", "param1"]        # Exec form (preferred)
ENTRYPOINT command param1                  # Shell form
```

Exec form receives `docker run` arguments; shell form ignores them.

#### COPY

Copy files from build context into the image.

```dockerfile
COPY [OPTIONS] <src>... <dest>
COPY [OPTIONS] ["<src>", ... "<dest>"]
```

Options: `--from`, `--chmod`, `--chown`, `--link`, `--parents`, `--exclude`

#### ADD

Add files from local or remote sources (auto-extracts archives).

```dockerfile
ADD [OPTIONS] <src>... <dest>
```

Options: `--keep-git-dir`, `--checksum`, `--chmod`, `--chown`, `--link`, `--exclude`

#### ENV

Set environment variables (persisted in the final image).

```dockerfile
ENV <key>=<value> [<key>=<value>...]
```

#### ARG

Define build-time variables (not persisted in image).

```dockerfile
ARG <name>[=<default value>]
```

Set via `--build-arg KEY=VALUE`. Scoped to the build stage where declared.

**Pre-defined ARGs:**

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY`, `HTTPS_PROXY`, `FTP_PROXY`, `NO_PROXY`, `ALL_PROXY` | Proxy settings (excluded from cache key) |
| `TARGETPLATFORM` | Target platform (e.g., `linux/amd64`) |
| `TARGETOS` | Target OS component |
| `TARGETARCH` | Target architecture |
| `TARGETVARIANT` | Target variant |
| `BUILDPLATFORM` | Builder's platform |
| `BUILDOS`, `BUILDARCH`, `BUILDVARIANT` | Builder's OS/arch/variant |

#### WORKDIR

Set working directory for subsequent instructions.

```dockerfile
WORKDIR /path/to/workdir
```

Created if it doesn't exist. Used by RUN, CMD, ENTRYPOINT, COPY, ADD.

#### EXPOSE

Document which ports the container listens on (informational only).

```dockerfile
EXPOSE <port>[/<protocol>]
```

Does not publish ports. Use `-p` at runtime to publish.

#### VOLUME

Create a mount point for externally mounted volumes.

```dockerfile
VOLUME ["/data"]
VOLUME /var/log /var/db
```

#### LABEL

Add metadata to the image.

```dockerfile
LABEL <key>=<value> [<key>=<value>...]
```

#### USER

Set user and optionally group for subsequent instructions.

```dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

#### HEALTHCHECK

Define container health check.

```dockerfile
HEALTHCHECK [OPTIONS] CMD <command>
HEALTHCHECK NONE
```

| Option | Default | Description |
|--------|---------|-------------|
| `--interval` | 30s | Time between checks |
| `--timeout` | 30s | Check timeout |
| `--start-period` | 0s | Grace period for startup |
| `--start-interval` | 5s | Check interval during start period |
| `--retries` | 3 | Failures before unhealthy |

Exit codes: 0 = healthy, 1 = unhealthy.

#### SHELL

Set default shell for shell-form instructions.

```dockerfile
SHELL ["executable", "parameters"]
```

Default: `["/bin/sh", "-c"]` on Linux, `["cmd", "/S", "/C"]` on Windows.

#### STOPSIGNAL

Set the system call signal for container exit.

```dockerfile
STOPSIGNAL signal
```

Default: `SIGTERM`.

#### ONBUILD

Register trigger instructions for downstream builds.

```dockerfile
ONBUILD <INSTRUCTION>
```

### Shell vs Exec Form

| Form | Syntax | Shell invocation | Variable expansion |
|------|--------|-----------------|-------------------|
| **Exec** | `["executable", "arg"]` | No | No |
| **Shell** | `executable arg` | `/bin/sh -c` | Yes |

Exec form is preferred for CMD and ENTRYPOINT. Shell form supports pipes, redirection, and variable substitution.

### Here-Documents

```dockerfile
RUN <<EOT bash
  set -ex
  apt-get update
  apt-get install -y vim
EOT

COPY <<EOF /greeting.txt
hello world
EOF
```

### Multi-Stage Builds

```dockerfile
# Build stage
FROM golang:1.22 AS build
WORKDIR /app
COPY . .
RUN go build -o /myapp ./cmd

# Production stage
FROM scratch
COPY --from=build /myapp /usr/bin/
ENTRYPOINT ["/usr/bin/myapp"]
```

- Name stages with `AS <name>`
- Copy between stages with `COPY --from=<stage>`
- Copy from external images: `COPY --from=nginx:latest /etc/nginx/nginx.conf /`
- Build up to a specific stage: `docker build --target build .`

### RUN Mount Options

#### Cache mount

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

#### Bind mount

```dockerfile
RUN --mount=type=bind,from=build,target=/src \
    cp /src/config.json .
```

#### Secret mount

```dockerfile
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/file .
```

#### SSH mount

```dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:user/repo.git
```

#### tmpfs mount

```dockerfile
RUN --mount=type=tmpfs,target=/tmp \
    compile_something
```

#### Network modes

```dockerfile
RUN --network=none pip install --find-links wheels mypackage
RUN --network=host curl https://example.com
```

### COPY/ADD Options

| Option | Description |
|--------|-------------|
| `--from=<stage\|image>` | Copy from a build stage or external image |
| `--chmod=<perms>` | Set file permissions (octal or symbolic) |
| `--chown=<user>:<group>` | Set file ownership |
| `--link` | Create independent layer for better cache reuse |
| `--parents` | Preserve parent directory structure |
| `--exclude=<pattern>` | Exclude files matching pattern |

### Environment Variable Substitution

Supported in: ADD, COPY, ENV, EXPOSE, FROM, LABEL, STOPSIGNAL, USER, WORKDIR.

```dockerfile
${VAR}            # Standard substitution
${VAR:-default}   # Use default if VAR is unset or empty
${VAR:+value}     # Use value if VAR is set and non-empty
```

### .dockerignore

Located at the build context root (or `<Dockerfile>.dockerignore` for Dockerfile-specific).

```
# Comment
*/temp*           # Exclude temp in immediate subdirectories
**/node_modules   # Exclude at any depth
*.md              # Exclude all markdown
!README.md        # Except README.md
.git
.env
```

Uses Go's `filepath.Match` rules. `**` matches any directory depth. Last matching rule wins.

### CMD vs ENTRYPOINT Interaction

|  | No ENTRYPOINT | `ENTRYPOINT ["exec", "p1"]` | `ENTRYPOINT exec p1` |
|--|---------------|---------------------------|---------------------|
| **No CMD** | Error, not allowed | `exec p1` | `/bin/sh -c exec p1` |
| **`CMD ["c1", "c2"]`** | `c1 c2` | `exec p1 c1 c2` | `/bin/sh -c exec p1` |
| **`CMD c1 c2`** | `/bin/sh -c c1 c2` | `exec p1 /bin/sh -c c1 c2` | `/bin/sh -c exec p1` |

---

## 3. Docker Compose Reference

### File Structure

Default filenames: `compose.yaml`, `compose.yml`, `docker-compose.yaml`, `docker-compose.yml`.

```yaml
name: my-project              # Project name

services:                     # Service definitions (required)
  webapp:
    image: nginx:latest

networks:                     # Custom networks
  frontend:

volumes:                      # Named volumes
  db-data:

configs:                      # Configuration objects
  app-config:
    file: ./config.yaml

secrets:                      # Sensitive data
  db-password:
    file: ./db-password.txt

include:                      # Include other Compose files
  - path: ./other-compose.yaml
```

### Service Configuration

#### Image and Build

```yaml
services:
  app:
    image: myapp:latest
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_VERSION: "20"
      target: production
      cache_from:
        - type=registry,ref=myapp:cache
      cache_to:
        - type=registry,ref=myapp:cache,mode=max
      platforms:
        - linux/amd64
        - linux/arm64
      secrets:
        - db_password
      ssh:
        - default
      labels:
        com.example.version: "1.0"
    pull_policy: missing        # always | never | missing | build
    platform: linux/amd64
```

#### Container Runtime

```yaml
services:
  app:
    command: ["python", "app.py"]           # Override CMD
    entrypoint: ["/docker-entrypoint.sh"]   # Override ENTRYPOINT
    working_dir: /app
    user: "1000:1000"
    init: true                              # Run init as PID 1
    hostname: myhost
    domainname: example.com
    container_name: my-app                  # Custom name (disables scaling)
    stdin_open: true                        # -i equivalent
    tty: true                               # -t equivalent
    read_only: true                         # Read-only root filesystem
    privileged: false
    stop_signal: SIGTERM
    stop_grace_period: 30s
```

#### Ports

```yaml
services:
  app:
    ports:
      # Short syntax
      - "8080:80"                 # HOST:CONTAINER
      - "443:443/udp"            # With protocol
      - "127.0.0.1:8080:80"     # Specific host IP
      - "80"                      # Random host port

      # Long syntax
      - target: 80
        published: 8080
        host_ip: 0.0.0.0
        protocol: tcp
        app_protocol: http
    expose:
      - "3000"                    # Internal only
```

#### Volumes

```yaml
services:
  app:
    volumes:
      # Short syntax
      - db-data:/var/lib/data            # Named volume
      - ./src:/app/src                   # Bind mount
      - /tmp/cache:/tmp/cache:ro         # Read-only bind

      # Long syntax
      - type: volume
        source: db-data
        target: /var/lib/data
        read_only: false
        volume:
          nocopy: true
          subpath: subdir
      - type: bind
        source: ./src
        target: /app/src
        bind:
          create_host_path: true
          selinux: z
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100000000
          mode: 1777
```

#### Environment Variables

```yaml
services:
  app:
    environment:
      # Map syntax
      NODE_ENV: production
      DB_HOST: db
      # Array syntax
      # - NODE_ENV=production
      # - DB_HOST=db

    env_file:
      - .env
      - path: .env.local
        required: false
        format: raw
```

#### Dependencies and Health

```yaml
services:
  app:
    depends_on:
      # Short syntax
      # - db
      # - redis

      # Long syntax
      db:
        condition: service_healthy
        restart: true
        required: true
      redis:
        condition: service_started

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s
    restart: unless-stopped        # no | always | on-failure[:max] | unless-stopped
```

#### Networking

```yaml
services:
  app:
    networks:
      frontend:
        aliases:
          - webapp
        ipv4_address: 172.16.238.10
        priority: 1000
      backend:
    network_mode: host             # none | host | service:{name}
    dns:
      - 8.8.8.8
    dns_search:
      - example.com
    extra_hosts:
      - "host.docker.internal:host-gateway"
    mac_address: "02:42:ac:11:00:02"
    links:
      - db:database                # Legacy
```

#### Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 512M
          pids: 100
        reservations:
          cpus: "0.5"
          memory: 128M

    # Legacy resource options (also supported)
    cpus: 2.0
    mem_limit: 512m
    mem_reservation: 128m
    memswap_limit: 1g
    shm_size: 64m
    pids_limit: 100
    oom_kill_disable: false
    oom_score_adj: -500

    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc: 65535
```

#### Security and Capabilities

```yaml
services:
  app:
    cap_add:
      - NET_ADMIN
      - SYS_PTRACE
    cap_drop:
      - ALL
    security_opt:
      - seccomp:unconfined
      - apparmor:docker-default
      - no-new-privileges:true
    userns_mode: "host"
    ipc: shareable
    pid: host
    sysctls:
      net.core.somaxconn: 1024
      net.ipv4.tcp_syncookies: 0
    devices:
      - /dev/sda:/dev/xvdc:rwm
    group_add:
      - "999"
```

#### Logging

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

#### Labels and Annotations

```yaml
services:
  app:
    labels:
      com.example.description: "My app"
      com.example.version: "1.0"
    annotations:
      com.example.note: "Annotation value"
```

#### Extends

```yaml
services:
  app:
    extends:
      file: common-services.yaml
      service: base-app
```

### Network Configuration (Compose)

```yaml
networks:
  frontend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: frontend
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 172.16.238.0/24
          gateway: 172.16.238.1
          ip_range: 172.16.238.128/25
    internal: false
    attachable: false
    labels:
      com.example.network: frontend

  external-net:
    external: true
    name: my-pre-existing-network

  default:                          # Customize default network
    driver: bridge
```

### Volume Configuration (Compose)

```yaml
volumes:
  db-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.1.0.1,rw
      device: ":/exported/path"
    labels:
      com.example.purpose: database

  existing-volume:
    external: true
    name: actual-volume-name
```

### Configs and Secrets

```yaml
configs:
  app-config:
    file: ./config.yaml           # From file
  inline-config:
    content: |                    # Inline content
      key: value
  env-config:
    environment: CONFIG_VAR       # From environment variable
  ext-config:
    external: true                # Pre-existing config
    name: actual-config-name

secrets:
  db-password:
    file: ./db-password.txt
  api-key:
    environment: API_KEY
  ext-secret:
    external: true
```

**Service-level access:**

```yaml
services:
  app:
    configs:
      - app-config                           # Short: mounted at /<config-name>
      - source: app-config                   # Long syntax
        target: /etc/app/config.yaml
        uid: "1000"
        gid: "1000"
        mode: 0440

    secrets:
      - db-password                          # Short: mounted at /run/secrets/<name>
      - source: api-key                      # Long syntax
        target: /run/secrets/my-api-key
        mode: 0400
```

### Deploy Specification

```yaml
services:
  app:
    deploy:
      mode: replicated             # replicated | global
      replicas: 3
      endpoint_mode: vip           # vip | dnsrr

      placement:
        constraints:
          - node.role == manager
        preferences:
          - spread: datacenter

      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback   # pause | continue | rollback
        monitor: 30s
        max_failure_ratio: 0.2
        order: start-first         # stop-first | start-first

      rollback_config:
        parallelism: 1
        delay: 5s
        failure_action: pause

      restart_policy:
        condition: on-failure      # none | any | on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

      resources:
        limits:
          cpus: "2.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 128M

      labels:
        com.example.service: app
```

### Develop / Watch Mode

```yaml
services:
  app:
    develop:
      watch:
        - path: ./src
          action: sync
          target: /app/src
          ignore:
            - "**/*.test.js"
        - path: ./package.json
          action: rebuild
        - path: ./config
          action: sync+restart
          target: /app/config
```

| Action | Description |
|--------|-------------|
| `sync` | Sync files to container (hot reload) |
| `rebuild` | Rebuild image and recreate container |
| `restart` | Restart the container |
| `sync+restart` | Sync then restart |
| `sync+exec` | Sync then run a command |

### Environment Variable Handling

**Interpolation syntax in Compose files:**

```yaml
services:
  app:
    image: "myapp:${TAG:-latest}"
    environment:
      DB_HOST: ${DB_HOST:?DB_HOST must be set}
```

| Syntax | Description |
|--------|-------------|
| `${VAR}` | Value of VAR |
| `${VAR:-default}` | Default if unset or empty |
| `${VAR-default}` | Default if unset |
| `${VAR:?error}` | Error if unset or empty |
| `${VAR?error}` | Error if unset |
| `${VAR:+replacement}` | Replacement if set and non-empty |
| `${VAR+replacement}` | Replacement if set |
| `$${ESCAPED}` | Literal `${ESCAPED}` |

**Precedence (highest to lowest):**

1. `docker compose run -e` or `--env-file` CLI flags
2. Shell environment variables
3. `.env` file in project directory
4. `environment:` attribute in Compose file
5. `env_file:` attribute in Compose file

### Profiles

```yaml
services:
  app:
    image: myapp
    # No profile: always started

  debug:
    image: debug-tools
    profiles:
      - debug                 # Only started when "debug" profile is active

  monitoring:
    profiles:
      - monitoring
      - debug                 # Started with either profile
```

Activate: `docker compose --profile debug up` or `COMPOSE_PROFILES=debug docker compose up`.

### Compose CLI Commands

```
docker compose up [OPTIONS] [SERVICE...]        # Create and start services
docker compose down [OPTIONS]                   # Stop and remove services
docker compose build [OPTIONS] [SERVICE...]     # Build or rebuild services
docker compose pull [OPTIONS] [SERVICE...]      # Pull service images
docker compose push [OPTIONS] [SERVICE...]      # Push service images
docker compose logs [OPTIONS] [SERVICE...]      # View service logs
docker compose ps [OPTIONS] [SERVICE...]        # List containers
docker compose exec SERVICE COMMAND [ARGS...]   # Execute in running service
docker compose run SERVICE [COMMAND] [ARGS...]  # Run one-off command
docker compose stop [OPTIONS] [SERVICE...]      # Stop services
docker compose start [SERVICE...]               # Start services
docker compose restart [OPTIONS] [SERVICE...]   # Restart services
docker compose pause [SERVICE...]               # Pause services
docker compose unpause [SERVICE...]             # Unpause services
docker compose kill [OPTIONS] [SERVICE...]      # Force stop services
docker compose rm [OPTIONS] [SERVICE...]        # Remove stopped containers
docker compose config [OPTIONS]                 # Validate and view config
docker compose cp SRC DEST                      # Copy files between service and host
docker compose top [SERVICE...]                 # Display running processes
docker compose images [OPTIONS] [SERVICE...]    # List images used by services
docker compose port SERVICE PRIVATE_PORT        # Print port binding
docker compose create [OPTIONS] [SERVICE...]    # Create containers
docker compose events [OPTIONS] [SERVICE...]    # Receive real-time events
docker compose watch [SERVICE...]               # Watch for file changes
docker compose alpha viz                        # Visualize Compose model
```

**Key `docker compose up` flags:**

| Flag | Description |
|------|-------------|
| `-d`, `--detach` | Run in background |
| `--build` | Build images before starting |
| `--force-recreate` | Recreate containers even if unchanged |
| `--no-deps` | Don't start dependencies |
| `--remove-orphans` | Remove containers not defined in file |
| `--scale SERVICE=NUM` | Scale service to NUM instances |
| `--wait` | Wait for services to be healthy |
| `--watch` | Watch for file changes and sync/rebuild |

**Key `docker compose down` flags:**

| Flag | Description |
|------|-------------|
| `-v`, `--volumes` | Remove named volumes |
| `--rmi all\|local` | Remove images |
| `--remove-orphans` | Remove containers not in file |
| `-t`, `--timeout` | Shutdown timeout (default: 10s) |

---

## 4. Networking

### Network Drivers

#### Bridge (default)

- Default for standalone containers
- User-defined bridges provide automatic DNS resolution between containers
- Containers on the same user-defined bridge can communicate by name
- Default bridge requires `--link` (legacy) or IP addresses for inter-container communication

```bash
docker network create --driver bridge my-network
docker run --network my-network --name web nginx
docker run --network my-network --name api myapi   # Can reach "web" by name
```

Key options:
- `com.docker.network.bridge.enable_ip_masquerade`: NAT for outbound traffic (default: true)
- `com.docker.network.bridge.enable_icc`: Inter-container communication (default: true)
- `com.docker.network.driver.mtu`: Set MTU
- Connection limit: 1000 containers per bridge

#### Host

- Container shares host's network namespace directly
- No network isolation: container uses host IP and ports
- Port mapping flags (`-p`) are ignored
- Best for performance-critical networking or large port ranges

```bash
docker run --network host nginx
```

#### Overlay

- Multi-host networking for Docker Swarm services
- Requires Swarm mode or `--attachable` flag for standalone containers
- Encryption via `--opt encrypted` (IPsec)
- Ports required: 2377/tcp, 4789/udp, 7946/tcp+udp

```bash
docker network create --driver overlay --attachable my-overlay
```

#### Macvlan

- Assigns a MAC address to each container, appearing as physical devices on the network
- Two modes: bridge (default) and 802.1Q trunk bridge
- Can't communicate with host directly
- Not supported on Docker Desktop

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 my-macvlan
```

#### IPvlan

- Similar to macvlan but shares host's MAC address
- Modes: L2 (default), L3, L3S
- Lower overhead than macvlan

```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  -o ipvlan_mode=l2 \
  -o parent=eth0 my-ipvlan
```

#### None

- Complete network isolation, only loopback device
- No IPv6 loopback address

```bash
docker run --network none alpine
```

### Port Publishing

```bash
-p 8080:80                    # Map host 8080 to container 80
-p 192.168.1.100:8080:80      # Bind to specific host IP
-p 8080:80/udp                # UDP protocol
-p 8080:80/tcp -p 8080:80/udp # Both TCP and UDP
-p 80                          # Random host port to container 80
-P                             # Publish all exposed ports to random ports
```

Gateway modes (per-network):
- `nat` (default): DNAT + masquerading
- `routed`: No NAT, direct routing
- `isolated`: No external access

### DNS and Service Discovery

- Embedded DNS server at `127.0.0.11` on user-defined networks
- Containers resolve each other by name or network alias
- DNS configuration flags: `--dns`, `--dns-search`, `--dns-opt`, `--hostname`
- Default bridge does NOT provide DNS resolution (use user-defined networks)

---

## 5. Storage and Volumes

### Volume Mounts

Named volumes are the preferred mechanism for persistent data. Managed by Docker.

```bash
# Create
docker volume create my-vol

# Mount (--mount syntax, preferred)
docker run --mount type=volume,src=my-vol,dst=/data myimage

# Mount (-v shorthand)
docker run -v my-vol:/data myimage

# Anonymous volume (auto-generated name)
docker run -v /data myimage

# Read-only
docker run --mount type=volume,src=my-vol,dst=/data,readonly myimage

# Sub-path mount
docker run --mount type=volume,src=my-vol,dst=/data,volume-subpath=subdir myimage
```

Key properties:
- Lifecycle independent of containers
- Sharable across multiple containers
- Data populated from container on first mount (if volume is empty)
- Can use remote drivers (NFS, cloud storage)

#### NFS Volume Example

```bash
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=10.1.0.1,rw \
  --opt device=:/exported/path \
  nfs-vol
```

### Bind Mounts

Direct mapping between host filesystem path and container path.

```bash
# --mount syntax (preferred)
docker run --mount type=bind,src=/host/path,dst=/container/path myimage

# -v shorthand
docker run -v /host/path:/container/path myimage

# Read-only
docker run --mount type=bind,src=/host/path,dst=/container/path,readonly myimage
docker run -v /host/path:/container/path:ro myimage

# SELinux labels
docker run -v /host/path:/container/path:z myimage    # Shared
docker run -v /host/path:/container/path:Z myimage    # Private
```

Key differences from volumes:
- `-v` auto-creates missing host path; `--mount` throws an error
- Rely on host filesystem structure
- Container processes can modify the host filesystem

### tmpfs Mounts

Temporary filesystem stored in host memory. Lost when container stops.

```bash
# --mount syntax
docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=100000000,tmpfs-mode=1777 myimage

# --tmpfs shorthand
docker run --tmpfs /tmp:size=100m,mode=1777 myimage
```

Properties:
- Not persisted to disk (may be written to swap)
- Not shared between containers
- Linux only
- Good for sensitive temporary data

### Storage Drivers

Storage drivers manage image layers and the container's writable layer.

| Driver | Description |
|--------|-------------|
| `overlay2` | Default, union filesystem (Linux 4.0+) |
| `btrfs` | Copy-on-write filesystem |
| `devicemapper` | Block device-based |
| `zfs` | Advanced filesystem |
| `vfs` | Simple, no copy-on-write (slow, testing only) |

Configure in `/etc/docker/daemon.json`:

```json
{
  "storage-driver": "overlay2"
}
```

Volumes are preferred over the writable layer for write-heavy workloads.

---

## 6. Build System (BuildKit / Buildx)

### docker buildx build Options

```
docker buildx build [OPTIONS] PATH | URL | -
```

See [Image Management > docker build](#docker-build--docker-buildx-build) for the full options table.

Additional buildx-specific options:

| Flag | Description |
|------|-------------|
| `--builder` | Override the configured builder instance |
| `--allow` | Allow extra privileged entitlements (`network.host`, `security.insecure`) |
| `--no-cache-filter` | Do not cache specific stages |

### Build Cache Strategies

#### Inline Cache

Embeds cache into the image itself. Simple but scales poorly with multi-stage builds.

```bash
docker buildx build \
  --cache-to type=inline \
  --cache-from type=registry,ref=myapp:latest \
  -t myapp:latest --push .
```

#### Registry Cache

Stores cache in a separate image in the registry. Recommended for CI/CD.

```bash
docker buildx build \
  --cache-to type=registry,ref=myapp:cache,mode=max \
  --cache-from type=registry,ref=myapp:cache \
  -t myapp:latest --push .
```

| Parameter | Values | Description |
|-----------|--------|-------------|
| `mode` | `min` (default), `max` | `max` caches all intermediate layers |
| `compression` | `gzip`, `estargz`, `zstd` | Compression algorithm |
| `oci-mediatypes` | `true` (default), `false` | Use OCI media types |

#### Local Cache

Stores cache as OCI layout in a local directory.

```bash
docker buildx build \
  --cache-to type=local,dest=/tmp/buildcache \
  --cache-from type=local,src=/tmp/buildcache .
```

#### GitHub Actions Cache

Uses GitHub Actions cache API. Recommended for GitHub workflows.

```bash
docker buildx build \
  --cache-to type=gha,mode=max \
  --cache-from type=gha .
```

#### S3 Cache

Uploads cache to AWS S3 or S3-compatible storage.

```bash
docker buildx build \
  --cache-to type=s3,region=us-east-1,bucket=my-cache,name=myapp \
  --cache-from type=s3,region=us-east-1,bucket=my-cache,name=myapp .
```

#### Multiple Cache Sources

```bash
docker buildx build \
  --cache-from type=registry,ref=myapp:branch-cache \
  --cache-from type=registry,ref=myapp:main-cache .
```

### Multi-Platform Builds

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
```

**Three strategies:**

1. **QEMU emulation** (easiest, slowest):
   ```bash
   docker run --privileged --rm tonistiigi/binfmt --install all
   docker buildx build --platform linux/amd64,linux/arm64 .
   ```

2. **Multiple native builder nodes** (faster):
   ```bash
   docker buildx create --use --name mybuild node-amd64
   docker buildx create --append --name mybuild node-arm64
   ```

3. **Cross-compilation** (fastest, language-dependent):
   ```dockerfile
   FROM --platform=$BUILDPLATFORM golang:1.22 AS build
   ARG TARGETOS TARGETARCH
   RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o /app .
   ```

### Build Secrets and SSH

#### Secrets

```bash
# Pass from file
docker buildx build --secret id=mytoken,src=./token.txt .

# Pass from environment variable
docker buildx build --secret id=mytoken,env=MY_TOKEN .
```

```dockerfile
# Mount as file (default: /run/secrets/<id>)
RUN --mount=type=secret,id=mytoken cat /run/secrets/mytoken

# Mount at custom path
RUN --mount=type=secret,id=mytoken,target=/etc/mytoken cat /etc/mytoken

# Mount as environment variable
RUN --mount=type=secret,id=mytoken,env=MY_TOKEN echo $MY_TOKEN
```

#### SSH

```bash
docker buildx build --ssh default .
```

```dockerfile
RUN --mount=type=ssh git clone git@github.com:user/private-repo.git
```

#### Git Authentication

```bash
# Bearer token (default)
GIT_AUTH_TOKEN=$(cat token.txt) docker buildx build \
  --secret id=GIT_AUTH_TOKEN \
  https://github.com/user/repo.git

# Per-host authentication
docker buildx build \
  --secret id=GIT_AUTH_TOKEN.gitlab.com,env=GITLAB_TOKEN \
  https://gitlab.com/user/repo.git
```

### Bake Reference

Bake is a high-level build orchestration tool. Supports HCL (recommended), YAML (Compose), and JSON.

#### Basic Example (docker-bake.hcl)

```hcl
variable "TAG" {
  default = "latest"
}

group "default" {
  targets = ["app", "api"]
}

target "app" {
  context    = "./app"
  dockerfile = "Dockerfile"
  tags       = ["myapp:${TAG}"]
  platforms  = ["linux/amd64", "linux/arm64"]
  cache-from = ["type=registry,ref=myapp:cache"]
  cache-to   = ["type=registry,ref=myapp:cache,mode=max"]
}

target "api" {
  context = "./api"
  tags    = ["myapi:${TAG}"]
  args = {
    NODE_VERSION = "20"
  }
}
```

```bash
docker buildx bake                    # Build default group
docker buildx bake app                # Build specific target
docker buildx bake --print            # Print resolved config
docker buildx bake --load             # Load to Docker
docker buildx bake --push             # Push to registry
TAG=v2.0 docker buildx bake           # Override variable
```

#### Target Properties

| Property | Type | Description |
|----------|------|-------------|
| `context` | string | Build context path |
| `dockerfile` | string | Dockerfile path |
| `dockerfile-inline` | string | Inline Dockerfile content |
| `tags` | list | Image tags |
| `args` | map | Build arguments |
| `labels` | map | Image labels |
| `target` | string | Build stage name |
| `platforms` | list | Target platforms |
| `cache-from` | list | Cache import |
| `cache-to` | list | Cache export |
| `output` | list | Output configuration |
| `secret` | list | Build secrets |
| `ssh` | list | SSH agent forwarding |
| `contexts` | map | Named build contexts |
| `no-cache` | bool | Disable cache |
| `pull` | bool | Always pull images |

#### Matrices

```hcl
target "app" {
  name = "app-${item.name}"
  matrix = {
    item = [
      { name = "web",  port = "8080" },
      { name = "api",  port = "3000" },
    ]
  }
  args = {
    PORT = item.port
  }
  tags = ["myapp-${item.name}:latest"]
}
```

#### Named Contexts

```hcl
target "base" {
  dockerfile = "base.Dockerfile"
}

target "app" {
  contexts = {
    baseapp  = "target:base"            # From another target
    alpine   = "docker-image://alpine"  # From image
    scripts  = "../scripts"             # From local path
  }
}
```

### Builder Drivers

| Driver | Auto-load | Full cache | Multi-platform | Use case |
|--------|-----------|-----------|----------------|----------|
| `docker` (default) | Yes | Limited | No | Simple local builds |
| `docker-container` | No | Yes | Yes | Advanced builds, multi-arch |
| `kubernetes` | No | Yes | Yes | Cloud-native CI/CD |
| `remote` | No | Yes | Yes | Pre-existing BuildKit |

```bash
# Create docker-container builder
docker buildx create --name mybuilder --driver docker-container --use --bootstrap

# List builders
docker buildx ls

# Switch builder
docker buildx use mybuilder

# Remove builder
docker buildx rm mybuilder
```

Key `docker-container` driver options:

| Option | Description |
|--------|-------------|
| `image` | BuildKit image (default: `moby/buildkit:buildx-stable-1`) |
| `memory` | Container memory limit |
| `network` | Network mode for the builder container |
| `default-load` | Auto-load images to Docker (default: false) |

### Build Exporters

```bash
# Load to Docker (single-platform only)
docker buildx build --load .
docker buildx build --output type=docker .

# Push to registry (supports multi-platform)
docker buildx build --push -t myapp:latest .
docker buildx build --output type=registry .

# Export to local directory
docker buildx build --output type=local,dest=./out .

# Export as OCI tarball
docker buildx build --output type=oci,dest=./image.tar .
```

---

## 7. Docker Engine Configuration

### Daemon Configuration

Configuration file: `/etc/docker/daemon.json` (Linux), `~/.config/docker/daemon.json` (rootless).

```json
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "log-level": "info",
  "debug": false,
  "hosts": ["unix:///var/run/docker.sock"],
  "tls": true,
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "default-runtime": "runc",
  "live-restore": true,
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5,
  "default-address-pools": [
    { "base": "172.17.0.0/12", "size": 24 }
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "bip": "172.17.0.1/16",
  "fixed-cidr": "172.17.0.0/16",
  "ipv6": false,
  "ip-forward": true,
  "iptables": true,
  "icc": true,
  "userns-remap": "default",
  "seccomp-profile": "/etc/docker/seccomp.json",
  "runtimes": {
    "custom": {
      "runtimeType": "/path/to/containerd-shim"
    }
  }
}
```

**Socket types:**

| Protocol | Example | Description |
|----------|---------|-------------|
| `unix://` | `unix:///var/run/docker.sock` | Unix domain socket (default) |
| `tcp://` | `tcp://0.0.0.0:2376` | TCP socket (use TLS) |
| `fd://` | `fd://` | systemd socket activation |
| `ssh://` | `ssh://user@host` | SSH tunneling |

**Key environment variables:**

| Variable | Description |
|----------|-------------|
| `DOCKER_HOST` | Daemon socket to connect to |
| `DOCKER_TLS_VERIFY` | Enable TLS verification |
| `DOCKER_CERT_PATH` | Path to TLS certificates |
| `DOCKER_CONFIG` | Client config directory |
| `DOCKER_CONTEXT` | Active context name |
| `DOCKER_API_VERSION` | Override API version |
| `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` | Proxy configuration |

### Engine API

RESTful API accessed via HTTP. Latest version: **v1.53** (Docker Engine 29.2).

```bash
# Via Unix socket
curl --unix-socket /var/run/docker.sock http://localhost/v1.53/containers/json

# Via TCP
curl https://docker-host:2376/v1.53/info

# Common endpoints
GET    /containers/json              # List containers
POST   /containers/create            # Create container
POST   /containers/{id}/start        # Start container
POST   /containers/{id}/stop         # Stop container
DELETE /containers/{id}              # Remove container
GET    /containers/{id}/json         # Inspect container
GET    /containers/{id}/logs         # Get container logs
POST   /containers/{id}/exec        # Create exec instance

GET    /images/json                  # List images
POST   /images/create               # Pull image
POST   /build                       # Build image
DELETE /images/{name}               # Remove image

GET    /networks                     # List networks
POST   /networks/create             # Create network
DELETE /networks/{id}               # Remove network

GET    /volumes                      # List volumes
POST   /volumes/create              # Create volume
DELETE /volumes/{name}              # Remove volume

GET    /info                         # System info
GET    /version                      # Version info
GET    /events                       # Monitor events
GET    /_ping                        # Ping (health check)
POST   /system/prune                # Prune unused data
```

### Security Features

#### Kernel Namespaces

Docker uses Linux kernel namespaces for container isolation:
- **PID**: Process isolation
- **Network**: Network stack isolation
- **IPC**: Inter-process communication isolation
- **Mount**: Filesystem isolation
- **UTS**: Hostname isolation
- **User**: UID/GID mapping

#### Control Groups (cgroups)

Resource accounting and limiting:
- Memory limits
- CPU shares, quotas, and pinning
- Block I/O limits
- PID limits
- Prevents containers from exhausting host resources

#### Linux Capabilities

Docker drops all capabilities except those needed:

```bash
# Add capabilities
docker run --cap-add NET_ADMIN --cap-add SYS_TIME myimage

# Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

#### Seccomp Profiles

Filters system calls available to containers.

```bash
# Use default profile (blocks ~44 syscalls)
docker run myimage

# Custom profile
docker run --security-opt seccomp=/path/to/profile.json myimage

# Disable seccomp (not recommended)
docker run --security-opt seccomp=unconfined myimage
```

#### AppArmor

Mandatory access control for Linux.

```bash
# Default profile (auto-loaded)
docker run myimage                                          # Uses docker-default

# Custom profile
docker run --security-opt apparmor=my-profile myimage

# Disable AppArmor
docker run --security-opt apparmor=unconfined myimage
```

#### User Namespace Remapping

Maps container root to an unprivileged host user.

```json
{
  "userns-remap": "default"
}
```

Requires `/etc/subuid` and `/etc/subgid` entries:

```
dockremap:231072:65536
```

#### Content Trust

Image signing and verification.

```bash
export DOCKER_CONTENT_TRUST=1
docker pull myapp:latest       # Verifies signature
docker push myapp:latest       # Signs image
```

### Logging Drivers

| Driver | Description |
|--------|-------------|
| `json-file` | Default. JSON-formatted logs |
| `local` | Optimized local logging with log rotation |
| `syslog` | Write to syslog facility |
| `journald` | Write to systemd journal |
| `gelf` | Graylog Extended Log Format (UDP) |
| `fluentd` | Forward to Fluentd |
| `awslogs` | Amazon CloudWatch Logs |
| `splunk` | Splunk HTTP Event Collector |
| `gcplogs` | Google Cloud Logging |
| `etwlogs` | Event Tracing for Windows |

```bash
# Per-container
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 myimage

# Default daemon config
# Set in daemon.json: "log-driver": "json-file"
```

**json-file options:**

| Option | Default | Description |
|--------|---------|-------------|
| `max-size` | `-1` (unlimited) | Max log file size (e.g., `10m`, `1g`) |
| `max-file` | `1` | Max number of rotated files |
| `labels` | | Comma-separated labels to include |
| `env` | | Comma-separated env vars to include |
| `compress` | `disabled` | Compress rotated files |

### Container Runtimes

Default runtime: **runc** (via containerd).

Available alternative runtimes:
- **gVisor** (`runsc`): Sandboxed execution
- **Kata Containers**: Lightweight VMs
- **youki**: runc-compatible Rust implementation
- **Wasmtime**: WebAssembly runtime

```json
{
  "default-runtime": "runc",
  "runtimes": {
    "gvisor": {
      "runtimeType": "io.containerd.runsc.v1"
    }
  }
}
```

```bash
docker run --runtime gvisor myimage
```

### Docker Contexts

Manage multiple Docker daemon connections.

```bash
# List contexts
docker context ls

# Create context
docker context create remote --docker host=ssh://user@remote-host
docker context create tls-remote --docker host=tcp://host:2376

# Switch context
docker context use remote

# Inspect
docker context inspect remote

# Export/import
docker context export remote --file remote.dockercontext
docker context import restored remote.dockercontext
```

**Selection precedence:**
1. `--context` CLI flag
2. `DOCKER_CONTEXT` environment variable
3. Active context (set with `docker context use`)
4. Default context (`unix:///var/run/docker.sock`)

---

## Quick Reference Cheat Sheet

### Most Common Commands

```bash
# Run a container
docker run -d --name web -p 8080:80 nginx

# Run interactively
docker run -it --rm ubuntu bash

# List running containers
docker ps

# Stop and remove
docker stop web && docker rm web

# Build an image
docker build -t myapp:latest .

# Push to registry
docker push myregistry/myapp:latest

# View logs
docker logs -f web

# Execute in running container
docker exec -it web bash

# Copy files
docker cp web:/var/log/nginx/access.log ./access.log

# Inspect
docker inspect web

# System cleanup
docker system prune -a --volumes
```

### Compose Quick Start

```bash
# Start services
docker compose up -d

# Stop and remove
docker compose down -v

# View logs
docker compose logs -f

# Rebuild and restart
docker compose up -d --build

# Scale a service
docker compose up -d --scale worker=3

# Execute command
docker compose exec app bash

# Watch mode (dev)
docker compose watch
```

### Build Quick Start

```bash
# Simple build
docker build -t myapp .

# Multi-platform build and push
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .

# Build with cache
docker buildx build \
  --cache-from type=registry,ref=myapp:cache \
  --cache-to type=registry,ref=myapp:cache,mode=max \
  -t myapp:latest --push .

# Build with secrets
docker buildx build --secret id=npmrc,src=$HOME/.npmrc .

# Bake
docker buildx bake --push
```
