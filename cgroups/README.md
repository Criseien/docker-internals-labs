# cgroups — Control Groups

## Scenario

A process is consuming too much CPU and memory on a shared node, affecting
other workloads. No container runtime available — you need to limit it
manually using cgroups directly.

Your job: create a cgroup, set CPU and memory limits, attach the process,
verify the limits are enforced, and freeze it without killing it.

---

## cgroups v2 — AlmaLinux 9

AlmaLinux 9 uses cgroups v2 (unified hierarchy). One tree, one interface.

```bash
# Verify cgroups v2 is active
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 ...   ← v2 confirmed

# v1 would show multiple mounts like:
# cgroup on /sys/fs/cgroup/cpu type cgroup ...
# cgroup on /sys/fs/cgroup/memory type cgroup ...
```

In v2, all controllers live under `/sys/fs/cgroup/`. You create a subdirectory
and write to its interface files to configure limits.

---

## How to Build a cgroup Manually

### Step 1 — Create the cgroup directory

```bash
mkdir /sys/fs/cgroup/myapp
ls /sys/fs/cgroup/myapp
# cgroup.controllers  cgroup.events  cgroup.freeze  cgroup.max.depth
# cgroup.procs        cpu.max        memory.max     memory.current
# ...
```

The kernel populates the directory automatically with interface files.

---

### Step 2 — Enable controllers

Controllers (cpu, memory, io) must be enabled in the parent cgroup before
a child cgroup can use them.

```bash
# Check which controllers are available
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc

# Enable cpu and memory for children
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control

# Verify the child cgroup has them
cat /sys/fs/cgroup/myapp/cgroup.controllers
# cpu memory
```

---

### Step 3 — Set CPU limit

`cpu.max` format: `quota period` — the process can use `quota` microseconds
every `period` microseconds.

```bash
# Limit to 50% of one CPU (50000 out of 100000 microseconds)
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max

# Limit to 200% (2 full CPUs)
echo "200000 100000" > /sys/fs/cgroup/myapp/cpu.max

# No limit (default)
echo "max 100000" > /sys/fs/cgroup/myapp/cpu.max

# Verify
cat /sys/fs/cgroup/myapp/cpu.max
# 50000 100000
```

---

### Step 4 — Set memory limit

```bash
# Limit to 256MB
echo $((256 * 1024 * 1024)) > /sys/fs/cgroup/myapp/memory.max
# or
echo "268435456" > /sys/fs/cgroup/myapp/memory.max

# Check current usage
cat /sys/fs/cgroup/myapp/memory.current
# 0   ← no processes attached yet

# Verify limit
cat /sys/fs/cgroup/myapp/memory.max
# 268435456
```

---

### Step 5 — Attach a process

```bash
# Get the PID of the process you want to limit
pidof myprocess
# 4821

# Add PID to the cgroup
echo 4821 > /sys/fs/cgroup/myapp/cgroup.procs

# Verify it's in the cgroup
cat /sys/fs/cgroup/myapp/cgroup.procs
# 4821
```

All child processes spawned by PID 4821 automatically join the same cgroup.

---

### Step 6 — Freeze the cgroup

Freezing suspends all processes in the cgroup without killing them. They stay
in memory, their state is preserved, but they get no CPU time.

```bash
# Freeze
echo 1 > /sys/fs/cgroup/myapp/cgroup.freeze

# Verify — processes still exist but are suspended
cat /sys/fs/cgroup/myapp/cgroup.freeze
# 1
ps aux | grep myprocess
# D state (uninterruptible) — frozen

# Unfreeze
echo 0 > /sys/fs/cgroup/myapp/cgroup.freeze
```

Use case: freeze a runaway process during incident investigation without
losing its state or logs.

---

### Step 7 — Verify limits are enforced

```bash
# Run a CPU stress test inside the cgroup
# Start the process, get its PID, attach it, then verify CPU stays at limit

# Watch CPU usage in real time
watch -n 1 "cat /sys/fs/cgroup/myapp/cpu.stat"
# usage_usec — total CPU time used
# throttled_usec — time the cgroup was throttled (above limit)
# nr_throttled — number of throttling events

# Memory pressure
cat /sys/fs/cgroup/myapp/memory.events
# oom — times the OOM killer fired
# oom_kill — processes killed by OOM
```

