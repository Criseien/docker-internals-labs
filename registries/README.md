# Registries — Pushing, pulling, and tagging: what breaks in image distribution

A registry is an HTTP server that stores OCI image manifests and layer blobs.
`docker push` uploads layers. `docker pull` downloads them. Tags are mutable
pointers to manifests — the same tag can point to a different image tomorrow.
Digests (`sha256:...`) are immutable. Most production incidents involving
"wrong image deployed" trace back to mutable tags used where a digest was needed.

---

## The 5 Problems You'll Actually Hit

### 1. `docker push` denied — not logged in or wrong image name format

**When it happens:** You build an image and push it. Authentication error.

```bash
docker push myapp:latest
# denied: requested access to the resource is denied
# or: unauthorized: authentication required
```

**Root cause (most common):** The image tag doesn't include the registry
hostname. Docker defaults to `docker.io` for bare names. If your target is
a private registry, the tag must include the full registry hostname.

**Fix:**

```bash
# Log in first
docker login registry.example.com
# or Docker Hub
docker login

# Tag with full registry path before push
docker tag myapp:latest registry.example.com/myorg/myapp:latest
docker push registry.example.com/myorg/myapp:latest
```

**Trap:** `docker tag` does not move the original — it creates an alias.
Both tags now point to the same image ID. You push the new tag, not the old one.

For GitHub Container Registry (ghcr.io):

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin
docker tag myapp:latest ghcr.io/<org>/myapp:latest
docker push ghcr.io/<org>/myapp:latest
```

---

### 2. `docker pull` rate limited — Docker Hub anonymous limit

**When it happens:** CI pipeline or `docker pull` on a fresh node fails with
a 429 error after pulling several images.

```bash
docker pull nginx:latest
# toomanyrequests: You have reached your pull rate limit.
# You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```

**Root cause:** Docker Hub limits unauthenticated pulls to 100/6h per IP.
Authenticated free accounts get 200/6h. CI runners often share IPs — the
limit is per IP, not per account.

**Fix (short term):** Log in to Docker Hub even with a free account:

```bash
docker login -u <username>    # doubles the limit to 200/6h
```

**Fix (long term):** Mirror images to a private registry or use a pull-through
cache. In K8s, configure `imagePullSecrets` with registry credentials:

```yaml
# In pod spec
imagePullSecrets:
  - name: docker-hub-creds
```

**Trap:** The rate limit applies to the number of manifest requests (each
`docker pull` counts), not the amount of data transferred. Pulling the same
image twice from the same IP counts twice.

---

### 3. Image pulled but it's the wrong version — mutable tag problem

**When it happens:** You deploy `myapp:latest` to staging and production.
Staging passes. Production gets a different image.

**Root cause:** Tags are mutable. `latest` (or any tag) is just a pointer in
the registry. If someone pushed a new image with the same tag between your
staging and production deploys, production gets a different layer set.

**Diagnose:**

```bash
# Check what digest your running container actually has
docker inspect <container> --format '{{.Image}}'
# sha256:abc123...   ← full digest of what's actually running

# Compare with what the registry says the tag points to
docker pull myapp:latest
docker inspect myapp:latest --format '{{.Id}}'
# sha256:def456...   ← different digest — tag moved
```

**Fix — use digests in production:**

```bash
# Get the digest of the image you tested
docker inspect myapp:latest --format '{{index .RepoDigests 0}}'
# myapp@sha256:abc123def456...

# Use the digest in production deployment
docker run myapp@sha256:abc123def456...
```

In K8s:

```yaml
image: myapp@sha256:abc123def456...   # immutable, guaranteed same image
```

**Trap:** Digest references bypass the tag entirely. Even if someone pushes a
new `latest`, a digest reference always pulls the exact layers it pointed to.

---

### 4. Private registry TLS error — certificate not trusted

**When it happens:** You set up a private registry (Harbor, Nexus, self-signed
cert). `docker pull` or `docker push` fails with a TLS error.

```bash
docker pull registry.internal:5000/myapp:latest
# x509: certificate signed by unknown authority
# or: http: server gave HTTP response to HTTPS client
```

**Root cause (self-signed cert):** Docker requires a trusted CA for HTTPS.
Self-signed certs aren't trusted by default.

**Fix — add the CA certificate:**

```bash
# Copy the registry's CA cert
mkdir -p /etc/docker/certs.d/registry.internal:5000
cp ca.crt /etc/docker/certs.d/registry.internal:5000/ca.crt
# No daemon restart needed — Docker reads this directory at request time
```

