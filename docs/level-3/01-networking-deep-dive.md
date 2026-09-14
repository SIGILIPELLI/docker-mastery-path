# 01 · Docker Networking Deep Dive

Level 1 and 2 used bridge networking without examining it closely. This
module compares Docker's network drivers directly and explains when you
need something beyond the default bridge.

## The drivers

| Driver | Scope | Typical use |
|---|---|---|
| `bridge` (default, or user-defined) | Single host | Most containers on one machine; the default for `docker run` and Compose |
| `host` | Single host, no isolation | Maximum network performance, at the cost of no port-mapping isolation |
| `overlay` | Multiple hosts | Swarm services that need containers on different machines to talk to each other as if on one network |
| `none` | No networking at all | Fully isolated batch jobs that shouldn't touch the network |
| `macvlan` | Single host, direct L2 | Giving a container its own MAC/IP on the physical LAN, as if it were a separate physical machine |

## `bridge`: the default, and why "user-defined" matters

```bash
docker network create mynet
docker run -d --name a --network mynet nginx
docker run -d --name b --network mynet alpine sleep 3600
docker exec b sh -c "getent hosts a"   # resolves — user-defined bridge networks get embedded DNS
```

Docker's original default bridge (`docker0`) predates user-defined
networks and does **not** provide automatic DNS resolution between
containers by name — only linking by IP or the legacy `--link` flag
works there. Any user-defined bridge network (`docker network create ...`,
or anything Compose creates) does get the embedded DNS server described
in Level 2. This is the concrete, practical reason to always create an
explicit network rather than relying on the legacy default bridge.

## `host`: no network namespace isolation

```bash
docker run -d --network host nginx
# nginx now listens on port 80 of the HOST directly — no -p mapping needed or possible
```

A `host`-networked container shares the host's network namespace
entirely: same interfaces, same IP, same port space. There is no
container-to-host port translation, which removes a small amount of
overhead but also means two containers can't both bind the same port,
and the container can see/bind every interface the host itself has —
appropriate for niche performance-critical cases, rarely appropriate as a
default choice given the isolation you give up.

## `overlay`: multi-host networking for Swarm

```bash
docker network create --driver overlay --attachable my-overlay
```

An overlay network makes containers on *different Docker hosts* (joined
into a Swarm, module 08) reach each other by service name as if they
were on one flat network, by encapsulating container traffic inside
VXLAN tunnels between hosts. `--attachable` allows standalone
`docker run` containers (not just Swarm services) to join it too — useful
for debugging a Swarm-based application from an ad hoc container.

## `macvlan`: a container as a first-class LAN citizen

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 mac-lan

docker run -d --network mac-lan --ip=192.168.1.50 nginx
```

The container gets its own MAC address and an IP from the physical LAN's
own address space, appearing to the rest of the network (switches, DHCP
servers, other physical machines) as an independent device rather than
something behind the host's IP via NAT — used when legacy network
tooling expects to see and address containers as if they were physical
hosts.

## Worked example: comparing `bridge` vs `host` latency characteristics

```bash
docker run -d --name bridged --network mynet -p 8080:80 nginx
docker run -d --name hosted --network host nginx
# hosted skips one layer of NAT/port-translation per packet compared to bridged,
# measurable under high-throughput/low-latency workloads via a benchmark tool
# such as wrk against each; not benchmarked here, but the mechanism below explains why
```

## How It Actually Works

**Bridge networking's per-packet cost is real and comes from netfilter
NAT, not from namespaces themselves.** A bridge-networked container's
published port (`-p 8080:80`) works via an `iptables`/`nftables` DNAT
rule the daemon installs on the host: an inbound packet to
`host_ip:8080` gets its destination rewritten to
`container_ip:80` before being forwarded across the veth pair into the
container's network namespace, and the reverse NAT happens on the return
path. Every packet pays this rewrite cost. `host` networking has no veth
pair and no DNAT rule at all — the container's process binds directly to
a socket in the host's own network namespace, so packets take the exact
same path they would for any other host process, which is the concrete
mechanism behind the "less overhead" claim, not merely a configuration
convenience.

**Why overlay networks need VXLAN, specifically.** A container's IP on an
overlay network is meaningful only within that overlay's virtual address
space, which has no relationship to the underlying hosts' real IP
addresses or routing. VXLAN solves this by encapsulating each container's
Ethernet frame inside a UDP packet addressed to the *destination host's*
real IP (looked up via a distributed key-value mapping the Swarm
control-plane maintains of "which host currently runs which overlay
IP"), which is then decapsulated back into a native frame on the
receiving host's `overlay` bridge before local delivery to the target
container's veth. This tunnel-in-a-tunnel design is precisely what
allows two containers with private, overlapping-possible overlay IPs to
communicate correctly even when the physical hosts sit on completely
unrelated real networks with firewalls in between (provided the VXLAN UDP
port itself is permitted).

## Exercise

Create two user-defined bridge networks, `net-a` and `net-b`. Start a
container attached to both, and one container each attached to only one.
Use `docker network connect`/`disconnect` to move the dual-homed
container off `net-a` while it's running, and confirm with `docker exec
... getent hosts` that name resolution for the peer on `net-a` stops
working immediately, without restarting any container.
