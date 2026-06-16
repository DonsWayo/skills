# Command Reference

Full reference for Apple's `container` CLI. Verified against `apple/container` `docs/command-reference.md` (v1.0.0 / `main`). Command availability varies by macOS version (network commands require macOS 26+).

## Contents

- [Running containers](#running-containers)
- [Container lifecycle](#container-lifecycle)
- [Images](#images)
- [Building](#building)
- [Volumes](#volumes)
- [Networks (macOS 26+)](#networks-macos-26)
- [Registry](#registry)
- [Container machines](#container-machines)
- [System service](#system-service)

## Running containers

`container run [<options>] <image> [<args>...]` — run a container. Foreground with stdin closed unless `-i`. `container create` takes the same flags but leaves it stopped; `container start [-a] [-i] <id>` starts a stopped one.

Most-used flags:

| Flag | Meaning |
|------|---------|
| `-i, --interactive` | Keep stdin open |
| `-t, --tty` | Allocate a TTY |
| `-d, --detach` | Run in background |
| `--name <name>` | Container name/ID |
| `--rm, --remove` | Remove after it stops |
| `-e, --env key=value` | Env var (or just `key` to inherit from host) |
| `--env-file <file>` | Env file (key=value, `#` comments ignored) |
| `-v, --volume <spec>` | Bind mount `host:container` or named `vol:container` |
| `--mount type=…,source=…,target=…[,readonly]` | Explicit mount |
| `-p, --publish [host-ip:]host-port:container-port[/proto]` | Publish a port |
| `--publish-socket host_path:container_path` | Publish a socket |
| `-c, --cpus <n>` | CPUs (default 4) |
| `-m, --memory <size>` | Memory, K/M/G/T/P suffix (default 1G) |
| `-w, --workdir <dir>` | Working directory |
| `-u, --user name\|uid[:gid]` | User |
| `--entrypoint <cmd>` | Override entrypoint |
| `--network <name>[,mac=…][,mtu=…]` | Attach to a network |
| `--dns <ip>` / `--dns-domain` / `--dns-search` / `--dns-option` / `--no-dns` | DNS config |
| `-a, --arch arm64\|amd64` | Architecture (default arm64) |
| `--os <os>` | OS (default linux) |
| `--platform os/arch[/variant]` | Takes precedence over `--os`/`--arch` |
| `--rosetta` | Enable Rosetta translation in the container |
| `--cap-add` / `--cap-drop <cap>` | Add/drop a Linux capability |
| `--read-only` | Read-only root filesystem |
| `--tmpfs <path>` | tmpfs mount |
| `--shm-size <size>` | `/dev/shm` size |
| `--init` | Run an init that reaps zombies / forwards signals |
| `--label key=value` | Container label |
| `--cidfile <path>` | Write container ID to a file |

There is **no `--restart` flag** (Docker restart policies have no equivalent).

```bash
container run -it ubuntu:latest /bin/bash
container run -d --name web -p 8080:80 nginx:latest
container run -e NODE_ENV=production --cpus 2 --memory 1G node:18
container run --arch amd64 ubuntu:latest        # x86 via Rosetta
```

## Container lifecycle

| Command | Purpose |
|---------|---------|
| `container list` (`ls`) `[--all] [-q] [--format json\|table\|yaml\|toml]` | List containers (running only unless `--all`) |
| `container stop [--all] [-s SIGNAL] [-t SECONDS] <ids…>` | Graceful stop (SIGTERM, 5s default) |
| `container kill [--all] [-s SIGNAL] <ids…>` | Immediate kill (SIGKILL) |
| `container delete` (`rm`) `[--all] [--force] <ids…>` | Remove (force if running) |
| `container exec [-it] [-e …] [-w …] [-u …] <id> <cmd>` | Run a command in a running container |
| `container logs [--boot] [-f] [-n N] <id>` | Logs (`--boot` = VM boot log) |
| `container inspect <ids…>` | JSON details |
| `container stats [--no-stream] [--format …] [<ids…>]` | Live resource usage (top-style) |
| `container copy` (`cp`) `<src> <dst>` | Copy host↔container (`<id>:/path`) |
| `container export [-o file] <id>` | Export stopped container FS as tar |
| `container prune` | Remove stopped containers |

## Images

| Command | Purpose |
|---------|---------|
| `container image pull [--platform …] [--arch …] [--os …] <ref>` | Pull |
| `container image push [--platform …] <ref>` | Push |
| `container image list` (`ls`) `[-q] [-v] [--format …]` | List local images |
| `container image tag <src> <target>` | Tag |
| `container image delete` (`rm`) `[--all] [--force] <imgs…>` | Delete |
| `container image prune [-a]` | Remove dangling (or all unused with `-a`) |
| `container image save -o <file> <refs…>` | Save to tar |
| `container image load -i <file>` | Load from tar |
| `container image inspect <imgs…>` | JSON details |

## Building

`container build [<options>] [<context-dir>]` — BuildKit-based build. Looks for `Dockerfile`, then `Containerfile`, unless `-f` is given.

| Flag | Meaning |
|------|---------|
| `-t, --tag <name>` | Image tag (repeatable for multiple tags) |
| `-f, --file <path>` | Dockerfile path |
| `--build-arg key=val` | Build-time variable |
| `--target <stage>` | Target build stage |
| `--no-cache` | Disable cache |
| `-a, --arch <arch>` | Add architecture (repeatable for multi-arch) |
| `--platform os/arch[/variant]` | Platform |
| `-c, --cpus` / `-m, --memory` | Builder resources (default 2 CPUs / 2048MB) |
| `-o, --output type=oci\|tar\|local[,dest=…]` | Output config |
| `--secret id=<key>[,env=…\|,src=…]` | Build secret |

> Known limit: Dockerfiles **≥ 16 KB** fail (apple/container #735).

Builder management: `container builder start [--cpus N] [--memory SIZE]`, `container builder status`, `container builder stop`, `container builder delete [--force]`. To resize a running builder you must stop → delete → start.

## Volumes

Named volumes are EXT4-formatted. **Anonymous volumes are not supported** (`-v /path` errors). Named volumes are **not auto-created** — create them first.

| Command | Purpose |
|---------|---------|
| `container volume create [-s SIZE] [--opt k=v] <name>` | Create (e.g. `-s 10G`) |
| `container volume list` (`ls`) | List |
| `container volume inspect <names…>` | JSON details |
| `container volume delete` (`rm`) `[--all] <names…>` | Delete (must not be in use) |
| `container volume prune` | Remove unreferenced volumes |

Driver `--opt` options (local driver): `size=<value>`, `journal=ordered\|writeback\|journal[:size]`.

## Networks (macOS 26+)

Unavailable on macOS 15. Each network is fully isolated from every other network.

| Command | Purpose |
|---------|---------|
| `container network create [--subnet CIDR] [--subnet-v6 CIDR] [--internal] [--label …] <name>` | Create |
| `container network list` (`ls`) | List |
| `container network inspect <names…>` | JSON details |
| `container network delete` (`rm`) `[--all] <names…>` | Delete (no attached containers) |
| `container network prune` | Remove unused (default/system networks preserved) |

See `networking.md` for the model, IP allocation, and DNS.

## Registry

| Command | Purpose |
|---------|---------|
| `container registry login [-u USER] [--password-stdin] <server>` | Log in (creds → macOS Keychain) |
| `container registry logout <server>` | Log out |
| `container registry list` | List logins |

Default registry domain is `docker.io`, changeable via `[registry] domain` in `config.toml`. Docker credential helpers are **not** honored (apple/container #820).

## Container machines

Persistent Linux VMs (run an init system, auto-map host home dir) — for development, as opposed to ephemeral app containers. Alias `m`. New in v1.0.0; standalone doc (`docs/container-machine.md`) currently lives on `main`.

| Command | Purpose |
|---------|---------|
| `container machine create [<opts>] <image>` | Create + boot (`--name`, `--cpus`, `--memory`, `--set-default`, `--no-boot`) |
| `container machine run [-n NAME] [--root] [<cmd>]` | Shell or one-off command (boots if stopped) |
| `container machine list` (`ls`) | List (default is marked) |
| `container machine inspect [<id>]` | JSON details |
| `container machine set [-n NAME] cpus=…\|memory=…\|home-mount=ro\|rw\|none` | Configure (effective after restart) |
| `container machine set-default <id>` | Set default |
| `container machine logs [--boot] [-f] [<id>]` | Logs |
| `container machine stop [<id>]` / `delete` (`rm`) `<id>` | Stop / delete |

Image must be a Linux image with `/sbin/init`. Memory defaults to half of host memory.

```bash
container machine create alpine:3.22 --name dev --set-default
container machine run -n dev uname -a
```

## System service

These manage the `container-apiserver` launch agent and helpers (macOS hosts only).

| Command | Purpose |
|---------|---------|
| `container system start [--enable-kernel-install\|--disable-kernel-install] [--timeout N]` | Start services (+ install kernel) |
| `container system stop [--prefix …]` | Stop services |
| `container system status` | Health check |
| `container system version [--format …]` | CLI + apiserver versions |
| `container system df [--format …]` | Disk usage (images/containers/volumes) |
| `container system logs [-f] [--last 5m]` | Service logs |
| `container system kernel set [--recommended\|--tar URL --binary MEMBER] [--arch …] [--force]` | Install/update the Linux kernel |
| `sudo container system dns create [--localhost IP] <domain>` | Create a local DNS domain |
| `sudo container system dns delete <domain>` / `container system dns list` | Manage DNS domains |
| `container system property list [--format toml\|json]` | **Read-only** effective config |

> v1.0.0 removed `container system property get` and `set`. Configuration is now the TOML file at `~/.config/container/config.toml` — see `troubleshooting.md`.