**Root cause (HTTP registry):** Registry runs HTTP, not HTTPS. Docker refuses
plain HTTP by default.

**Fix — allow insecure registry (non-production only):**

```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["registry.internal:5000"]
}
```

```bash
systemctl reload docker
```

**Trap:** The `insecure-registries` setting disables TLS verification for that
registry. Never use it for a registry that holds production images — the
connection can be intercepted.

---

### 5. Tag pushed but `docker pull` returns "not found" — namespace or tag mismatch

**When it happens:** You pushed an image, checked the registry UI, it's there.
Another machine or CI runner can't pull it.

```bash
docker pull registry.example.com/myorg/myapp:v1.2.3
# Error response from daemon: manifest for registry.example.com/myorg/myapp:v1.2.3 not found
```

**Diagnose:**

```bash
# List what tags actually exist
curl -s https://registry.example.com/v2/myorg/myapp/tags/list | python3 -m json.tool
# {
#   "name": "myorg/myapp",
#   "tags": ["v1.2.2", "latest"]   ← v1.2.3 is not there
# }
```

**Root cause (most common):** The push used a different tag string. Check push
history:

```bash
docker images myapp --format "{{.Tag}}"
# v1.2.3-rc1    ← pushed the rc tag, not the release tag
```

**Fix:**

```bash
# Retag and push the correct tag
docker tag myapp:v1.2.3-rc1 registry.example.com/myorg/myapp:v1.2.3
docker push registry.example.com/myorg/myapp:v1.2.3
```

---

## Decision: mutable tags vs digests; Docker Hub vs private registry

Use **mutable tags** (`latest`, `v1`, branch names) for human-readable
references in development and for CI pipelines that always pull fresh.

Use **digest references** in production deployments where you need to guarantee
the exact image is used — `image@sha256:...` in your pod spec. GitOps tools
(ArgoCD, Flux) with image updater can automate this: they detect a new tag,
resolve its digest, and commit the digest to the repo.

Use **Docker Hub** for public base images (`nginx:alpine`, `python:3.11-slim`).
Use a **private registry** (ECR, GCR, ghcr.io, Harbor) for anything you build.
Never push images with credentials, private data, or internal configs to Docker Hub.

The rule for production: tag for humans, deploy by digest.

---

## Quick Reference

```bash
# Login
docker login                                    # Docker Hub
docker login registry.example.com              # private registry
echo $TOKEN | docker login ghcr.io -u USER --password-stdin

# Tag and push
docker tag myapp:latest registry.example.com/org/myapp:v1.2.3
docker push registry.example.com/org/myapp:v1.2.3

# Pull
docker pull registry.example.com/org/myapp:v1.2.3
docker pull myapp@sha256:abc123...              # by digest (immutable)

# Inspect what's running
docker inspect <container> --format '{{.Image}}'          # digest of running image
docker inspect <image> --format '{{index .RepoDigests 0}}' # digest for a local tag

# Registry API (v2)
curl -s https://registry.example.com/v2/<name>/tags/list  # list tags
curl -s https://registry.example.com/v2/<name>/manifests/<tag>  # get manifest

# TLS fix
mkdir -p /etc/docker/certs.d/<registry>:<port>
cp ca.crt /etc/docker/certs.d/<registry>:<port>/ca.crt

# Insecure registry (dev only)
# /etc/docker/daemon.json → "insecure-registries": ["<host>:<port>"]
systemctl reload docker
```

---

## K8s Connection

```yaml
# Pull from private registry in K8s
spec:
  imagePullSecrets:
    - name: my-registry-secret       # kubectl create secret docker-registry ...
  containers:
    - name: app
      image: registry.example.com/org/myapp@sha256:abc123...   # digest = immutable
```

On EKS or GKE, the node IAM role grants pull access from ECR/GCR automatically
— no `imagePullSecrets` needed for images in the same cloud account.

Image pull failures appear in pod events:

```bash
kubectl describe pod <pod-name>
# Events:
#   Warning  Failed   ... Failed to pull image: ... unauthorized
#   Warning  Failed   ... ErrImagePull / ImagePullBackOff
```

`ImagePullBackOff` means the kubelet already retried and is backing off
exponentially. Fix the credentials or tag, then `kubectl delete pod` to
force a fresh attempt — the backoff timer doesn't reset otherwise.
