# Troubleshooting, Limitations & Configuration

> Note: the apple/container README links to a `docs/troubleshooting.md`, but that file does not exist in the repo (dead link). There is also no FAQ doc. The guidance below is assembled from the README, `technical-overview.md`, `how-to.md`, and `bug-report-how-to.md` — do not look for an upstream troubleshooting page.

## First, the two universal checks

```bash
container system status        # is the apiserver running?
container --version            # confirm the CLI is installed
```

Most failures are one of: the service isn't running, the Linux kernel isn't installed, or you're on macOS 15 hitting networking limits.

## Common errors and fixes

| Symptom | Cause / fix |
|---------|-------------|
| `No default kernel configured` / prompt on `system start` | Accept the kernel install, or `container system kernel set --recommended`, or `container system start --enable-kernel-install` |
| Commands hang or "cannot connect" | Service not running: `container system start`. If wedged: `container system stop && container system start` |
| `anonymous volumes are not supported` | `-v /path` without a name isn't allowed. Use a named volume (`container volume create v && -v v:/path`) or a bind mount (`-v ./host:/path`) |
| Named volume "not found" | Named volumes aren't auto-created — `container volume create <name>` first |
| Postgres/MySQL won't initialize on a fresh volume | EXT4 volume has a `lost+found` dir → not "empty". Mount a subdir as the data dir, or bind-mount an empty host dir (#1730) |
| amd64 image fails immediately / build errors on fresh Mac | Rosetta not installed: `softwareupdate --install-rosetta` |
| Registry login "works" but ECR/GCR pull/push 401 | Docker credential helpers aren't honored (#820). Log in with an explicit token via `container registry login` |
| Container-to-container by IP fails | Expected on macOS 15 (network isolation). Requires macOS 26 |
| All networking fails on macOS 15 | vmnet/helper subnet disagreement (default `192.168.64.1/24`). Use macOS 26 |
| `nested virtualization is not supported on the platform` | `--virtualization` needs M3+ and a virt-enabled kernel |
| Networking dies after connecting to a VPN | Known interaction (#1307) |

## Debugging tools

```bash
container --debug <command>    # verbose CLI output for any command
container logs <id>            # container stdio
container logs --boot <id>     # VM boot log (kernel/init/vminitd)
container system logs -f       # apiserver + helper logs (unified logging)
container system logs --last 1h
```

To file a useful bug report (per `docs/bug-report-how-to.md`) include: `sw_vers`, `xcodebuild -version`, `container --version`, exact repro commands, full error text, and `--debug` / `logs` output.

## Hard limitations to know up front

- **macOS 26 + Apple Silicon required.** macOS 15 runs but is degraded (networking) and unsupported for fixes; Intel Macs are not supported at all.
- **Memory is not returned to the host.** The Virtualization framework only partially supports ballooning — pages freed inside the guest aren't relinquished to macOS. Restart long-running, memory-heavy containers to reclaim host RAM.
- **Dockerfiles ≥ 16 KB fail** (#735).
- **No Docker daemon / socket / Engine API, no Docker-in-Docker, no native Compose, no `--restart`, no `--gpus`.** See `docker-migration.md`.
- **Young project.** Stability is guaranteed only within patch versions; v1.0.0 is the first stability milestone.

## Configuration (TOML)

v1.0.0 replaced the old UserDefaults "system properties" with a TOML file. **`container system property get` and `set` were removed**; `container system property list` is now read-only and prints the merged effective config.

Config file (first match wins, read once at service startup):
1. `~/.config/container/config.toml` (user)
2. `<installRoot>/etc/container/config.toml` (optional, package-shipped)
3. Built-in defaults

Changes take effect only after `container system stop && container system start`.

```toml
[container]                 # defaults when run/create omit them
cpus = 4
memory = "1g"

[build]
rosetta = true              # set false to force native arm64-only builds
cpus = 2
memory = "2048mb"

[registry]
domain = "docker.io"        # default registry

[dns]
domain = "test"             # default DNS domain appended to container names

[network]
subnet = "192.168.64.1/24"  # default subnet for newly created networks
```

Memory sizes are quoted, binary units (1024), case-insensitive (`b`/`k`/`m`/`g`/`t`/`p`); a bare integer is bytes. Default per-container resources are 4 CPUs / 1 GiB; the builder defaults to 2 CPUs / 2 GiB.

**Kernel version is release-specific** — don't hardcode it. Inspect the effective kernel with `container system property list`, and install/update with `container system kernel set --recommended`.

Data lives under the app-data root (`container system start --app-root`); installer files go under `/usr/local` (including `uninstall-container.sh`).
