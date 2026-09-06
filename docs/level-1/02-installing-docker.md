# 02 · Installing Docker

Docker ships in a few different distributions depending on your OS. This
module covers what to install, and the commands you'll use immediately
afterward to confirm everything works — described accurately against
documented Docker behavior; exact installer screens change over time, so
always cross-check the current steps at [docs.docker.com](https://docs.docker.com/get-docker/)
if something here looks out of date.

## Choosing what to install

| Platform | Recommended install | Notes |
|---|---|---|
| macOS | **Docker Desktop** | Runs a lightweight Linux VM under the hood; provides the daemon, CLI, and a GUI |
| Windows | **Docker Desktop** (with WSL2 backend) | Needs WSL2 enabled; also runs Linux containers inside a Linux environment |
| Linux | **Docker Engine** (native packages) | Runs natively — no VM needed, since the host kernel *is* Linux |

On Linux you install the Docker Engine directly from your distribution's
package manager (or Docker's official `apt`/`yum` repositories), which
gives you `dockerd` and the `docker` CLI without any GUI. On macOS and
Windows, Docker Desktop bundles the daemon, CLI, a small Linux VM, and a
management GUI into one installer.

## Installing on Linux (Ubuntu/Debian example)

```bash
# Remove any old versions first
sudo apt-get remove docker docker-engine docker.io containerd runc

# Set up Docker's official apt repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine, CLI, containerd, and the compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

After install, the `docker` command talks to the daemon over a Unix socket
that by default is only writable by `root` and members of the `docker`
group. To run `docker` without `sudo`:

```bash
sudo usermod -aG docker $USER
# log out and back in (or `newgrp docker`) for the group change to apply
```

!!! warning "Security note"
    Adding a user to the `docker` group is effectively equivalent to
    giving that user root on the host — a container can be used to mount
    the host filesystem. Only add trusted users.

## Installing on macOS / Windows

1. Download **Docker Desktop** from docker.com for your platform.
2. Run the installer. On Windows, this requires WSL2 (the installer will
   guide you through enabling it if it isn't already).
3. Launch Docker Desktop and wait for the whale icon in the menu bar/tray
   to show it's running.
4. Docker Desktop places the `docker` CLI on your `PATH` automatically.

## Verifying the install

Once installed, three commands confirm everything is working:

```bash
docker --version
# Docker version 27.x.x, build xxxxxxx

docker info
# Prints daemon details: number of containers/images, storage driver,
# server version, and whether it's reachable at all.

docker run hello-world
```

`docker run hello-world` is the standard smoke test. Behind the scenes it:

1. Checks whether the `hello-world` image exists locally.
2. If not, pulls it from Docker Hub.
3. Creates a container from that image and starts it.
4. The container prints a confirmation message and exits.

Expected output looks like:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

If you see that message, the daemon is running, the CLI can talk to it,
and it has network access to pull images — the three things that most
commonly go wrong during setup.

## Common setup problems

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Daemon isn't running | Start Docker Desktop, or `sudo systemctl start docker` on Linux |
| `permission denied` on the socket | User isn't in the `docker` group (Linux) | Run `sudo usermod -aG docker $USER`, then re-login |
| Pull is very slow or times out | Network/proxy or Docker Hub rate limits | Check network; configure a registry mirror or authenticate to Docker Hub |
| WSL2 errors on Windows | WSL2 not enabled or outdated | Run `wsl --update`, enable virtualization in BIOS if needed |

## How It Actually Works

Installing "Docker" actually installs several separate binaries that talk
to each other over Unix sockets — understanding that layering explains
most of the setup problems above.

**The daemon/CLI split.** `docker` (the CLI) is a thin client. It doesn't
build images or run containers itself — every command you type is
serialized into an HTTP request and sent to `dockerd` (the daemon) over a
Unix domain socket, normally `/var/run/docker.sock`. That's why
`permission denied` errors happen: the socket file has the permissions of
a regular file (owned by `root`, group `docker`), and the kernel enforces
normal Unix filesystem permissions on it exactly like any other file —
there's no Docker-specific permission system involved, which is also why
adding a user to the `docker` group is equivalent to root: anyone who can
write to that socket can ask the daemon to bind-mount `/` from the host
into a new container and get a root shell on the host filesystem.

**containerd and runc underneath dockerd.** `dockerd` itself doesn't
create containers directly either. It delegates to **containerd** (a
separate daemon, also installed by `docker-ce`), which manages image
storage and container lifecycle, and containerd in turn shells out to
**runc** for each container start. `runc` is the piece that actually
issues the `clone()`/`unshare()` syscalls to create namespaces and writes
the cgroup files, per the OCI (Open Container Initiative) runtime spec —
then exits once the container is running, handing supervision back to
`containerd-shim`. This is why you'll see `containerd`, `containerd-shim`,
and `runc` processes on a Linux host even though you only ever typed
`docker` commands: it's a chain of four separate programs, each doing one
job, with the OCI spec as the contract between them.

**Why Docker Desktop needs a VM.** `runc` calls Linux-specific syscalls
(`clone` with namespace flags, cgroup v2 file writes) that simply don't
exist on the macOS or Windows kernel. Docker Desktop's Linux VM (via
Apple's Virtualization.framework, or WSL2's real Linux kernel on Windows)
exists solely to give `containerd`/`runc` an actual Linux kernel to issue
those syscalls against — your containers are Linux processes running
inside that VM's kernel, and the Docker Desktop GUI/CLI on the host side
is just a client proxying commands into the VM.

## Exercise

Install Docker on your machine (or verify it's already installed), then
run these three commands and record their output:

```bash
docker --version
docker info
docker run hello-world
```

If `docker info` reports a `Server` section with a version number, your
daemon is reachable — that's the thing to confirm before moving on, since
every later module assumes a working daemon.
