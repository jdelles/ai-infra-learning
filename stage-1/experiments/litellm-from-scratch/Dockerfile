# LiteLLM proxy — written from scratch as a learning exercise
# Demonstrates: multi-stage build, non-root user, .dockerignore, layer caching

# ── Stage 1: builder ────────────────────────────────────────────────────────
# Use a full Python image to compile dependencies with native extensions.
# We'll copy only the installed packages into the final stage — the compiler
# toolchain stays behind and never ships in the production image.
FROM python:3.11-slim AS builder

# Install build dependencies (needed to compile some Python C extensions)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Copy requirements first — this layer is cached as long as requirements.txt
# doesn't change, even if your config files do. This is the single most
# important caching trick in Dockerfile writing.
COPY requirements.txt .

# Install into a prefix directory we can copy wholesale into the final image
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ── Stage 2: runtime ─────────────────────────────────────────────────────────
# Minimal Python image — no compiler, no apt cache, no build tools.
# Only the installed packages (from /install) come along.
FROM python:3.11-slim AS runtime

# Copy installed packages from builder stage
COPY --from=builder /install /usr/local

# Create a non-root user to run the process.
# Containers run as root by default — a vulnerability if the process is
# compromised, because root inside the container maps to root on the host
# (unless user namespaces are configured). Non-root is the correct default.
RUN groupadd --gid 1001 litellm && \
    useradd --uid 1001 --gid litellm --shell /bin/sh --create-home litellm

WORKDIR /app

# Copy the LiteLLM config file
COPY config.yaml .

# Transfer ownership so our non-root user can read the config
RUN chown -R litellm:litellm /app

# Switch to non-root user for all subsequent instructions and at runtime
USER litellm

# Document the port — this doesn't actually publish it.
# You still need -p 4000:4000 at docker run, or ports: in Compose.
EXPOSE 4000

# Start the LiteLLM proxy pointing at our config file.
# Using exec form (JSON array) so the process receives signals directly —
# shell form wraps in /bin/sh -c, which swallows SIGTERM on docker stop.
CMD ["litellm", "--config", "/app/config.yaml", "--port", "4000"]
