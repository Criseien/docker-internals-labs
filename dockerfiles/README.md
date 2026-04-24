# Dockerfiles — Production image builds: what causes bloated, broken, and insecure images

A Dockerfile is a layer recipe. Each `RUN`, `COPY`, and `ADD` instruction creates
an immutable layer. The order of instructions determines cache behavior, image
size, and what ends up in production. Most image problems — size, broken builds,
dev deps leaking to prod — come from three root causes: wrong layer order,
single-stage when multi-stage is needed, or wrong base image choice.

---

## The 5 Problems You'll Actually Hit

### 1. Build cache invalidated on every run — slow builds

**When it happens:** Every `docker build` reruns `apt-get install` or
`pip install` even when dependencies haven't changed. Build takes 3 minutes
instead of 10 seconds.

**Root cause:** Once a layer is invalidated, all subsequent layers are rebuilt.
`COPY . .` before `RUN pip install` means any source file change — including
a comment edit — invalidates the install layer.

**Wrong order (cache-busting):**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .                         # ← any file change busts the cache here
RUN pip install -r requirements.txt   # ← always reruns
```

**Fix — dependencies first, source after:**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .          # ← only busted when requirements.txt changes
RUN pip install -r requirements.txt
COPY . .                         # ← source goes last
```

Same pattern for Node.js:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .
```

**Trap:** `npm install` vs `npm ci`. `npm ci` is deterministic (uses lockfile
exactly), fails if lockfile is missing, and is correct for CI/CD. `npm install`
can silently update deps and is wrong in a Dockerfile.

---

### 2. Dev dependencies in production image — bloated and insecure

**When it happens:** Your Python or Node image is 800MB when it should be
under 200MB. Security scans find dev tool CVEs in production.

**Diagnose:**

```bash
docker images myapp
# REPOSITORY   TAG    IMAGE ID       SIZE
# myapp        latest abc123         812MB   ← suspicious

docker run --rm myapp pip list | grep pytest
# pytest 7.4.0   ← test framework in production image
```

**Root cause:** Installing all dependencies (including dev/test) in a single
stage with no cleanup step.

**Fix — multi-stage build:**

```dockerfile
# Stage 1: build with all deps
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements*.txt ./
RUN pip install --target /install -r requirements.txt   # prod only

# Stage 2: production image — only what runs
FROM python:3.11-slim
COPY --from=builder /install /usr/local/lib/python3.11/site-packages/
COPY . .
CMD ["python", "app.py"]
```

For Node.js, use `--omit=dev` to strip dev deps from the final stage:

```dockerfile
FROM node:20-slim AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev     # installs only production deps

