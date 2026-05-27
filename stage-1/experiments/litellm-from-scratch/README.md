# litellm-from-scratch

A Dockerfile for the LiteLLM proxy written from scratch as a Stage 1 learning exercise. Not using the official `ghcr.io/berriai/litellm` image — the point is to understand every layer.

## Concepts demonstrated

- **Multi-stage build** — `builder` stage compiles deps with full toolchain; `runtime` stage only gets the installed packages. Keeps the production image small and free of compilers.
- **Layer caching** — `COPY requirements.txt` before `COPY . .` so pip only re-runs when dependencies actually change.
- **Non-root user** — process runs as uid 1001 (`litellm`), not root.
- **Exec form CMD** — `CMD ["litellm", ...]` vs shell form `CMD litellm ...`: exec form means the process is PID 1 and receives signals directly. SIGTERM on `docker stop` reaches LiteLLM instead of being swallowed by `/bin/sh`.
- **`.dockerignore`** — keeps `.env` files and `__pycache__` out of the build context.
- **Runtime secrets** — API keys passed via `-e` at `docker run`, never baked into the image.

## Build and run

```bash
# Build
docker build -t litellm-scratch:latest .

# Run (pass API keys at runtime, not in the image)
docker run -d \
  -p 4000:4000 \
  -e OPENAI_API_KEY=sk-... \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -e LITELLM_MASTER_KEY=sk-master-... \
  --name litellm \
  litellm-scratch:latest

# Check it's running
curl http://localhost:4000/health

# Tail logs
docker logs -f litellm
```

## Compare to the official image

```bash
# What you'd normally do (pulls a pre-built image you didn't write)
docker pull ghcr.io/berriai/litellm:main-latest

# What this exercise teaches you is what's inside that image
docker image history ghcr.io/berriai/litellm:main-latest
```
