---
name: apple-container
description: Use when working with Apple's `container` CLI on macOS — running Linux containers as lightweight VMs on Apple Silicon, as a Docker Desktop replacement. Covers installation, Docker command equivalents, images, containers, volumes, networks, container machines, the system service, and multi-container orchestration with Container-Compose. Use when the user mentions `container run`, `container image`, `container system`, `container-compose`, or asks how to run containers on macOS without Docker.
---

# Apple Container

Apple's `container` CLI runs Linux containers as lightweight virtual machines on Apple Silicon Macs. It is OCI-compatible and works with standard container images from any registry. Unlike Docker, each container runs in its own lightweight VM with full kernel isolation rather than sharing the host kernel via namespaces.

**Requirements:** Apple Silicon Mac + macOS 26 (Tahoe) or later. Some features (user-defined networks) require macOS 26+; the tool runs on macOS 15 with reduced functionality (no automatic DNS).

## Installation

Download and run the signed installer (`.pkg`) from the [GitHub releases page](https://github.com/apple/container/releases), then start the system service:

```bash
container system start
# On first run, prompts to install the default Linux kernel — answer Y
```

To install the kernel non-interactively:

```bash
container system start --enable-kernel-install
```

The service must be running for any container command to work. It does not auto-start on boot — run `container system start` again after a reboot, or add it as a login item.

## Docker → container command equivalents

| Docker | container |
|--------|-----------|
| `docker run` | `container run` |
| `docker create` | `container create` |
| `docker start` | `container start` |
| `docker ps` | `container list` (`ls`) |
| `docker ps -a` | `container list --all` |
| `docker stop` | `container stop` |
| `docker kill` | `container kill` |
| `docker rm` | `container delete` (`rm`) |
| `docker exec` | `container exec` |
| `docker logs` | `container logs` |
| `docker inspect` | `container inspect` |
| `docker stats` | `container stats` |
| `docker cp` | `container copy` (`cp`) |
| `docker build` | `container build` |
| `docker pull` | `container image pull` |
| `docker push` | `container image push` |
| `docker images` | `container image list` |
| `docker rmi` | `container image delete` (`rm`) |
| `docker tag` | `container image tag` |
| `docker save` | `container image save` |
| `docker load` | `container image load` |
| `docker login` | `container registry login` |
| `docker volume *` | `container volume *` |
| `docker network *` | `container network *` |
| `docker system prune` | `container system df` + `prune` subcommands |

## Running containers

```bash
# Basic run (foreground; stdin closed unless -i)
container run ubuntu:latest

# Interactive shell
container run -it ubuntu:latest /bin/bash

# Named container, auto-remove on stop
container run --name web --rm nginx:latest

# Detached (background)
container run -d --name web nginx:latest

# Port mapping (format: [host-ip:]host-port:container-port[/protocol])
container run -p 8080:80 nginx:latest

# Volume bind mount
container run -v /host/path:/container/path ubuntu:latest

# Named volume
container run -v myvolume:/data ubuntu:latest

# Environment variables
container run -e NODE_ENV=production -e PORT=3000 node:18

# Env file
container run --env-file .env node:18

# Resource limits
container run --cpus 2 --memory 1G node:18

# Override entrypoint
container run --entrypoint /bin/sh ubuntu:latest
```

> Note: `--rm` removes the container, but anonymous volumes are NOT auto-removed (unlike Docker). Clean them up manually with `container volume rm`.

## Container lifecycle

```bash
container list                  # running containers
container list --all            # include stopped
container list -q               # IDs only
container stop <id>             # graceful stop (SIGTERM, 5s timeout)
container stop --all            # stop everything
container kill <id>             # immediate SIGKILL
container delete <id>           # remove (add --force if running)
container delete --all          # remove all containers
container exec -it <id> /bin/bash   # run command in a running container
container logs <id>             # view logs
container logs -f <id>          # follow
container logs --boot <id>      # VM boot log
container inspect <id>          # JSON details
container stats                 # live resource usage (top-style)
container stats --no-stream <id># single snapshot
container cp ./file.txt <id>:/tmp/  # copy host → container
container cp <id>:/var/log/app.log ./    # copy container → host
container prune                 # remove stopped containers
```

## Image management

```bash
container image pull ubuntu:latest
container image pull --platform linux/arm64 ubuntu:latest
container image list                 # local images
container image tag src:latest new:latest
container image push myregistry.com/my-image:latest
container image delete ubuntu:latest # remove image (rm)
container image prune                # remove dangling images only
container image prune -a             # remove all unused images
container image save -o img.tar my-image:latest
container image load -i img.tar
container image inspect ubuntu:latest
```

## Building images

```bash
# Build from Dockerfile (falls back to Containerfile) in current dir
container build -t my-app:latest .

# Custom Dockerfile
container build -f docker/Dockerfile.prod -t my-app:prod .

# Build args, target stage, no cache
container build --build-arg NODE_VERSION=18 --target production --no-cache -t my-app .

# Multiple tags
container build -t my-app:latest -t my-app:v1.0.0 .
```

Builds run in an isolated BuildKit builder. Manage it with `container builder start|status|stop|delete`.

## Volumes

```bash
container volume create myvolume
container volume create -s 10G myvolume        # with size
container volume list
container volume inspect myvolume
container volume delete myvolume               # rm; must not be in use
container volume prune                         # remove unreferenced volumes
```

## Networks (macOS 26+)

```bash
container network create mynet
container network create --subnet 192.168.100.0/24 mynet
container network list
container network inspect mynet
container network delete mynet
container network prune

# Attach a container to a network
container run --network mynet ubuntu:latest
```

### DNS / hostname resolution

Containers do **not** automatically resolve each other by name. To enable a local DNS domain (requires sudo):

```bash
sudo container system dns create test               # create a .test domain
container run --dns-domain test --name web nginx     # web.test now resolves
container system dns list
sudo container system dns delete test
```

## Container machines

Container machines are persistent Linux VMs you can run commands in or open a shell into. Alias: `container m`.

```bash
# Create and boot a machine from an image (name via --name/-n)
container machine create alpine:3.22 --name dev

# Create with custom resources, set as default
container machine create --cpus 4 --memory 8G --set-default alpine:3.22

# Create without booting
container machine create --no-boot alpine:3.22

container machine list                 # ls; default is marked
container machine run                   # interactive shell in default machine
container machine run -n dev uname -a   # run a command in a named machine
container machine inspect dev
container machine set cpus=4 memory=8G  # takes effect after restart
container machine set-default dev
container machine logs dev
container machine stop dev
container machine delete dev            # rm; stops first
```

## System service management

```bash
container system start          # start services (+ install kernel if needed)
container system stop           # stop services
container system status         # is it running?
container system version        # CLI + apiserver versions
container system df             # disk usage (images/containers/volumes)
container system logs           # service logs
container system logs -f        # follow
container system kernel set --recommended   # install/update the Linux kernel
```

## Registry authentication

```bash
container registry login ghcr.io
container registry login -u myuser --password-stdin registry.example.com
container registry list
container registry logout ghcr.io
```

## Multi-container orchestration (Docker Compose alternative)

Apple's `container` has no built-in Compose equivalent. Use **[Container-Compose](https://github.com/Mcrich23/Container-Compose)** by Mcrich23 — it parses standard `docker-compose.yml` files and orchestrates them on Apple Container.

### Install

```bash
brew update
brew install container-compose
```

### Usage

```bash
container-compose up                    # start all services
container-compose up -d                 # detached (background)
container-compose up -b                 # build images before starting
container-compose up --no-cache         # build without cache
container-compose up web db             # start specific services only
container-compose build                 # build images without starting
container-compose down                  # stop all services
container-compose down web              # stop specific services
container-compose version

# Use a compose file at a custom path
container-compose up -f ./path/to/docker-compose.yml
```

It supports Compose features including service definitions, `depends_on` ordering, `.env` files, and volume/network mapping to Apple Container equivalents.

> Notes:
> - If you do NOT use `-d`, killing the `up` process does not stop the containers — run `container-compose down` to stop them.
> - DNS between services works best on macOS 26 (Tahoe). On macOS 15 (Sequoia), automatic DNS is not configured.
> - Container-Compose support is intentionally limited — it is a bridge, not a full Docker Compose reimplementation.

## Key differences from Docker

- Containers run as **lightweight VMs** with full kernel isolation, not shared-kernel namespaces.
- **Apple Silicon only**, **macOS 26+** recommended (some features unavailable on macOS 15).
- `container system start` must be run once per boot.
- No native Compose — use Container-Compose for multi-container workflows.
- Anonymous volumes are not auto-removed by `--rm`; clean them up manually.
- Inter-container name resolution requires setting up a DNS domain (`container system dns create`, sudo).
