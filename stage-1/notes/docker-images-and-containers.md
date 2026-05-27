# Docker Images and Containers
_Source: Docker Deep Dive — Nigel Poulton, ch 5–6 | Date: 2026-05-27_

---

## Images

### What an image actually is

An image is a read-only, layered filesystem — not a single flat file. Each layer is a content-addressed diff on top of the one below it. When you pull `node:22-alpine`, you're pulling a stack of tarballs (layers) plus a JSON manifest that describes their order and metadata.

The layers are stored on disk in `/var/lib/docker/overlay2/` and shared across images. If two images share a base layer (e.g., both start `FROM alpine`), Docker stores that layer **once**. This is why layer order in your Dockerfile matters for build cache and disk usage.

### The union filesystem: overlay2

Docker uses the **overlay2** storage driver to merge layers into a single coherent filesystem view. The mechanism:

- **lowerdir** — the read-only image layers, stacked
- **upperdir** — a writable layer created when a container starts (the container layer)
- **merged** — the unified view the process sees

When a container writes a file that exists in a lower layer, overlay2 copies it up to the upperdir first (copy-on-write). The original layer is untouched. This is why image layers stay immutable.

```
Container layer (upperdir) — writable, ephemeral
────────────────────────────────────────────────
Image layer 3: RUN npm install          ← read-only
Image layer 2: COPY . /app              ← read-only
Image layer 1: FROM node:22-alpine      ← read-only
```

### Image digests vs. tags

| Concept | What it is | Mutable? |
|---------|-----------|----------|
| **Tag** | A human-readable pointer (e.g., `node:22-alpine`) | Yes — can be moved to a new image |
| **Digest** | SHA256 of the image manifest (e.g., `sha256:abc123...`) | No — cryptographically tied to exact content |

Tags are convenient but unreliable for reproducibility. `node:22-alpine` today might not be the same bytes as `node:22-alpine` next month. In production (and FedRAMP), you pin by digest:

```dockerfile
FROM node:22-alpine@sha256:abc123...
```

### Useful inspection commands

```bash
# Show layer history — what each instruction added and how much size
docker image history node:22-alpine

# Full JSON manifest — architecture, config, layer digests
docker image inspect node:22-alpine

# Show digest for a pulled image
docker image ls --digests

# Pull by digest (immutable)
docker pull node:22-alpine@sha256:<digest>
```

`docker image history` is the first thing to run when debugging a bloated image. You'll often find a `RUN apt-get install` layer that's 400MB and realize the build didn't clean up the apt cache.

### Multi-arch images

A single tag like `python:3.11-slim` resolves to different image layers depending on your CPU architecture (amd64, arm64, etc.). The tag actually points to a **manifest list** (or image index), which maps each platform to a platform-specific manifest. Docker pulls the right one for your machine automatically.

This matters when you build on an M-series Mac (arm64) and deploy to a GCP node (amd64) — you need to build for the target platform:

```bash
docker buildx build --platform linux/amd64 -t myimage:latest .
```

---

## Containers

### What a container actually is

A container is a running process with:
- An isolated view of the filesystem (via mount namespace — overlay2 merged view)
- Its own network stack (via net namespace)
- Its own process tree (via pid namespace — PID 1 inside the container)
- A writable layer on top of the image (the upperdir)
- Resource limits (via cgroups)

The writable layer is **ephemeral** — it disappears when `docker rm` is called. Anything you want to persist must be in a volume or bind mount.

### Container lifecycle

```
docker run   → creates + starts (pull image if needed, create container layer, start PID 1)
docker stop  → sends SIGTERM to PID 1, waits grace period (10s default), then SIGKILL
docker start → restarts a stopped container (container layer preserved)
docker rm    → deletes container + its writable layer
docker rm -f → force-stop + delete in one step
```

Key distinction: **stopped ≠ deleted**. `docker stop` preserves the container (and its writable layer). `docker rm` actually destroys it.

### Essential runtime commands

```bash
# Run detached, port-mapped, named
docker run -d -p 8080:8080 --name myapp myimage:latest

# Exec into a running container (new process, not PID 1)
docker exec -it myapp /bin/sh

# Follow logs in real time
docker logs -f myapp

# Inspect runtime state (IP, mounts, env vars, restart policy)
docker inspect myapp

# See all containers including stopped ones
docker ps -a

# Copy a file out of a container without exec-ing in
docker cp myapp:/app/config.json ./config.json
```

### Restart policies

Set on `docker run` with `--restart`:

| Policy | Behavior |
|--------|----------|
| `no` (default) | Never restart |
| `always` | Restart always, including on daemon start |
| `unless-stopped` | Restart always, except if manually stopped |
| `on-failure[:n]` | Restart only on non-zero exit; optional max retries |

`unless-stopped` is the right default for long-running services. `always` will restart even containers you deliberately stopped — usually not what you want.

### Container ≠ process (but close)

The container is the Docker-managed metadata wrapper around the process: config, network settings, volume mounts, restart policy, the overlay2 layer. The **process** is what actually runs. When PID 1 inside the container exits, the container stops. If PID 1 is a shell and you `exec` into it, you're starting a second process in the same namespace — `exec` doesn't replace PID 1.

This matters for signal handling: `docker stop` sends SIGTERM to PID 1. If your Dockerfile has `CMD ["npm", "start"]` and npm doesn't forward signals to the Node process, your app never gets the graceful shutdown signal. Fix: use `CMD ["node", "server.js"]` directly, or use `tini` as PID 1.

---

## Key Dockerfile Instructions (ch 5 reference)

| Instruction | What it does |
|-------------|-------------|
| `FROM` | Sets the base image / starting layer |
| `RUN` | Executes a command during build; creates a new layer |
| `COPY` | Copies files from build context into the image |
| `ADD` | Like COPY but also handles URLs and tar extraction (prefer COPY) |
| `WORKDIR` | Sets the working directory for subsequent instructions |
| `ENV` | Sets environment variables baked into the image |
| `ARG` | Build-time variable (not available at runtime) |
| `EXPOSE` | Documents which port the container listens on (doesn't actually open it) |
| `CMD` | Default command when container starts; overridable at `docker run` |
| `ENTRYPOINT` | Fixed command; CMD becomes its arguments; harder to override |

**ENTRYPOINT vs CMD pattern:**
```dockerfile
ENTRYPOINT ["node"]   # fixed: always run node
CMD ["server.js"]     # default arg: run server.js unless overridden
```
Running `docker run myimage other.js` would execute `node other.js`.

---

## The .dockerignore file

Like `.gitignore` but for the Docker build context. Everything you don't `.dockerignore` gets sent to the daemon on every `docker build`. If you have model weights or large data files in your project directory, you *will* accidentally send gigabytes to the daemon without this.

```
node_modules/
.git/
*.log
.env
weights/
```
