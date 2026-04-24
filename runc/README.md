# runc — The OCI runtime: what actually creates your containers at the kernel level

runc is the last step in the container stack. It creates the Linux namespaces
and cgroups, then exits. The container stays alive via `containerd-shim` — runc
is not a daemon. Every `docker run` and every pod created by kubelet goes through
runc (or a compatible OCI runtime). Understanding the two-step `create → start`
model explains why CNI failures surface as `CreateContainerError` in K8s.

---

## The 5 Problems You'll Actually Hit

### 1. `ip netns` can't see the container namespace

**When it happens:** You've created the container with `runc create`. You want
to configure networking before `runc start`. `ip netns list` shows nothing.

```bash
ip netns list
# (empty)

ip netns exec nginx-test ip a
# Cannot open network namespace "nginx-test": No such file or directory
```

**Root cause:** runc creates the network namespace under `/proc/<pid>/ns/net`
but does not create the symlink in `/run/netns/` that `ip netns` uses. Without
the symlink, the `ip netns` subcommand is blind to the namespace.

**Fix:**

```bash
# Get runc init PID from runc state
RUNC_PID=$(runc state nginx-test | python3 -c "import sys,json; print(json.load(sys.stdin)['pid'])")

# Create the symlink manually
mkdir -p /run/netns
ln -sT /proc/$RUNC_PID/ns/net /run/netns/nginx-test

# Now ip netns can see it
ip netns list
# nginx-test

ip netns exec nginx-test ip a
# lo   UNKNOWN   127.0.0.1/8
```

**Trap:** The symlink must be created while `runc init` is still running (i.e.,
after `runc create`, before `runc start` or before the container exits). Once
the PID is gone, the symlink is stale.

---

### 2. Container exits immediately after `runc start`

**When it happens:** `runc start` succeeds, but `runc state` shows `stopped`
within seconds.

```bash
runc start nginx-test
runc state nginx-test
# "status": "stopped"
```

**Diagnose:**

```bash
runc list
# ID           PID   STATUS
# nginx-test   0     stopped   ← PID 0 means the process exited
```

**Root cause:** nginx forks to background by default. When PID 1 of the
container exits (the nginx master that forked), the container stops. There
is no init process holding it open.

**Fix:** Force foreground mode in `config.json`:

```json
"args": ["nginx", "-g", "daemon off;"]
```

Also set `"terminal": false` — nginx doesn't need a TTY and the default spec
has it enabled.

---

### 3. `runc create` fails — container already exists

**When it happens:** You're redoing the lab or a previous run failed mid-way.

```bash
runc create nginx-test
# container with id exists: nginx-test
```

**Fix:**

```bash
runc kill nginx-test SIGKILL 2>/dev/null; sleep 1
runc delete nginx-test 2>/dev/null
runc create nginx-test
```

**Trap:** `runc delete` fails if the container is still in `running` or
`created` state. Always `kill` first, wait a moment, then `delete`.

---

### 4. `curl` fails from the host — networking not configured before start

**When it happens:** Container is running, but `curl http://192.168.0.2:80`
returns "No route to host" or hangs.

**Diagnose:**

```bash
runc state nginx-test
# "status": "running"  ← container is alive

ip netns exec nginx-test ip a
# lo   UNKNOWN   127.0.0.1/8
# (no veth interface — namespace has no usable network)
```

**Root cause:** The veth pair was configured after `runc start`. By the time
the interface was moved into the namespace, nginx was already running and
bound to loopback only.

**Root cause (deeper):** The `create → start` gap is the only window where the
namespace exists but the process hasn't started yet. Network must be wired up
in that window. This is exactly what CNI plugins do in K8s.

**Fix:** Stop, delete, and redo. Wire networking between `create` and `start`:

```bash
runc create nginx-test        # namespace exists, runc init holds it open

# Wire networking now (before start)
ip link add veth-host type veth peer name veth-ctr
ip link set veth-ctr netns nginx-test
ip addr add 192.168.0.1/24 dev veth-host
ip link set veth-host up
ip netns exec nginx-test ip addr add 192.168.0.2/24 dev veth-ctr
ip netns exec nginx-test ip link set veth-ctr up
ip netns exec nginx-test ip link set lo up

runc start nginx-test         # nginx starts with networking already configured
curl http://192.168.0.2:80    # works
```

---

### 5. rootfs export truncated or nginx missing from rootfs

**When it happens:** `runc start` fails with "exec: nginx: no such file" or
rootfs directory is empty.

```bash
runc start nginx-test
# container_linux.go: starting container process: exec: "nginx": executable file not found
```

**Diagnose:**

```bash
ls rootfs/usr/sbin/nginx   # check if the binary exists in the rootfs
```

**Root cause:** `docker export` pipe to `tar -x` failed silently — usually
because the intermediate container ID was wrong, or `docker create` wasn't
given enough time before the export.

**Fix:**

```bash
# Use a named container to avoid race
docker create --name temp-nginx nginx:latest
docker export temp-nginx | tar -xC rootfs/
docker rm temp-nginx

# Verify the binary is there
ls -la rootfs/usr/sbin/nginx
```

---

## Decision: `runc create` + `runc start` vs `runc run`

Use **`runc run`** when you just want to quickly test that a bundle works. It
combines both steps but gives you no window to configure networking.

Use **`runc create` + `runc start`** whenever you need to do anything between
namespace creation and process start: wire a veth pair, set up mounts, or
inspect the namespace before the workload begins. This is the model containerd
uses — CNI runs in the gap between these two calls.

**K8s translation:** `CreateContainerError` means the CNI plugin failed between
`runc create` and `runc start`. The namespaces were created successfully, but
the network wasn't wired up, so `runc start` was never called.

---

## Quick Reference

```bash
# Bundle setup
mkdir mybundle && cd mybundle
runc spec                                             # generate config.json
mkdir rootfs
docker create --name temp nginx:latest
docker export temp | tar -xC rootfs/
docker rm temp

# Two-step lifecycle
runc create <id>                  # namespaces created, runc init holding them
runc state <id>                   # check PID and status ("created")
# ↑ configure networking here ↑
runc start <id>                   # replaces runc init with actual process
runc kill <id> SIGTERM            # send signal to PID 1
runc delete <id>                  # remove after stopped

# One-step (no networking window)
runc run <id>

# List all containers
runc list

# ip netns symlink trick
mkdir -p /run/netns
ln -sT /proc/<runc-init-pid>/ns/net /run/netns/<id>
ip netns exec <id> ip a
ip netns exec <id> ip r

# Cleanup
runc kill <id> SIGKILL 2>/dev/null; sleep 1
runc delete <id>
rm /run/netns/<id>
ip link del veth-host
rm -rf /path/to/bundle
```

---

## K8s Connection

containerd uses exactly the two-step model for every pod:

1. `runc create` — namespaces ready, `runc init` holds them open
2. CNI plugin runs — sets up veth pair, IP address, routes (the manual steps above, automated)
3. `runc start` — actual container process starts with networking already in place

On a K8s node, use `crictl` to inspect this layer:

```bash
crictl ps                             # running containers (containerd, not dockerd)
crictl inspect <container-id>         # OCI state including PID
crictl logs <container-id>
```

`CreateContainerError` in K8s events → CNI failed between steps 1 and 3.
Check CNI plugin logs: `journalctl -u containerd | grep -i cni`.
