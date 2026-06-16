# Networking & DNS

How container networking works on Apple `container`. Verified against `docs/how-to.md` and `docs/technical-overview.md`. **Most of this requires macOS 26.** macOS 15 is degraded — see the bottom of this file.

## Model

- `container system start` creates a vmnet network named **`default`**. Containers attach to it unless `--network` says otherwise.
- The default subnet is **`192.168.64.0/24`**, gateway **`192.168.64.1`**. Containers get sequential IPs (`.2`, `.3`, …). The BuildKit **builder** container usually takes `.2`.
- Each running container's IP appears in `container list` (IP column) and `container inspect` (`networks[].address`, `.gateway`, `.hostname`).
- Because each container is its own VM, there is **no shared Docker-style bridge** — every container has its own dedicated IP.
- v1.0.0 ties IP leases to the container's XPC session, so IPs are freed automatically when the container stops (fixes earlier IP leaks).

Change the default subnet for newly created networks in `~/.config/container/config.toml`:

```toml
[network]
subnet = "192.168.100.1/24"
subnetv6 = "fd00:abcd::/64"
```

## Container ↔ container

On **macOS 26**, containers on the *same* network reach each other directly by IP:

```bash
container run -d --name api -p 8000:8000 my-api
container inspect api | grep address          # e.g. 192.168.64.3
container run -it --rm curlimages/curl http://192.168.64.3:8000
```

Containers on *different* networks cannot reach each other — each created network is fully isolated. On **macOS 15 this does not work at all** (see below).

## DNS / resolve containers by name

Containers do **not** resolve each other by name out of the box. Create a local DNS domain (needs sudo — it writes `/etc/resolver/<domain>`):

```bash
sudo container system dns create test                  # create a .test domain
container run -d --name web --dns-domain test nginx    # now resolvable as web.test
container system dns list
sudo container system dns delete test
```

Set a default domain so you don't pass `--dns-domain` each time, in `config.toml`:

```toml
[dns]
domain = "test"
```

Per-container DNS overrides on `run`/`create`: `--dns <ip>`, `--dns-domain`, `--dns-search`, `--dns-option`, `--no-dns`.

## Host → container (publish ports)

```bash
container run -d -p 8080:80 nginx                 # all interfaces
container run -d -p 127.0.0.1:8080:80 nginx       # IPv4 localhost only
container run -d -p '[::1]:8080:80' nginx          # IPv6 localhost
container run -d -p 5353:53/udp my-dns            # UDP
```

Format: `[host-ip:]host-port:container-port[/protocol]`. If a container is on multiple networks, published ports forward to the IP of the **first** network's interface. Also `--publish-socket host_path:container_path` for sockets.

Inside a container it's safe to bind `0.0.0.0` — external systems have no route to the virtual network.

## Container → host service

To let a container reach a service running on the macOS host, map a domain to a host IP (sudo):

```bash
sudo container system dns create host.container.internal --localhost 203.0.113.1
```

**Security caveats (macOS):**
- Creating a localhost domain **disables Private Relay** while it exists.
- The local-domain packet-filter rule is **removed on restart** — recreate it after rebooting.
- Pick an IP unlikely to collide (documentation/reserved ranges: `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, or `172.16.0.0/12`).

There is no built-in `host.docker.internal`; this DNS mechanism is the supported substitute.

## Custom networks & MAC (macOS 26+)

```bash
container network create backend --subnet 192.168.100.0/24
container run -d --network backend --name db postgres:16
container run -d --network backend,mac=02:42:ac:11:00:02 my-app
```

For a custom MAC, set the two least-significant bits of the first octet to `10` (locally-administered unicast). Auto-generated MACs start with first nibble `f`.

## macOS 15 (Sequoia) limitations

`container` runs on macOS 15 but networking is severely limited and **not officially supported for bug fixes**:

- **No container-to-container communication** over the virtual network (vmnet only provides isolated networks). Confirmed in apple/container #326, #345.
- **`container network …` commands are unavailable**; `--network` on run/create **errors**. All containers share the single `default` network.
- **Networking can fail entirely** when the network helper and vmnet disagree on the subnet (default `192.168.64.1/24`), leaving containers cut off.

If you need real container networking, use macOS 26.
