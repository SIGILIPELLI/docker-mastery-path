---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Docker Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A Dockerfile has these lines. After editing only app.py, which steps rebuild (cache miss)?\n```dockerfile\nFROM python:3.12-slim\nWORKDIR /app\nCOPY . .\nRUN pip install -r requirements.txt\nCMD [\"python\", \"app.py\"]\n```\n",
    "options": [
      "Nothing",
      "Only CMD",
      "Only COPY . .",
      "COPY . . and then RUN pip install (every later layer misses once one layer misses)"
    ],
    "correctIndex": 3,
    "explanation": "COPY . . has a changed checksum, so it and every layer after it rebuild. Verified with a small cache simulation. Copying requirements.txt first avoids the reinstall."
  },
  {
    "question": "After reordering to `COPY requirements.txt .` → `RUN pip install` → `COPY . .`, you edit only app.py. What rebuilds?",
    "options": [
      "Only RUN pip install",
      "Everything",
      "Only COPY . .; the pip install layer is CACHED",
      "The FROM layer"
    ],
    "correctIndex": 2,
    "explanation": "The dependency manifest didn't change, so the install layer is reused. Verified by simulation."
  },
  {
    "question": "`docker run --name web nginx` worked yesterday and fails today with a name conflict. Why, and what fixes it?",
    "options": [
      "docker run is create + start, so the container web still exists (Exited); docker rm web, or docker start web",
      "The name is reserved",
      "The image was deleted",
      "Port 80 is in use"
    ],
    "correctIndex": 0,
    "explanation": "A stopped container keeps its name until removed."
  },
  {
    "question": "A container exits with code 137 and no error in its logs. What should you check first?",
    "options": [
      "The registry login",
      "Whether the kernel OOM killer stopped it: docker inspect -f '{{.State.OOMKilled}}'",
      "The image tag",
      "The network driver"
    ],
    "correctIndex": 1,
    "explanation": "Exceeding --memory gets the process killed rather than throttled."
  },
  {
    "question": "`EXPOSE 8000` is in the Dockerfile, but `curl localhost:8000` from the host fails. Why?",
    "options": [
      "The image is too old",
      "EXPOSE is only documentation; you must publish with -p 8000:8000 (or -P)",
      "Port 8000 is reserved",
      "EXPOSE needs a firewall rule"
    ],
    "correctIndex": 1,
    "explanation": "-p creates the NAT rule from host port to container port."
  },
  {
    "question": "In Compose, `depends_on: [db]` is set, yet the api crashes because Postgres isn't accepting connections yet. What is the fix?",
    "options": [
      "Add a healthcheck to db and use depends_on with condition: service_healthy",
      "Use a longer sleep in the Dockerfile",
      "Expose port 5432",
      "Rename the service"
    ],
    "correctIndex": 0,
    "explanation": "depends_on alone only controls start order, not readiness."
  },
  {
    "question": "You run `docker compose down` on a stack with a named volume pgdata. What happens to the database data?",
    "options": [
      "It's compressed",
      "It's kept: down removes containers and networks but keeps named volumes (unless -v is given)",
      "It moves to the image",
      "It's deleted"
    ],
    "correctIndex": 1,
    "explanation": "Named volumes outlive containers."
  },
  {
    "question": "A .env file next to docker-compose.yml has APP_PORT=8080 and the service uses `ports: [\"${APP_PORT}:8000\"]`. Is APP_PORT available as an environment variable inside the container?",
    "options": [
      "Only for build",
      "Yes, automatically",
      "No: .env substitutes values into the compose file; add environment: - APP_PORT=${APP_PORT} to pass it through",
      "Only in Swarm"
    ],
    "correctIndex": 2,
    "explanation": "The three mechanisms differ: ENV in the image, environment:/-e at runtime, and .env for compose-file substitution."
  },
  {
    "question": "Which setup stops the proxy container from reaching the database?",
    "options": [
      "Name the services differently",
      "Put db on an internal network shared only with the api, and the proxy on a public network only",
      "Publish db's port",
      "Use depends_on"
    ],
    "correctIndex": 1,
    "explanation": "Services need a shared network to talk. Verified by parsing a compose file: proxy->db False, api->db True."
  },
  {
    "question": "Which image reference means an official Docker Hub image?",
    "options": [
      "nginx:1.27 (shorthand for docker.io/library/nginx:1.27)",
      "bitnami/nginx",
      "ghcr.io/org/nginx",
      "localhost:5000/nginx"
    ],
    "correctIndex": 0,
    "explanation": "No namespace means library/ on Docker Hub."
  },
  {
    "question": "Why is `myapp:latest` a poor tag to deploy in production?",
    "options": [
      "It's too short",
      "Registries reject it",
      "It's mutable and can silently point at a different image; pin to a git SHA or version",
      "It disables caching"
    ],
    "correctIndex": 2,
    "explanation": "Immutable tags make deployments reproducible and traceable."
  },
  {
    "question": "You want the smallest, hardest-to-abuse final image for a Node API. Which Dockerfile ending is best?\n```dockerfile\nFROM gcr.io/distroless/nodejs20-debian12\nCOPY --from=build /app/dist ./dist\nUSER nonroot\nCMD [\"dist/server.js\"]\n```\n",
    "options": [
      "Switch to FROM node:20 (full image)",
      "Run as root for simplicity",
      "This one: distroless has no shell or package manager, runs as non-root, and uses a build stage artifact",
      "Add RUN apt-get install curl"
    ],
    "correctIndex": 2,
    "explanation": "A shell-less image gives an attacker nothing to pivot with."
  },
  {
    "question": "Which run flags make the root filesystem immutable while still allowing /tmp?",
    "options": [
      "--cap-add=ALL",
      "--network host",
      "--read-only --tmpfs /tmp",
      "--privileged"
    ],
    "correctIndex": 2,
    "explanation": "--tmpfs gives in-memory writable paths without weakening the read-only guarantee."
  },
  {
    "question": "How do you give a Swarm service a database password without an environment variable value?",
    "options": [
      "Put it in .env",
      "Print it in logs",
      "Create a Docker secret and read it from /run/secrets/<name>, e.g. POSTGRES_PASSWORD_FILE",
      "Hard-code it in the image"
    ],
    "correctIndex": 2,
    "explanation": "Secrets are mounted in memory as files, never in the writable layer."
  },
  {
    "question": "A new image version must replace the old one with no dropped requests. Which combination is right?",
    "options": [
      "Restart all replicas at once",
      "Health-gated rolling update (start-first, rollback on failure) plus graceful shutdown",
      "docker stop then docker run",
      "Remove the health check to start faster"
    ],
    "correctIndex": 1,
    "explanation": "Never remove capacity before its replacement is provably ready."
  },
  {
    "question": "A volume holds production data. Which backup is correct?",
    "options": [
      "docker commit the container",
      "Copy /var/lib/docker while running",
      "Back up the image",
      "A throwaway container mounting the volume read-only and writing a tar to a host directory, plus a tested restore into a new volume"
    ],
    "correctIndex": 3,
    "explanation": "Test restores; a backup never restored is a hope, not a plan."
  },
  {
    "question": "Containers on the default bridge can't reach each other by name, but they can on a network created with `docker network create app-net`. Why?",
    "options": [
      "User-defined bridge networks provide embedded DNS name resolution",
      "app-net has more IPs",
      "The default bridge is encrypted",
      "Names need a registry"
    ],
    "correctIndex": 0,
    "explanation": "The legacy default bridge only supports IPs and --link."
  },
  {
    "question": "In CI you push images tagged with the commit SHA. What does that buy you?",
    "options": [
      "Faster pulls",
      "Smaller images",
      "Traceability: every image maps to the exact source that built it",
      "Automatic rollbacks"
    ],
    "correctIndex": 2,
    "explanation": "It's essential when debugging production weeks later."
  },
  {
    "question": "When is Kubernetes justified over Compose or Swarm?",
    "options": [
      "Many hosts with varied hardware, multiple teams needing RBAC and namespaces, metric-based autoscaling, or the Helm/operator ecosystem",
      "Because Docker is deprecated",
      "To look modern",
      "Whenever you have two containers"
    ],
    "correctIndex": 0,
    "explanation": "For a single team with a handful of hosts and predictable load, Swarm or Compose is often the right size."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Isolate the tiers

Fill in the Compose file so that the proxy can reach the api, the api can reach the db, but the proxy cannot reach the db; only the proxy publishes a port; and the data network is internal. The checker parses your YAML and prints the results. (Needs PyYAML: pip install pyyaml.)

**Starter code**

```python
import yaml
COMPOSE = """
services:
  proxy:
    image: nginx:1.25
    # TODO: ports and networks
  api:
    build: ./api
    # TODO: networks
  db:
    image: postgres:16
    volumes: ["pgdata:/var/lib/postgresql/data"]
    # TODO: networks
# TODO: declare the networks (data must be internal)
volumes:
  pgdata:
"""
cfg = yaml.safe_load(COMPOSE)
svc = cfg["services"]
def can_reach(a, b):
    return bool(set(svc[a].get("networks", [])) & set(svc[b].get("networks", [])))
for a, b in [("proxy", "db"), ("api", "db"), ("proxy", "api")]:
    print(f"{a}->{b}: {can_reach(a, b)}")
print("published:", sorted(n for n, s in svc.items() if s.get("ports")))
print("internal networks:", sorted(n for n, v in cfg["networks"].items() if v and v.get("internal")))
```

**Expected output**

```text
proxy->db: False
api->db: True
proxy->api: True
published: ['proxy']
internal networks: ['data']
```

<details><summary>Show solution and explanation</summary>

<pre><code>import yaml
COMPOSE = &quot;&quot;&quot;
services:
  proxy:
    image: nginx:1.25
    ports: [&quot;8080:80&quot;]
    networks: [public]
  api:
    build: ./api
    networks: [public, data]
  db:
    image: postgres:16
    volumes: [&quot;pgdata:/var/lib/postgresql/data&quot;]
    networks: [data]
networks:
  public:
  data:
    internal: true
volumes:
  pgdata:
&quot;&quot;&quot;
cfg = yaml.safe_load(COMPOSE)
svc = cfg[&quot;services&quot;]
def can_reach(a, b):
    return bool(set(svc[a].get(&quot;networks&quot;, [])) &amp; set(svc[b].get(&quot;networks&quot;, [])))
for a, b in [(&quot;proxy&quot;, &quot;db&quot;), (&quot;api&quot;, &quot;db&quot;), (&quot;proxy&quot;, &quot;api&quot;)]:
    print(f&quot;{a}-&gt;{b}: {can_reach(a, b)}&quot;)
print(&quot;published:&quot;, sorted(n for n, s in svc.items() if s.get(&quot;ports&quot;)))
print(&quot;internal networks:&quot;, sorted(n for n, v in cfg[&quot;networks&quot;].items() if v and v.get(&quot;internal&quot;)))
</code></pre>

<p>Services can talk only if they share a network. proxy is on public, db on data, and api on both, so api bridges them. internal: true stops the data network from reaching outside. Verified by running the checker (PyYAML) against the solution.</p>
</details>

### Reorder for the build cache

The NAIVE Dockerfile reinstalls dependencies on every code edit. Write GOOD so that editing app.py rebuilds only COPY . ., while editing requirements.txt rebuilds the install too. The simulation applies the rule 'once one layer misses the cache, every later layer misses'.

**Starter code**

```python
NAIVE = """FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]"""

GOOD = """FROM python:3.12-slim
# TODO: reorder so dependencies are installed before the source is copied
"""

def rebuilt(dockerfile, changed):
    """Layers that must rebuild: once one layer misses the cache, every later layer misses too."""
    missed, out = False, []
    for line in dockerfile.splitlines():
        op, _, rest = line.partition(" ")
        if op not in ("FROM", "WORKDIR", "COPY", "RUN"):
            continue  # CMD only adds metadata
        if op == "COPY":
            src = rest.split()[0]
            if src == "." or src in changed:
                missed = missed or bool(src == "." and changed) or src in changed
        if missed:
            out.append(line)
    return out

print("naive, edit app.py ->", rebuilt(NAIVE, {"app.py"}))
print("good, edit app.py ->", rebuilt(GOOD, {"app.py"}))
print("good, edit requirements.txt ->", rebuilt(GOOD, {"requirements.txt"}))
```

**Expected output**

```text
naive, edit app.py -> ['COPY . .', 'RUN pip install -r requirements.txt']
good, edit app.py -> ['COPY . .']
good, edit requirements.txt -> ['COPY requirements.txt .', 'RUN pip install -r requirements.txt', 'COPY . .']
```

<details><summary>Show solution and explanation</summary>

<pre><code>NAIVE = &quot;&quot;&quot;FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD [&quot;python&quot;, &quot;app.py&quot;]&quot;&quot;&quot;

GOOD = &quot;&quot;&quot;FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD [&quot;python&quot;, &quot;app.py&quot;]&quot;&quot;&quot;

def rebuilt(dockerfile, changed):
    &quot;&quot;&quot;Layers that must rebuild: once one layer misses the cache, every later layer misses too.&quot;&quot;&quot;
    missed, out = False, []
    for line in dockerfile.splitlines():
        op, _, rest = line.partition(&quot; &quot;)
        if op not in (&quot;FROM&quot;, &quot;WORKDIR&quot;, &quot;COPY&quot;, &quot;RUN&quot;):
            continue  # CMD only adds metadata
        if op == &quot;COPY&quot;:
            src = rest.split()[0]
            if src == &quot;.&quot; or src in changed:
                missed = missed or bool(src == &quot;.&quot; and changed) or src in changed
        if missed:
            out.append(line)
    return out

print(&quot;naive, edit app.py -&gt;&quot;, rebuilt(NAIVE, {&quot;app.py&quot;}))
print(&quot;good, edit app.py -&gt;&quot;, rebuilt(GOOD, {&quot;app.py&quot;}))
print(&quot;good, edit requirements.txt -&gt;&quot;, rebuilt(GOOD, {&quot;requirements.txt&quot;}))
</code></pre>

<p>Copy the manifest and install dependencies first, then copy the source. Source edits then only invalidate the last COPY. Verified by running the simulation against the solution.</p>
</details>
