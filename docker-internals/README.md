# Docker Internals — The dockerd → containerd → runc chain and what breaks in each layer

The Docker stack is five components deep. When something breaks, the error
message tells you the symptom but not the layer. Knowing the chain tells you
where to look: `docker CLI → dockerd → containerd → containerd-shim → runc → kernel`.
The critical detail: **runc exits after creating the container**. containerd-shim
keeps the container alive independently of dockerd. This is why restarting
dockerd in production does not kill running containers.

---

## The 5 Problems You'll Actually Hit

### 1. `docker ps` returns nothing on a K8s node — dockerd is not in the chain

**When it happens:** You SSH into a K8s worker node to debug a pod. `docker ps`
returns an empty list even though pods are clearly running.

**Root cause:** On a Kubernetes node, the stack is
`kubelet → containerd (CRI) → runc`. dockerd is not installed or not used.
`docker ps` talks to dockerd — which doesn't know about containers started
by kubelet directly through containerd.

**Fix:** Use the right tool for the right layer:

```bash
crictl ps                             # containerd's containers (what kubelet sees)
crictl ps --name <pod-name>
crictl logs <container-id>
crictl inspect <container-id>         # full OCI state, PID, mounts

# Lower-level: bypass CRI, talk directly to containerd
ctr -n k8s.io containers ls
```

**Trap:** `crictl` requires `--runtime-endpoint` on some setups:

```bash
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
```

---

### 2. `systemctl stop docker` — containers keep running

**When it happens:** You stop Docker to apply a config change. You expect
containers to stop. They don't.

```bash
systemctl stop docker
ps aux | grep containerd-shim
# containerd-shim-runc-v2 -namespace moby -id <container-id>  ← still running
```

**Root cause:** Each running container has its own `containerd-shim` process.
The shim's job is to hold the container alive independently of both dockerd
and containerd. Stopping dockerd severs the management plane — you lose `docker ps`,
`docker logs`, and `docker exec` — but the container process keeps running.

This is by design: it allows daemon upgrades without container downtime.

**To actually stop all containers:**

```bash
# Stop all containers gracefully first
docker stop $(docker ps -q)

# Then stop the daemon
systemctl stop docker
```

**To kill everything including shims:**

```bash
systemctl stop containerd   # kills shims and all containers
```

---

### 3. Container writes disappear after `docker rm`

**When it happens:** You write a file inside a running container. After stopping
and removing it, the file is gone.

**Root cause:** Container writes go to `UpperDir` in the overlay2 filesystem.
`docker rm` deletes `UpperDir`. The image layers in `LowerDir` are untouched —
they're shared across all containers using that image.

```bash
# Inspect where a container's writes live
docker inspect mycontainer --format '{{json .GraphDriver.Data}}' | python3 -m json.tool
# {
#   "UpperDir":  "/var/lib/docker/overlay2/<id>/diff",   ← deleted on docker rm
#   "LowerDir":  "/var/lib/docker/overlay2/<id>/diff:...",  ← image, never deleted
#   "MergedDir": "/var/lib/docker/overlay2/<id>/merged"  ← what container sees as /
# }
```

**Fix:** Mount a volume for anything that must persist:

```bash
docker run -v /host/path:/container/path myimage
docker run -v myvolume:/data myimage
```

**Trap:** `docker stop` + `docker start` (not `rm`) preserves `UpperDir` — the
container's writes survive. It's `rm` that deletes them.

---

### 4. Binary in PATH returns "not found" — permissions, not a missing binary

**When it happens:** Container starts but a binary that clearly exists throws
"not found" or "permission denied" when executed.

```bash
docker exec mycontainer /usr/bin/runc version
# bash: /usr/bin/runc: Permission denied
# or: exec /usr/bin/runc: no such file or directory  (misleading error)
```

**Diagnose:**

```bash
docker exec mycontainer ls -la /usr/bin/runc
# ---x--x--x  ← execute bit set, but no read bit
```

**Root cause:** The binary exists but has no read permission. The OS loader
needs the read bit to map the binary into memory. Without it, the exec syscall
fails with `EACCES`, which the shell reports as "not found" on some systems.

**Fix:** Set the correct permissions — don't reinstall:

```bash
chmod 755 /usr/bin/runc
```

---

### 5. `docker exec` fails — container is not running

**When it happens:** You try to enter a container to debug it. docker exec
returns an error.

```bash
docker exec -it mycontainer sh
# Error response from daemon: Container mycontainer is not running
```

**Root cause:** `docker exec` creates a new process inside an existing
running container's namespaces via `setns()`. If the container is stopped
or crashed, its namespaces are gone — there is nothing to exec into.

**Fix:** Check what happened before the container exited:

```bash
docker logs mycontainer              # stdout/stderr from the last run
docker logs --tail 50 mycontainer
docker inspect mycontainer --format '{{.State.ExitCode}}'
```

If you need to explore the filesystem of a crashed container, use the stopped
container's overlay or start a new one with a shell override:

```bash
docker run --entrypoint sh -it myimage   # starts fresh container with sh
```

**Trap:** `docker run -it myimage sh` is NOT the same as exec into the running
container — it creates a new container from the image, with a clean filesystem.

---

## Decision: `docker exec` vs `docker run` for debugging

`docker exec` enters an existing running container. Use it when you need to
debug the live state: check environment variables, inspect open files, or run
a diagnostic inside the running process's namespace.

`docker run --entrypoint sh` starts a fresh container from the same image.
Use it when the container is crashed (can't exec) or when you want to inspect
the image's filesystem without side effects.

Never use `docker run` thinking you're entering the running container — you're
not. You're creating a sibling container from the same image with a clean slate.

---

## Quick Reference

```bash
# Architecture chain
# docker CLI → dockerd → containerd → containerd-shim → runc → kernel

# Run
docker run -it alpine sh
docker run -d -e VAR=value --restart always myimage
docker run --entrypoint sh myimage             # override entrypoint for debug

# Exec into running container
docker exec -it mycontainer sh
docker exec mycontainer env                    # check env vars
docker exec mycontainer cat /etc/hosts

# Inspect overlay2 filesystem
docker inspect <id> --format '{{json .GraphDriver.Data}}' | python3 -m json.tool

# Container PID on host
docker inspect <id> --format '{{.State.Pid}}'

# Enter container namespace from host (without docker exec)
nsenter --target <pid> --mount --uts --ipc --net --pid

# Lifecycle
docker stop <id>         # SIGTERM → wait 10s → SIGKILL
docker kill <id>         # SIGKILL immediately
docker rm -f <id>        # force stop + remove
docker update --restart on-failure --cpus 0.5 <id>

# Logs
docker logs <id>
docker logs --follow --tail 50 <id>

# K8s node tools
crictl ps
crictl logs <container-id>
crictl inspect <container-id>
ctr -n k8s.io containers ls

# Diagnose daemon
journalctl -xeu docker.service
journalctl -u containerd
```

---

## K8s Connection

```
kubelet → containerd (CRI) → containerd-shim → runc → kernel
```

dockerd is not in the Kubernetes chain. The overlay2 filesystem, OCI layers,
and container lifecycle (pause, kill, signals) are identical — K8s just
orchestrates them without the Docker management layer.

| Docker | K8s equivalent |
|--------|---------------|
| `docker exec` | `kubectl exec` (same `nsenter`/`setns` under the hood) |
| `docker logs` | `kubectl logs` (reads same stdout/stderr pipe) |
| `docker inspect` | `kubectl describe pod` + `crictl inspect` |
| `docker ps` | `crictl ps` on the node |
| `containerd-shim` | same binary, started by kubelet's containerd CRI |
