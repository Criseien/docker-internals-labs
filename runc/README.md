# runc — Container from Scratch

## Scenario

No Docker, no containerd. Start an nginx container manually using only runc —
prepare the bundle, configure networking with a veth pair, start the process,
and verify it's accessible from the host.

This is what Docker does automatically every time you run a container.

---

## The Two-Step Model: create vs start

runc splits container startup into two steps. This is important to understand:

```
runc create  →  creates namespaces + starts runc init stub (holds namespaces open)
runc start   →  replaces runc init with the actual process (nginx)
```

`runc run` combines both in one call, but doesn't expose the intermediate state.
The two-step model is what container runtimes like containerd use — they wire
up networking between `create` and `start`, while namespaces already exist.

---

## How to Run a Container with runc

### Step 1 — Create the bundle directory

```bash
mkdir /tmp/nginx-bundle
cd /tmp/nginx-bundle
```

The bundle is a directory containing exactly two things: `config.json` and `rootfs/`.

---

### Step 2 — Generate config.json

```bash
runc spec
ls
# config.json   ← OCI runtime spec generated with defaults
```

---

### Step 3 — Prepare the rootfs from nginx:latest

```bash
mkdir rootfs

# Export nginx image filesystem into rootfs/
docker export $(docker create nginx:latest) | tar -xC rootfs/

ls rootfs/
# bin  dev  docker-entrypoint.d  etc  home  lib  media  mnt  opt
# proc  root  run  sbin  srv  sys  tmp  usr  var
```

---

### Step 4 — Configure process.args in config.json

The default spec runs `sh`. Change it to run nginx as a foreground process:

```bash
# Edit config.json — find the process.args field
# Change: "args": ["sh"]
# To:     "args": ["nginx", "-g", "daemon off;"]
```

`-g daemon off;` is required — without it nginx forks to background and PID 1
exits immediately, which causes the container to stop.

Also set `"terminal": false` since nginx doesn't need a TTY.

---

### Step 5 — Create the container

```bash
runc create nginx-test

# Check state — created but not running yet
runc state nginx-test
# {
#   "status": "created",
#   "pid": 12345,    ← this is runc init, NOT nginx yet
#   ...
# }

runc list
# ID           PID    STATUS    BUNDLE
# nginx-test   12345  created   /tmp/nginx-bundle
```

At this point, the namespaces exist and `runc init` is holding them open.
nginx has not started yet.

---

### Step 6 — Configure networking

By default the container's network namespace has only `lo`. To reach nginx
from the host, set up a veth pair and move one end into the container's namespace.

**The trap:** `ip netns` won't see runc's network namespaces because runc
doesn't create the symlink in `/run/netns/`. Fix it manually:

```bash
# Get runc init PID
RUNC_INIT_PID=$(runc state nginx-test | python3 -m json.tool | grep '"pid"' | awk '{print $2}' | tr -d ',')

# Create the symlink so ip netns can see the namespace
mkdir -p /run/netns
ln -sT /proc/$RUNC_INIT_PID/ns/net /run/netns/nginx-test

# Verify ip netns can see it now
ip netns list
# nginx-test
```

Now configure the veth pair:

```bash
# Create veth pair
ip link add veth-host type veth peer name veth-ctr

# Move one end into the container's namespace
ip link set veth-ctr netns nginx-test

# Configure host side
ip addr add 192.168.0.1/24 dev veth-host
ip link set veth-host up

# Configure container side
ip netns exec nginx-test ip addr add 192.168.0.2/24 dev veth-ctr
ip netns exec nginx-test ip link set veth-ctr up
ip netns exec nginx-test ip link set lo up

# Verify
ip netns exec nginx-test ip -br a
# lo        UNKNOWN  127.0.0.1/8
# veth-ctr  UP       192.168.0.2/24
```

---

### Step 7 — Start the container

```bash
runc start nginx-test

# Verify nginx is running
runc state nginx-test
# "status": "running"

runc list
# ID           PID    STATUS    BUNDLE
# nginx-test   12389  running   /tmp/nginx-bundle
# PID changed — runc init was replaced by nginx
```

---

### Step 8 — Verify nginx is accessible from the host

```bash
curl http://192.168.0.2:80
# <!DOCTYPE html>
# <html>
# <head><title>Welcome to nginx!</title>...
```

---

### Step 9 — Cleanup

```bash
# Stop the container
runc kill nginx-test SIGTERM

# Wait for it to stop, then delete
runc state nginx-test   # check status is "stopped"
runc delete nginx-test

# Remove the netns symlink
rm /run/netns/nginx-test

# Remove the veth pair on the host
ip link del veth-host   # deletes both ends

# Remove the bundle
rm -rf /tmp/nginx-bundle
```

Verify everything is clean:

```bash
runc list         # empty
ip link show      # no veth-host
ip netns list     # no nginx-test
```

---

## The 90% — runc Failure Modes

### 1. ip netns can't see the container namespace

**Symptom:**

```bash
ip netns list
# (empty — doesn't show nginx-test)
```

runc doesn't create the `/run/netns/` symlink automatically. Without it,
`ip netns exec` can't target the container's namespace.

**Fix:**

```bash
mkdir -p /run/netns
ln -sT /proc/$RUNC_INIT_PID/ns/net /run/netns/nginx-test
```

---

### 2. nginx exits immediately after runc start

**Symptom:**

```bash
runc start nginx-test
runc state nginx-test
# "status": "stopped"
```

nginx forked to background and PID 1 exited, killing the container.

**Fix:** `process.args` must include `-g daemon off;`:

```json
"args": ["nginx", "-g", "daemon off;"]
```

---

### 3. container already exists on runc create

**Symptom:**

```bash
runc create nginx-test
# container with id exists: nginx-test
```

**Fix:**

```bash
runc kill nginx-test SIGKILL 2>/dev/null
runc delete nginx-test 2>/dev/null
runc create nginx-test
```

---

### 4. curl fails — veth not configured before runc start

**Symptom:** container is running but curl returns "Connection refused" or
"No route to host".

The veth pair must be configured between `runc create` and `runc start`.
If you ran `runc start` before configuring the veth, the container is running
but its network namespace has no usable interface.

**Fix:** stop, delete, and redo — configure networking after `create`,
before `start`.

---

## Key Commands

```bash
# Bundle setup
mkdir mybundle && cd mybundle
runc spec                                            # generate config.json
mkdir rootfs
docker export $(docker create nginx:latest) | tar -xC rootfs/

# Container lifecycle (two-step)
runc create <id>                  # create namespaces, hold with runc init
runc state <id>                   # check status and PID
runc start <id>                   # replace runc init with actual process
runc kill <id> SIGTERM            # stop
runc delete <id>                  # remove

# One-step (skips networking window)
runc run <id>

# List
runc list

# Networking trick
mkdir -p /run/netns
ln -sT /proc/<runc-init-pid>/ns/net /run/netns/<id>
ip netns exec <id> ip a
```

## K8s Connection

containerd uses exactly this two-step model for every pod:

1. `runc create` — namespaces ready, `runc init` holding them
2. CNI plugin runs — sets up veth pair, bridge, IP (the manual steps above, automated)
3. `runc start` — actual container process starts

This is why pod networking must be configured before the process starts, and
why CNI failures cause `CreateContainerError` — the `runc start` step never
gets called if CNI fails between create and start.
