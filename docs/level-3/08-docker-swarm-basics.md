# 08 · Docker Swarm Basics

**Swarm** is Docker's own built-in orchestrator — no separate
installation required, since it ships inside the Docker Engine itself —
for running services across a cluster of multiple hosts, with
replication, rolling updates, and the overlay networking covered in
module 01.

## Initializing a cluster

```bash
docker swarm init --advertise-addr 192.168.1.10
```

This turns the current host into a Swarm **manager** and prints a
`docker swarm join` command (with a join token) to run on other hosts to
add them as **workers**.

```bash
# on a second host
docker swarm join --token SWMTKN-1-xxxx 192.168.1.10:2377
```

```bash
docker node ls
# lists every node in the cluster, its role (Manager/Worker), and status
```

## Services vs. plain containers

In Swarm mode, you deploy **services** rather than individual
containers — a service describes a desired state (which image, how many
replicas, which ports) and Swarm's manager continuously reconciles the
actual cluster toward it.

```bash
docker service create --name web --replicas 3 -p 8080:80 nginx:1.25
docker service ls
docker service ps web
# shows which physical node each of the 3 replicas landed on
```

If a node running one of the replicas goes down, the manager detects it
and schedules a replacement replica on a healthy node automatically —
this self-healing behavior is the core value Swarm adds over
plain `docker run`.

## Scaling and updating a service

```bash
docker service scale web=5
docker service update --image nginx:1.26 --update-parallelism 1 --update-delay 10s web
```

`--update-parallelism 1 --update-delay 10s` performs a **rolling
update**: replace one replica, wait 10 seconds (giving it time to become
healthy), then move to the next — rather than replacing all replicas
simultaneously and risking a total outage if the new image is broken.

## Deploying a full stack with `docker stack deploy`

Swarm reuses Compose file syntax for multi-service deployments:

```yaml
# stack.yml
version: "3.9"
services:
  web:
    image: nginx:1.25
    ports:
      - "8080:80"
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
  api:
    image: registry.example.com/team/api:1.0.0
    deploy:
      replicas: 2
    networks:
      - backend

networks:
  backend:
    driver: overlay
```

```bash
docker stack deploy -c stack.yml myapp
docker stack services myapp
docker stack rm myapp
```

Note `networks.backend.driver: overlay` — this is where module 01's
overlay networking becomes necessary: `api`'s replicas can land on any
node in the cluster, and only an overlay network lets them discover and
reach each other by service name regardless of which physical host each
replica runs on.

## Worked example: surviving a node failure

```bash
docker service create --name resilient --replicas 3 nginx
docker service ps resilient
# note which nodes host each replica

# simulate a node failure by draining it
docker node update --availability drain <node-id>
docker service ps resilient
# the replica that was on the drained node is now Shutdown, and a
# replacement has been scheduled on a remaining active node automatically
```

## How It Actually Works

**How the manager decides the cluster is converged, mechanically.** Swarm
managers run a Raft consensus group among themselves (an odd number of
managers, typically 3 or 5, is recommended specifically because Raft
needs a majority quorum to keep operating through manager failures) that
holds the cluster's desired-state store — every service spec, its
replica count, and current task placement. Each manager continuously
diffs desired state (from that Raft-replicated store) against observed
actual state (worker heartbeats reporting which tasks are running where)
and issues scheduling decisions to close any gap — a crashed replica
simply stops sending "I'm running" heartbeats, which the reconciliation
loop notices within its heartbeat-timeout window and responds to by
scheduling a replacement task elsewhere, all without any human
intervention or external orchestrator.

**Why overlay networking (module 01) is what makes service-name discovery
work identically to single-host Compose.** Each Swarm service gets a
**VIP** (virtual IP) on its overlay network in addition to individual
task IPs; the embedded DNS server, extended for Swarm mode, resolves the
service name to that VIP, and the Linux kernel's IPVS (IP Virtual Server)
module — the same load-balancing technology used elsewhere in the Linux
networking stack — transparently load-balances connections to that VIP
across the current set of healthy task IPs behind it. This is a strictly
more capable version of the "multiple containers register the same DNS
alias" mechanism from Compose's `--scale` (Level 2, module 09): here the
balancing happens at the kernel/VIP level across potentially many
physical hosts, rather than relying on DNS round-robin alone across
containers on one bridge.

## Exercise

Set up a two-node Swarm (a manager and one worker, both can be VMs or
containers-in-a-VM if you don't have two physical machines available),
deploy a 4-replica `nginx` service, confirm with `docker service ps` that
replicas are distributed across both nodes, then drain the worker node
and confirm all 4 replicas migrate to the manager node automatically.
