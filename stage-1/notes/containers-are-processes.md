| Base image | Why it exists | Typical size |
|---|---|---|
| `FROM scratch` | Static binary, zero runtime deps | ~2 MB |
| `alpine:3` | Minimal musl libc + shell | ~7 MB |
| `python:3.11-slim` | Python + glibc + pip, no extras | ~130 MB |
| `python:3.11` | Full Debian + Python + build tools | ~1 GB |