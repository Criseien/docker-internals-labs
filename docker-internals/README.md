# Docker Internals

## Scenario

containerd is running but `docker ps` fails. A container that was working
yesterday is gone after `systemctl stop docker`. A binary in PATH returns
"not found". Container writes disappeared after removal.

Your job: understand each layer well enough to know which one broke and fix
it without guessing.

---

## The Architecture Chain

```
docker CLI → dockerd → containerd → containerd-shim → runc → kernel
```

| Layer | Responsibility |
|---|---|
| `docker CLI` | Translates commands to API calls to dockerd |
| `dockerd` | General manager — images, networks, volumes |
| `containerd` | Container lifecycle — pull image, prepare filesystem, start/stop |
| `containerd-shim` | Per-container process — keeps container alive independently of dockerd |
| `runc` | Creates namespaces and cgroups, then exits |

The critical detail: **runc exits after creating the container**. The container
stays alive via containerd-shim. This is why `systemctl stop docker` does not
kill running containers — and why you can restart dockerd in production without
downtime.

```bash
systemctl stop docker
ps aux | grep containerd-shim   # still running — containers are alive
```

---

## docker run Flags

```bash
# -i  keep STDIN open — for pipes, even without a terminal
echo "hello" | docker run -i alpine cat

# -t  allocate a pseudo-TTY — terminal simulation
# -it  interactive session with a terminal (combine both)
docker run -it alpine sh

# -d  detached — run in background, print container ID
docker run -d nginx

# -e  set environment variable inside the container
docker run -e DB_HOST=10.0.0.5 -e DB_PORT=5432 myapp

# --entrypoint  override the default command defined in the image
docker run --entrypoint /bin/sh nginx        # open a shell instead of starting nginx
docker run --entrypoint "" nginx /bin/sh     # clear entrypoint entirely
```

**The trap with `-t` in pipes:**

```bash
echo "data" | docker run -it alpine cat
# input device is not a TTY — fails
# -t requires a real terminal. In scripts and pipes, use -i only.
```

**Alpine uses `sh`, not `bash`:**

```bash
docker run -it alpine bash    # OCI runtime exec failed — bash not found
docker run -it alpine sh      # correct
```

This applies to K8s debugging: `kubectl exec -it pod -- bash` fails on
Alpine-based pods. Use `sh`.

---

## docker exec vs docker run

This is the most common mental model mistake.

```bash
# docker run — creates a NEW container from the image
docker run -it nginx sh       # starts a brand new nginx container

# docker exec — runs a command in an EXISTING running container
docker exec -it mycontainer sh   # enters the container that's already running
```

If you have a container named `myapp` running and you want to debug it,
`docker run` creates a separate container — it doesn't touch `myapp`.
`docker exec` is what enters the running one.

```bash
# Wrong — creates a new container, doesn't debug the running one
docker run -it myapp-image sh

# Right — enters the running container
docker exec -it myapp sh
```

`docker exec` requires the container to be running. If the container crashed,
use `docker logs` to see what happened before it exited.

---

## overlay2 Filesystem

Every container's filesystem is an overlay of image layers plus a writable layer.

```bash
docker inspect mycontainer --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
# {
#   "LowerDir":  "/var/lib/docker/overlay2/<id>/diff:...",  ← read-only image layers (stacked)
#   "UpperDir":  "/var/lib/docker/overlay2/<id>/diff",      ← container's writes
#   "WorkDir":   "/var/lib/docker/overlay2/<id>/work",      ← internal scratch (kernel use)
#   "MergedDir": "/var/lib/docker/overlay2/<id>/merged"     ← unified view = rootfs the container sees
# }
```

`MergedDir` is what the container sees as `/`. `LowerDir` contains the image
layers (read-only, stacked right-to-left — last layer is topmost). `UpperDir`
captures all writes the container makes.

When you `docker rm`, `UpperDir` is deleted. `LowerDir` (the image) is untouched.
This is why container changes disappear — they only lived in `UpperDir`.

---

## OCI Image Layout

```bash
# Pull image as OCI layout to inspect it locally
crane pull nginx --format oci /tmp/nginx-oci
ls /tmp/nginx-oci
# index.json   blobs/

cat /tmp/nginx-oci/index.json    # contains digest of the manifest
# follow the digest → manifest → lists layers in order
# each layer blob is a compressed tar of filesystem diffs
```

Layer order: layer 0 = base image, last layer = most recent change. Each layer
is only what changed from the previous one. `LowerDir` in overlay2 stacks
these in the same order.

---

## docker export — Extracting a Flat Filesystem

`docker export` extracts a container's complete filesystem as a single tar —
no layers, just the merged result. Useful for building runc bundles or
inspecting the rootfs.