---

## The 90% — cgroup Failure Modes

### 1. Controller not available in the cgroup

**Symptom:**

```bash
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
# bash: /sys/fs/cgroup/myapp/cpu.max: No such file or directory
```

The `cpu` controller was not enabled in the parent.

**Fix:**

```bash
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control
# Now cpu.max exists in child cgroups
```

---

### 2. PID not moving to cgroup — already in a non-root cgroup

**Symptom:**

```bash
echo 4821 > /sys/fs/cgroup/myapp/cgroup.procs
# bash: echo: write error: No such file or directory
# or the PID appears in the file then disappears
```

A process can only be in one cgroup leaf. If it's already managed by
systemd or Docker, it's in a different cgroup.

**Diagnose:**

```bash
cat /proc/4821/cgroup
# 0::/system.slice/myservice.service   ← already in systemd's cgroup
```

**Fix:** you can still move it, but the existing cgroup manager (systemd,
containerd) may move it back. For persistent limits, configure them at
the service level:

```bash
systemctl set-property myservice.service CPUQuota=50% MemoryMax=256M
```

---

### 3. OOM killer fires — process dies instead of waiting

**Symptom:** process dies under memory pressure.

```bash
cat /sys/fs/cgroup/myapp/memory.events
# oom 3
# oom_kill 1
```

By default, when a cgroup hits `memory.max`, the OOM killer fires and kills
the most expensive process.

**Fix:** set a swap limit or configure OOM behavior:

```bash
# Allow some swap before OOM
echo $((512 * 1024 * 1024)) > /sys/fs/cgroup/myapp/memory.swap.max
```

---

### 4. Limits disappear after reboot

**Symptom:** cgroup directory and limits are gone after restart.

`/sys/fs/cgroup/` is a virtual filesystem in memory — everything written
there is lost on reboot. For persistent limits, use systemd:

```bash
# Persistent CPU and memory limits for a service
systemctl set-property myservice.service CPUQuota=50% MemoryMax=256M --runtime=false

# Verify
systemctl show myservice.service | grep -E "CPU|Memory"
```

---

## Key Commands

```bash
# Inspect
mount | grep cgroup                           # verify v2
cat /sys/fs/cgroup/cgroup.controllers         # available controllers
cat /proc/<pid>/cgroup                        # which cgroup a process is in

# Build
mkdir /sys/fs/cgroup/myapp
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control

# Limits
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max    # 50% of 1 CPU
echo "268435456"    > /sys/fs/cgroup/myapp/memory.max  # 256MB

# Attach
echo <pid> > /sys/fs/cgroup/myapp/cgroup.procs

# Freeze / unfreeze
echo 1 > /sys/fs/cgroup/myapp/cgroup.freeze
echo 0 > /sys/fs/cgroup/myapp/cgroup.freeze

# Monitor
cat /sys/fs/cgroup/myapp/cpu.stat
cat /sys/fs/cgroup/myapp/memory.current
cat /sys/fs/cgroup/myapp/memory.events

# Persistent (systemd)
systemctl set-property <service> CPUQuota=50% MemoryMax=256M
```

## K8s Connection

Every pod's resource limits (`resources.limits.cpu` and `resources.limits.memory`
in the pod spec) map directly to cgroup files on the node. Kubernetes creates
a cgroup hierarchy under `/sys/fs/cgroup/kubepods/` — one cgroup per pod,
one per container.

```bash
# On a K8s node — find a pod's cgroup
systemd-cgls | grep <pod-uid>
cat /sys/fs/cgroup/kubepods/burstable/<pod-uid>/cpu.max
```

`cpu.max` corresponds to `resources.limits.cpu`. `memory.max` corresponds
to `resources.limits.memory`. When a container is OOM-killed in K8s, the
same `memory.events` file is what the kubelet reads to report the event.

`docker pause` and `kubectl` pod suspension use the same `cgroup.freeze`
interface practiced here.
