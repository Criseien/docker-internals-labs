# cgroups — Resource enforcement on the node (what `resources.limits` actually writes)

cgroups v2 is the kernel interface that enforces CPU and memory limits on any
process group — with or without a container runtime. Every `resources.limits.cpu`
and `resources.limits.memory` in a pod spec maps directly to `cpu.max` and
`memory.max` files under `/sys/fs/cgroup/kubepods/`. When a container is
OOM-killed in K8s, the kubelet reads that event from `memory.events` in the
same hierarchy you manipulate here.

---

## The 5 Problems You'll Actually Hit

### 1. `cpu.max` written but CPU still maxes out — controllers not enabled

**When it happens:** You create a cgroup directory and try to write `cpu.max`
immediately. The file doesn't exist.

```bash
mkdir /sys/fs/cgroup/myapp
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
# bash: /sys/fs/cgroup/myapp/cpu.max: No such file or directory
```

**Root cause:** In cgroups v2, controllers must be explicitly enabled in the
parent before child cgroups inherit them.

```bash
# Check what's available at the root
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc

# What's enabled for children right now
cat /sys/fs/cgroup/cgroup.subtree_control
# (empty — nothing enabled by default)
```

**Fix:**

```bash
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control

# Now the interface files exist
cat /sys/fs/cgroup/myapp/cgroup.controllers
# cpu memory
```

---

### 2. Process can't be moved to cgroup — systemd or Docker moves it back

**When it happens:** You write a PID to `cgroup.procs`. It appears briefly,
then disappears.

**Diagnose:**

```bash
cat /proc/4821/cgroup
# 0::/system.slice/myservice.service   ← already owned by systemd
```

**Root cause:** A process can only live in one leaf cgroup. If systemd or
containerd manages the service, it reassigns the PID back to its own
cgroup hierarchy on the next unit state change.

**Fix:** Configure limits at the manager level — don't fight the manager:

```bash
# systemd service — persistent, survives restarts
systemctl set-property myservice.service CPUQuota=50% MemoryMax=256M

# Verify it stuck
systemctl show myservice.service | grep -E "^(CPUQuota|MemoryMax)"
```

For Docker:

```bash
docker run --cpus 0.5 --memory 256m myimage
# Writes cpu.max and memory.max in Docker's cgroup subtree automatically

# Update a running container without restart
docker update --cpus 0.5 --memory 256m mycontainer
```

---

### 3. Process hits `memory.max` → OOM kill instead of waiting

**When it happens:** Memory-intensive process dies unexpectedly with no
application-level crash. Sudden termination.

**Diagnose:**

```bash
cat /sys/fs/cgroup/myapp/memory.events
# oom 3
# oom_kill 1
```

**Root cause:** CPU has a throttle mechanism — process waits for the next
period when it exceeds quota. Memory has no equivalent. When a cgroup hits
`memory.max`, the kernel OOM killer fires and kills the most expensive
process in the cgroup immediately.

**Fix:** Set `memory.max` with headroom, or add a swap buffer before OOM fires:

```bash
echo $((512 * 1024 * 1024)) > /sys/fs/cgroup/myapp/memory.swap.max
```

**Trap:** `memory.current` shows live usage but takes a moment to update after
attaching a process. Don't read it immediately after `echo <pid> > cgroup.procs`.

---

### 4. Cgroup limits disappear after reboot

**When it happens:** Limits set directly in `/sys/fs/cgroup/` work fine, but
they're gone after the next reboot.

**Root cause:** `/sys/fs/cgroup/` is a virtual filesystem in memory (type
`cgroup2`). It is rebuilt from scratch by the kernel at boot. Nothing written
there persists.

**Fix:** Persist via systemd:

```bash
# --runtime=false writes a drop-in file, not just a runtime change
systemctl set-property myservice.service CPUQuota=50% MemoryMax=256M --runtime=false

# Verify the drop-in file was written to disk
cat /etc/systemd/system/myservice.service.d/50-CPUQuota.conf
```

---

### 5. `cgroup.freeze` written but process looks alive in `ps`

**When it happens:** You freeze a runaway process but `ps aux` still shows it
running. You think the freeze didn't work.

**Diagnose:**

```bash
cat /sys/fs/cgroup/myapp/cgroup.freeze
# 1

ps aux | grep myprocess
# myprocess  D  ← D state (uninterruptible sleep) — this IS frozen
```

**Root cause:** Frozen processes enter uninterruptible sleep (D state). `ps`
still shows them — they exist in memory with their state intact, but they
receive zero CPU time. This is correct behavior.

**Use case:** Freeze a runaway process during an incident to stop the damage
without losing its memory state, open file handles, or log context.

```bash
# Freeze
echo 1 > /sys/fs/cgroup/myapp/cgroup.freeze

# Investigate safely — process is suspended, not dead
cat /proc/<pid>/fd | wc -l    # open file descriptors still intact
cat /proc/<pid>/status         # full process state readable

# Resume
echo 0 > /sys/fs/cgroup/myapp/cgroup.freeze
```

---

## Decision: raw cgroup vs systemd properties vs Docker flags

Use **Docker flags** (`--cpus`, `--memory`) for containerized workloads — Docker
writes to the correct cgroup path and limits survive container restarts.
`docker update` changes limits without a restart.

Use **systemd properties** (`systemctl set-property`) for system services.
Persistent across reboots. Never write directly to the cgroup of a systemd
service — systemd will overwrite it on the next reload.

Use **raw cgroup interfaces** only when the process is not managed by any
runtime: an ad-hoc script, a one-off process, or when you need to freeze a
runaway process during an incident without killing it.

The rule: whoever started the process owns its cgroup. Writing directly into
another manager's cgroup is a race you'll lose.

---

## Quick Reference

```bash
# Verify cgroups v2
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 ...

# Setup
mkdir /sys/fs/cgroup/myapp
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control

# Limits
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max    # 50% of 1 CPU
echo "268435456"    > /sys/fs/cgroup/myapp/memory.max  # 256MB
# cpu.max format: <quota_usec> <period_usec>
# 50000/100000 = 50%  |  200000/100000 = 2 CPUs  |  "max 100000" = unlimited

# Attach process
echo <pid> > /sys/fs/cgroup/myapp/cgroup.procs

# Freeze / unfreeze
echo 1 > /sys/fs/cgroup/myapp/cgroup.freeze
echo 0 > /sys/fs/cgroup/myapp/cgroup.freeze

# Monitor
cat /sys/fs/cgroup/myapp/cpu.stat          # throttled_usec, nr_throttled
cat /sys/fs/cgroup/myapp/memory.current    # live usage in bytes
cat /sys/fs/cgroup/myapp/memory.events     # oom, oom_kill counts

# Which cgroup is a process in?
cat /proc/<pid>/cgroup

# Persistent via systemd
systemctl set-property <service> CPUQuota=50% MemoryMax=256M
```

---

## K8s Connection

```bash
# Find a pod's cgroup on the node
systemd-cgls | grep <pod-uid>
cat /sys/fs/cgroup/kubepods/burstable/<pod-uid>/cpu.max
```

`resources.limits.cpu: "500m"` → `500000 1000000` in `cpu.max` (500ms per second).
`resources.limits.memory: "256Mi"` → `268435456` in `memory.max`.

OOM-killed container in K8s: the kubelet reads `memory.events` from the pod's
cgroup to set `OOMKilled: true` in pod status. `docker pause` and pod
suspension both use the `cgroup.freeze` interface practiced here.
