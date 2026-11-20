# Docker Multi-Stage Builds — Smaller, Safer Images

> Daily note, started 2026-11-20. This file lives in
> `00 - Docker/` alongside `docker-command-cheatsheet.md` and will
> be extended over the next few days — one topic, one file.
> Next: a multi-stage build for the FastAPI Medical AI backend.

## 2026-11-20 — Part 1: Why multi-stage, and the pattern to copy

### The problem with a single-stage image

A typical Python image installs a compiler, build tools, and dev
dependencies, then ships all of them to production. The result is
a large image, a slow pull, and a bigger attack surface — the
build tools are still there for anyone who gets into the container.

### The multi-stage pattern

Build in one stage, run in another. Only the runtime artefacts
move across.

```dockerfile
# ---- Stage 1: build ----
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir --target /app/deps -r requirements.txt
# (or: uv sync --frozen --no-dev)

# ---- Stage 2: runtime ----
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY . .
ENV PYTHONPATH=/app/deps PATH=/app/deps/bin:$PATH
# Run as a non-root user — do not run as root in production
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### What to check after the build

```bash
docker build -t my-api:slim .
docker images my-api:slim          # compare size with the single-stage image
docker run --rm my-api:slim python -c "import sys; print(sys.version)"
docker history my-api:slim         # which layers are biggest?
```

A realistic target for a FastAPI image is well under 300 MB;
a single-stage image with build tools is often 800 MB+.

### Rules that save you later

- **Pin the base image** — `python:3.12-slim`, not `python:latest`.
  A rebuild next month should not silently change the runtime.
- **`.dockerignore` matters** — exclude `.git`, `__pycache__`,
  `.venv`, `*.db`, `.uploads/`, test files. Otherwise they are
  copied into the image and invalidate the cache.
- **One process, one container** — the API in one container, the
  worker / background job in another. Do not run a database in
  the same container as the app in production.
- **No secrets in the image** — API keys come from the environment
  (or Azure Key Vault / AWS Secrets Manager), never from a
  `COPY .env` line or a baked-in `ENV` value.

### What I will add next in this file

- 2026-11-21: the actual multi-stage Dockerfile for the Medical
  AI FastAPI backend, and the measured size before / after.
- 2026-11-22: Docker Compose for local dev — API + a local
  vector-store stand-in, with healthchecks.
- Later: the same image deployed to Azure Container Apps vs
  AWS ECS Fargate — what changes, what stays the same.
