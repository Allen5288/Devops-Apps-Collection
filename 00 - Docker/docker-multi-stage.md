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

- 2026-11-26: Docker Compose for local dev — done, see Part 3 below.
- Next: the same image deployed to Azure Container Apps vs AWS
  ECS Fargate — what changes, what stays the same.

## 2026-11-23 — Part 2: a multi-stage Dockerfile for the Medical AI FastAPI backend

Part 1 set the pattern; today it lands on a real service — the
Medical AI Assistant backend (FastAPI + LangChain, Groq LLaMA 3,
Pinecone). That backend has no build step and no compiled
dependencies, which makes it a good test of whether multi-stage
still earns its keep: the win here is not compilation, it is
leaving the dev tooling and the package cache behind.

### The Dockerfile

```dockerfile
# ---- Stage 1: build ----
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir --target /app/deps -r requirements.txt

# ---- Stage 2: runtime ----
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY app/ ./app/
ENV PYTHONPATH=/app/deps PATH=/app/deps/bin:$PATH
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

What is deliberate here:

- **No `COPY . .`** — only `app/` is copied, so tests, eval
  scripts, and local PDFs never enter the image. The `.dockerignore`
  below is the second line of defence, not the first.
- **No dev dependencies in the image** — `requirements.txt` is
  the runtime list only; `requirements-dev.txt` (pytest, linters)
  is never copied.
- **The healthcheck hits a real endpoint** — `/health` must exist
  in the FastAPI app and return 200 without calling the LLM or the
  vector store, or a slow model call will mark a healthy container
  as dead. `start-period=10s` gives uvicorn time to boot.
- **No secrets** — the Groq key, Pinecone key, and Google key come
  from the environment at run time, never from the image.

### `.dockerignore` for this service

```text
.git
__pycache__
*.pyc
.venv
.env
*.db
.uploads/
eval/
tests/
README.md
```

### Measure it — the commands, and what to record

```bash
docker build -t medical-ai:slim .
docker images medical-ai:slim
docker history medical-ai:slim
docker run --rm -p 8000:8000 --env-file .env medical-ai:slim
curl http://localhost:8000/health
```

Record the numbers in the table below after the first build —
targets are from Part 1, not measurements:

| Image | Size | Notes |
|---|---|---|
| Single-stage (build tools + dev deps) | — measure | baseline to beat |
| Multi-stage (this Dockerfile) | — measure | target: well under 300 MB |


## 2026-11-26 — Part 3: Docker Compose for local dev — the whole stack in one command

Parts 1–2 made one image small and safe. But a developer does not
run one image — they run the API, the thing the API talks to, and
the seed data, all at once. Without Compose that is three terminals,
three commands, and a README section everyone skips. With Compose
it is `docker compose up`, and the stack matches what the
Dockerfile promises.

### The stack for the Medical AI backend

The API needs a vector store. In production that is Pinecone; on a
laptop it is a stub — a tiny HTTP service that accepts the same
search call and returns canned chunks, so the API code never knows
the difference. The point of the exercise is the wiring (depends_on,
healthchecks, env), not the stub.
```yaml
# docker-compose.yml — run from the repo root of the Medical AI backend
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile        # the multi-stage build from Part 2
    ports:
      - "8000:8000"
    environment:
      GROQ_API_KEY: ${GROQ_API_KEY:?set GROQ_API_KEY in .env}
      PINECONE_API_KEY: ${PINECONE_API_KEY:?set PINECONE_API_KEY in .env}
      VECTOR_STORE_URL: http://vector-store:9000
    env_file:
      - .env                        # never committed; see .gitignore
    depends_on:
      vector-store:
        condition: service_healthy  # do not start the API until the store answers
    healthcheck:
      test: ["CMD", "python", "-c",
             "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 5s
      start_period: 10s
    volumes:
      - ./app:/app/app:ro           # live-reload app code without rebuilding
    command: python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  vector-store:
    build:
      context: ./vector-store-stub
    ports:
      - "9000:9000"
    healthcheck:
      test: ["CMD", "python", "-c",
             "import urllib.request; urllib.request.urlopen('http://localhost:9000/health')"]
      interval: 10s
      timeout: 3s
      start_period: 5s
```

### What each line is doing, and why

- **`depends_on` with `condition: service_healthy`.** Plain
  `depends_on` only waits for the container to start, not for the
  service to answer — the API would boot, fail its first search,
  and the developer would blame the code. The healthcheck makes
  the ordering real. (Compose v2 syntax; the old `version:` key is
  gone — if a tutorial still shows it, the tutorial is out of date.)
- **`/health` must be cheap.** Same rule as the Dockerfile
  HEALTHCHECK in Part 2: the endpoint returns 200 without calling
  the LLM or the vector store, or a slow model call marks a healthy
  container dead and Compose restarts it in a loop.
- **Secrets via `${VAR:?message}` and `env_file`.** The `:?`
  makes Compose fail fast with a readable message instead of
  starting the API with an empty key and failing on the first
  question. `.env` is git-ignored; the committed file is
  `.env.example` with empty values.
- **The `:ro` volume mount.** App code is mounted read-only and
  `--reload` picks up changes, so the edit-run loop is seconds, not
  a rebuild. The mount is read-only so a bug in the container
  cannot rewrite the developer's source. Production never mounts
  code — the image from Part 2 is self-contained.
- **One network, service names as hostnames.** `api` reaches the
  stub at `http://vector-store:9000` — Compose creates the network
  and the DNS for it. No `localhost` ports in app config, no
  `extra_hosts` hacks.

### Commands that matter

```bash
docker compose up --build     # first run, or after a Dockerfile change
docker compose up             # day-to-day start
docker compose ps             # which services are up, and healthy?
docker compose logs -f api    # follow the API logs only
docker compose down           # stop and remove containers + network
docker compose down -v        # also remove named volumes (fresh data)
```

Add a `profiles:` key for services not every developer needs
(the eval runner, the seed-data loader) so `up` stays fast and
`docker compose --profile eval up` starts the extras on demand.

### What I will add next in this file

- Next: the same image deployed to Azure Container Apps vs AWS
  ECS Fargate — what changes, what stays the same.
