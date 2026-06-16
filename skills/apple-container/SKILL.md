---
name: apple-container
description: Use when running Linux containers on macOS with Apple's `container` CLI (github.com/apple/container) — the per-VM, Apple-silicon-native alternative to Docker Desktop. Covers install, the run/build/image/volume/network/system commands, Docker command equivalents, networking and DNS, migrating off Docker Desktop, multi-container orchestration with Container-Compose, and troubleshooting. Triggers on `container run`, `container build`, `container system`, `container-compose`, "run containers on macOS without Docker", or migrating a Docker/Compose workflow to Apple Silicon. Do NOT use for Docker Desktop, Podman, Colima, or Linux/Windows container tooling.
---

# Apple Container

Apple's `container` CLI runs Linux containers on Apple Silicon Macs. Unlike Docker (one shared Linux VM hosting all containers), `container` runs **one lightweight VM per container** via the macOS Virtualization framework, giving each container full-VM isolation. It is OCI-compatible and works with any standard registry (default: Docker Hub).

**Requirements:** Apple Silicon Mac + **macOS 26 (Tahoe) or later**. It runs on macOS 15 (Sequoia) but with serious networking limitations and no official bug support — treat macOS 26 as the real baseline. Current release: v1.0.0.

This SKILL.md covers the everyday workflow. For depth, read the reference file that matches the task:

- **`references/commands.md`** — full command reference (run/create/exec, image, volume, network, builder, registry, machine, system) with every flag.
- **`references/networking.md`** — networking model, IP allocation, port publishing, DNS setup, container↔container and host↔container traffic, multi-network.
- **`references/docker-migration.md`** — moving off Docker Desktop: command map, and the behavioral differences that bite (no daemon/socket, no Compose, volume rules, Rosetta, registry auth, no `--restart`).
- **`references/container-compose.md`** — Container-Compose (Docker Compose support): install, commands, and the exact `docker-compose.yml` field support matrix (what works vs. what is silently ignored).
- **`references/troubleshooting.md`** — common errors and fixes, hard limitations, and TOML configuration.

## Quick start

```bash
# 1. Start the system service (first run prompts to install the Linux kernel — answer Y)
container system start

# 2. Pull and run a container
container run --name web --detach --rm -p 8080:80 nginx:latest

# 3. See it, exec into it, read logs
container list
container exec -it web /bin/sh
container logs -f web

# 4. Stop it (--rm removes it on stop) and stop the service
container stop web
container system stop
```

The service does **not** auto-start on boot — run `container system start` again after a reboot.

## Build → run → push workflow

```bash
container build -t my-app:latest .                 # build from ./Dockerfile (or ./Containerfile)
container run -d --name my-app -p 3000:3000 my-app:latest
container registry login ghcr.io                   # creds stored in macOS Keychain
container image tag my-app:latest ghcr.io/me/my-app:latest
container image push ghcr.io/me/my-app:latest
```

The first build boots a BuildKit builder VM (2 CPUs / 2 GiB by default). Resize with `container builder start --cpus 8 --memory 16g`.

## Command cheat sheet (Docker → container)

| Task | Docker | container |
|------|--------|-----------|
| Run | `docker run` | `container run` |
| List running | `docker ps` | `container list` (`ls`) |
| List all | `docker ps -a` | `container list --all` |
| Stop / kill | `docker stop` / `kill` | `container stop` / `kill` |
| Remove container | `docker rm` | `container delete` (`rm`) |
| Exec | `docker exec` | `container exec` |
| Logs | `docker logs` | `container logs` |
| Inspect / stats | `docker inspect` / `stats` | `container inspect` / `stats` |
| Copy | `docker cp` | `container copy` (`cp`) |
| Build | `docker build` | `container build` |
| Pull / push | `docker pull` / `push` | `container image pull` / `push` |
| List / remove images | `docker images` / `rmi` | `container image list` / `delete` |
| Volumes | `docker volume …` | `container volume …` |
| Networks | `docker network …` | `container network …` |
| Login | `docker login` | `container registry login` |
| Compose | `docker compose …` | `container-compose …` (third-party — see reference) |

## The five things that trip up Docker users

These are the differences most likely to surprise you. Full detail in `references/docker-migration.md`.

1. **No Docker daemon, socket, or Engine API.** Anything wired to `/var/run/docker.sock` or `DOCKER_HOST` (Testcontainers, Traefik's docker provider, etc.) will not work. There is no Docker-in-Docker.
2. **No native Docker Compose.** Use Container-Compose (`references/container-compose.md`) — but know that `restart`, `healthcheck`, `deploy`, `secrets`, and `configs` are parsed and **silently ignored**.
3. **Volumes are stricter.** Anonymous volumes (`-v /data`) are **not supported** and error out. Named volumes are **not auto-created** — run `container volume create myvol` first. Named volumes are EXT4-formatted with a `lost+found` dir, which **breaks DB images** (Postgres/MySQL) that require an empty data directory.
4. **No `--restart` policy.** Crashed or reboot-killed containers do not come back on their own; handle restarts externally.
5. **x86/amd64 runs via Rosetta but is less reliable**, and Rosetta may not even be installed on a fresh machine (builds error until it is). Docker credential helpers (ECR/GCR `gcloud`) are **not** honored.

## Multi-container apps

There is no built-in Compose. Install [Container-Compose](https://github.com/Mcrich23/Container-Compose):

```bash
brew install container-compose
container-compose up -d        # parses docker-compose.yml, runs services on Apple Container
container-compose down
```

It supports the common fields (services, image/build, ports, volumes, environment, env_file, depends_on ordering, networks, command, entrypoint, cpu/memory limits) but **silently drops** `restart`, `healthcheck`, `deploy` (beyond cpu/mem), `secrets`, `configs`, and all network/volume driver options. See `references/container-compose.md` for the exact field-by-field matrix before relying on a compose file.

## When things break

Most issues are the kernel not installed, the service not running, or macOS-15 networking. Quick checks:

```bash
container system status        # is the service up?
container system start --enable-kernel-install   # install the Linux kernel non-interactively
container --debug <command>    # verbose output for any command
container logs --boot <id>     # VM boot log (init/kernel issues)
```

See `references/troubleshooting.md` for specific errors, the macOS 15 networking caveats, the memory-not-returned-to-host limitation, and TOML configuration (`~/.config/container/config.toml`; note `container system property get/set` were removed in v1.0.0 — config is now the TOML file).
