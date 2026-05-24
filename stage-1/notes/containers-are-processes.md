https://iximiuz.com/en/posts/not-every-container-has-an-operating-system-inside/

Core Idea: "A container is a process that has been given an isolated view of the filesystem, network, and process table via Linux namespaces, and a resource budget via cgroups. The "OS" inside is just libraries your process needs at runtime."

| Base image | Why it exists | Typical size |
|---|---|---|
| `FROM scratch` | Static binary, zero runtime deps | ~2 MB |
| `alpine:3` | Minimal musl libc + shell | ~7 MB |
| `python:3.11-slim` | Python + glibc + pip, no extras | ~130 MB |
| `python:3.11` | Full Debian + Python + build tools | ~1 GB |