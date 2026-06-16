# Migrating from Docker Desktop

Moving a Docker/Docker Desktop workflow to Apple `container`. The CLI is deliberately Docker-like, but the architecture (one VM per container, no daemon) creates real behavioral differences. The command map is easy; the gotchas below are what actually break migrations. Citations are apple/container issue numbers.

## Command map

| Docker | container |
|--------|-----------|
| `docker run` | `container run` |
| `docker create` / `start` | `container create` / `start` |
| `docker ps` / `ps -a` | `container list` / `list --all` |
| `docker stop` / `kill` / `rm` | `container stop` / `kill` / `delete` |
| `docker exec` / `logs` / `inspect` / `stats` / `cp` | `container exec` / `logs` / `inspect` / `stats` / `copy` |
| `docker build` | `container build` |
| `docker pull` / `push` | `container image pull` / `push` |
| `docker images` / `rmi` / `tag` / `save` / `load` | `container image list` / `delete` / `tag` / `save` / `load` |
| `docker volume …` / `network …` | `container volume …` / `network …` |
| `docker login` / `logout` | `container registry login` / `logout` |
| `docker compose …` | `container-compose …` (third-party) |

## Gotchas that break migrations

### 1. No Docker daemon, socket, or Engine API
There is no `dockerd`, no `/var/run/docker.sock`, and no Docker Engine REST API. Architecture is XPC helpers + one lightweight VM per container.
- Tools wired to the Docker socket or `DOCKER_HOST` **silently no-op**: Testcontainers, Traefik's docker provider, Grafana Alloy `loki.source.docker`, OTel `dockerstats`, CI scripts that talk to the API, etc.
- Requests to add Docker Engine API compatibility were **closed without implementation** (#66, #1475, #1476).
- **No Docker-in-Docker** (#87).

### 2. No native Docker Compose
The official request (#230) was closed with no native support. Use Container-Compose (see `container-compose.md`) — but verify field support first; several common fields are silently ignored.

### 3. Volumes are stricter than Docker
- **Anonymous volumes are not supported.** `container run -v /data alpine` → `Error: invalidArgument: "anonymous volumes are not supported"` (#726).
- **Named volumes are not auto-created.** Referencing a missing named volume **fails** instead of creating it like Docker (#690). Run `container volume create myvol` first.
- **Named volumes are EXT4-formatted and contain a `lost+found` dir**, so they look non-empty. This **breaks database images** (Postgres `initdb`, MySQL) that require an empty data directory (#1730). Workaround: mount a subdirectory of the volume as the data dir, or use a bind mount to an empty host dir.
- Bind mounts (`-v host:container`, `--mount source=,target=`) work normally.

### 4. No `--restart` policy
`container run`/`create` have no `--restart` flag (#286). `always` / `unless-stopped` / `on-failure` have no equivalent — auto-restart on crash or reboot must be handled externally (e.g. a launchd agent or a supervisor in the container).

### 5. x86/amd64 via Rosetta — works, but with caveats
- amd64 images run under Rosetta automatically (`--arch amd64`, or `--platform linux/amd64`); multi-arch builds use `--arch arm64 --arch amd64`.
- **Rosetta may not be installed on a fresh machine**, and it's enabled by default without verifying — builds/runs error until Rosetta is installed (#1136). Install it with `softwareupdate --install-rosetta`.
- amd64 emulation is less reliable than Docker Desktop's: Python hangs (#966), debuggers fail (#766), some DBs segfault under linux/amd64 (#986). Prefer arm64 images where possible.

### 6. Registry auth differences
- Auth is `container registry login <host>`, credentials stored in **macOS Keychain**.
- **Docker credential helpers are not honored** — `~/.docker/config.json` `credHelpers`/`credsStore` (ECR `docker-credential-ecr-login`, GCR/GAR via `gcloud`, etc.) do nothing (#820). Log in manually with a token instead.
- `push` may trigger a Keychain access prompt because a separate helper (`container-core-images`) reads the credential (#1253).
- Some non-Docker-Hub registries have interop bugs (ECR push 401 #1707, ACR pull 401 #254). Pulls can succeed while pushes fail on the same registry.

### 7. Build differences (not `docker buildx`)
- `container build` uses a BuildKit builder running in a utility VM — there is no `docker buildx`, no `buildx bake`, limited BuildKit option passthrough (#701).
- Builder defaults to 2 GiB / 2 CPUs; pre-start it larger for heavy builds: `container builder start --cpus 8 --memory 16g`.
- Dockerfiles **≥ 16 KB** fail (#735). Known gaps: `COPY --chown` (#371), `VOLUME` parsing (#434).

### 8. Other differences
- **Memory is not returned to the host.** Pages freed inside the guest aren't relinquished to macOS (Virtualization-framework ballooning limitation); long-running memory-heavy containers may need periodic restart. Postgres can report much higher memory than under Docker (#1698).
- **No `container system prune`** catch-all yet (#891) — prune per resource type (`container prune`, `container image prune`, `container volume prune`, `container network prune`).
- **No `--gpus` passthrough** (#1511).
- **CLI plugins are lost on every upgrade** (#1617).
- **VPNs can break container networking** (#1307).

## A reasonable migration order

1. Confirm macOS 26 + Apple Silicon. On macOS 15, expect no container-to-container networking.
2. Install Rosetta (`softwareupdate --install-rosetta`) if you use any amd64 images.
3. Replace `docker` commands per the map above (or alias `docker=container` for simple cases — it won't cover compose/daemon-dependent tools).
4. Pre-create named volumes; for databases, avoid mounting the EXT4 volume root directly.
5. Re-login to registries with `container registry login` (credential helpers won't carry over).
6. Convert `docker-compose.yml` via Container-Compose, checking the support matrix; replace `restart:`/`healthcheck:` reliance with external supervision.
