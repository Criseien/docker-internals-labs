# docker-internals-labs

Hands-on labs covering Docker internals and the Linux kernel primitives underneath — cgroups, network namespaces, overlay filesystems, and the OCI runtime. Each lab starts from a real failure scenario, traces the problem to the kernel level, and connects explicitly to how Kubernetes uses the same mechanism.

This is part of a structured path toward K8s Networking & Troubleshooting specialization. The approach is bottom-up: understand what happens at the kernel before abstracting it with Kubernetes.

---

## Why this matters for Kubernetes

Most engineers know how to run `kubectl apply`. Fewer know why a pod can't reach the internet when `ip_forward` is 0, what `CreateContainerError` means at the runc level, or why an OOM-killed container maps directly to a cgroup `memory.events` entry.

These labs build that mental model.

---

## Labs

### [cgroups](./cgroups/) — Control Groups v2

**Scenario:** A process is consuming too much CPU and memory on a shared node. No container runtime available — limit it manually using raw cgroup interfaces.

Covers: cgroups v2 unified hierarchy, enabling controllers, `cpu.max` quota/period format, `memory.max`, attaching processes, and `cgroup.freeze` to suspend without killing.

**K8s connection:** `resources.limits.cpu` and `resources.limits.memory` in the pod spec write directly to `cpu.max` and `memory.max` under `/sys/fs/cgroup/kubepods/`. When a container is OOM-killed in K8s, the kubelet reads the event from `memory.events`. `kubectl` pod suspension and `docker pause` both use the `cgroup.freeze` interface practiced here.

---

### [runc](./runc/) — Container from scratch

**Scenario:** No Docker, no containerd. Start nginx manually using only runc — prepare the OCI bundle, configure a veth pair, and verify access from the host.

Covers: the `runc create` → wire networking → `runc start` two-step model, generating `config.json` with `runc spec`, extracting a rootfs via `docker export`, the `/run/netns/` symlink trick so `ip netns` can see the container namespace, and common failure modes.

**K8s connection:** containerd uses exactly this two-step model for every pod. CNI plugins run between `create` and `start` — after namespaces exist but before the process starts. `CreateContainerError` means the CNI plugin failed in that window and `runc start` was never called.

---

### [docker-internals](./docker-internals/) — Architecture and container lifecycle

**Scenario:** `docker ps` returns nothing with containerd running. A container disappears after `systemctl stop docker`. Binary in PATH returns "not found". Container writes disappear on `docker rm`.

Covers: the full `docker CLI → dockerd → containerd → containerd-shim → runc → kernel` chain, why `systemctl stop docker` does not kill running containers (containerd-shim keeps them alive), overlay2 filesystem (LowerDir/UpperDir/MergedDir), `docker exec` vs `docker run`, OCI image layout, `docker export`, and `ctr` for talking to containerd directly.

**K8s connection:** On a K8s node the chain is `kubelet → containerd (CRI) → runc`. `docker` is not in it — `crictl` is the right tool. `kubectl exec` enters the container's namespace the same way `nsenter` does via `/proc/<pid>/ns/net`. `kubectl logs` reads from the same place `docker logs` does.

---

### [docker-networking](./docker-networking/) — Network namespaces and network modes

**Scenario:** nginx unreachable from the host (port not published). Monitoring container can't see host connections (wrong network mode). Security-sensitive workload needs zero network access.

Covers: bridge, host, and none modes at the network namespace level, veth pairs, `docker0` bridge, iptables MASQUERADE for container-to-internet traffic, and the four most common networking failure modes.

**K8s connection:** Every Kubernetes pod is a network namespace — exactly what Docker creates in bridge mode, but managed by the CNI plugin instead of Docker. The veth pair, the bridge, and the MASQUERADE rule are all still there. `hostNetwork: true` in a pod spec is host mode — same tradeoff: full host visibility, no network isolation.

---

## Prerequisites

- Linux with cgroups v2 (AlmaLinux 9, RHEL 9, Ubuntu 22.04+)
- Docker installed (`docker`, `containerd`, `runc`)
- `iproute2`, `iptables`, `nsenter`
- For the runc lab: Docker available to export the rootfs

## Recommended order

`cgroups` → `runc` → `docker-internals` → `docker-networking`

The first two are the raw kernel primitives. The second two are how Docker orchestrates them. That order makes everything click.
