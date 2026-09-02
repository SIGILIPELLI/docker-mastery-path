# 09 · Networking Basics

Containers need to talk to the outside world, to the host, and to each
other. Docker sets up virtual networking for this automatically, built
around a small set of network **drivers**.

## The default: bridge networking

When Docker is installed, it creates a default network named `bridge`
(using the `bridge` driver). Unless told otherwise, `docker run` attaches
a new container to this network.

```bash
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# 8f2a1b3c...    bridge    bridge    local
# ...            host      host      local
# ...            none      null      local
```

A bridge network works like a private virtual switch on the host:

- Docker creates a virtual network interface on the host (commonly
  `docker0`).
- Each container gets its own virtual ethernet interface, connected to
  that bridge, and its own private IP address on an internal subnet
  (e.g. `172.17.0.2`).
- Containers on the same bridge network can reach each other directly by
  IP.
- Traffic leaving the bridge to the outside internet is NATed through the
  host.
- Nothing *outside* the host can reach a container's ports unless you
  explicitly publish them with `-p` (module 04) — this is deliberate:
  bridge networking isolates containers from the host's external network
  by default.

```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx
# 172.17.0.2
```

## Port mapping recap

```bash
docker run -d -p 8080:80 nginx
```

`-p <host>:<container>` creates a NAT rule (via iptables on Linux) that
forwards traffic hitting `<host_port>` on the host to `<container_port>`
inside the container's network namespace. Without `-p`, the container is
reachable from other containers on the same bridge network (by its
internal IP), but not from the host's regular network interface at the
mapped port.

## User-defined bridge networks (and container name resolution)

The **default** bridge network only lets containers reach each other by
IP address, which is fragile (IPs can change on restart). Creating your
own bridge network gets you automatic DNS-based service discovery by
container name:

```bash
docker network create app-net

docker run -d --name backend --network app-net myapp-backend
docker run -d --name frontend --network app-net myapp-frontend
```

Now, from inside the `frontend` container, the hostname `backend`
resolves automatically to `backend`'s IP address on `app-net` — Docker
runs an embedded DNS server for user-defined networks that resolves
container names (and any `--network-alias`es) to their current IPs. This
does **not** work on the default `bridge` network, only on user-defined
ones — a small but important distinction, and one reason user-defined
networks are recommended over the default bridge for anything with more
than one container.

```bash
docker exec frontend ping -c 2 backend
# PING backend (172.19.0.2): 56 data bytes
# 64 bytes from 172.19.0.2: icmp_seq=0 ttl=64 time=0.09 ms
```

## Other network drivers, briefly

| Driver | What it does |
|---|---|
| `bridge` | The default; an isolated private network on a single host (covered above) |
| `host` | The container shares the host's network namespace directly — no isolation, no port mapping needed, container ports *are* host ports |
| `none` | The container gets no network interface at all beyond loopback |
| `overlay` | Spans multiple Docker hosts (used in Swarm/multi-host setups) — covered in Level 3 |

```bash
docker run -d --network host nginx
# nginx's port 80 is now directly the host's port 80 -- no -p needed,
# but also no isolation and no remapping possible
```

`host` networking trades isolation for a small performance/simplicity
gain, and is mostly used for specific cases (certain monitoring agents,
performance-sensitive services) rather than as a default choice.

## Inspecting network details

```bash
docker network inspect app-net
```

Returns JSON describing the subnet, gateway, and every container
currently attached, along with each container's IP on that network —
useful for debugging "why can't service A reach service B."

```bash
docker network connect app-net some-other-container   # attach after the fact
docker network disconnect app-net some-other-container
```

A running container can be connected to multiple networks simultaneously,
and can be added to or removed from a network without being restarted.

## Worked example: two containers talking by name

```bash
docker network create demo-net

docker run -d --name redis-srv --network demo-net redis:7

docker run --rm --network demo-net redis:7 \
  redis-cli -h redis-srv ping
# PONG
```

The second container reaches the first purely by the name `redis-srv` —
no IP address needed, because both are on the same user-defined network
and Docker's embedded DNS resolved the name.

## Exercise

Create a user-defined bridge network called `lab-net`. Start an `nginx`
container named `web` attached to it (no need to publish any port to the
host). Then run a throwaway `alpine` container on the same network
(`docker run --rm --network lab-net alpine wget -qO- http://web`) and
confirm it can fetch nginx's default page purely by the name `web`. Then
try the same `wget` command with a container **not** attached to
`lab-net` and confirm it fails to resolve `web` at all — demonstrating
that name resolution is scoped to the network.