```bash
# Export a running or stopped container's filesystem
docker export mycontainer -o /tmp/rootfs.tar

# Or via intermediate container (useful when you only have an image)
docker create --name temp nginx       # creates container without starting it
docker export temp -o /tmp/rootfs.tar
docker rm temp

# Extract
mkdir /tmp/rootfs
tar -xf /tmp/rootfs.tar -C /tmp/rootfs
ls /tmp/rootfs
# bin  etc  lib  usr  var  ...  ← flat filesystem, no layer metadata
```

`docker export` vs `docker save`:

| Command | Output | Contains |
|---|---|---|
| `docker export` | Flat tar of the container filesystem | No layer metadata, no history |
| `docker save` | Tar of image layers | Full layer stack, config, history |

Use `export` when you need a rootfs for runc or inspection.
Use `save` when you need to transfer the full image to another host.

---

## ctr — containerd CLI

`ctr` talks directly to containerd, bypassing dockerd entirely. It requires
full image references and does not configure networking automatically.

```bash
# Pull an image (full reference required — no implicit defaults)
ctr images pull docker.io/library/nginx:latest

# List images
ctr images ls

# Create a container (does not start it)
ctr containers create docker.io/library/nginx:latest mynginx

# Start the container's task (the actual process)
ctr tasks start mynginx

# List running tasks
ctr tasks ls

# Execute a command inside a running container
ctr tasks exec --exec-id myshell mynginx sh

# Kill and delete
ctr tasks kill mynginx
ctr containers delete mynginx
```

**Networking with ctr:** ctr does not set up veth pairs or bridge connections
automatically. The container starts with no network. To access its network
namespace:

```bash
# Get container PID
ctr tasks ls   # shows PID

# Enter the container's network namespace from the host
nsenter --net=/proc/<pid>/ns/net ip a    # see interfaces inside container
nsenter --net=/proc/<pid>/ns/net ip r    # see routes
```

This is the same `nsenter` pattern used for K8s pod debugging — containerd
is the runtime in both cases.

---

## Container Lifecycle

```bash
# Stop — sends SIGTERM, waits 10s, then SIGKILL
docker stop mycontainer
docker stop -t 30 mycontainer    # wait 30s before SIGKILL

# Kill — sends signal immediately (default SIGKILL)
docker kill mycontainer
docker kill --signal SIGUSR1 mycontainer    # send any signal to PID 1

# Remove
docker rm mycontainer            # must be stopped first
docker rm -f mycontainer         # force — stops and removes

# Pause/unpause — SIGSTOP/SIGCONT via cgroups freezer
docker pause mycontainer         # process suspended, still in memory
docker unpause mycontainer

# Restart policies
docker run --restart no nginx              # never restart (default)
docker run --restart on-failure nginx      # restart on non-zero exit only
docker run --restart always nginx          # always restart, survives daemon restart
docker run --restart unless-stopped nginx  # like always, respects manual stop

# Change restart policy on a running container
docker update --restart on-failure mycontainer
```

Source of truth for restart policy values: `man docker-run`, not `--help`.
`--help` shows the flag exists but not the valid strings.

---

## The 90% — Failure Modes

### 1. Binary in PATH returns "not found"

```bash
ls -la /usr/bin/runc
# ---x--x--x   ← no read bit
chmod 755 /usr/bin/runc    # fix permissions, not reinstall
```

### 2. docker stop doesn't kill containers

By design — containerd-shim keeps them alive. Stop containerd to kill all:

```bash
systemctl stop containerd
```

### 3. Container changes disappear after removal

UpperDir is deleted on `docker rm`. Use volumes for persistence:

```bash
docker run -v /host/path:/container/path myimage
```

### 4. docker ps empty on K8s node

kubelet talks to containerd directly, not through dockerd:

```bash
crictl ps    # correct tool on K8s nodes
```

---

## Key Commands

```bash
# Run
docker run -it alpine sh
docker run -d -e VAR=value --restart always myimage
docker run --entrypoint /bin/sh nginx

# Exec into running container
docker exec -it mycontainer sh

# Inspect
docker inspect <container> --format '{{.State.Pid}}'
docker inspect <container> --format '{{json .GraphDriver.Data}}'

# Export filesystem
docker export mycontainer -o rootfs.tar

# ctr (containerd direct)
ctr images pull docker.io/library/nginx:latest
ctr tasks start <container>
ctr tasks exec --exec-id s1 <container> sh
nsenter --net=/proc/<pid>/ns/net ip a

# Diagnose chain
journalctl -xeu docker.service
journalctl -u containerd
crictl ps
```

## K8s Connection

On a Kubernetes node:

```
kubelet → containerd (CRI) → containerd-shim → runc → kernel
```

dockerd is not in this chain. `crictl` is the right tool, `ctr` is the
lower-level option. The overlay2 filesystem, OCI layers, and container
lifecycle (pause, kill, signals) are identical — K8s just orchestrates
them at scale.

`docker exec` → `kubectl exec`. `docker logs` → `kubectl logs`.
`docker inspect` → `kubectl describe pod` + `crictl inspect`.
