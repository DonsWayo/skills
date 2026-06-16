# Container-Compose

[Container-Compose](https://github.com/Mcrich23/Container-Compose) by Mcrich23 brings (limited) Docker Compose support to Apple `container`. It parses a standard `docker-compose.yml` and translates each service into a `container run` invocation. It is not a Docker wrapper.

The critical thing to understand: **a field parsing successfully does not mean it does anything.** Container-Compose decodes many Compose fields but only emits `container run` flags for some. Unsupported fields are dropped — often **silently** — so a compose file can appear to work while quietly losing `restart`, `healthcheck`, secrets, and more. The matrix below is from reading the source (`Sources/Container-Compose/`, especially the Codable structs and `ComposeUp.configService`).

## Install

```bash
brew update
brew install container-compose
# or build from source: git clone … && make build && make install (needs Swift toolchain)
```

Latest release: 1.0.0. Requires Apple `container`. DNS between services works best on macOS 26; on macOS 15 automatic DNS is not configured.

## Commands

| Command | Purpose |
|---------|---------|
| `container-compose up [services…]` | Start services (topologically ordered) |
| `container-compose up -d` | Detached (background) |
| `container-compose up -b` / `--build` | Build images before starting |
| `container-compose up --no-cache` | Build without cache |
| `container-compose build [services…]` | Build images without starting |
| `container-compose build --no-cache` | Build without cache |
| `container-compose down [services…]` | Stop services |
| `container-compose version` | Version |
| `-f, --file <path>` | Path to the compose file (on any command) |

> If you do **not** pass `-d`, killing the `up` process does not stop the containers — run `container-compose down`.

## Field support matrix

### Top-level keys

| Key | Parsed | Effect |
|-----|:------:|--------|
| `services` | ✅ | Core path (required) |
| `version` | ✅ | Ignored (prints a note) |
| `name` | ✅ | Only used to name containers `<name>-<service>` |
| `volumes` | ✅ | **Faked** — creates a host dir and bind-mounts it, not a real named volume |
| `networks` | ✅ | Creates the network by name only; all options ignored |
| `configs` | ✅ | **Decoded, never used** |
| `secrets` | ✅ | **Decoded, never used** |

### Per-service — these genuinely translate to `container run`

`image`, `build` (with `--build-arg`, context, `--os`/`--arch`), `platform`, `container_name` (`--name`), `user`, `volumes` (`-v`), `environment` (map or list → `-e`), `env_file`, `ports` (`-p`), `command`, `entrypoint`, `networks` (`--network`), `hostname`, `working_dir`, `privileged`, `read_only`, `stdin_open` (`-i`), `tty` (`-t`), `deploy.resources.limits.cpus` (`--cpus`), `deploy.resources.limits.memory` (`--memory`).

A service must define `image` or `build`, or decoding fails.

### Per-service — parsed but SILENTLY IGNORED (the dangerous set)

These decode without error and do nothing:

- **`restart`** — `container run` has no restart policy; dropped.
- **`healthcheck`** — fully parsed (`test`/`interval`/`retries`/…) but no `--health*` flag is ever emitted.
- **`deploy.*` beyond `resources.limits`** — `mode`, `replicas`, `restart_policy`, `resources.reservations` ignored.
- **service-level `configs` / `secrets`** — logged, never attached.
- **all network options** — `driver`, `driver_opts`, `attachable`, `enable_ipv6`, `internal`, `labels`: "Detected, But Not Supported."
- **volume options** — `driver`, `driver_opts`, `labels`, `external`: decoded, never used.

### Not in the schema at all (dropped by the YAML parser, no warning)

`labels`, `expose`, `extra_hosts`, `cap_add`/`cap_drop`, `devices`, `dns`, `sysctls`, `ulimits`, `logging`, `profiles`, `extends`, `include`, `cgroup_parent`, `init`, `stop_signal`, `stop_grace_period`.

## depends_on — ordering only, not health

`depends_on` is honored as **startup order** (topological sort; errors on cycles). Services start sequentially and the tool polls the container API up to ~30s for status `running` before continuing. But the **`condition:` form (e.g. `service_healthy`) is ignored**, and since `healthcheck` is also ignored, there is **no real "wait until healthy"** — only "process is running" (#68). For a service that must wait for a dependency to be truly ready, add an in-container wait (e.g. `wait-for-it`, retry loop) rather than relying on `condition: service_healthy`.

## Named volumes are faked

Apple `container` doesn't support named-volume refs in `container run -v`, so Container-Compose creates a host directory `~/.containers/Volumes/<project>/<name>` and bind-mounts it. Bind mounts (`./path`, `/path`) resolve to absolute host paths and the dir is auto-created. Consequence: volume `driver`/`external` semantics don't apply, and data lives under your home dir, not in a managed volume.

## Known limitations / open issues

- #96 volumes mount parent/root dir; #95 `replicas` with an env var → decode error; #81 no inter-container DNS; #70 networks unsupported; #68 `depends_on` health; #67/#33 `-f` file vs. auto-detected `docker-compose.yml`; #48 `include`; #7 `extends`; #4 relative bind mounts.
- README itself describes it as "(limited) Docker Compose support."

## Bottom line

Use Container-Compose for straightforward stacks (web + db + cache) defined with the supported fields. Before relying on a compose file, scan it for the silently-ignored set — **`restart`, `healthcheck`, `deploy` (beyond cpu/mem), `secrets`, `configs`, and network/volume driver options** — and replace that behavior with explicit setup, since the tool will run without complaint while dropping it.