FROM node:20-slim
COPY --from=deps /app/node_modules ./node_modules
COPY . .
CMD ["node", "src/index.js"]
```

---

### 3. System library missing at runtime — container crashes on start

**When it happens:** Build succeeds. Container starts and immediately exits.
`docker logs` shows a library error.

```bash
docker logs mycontainer
# ImportError: libmagic.so.1: cannot open shared object file: No such file or directory
# or: error while loading shared libraries: libssl.so.1.1
```

**Root cause:** Python and Node packages with C extensions require OS-level
shared libraries. `pip install python-magic` installs the Python wrapper but
not `libmagic1` — that's an OS package, not a pip package.

**Fix — install OS deps before pip/npm:**

```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y \
    libmagic1 \
    && rm -rf /var/lib/apt/lists/*    # ← always clean apt cache in same layer
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

**Trap:** `apt-get update` and `apt-get install` must be in the same `RUN`
instruction. If they're split into two `RUN` instructions, the `update` layer
gets cached while the `install` layer doesn't — you get stale package lists and
broken installs.

```dockerfile
# Wrong
RUN apt-get update
RUN apt-get install -y libmagic1

# Correct
RUN apt-get update && apt-get install -y libmagic1 && rm -rf /var/lib/apt/lists/*
```

---

### 4. Multi-stage build copies nothing — wrong path or wrong stage name

**When it happens:** Multi-stage build completes but the final image is empty
or missing the built binary.

```bash
docker run --rm myapp ls /app
# (empty or binary not found)
```

**Root cause:** Either the `COPY --from` references the wrong stage name,
or the path in the builder stage doesn't match the path in the `COPY` command.

**Fix — verify paths in each stage:**

```dockerfile
# Go example — common pattern
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o /out/server .   # binary written to /out/server

FROM debian:bookworm-slim
COPY --from=builder /out/server /usr/local/bin/server   # ← path must match
CMD ["/usr/local/bin/server"]
```

**Diagnose — run the builder stage interactively:**

```bash
# Build only up to the builder stage
docker build --target builder -t myapp-builder .

# Explore what's actually there
docker run --rm -it myapp-builder sh
ls /out/   # verify the binary exists at the expected path
```

**Trap:** `COPY --from=0` uses stage index. `COPY --from=builder` uses the
name from `AS builder`. The name is more readable and won't break if you add
a stage before it.

---

### 5. TypeScript or compiled app: source files in production image

**When it happens:** Node TypeScript app image contains `.ts` source files
and `node_modules` with dev deps. The `dist/` output is there but so is
everything else.

**Root cause:** No multi-stage build. `tsc` compiles to `dist/` in the same
stage where all source and dev tools are present, and everything gets included.

**Fix — compile in builder, ship only `dist/`:**

```dockerfile
# Stage 1: compile TypeScript
FROM node:20 AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src/ ./src/
RUN npx tsc                    # compiles src/*.ts → dist/*.js

# Stage 2: production — only compiled output + prod deps
FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --omit=dev          # only production dependencies
CMD ["node", "dist/index.js"]
```

Final image contains: `dist/`, `node_modules/` (prod only). No `.ts` files,
no `typescript` package, no `tsconfig.json`.

---

## Decision: single-stage vs multi-stage; alpine vs debian-slim

Use **single-stage** only for interpreted languages with no compilation step and
no dev/prod dependency split (simple Python scripts, shell tools). If there's a
build step or dev deps, use multi-stage.

Use **multi-stage** when: you compile (Go, TypeScript, Java), when dev tools must
not reach production, or when you need to run tests in the build pipeline.

For the **base image**:

- `debian-slim` — predictable, well-supported, easy to add packages with `apt-get`. Start here.
- `alpine` — smaller, but uses `musl` libc. C extensions (most Python/Node native modules) break silently or require rebuilding. Only use if you know your deps are `musl`-compatible.
- `distroless` — no shell, no package manager, minimal attack surface. Use in production for Go or Java. Hard to debug — you lose `exec` access.

The rule: optimize for correctness first, size second. A working 200MB image
beats a broken 20MB one. Go distroless after the app is stable.

---

## Quick Reference

```dockerfile
# Layer order for cache efficiency
COPY <lockfile> .
RUN <install>          # cached until lockfile changes
COPY . .               # source always last

# apt-get pattern (always in one RUN)
RUN apt-get update && apt-get install -y <pkg> && rm -rf /var/lib/apt/lists/*

# npm patterns
RUN npm ci                       # CI/CD — deterministic, uses lockfile
RUN npm ci --omit=dev            # production deps only
RUN npm install                  # never in Dockerfile (non-deterministic)

# Multi-stage reference
FROM <image> AS builder          # name the stage
COPY --from=builder /src /dst    # reference by name
docker build --target builder .  # build up to a specific stage

# Debug a stage
docker build --target builder -t debug-img .
docker run --rm -it debug-img sh
```

```bash
# Inspect layers and size
docker history myimage
docker images myimage

# Check what's in the final image
docker run --rm myimage find / -name "*.ts" 2>/dev/null   # should be empty
docker run --rm myimage pip list                           # verify no dev deps
```

---

## K8s Connection

Image size directly affects pod startup time — the kubelet pulls the image on
every node that schedules the pod. A 1GB image pulled cold on 10 nodes before
a deploy = 10 minutes of startup latency.

`imagePullPolicy: Always` means the kubelet checks the registry digest on every
pod start. `IfNotPresent` uses the local cache if the tag exists. In production,
pin images to a digest (`image: myapp@sha256:abc...`) to guarantee the same
layer is used regardless of tag mutation.

Multi-stage builds and `distroless` also reduce the CVE surface area that
security scanners (Trivy, Grype) will flag in your pipeline.
