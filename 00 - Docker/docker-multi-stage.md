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

## 2026-11-27 — Part 4: the same image on Azure Container Apps vs AWS ECS Fargate

Parts 1–3 made one image small and safe and gave it a local stack.
Today the image leaves the laptop: the same multi-stage build from
Part 2, deployed to two serverless container platforms. The image
does not change — the platform contract around it does.

### What stays the same on both platforms

- The image: same Dockerfile, same non-root user, same
  `/app/deps` layout. If the image changes per platform, the
  "works on my machine" problem just moved to the cloud.
- The health contract: `/health` returns 200 without calling the
  LLM or the vector store (Parts 2–3). Both platforms probe it;
  both will restart a container whose health check fails, so a
  slow model call in `/health` marks a healthy container dead on
  either platform.
- Secrets from the environment, never the image (Part 1): ACA
  secrets / Key Vault references, ECS secrets from Secrets
  Manager — same rule, different store.
- No code mounts in production: the image is self-contained. The
  `:ro` volume from the Compose file (Part 3) is a dev loop, not a
  deploy pattern.

### Azure Container Apps — the simple one

```bash
az containerapp create --name medical-ai --resource-group <rg> \
  --environment <env> --image medical-ai:slim \
  --target-port 8000 --ingress external \
  --min-replicas 0 --max-replicas 5 \
  --cpu 0.5 --memory 1Gi \
  --env-vars GROQ_API_KEY=secretref:groq-key \
             PINECONE_API_KEY=secretref:pinecone-key
```

- **Scale to zero.** `--min-replicas 0` means an idle demo costs
  ~$0 — the container stops, and the first request starts it
  (a cold start of a few seconds, which the client feels once).
  That is the right shape for a demo RAG backend that gets
  questions in bursts, not a steady stream.
- **Scaling is KEDA rules, not a VM count.** Scale on HTTP
  concurrency or on queue length (a Service Bus queue trigger
  scales the worker the same way the HTTP trigger scales the API).
  You declare "scale when this is busy", not "run three of these".
- **Revisions are the deploy.** Every deploy is a new revision;
  traffic splits between revisions (90/10 canary) without a load
  balancer change. A bad revision is a traffic flip back, not a
  redeploy.
- **Managed environment, no cluster.** No VPC to design, no node
  pool to size. The cost is control: no VNet on the Consumption
  plan, and the networking knobs are fewer than ECS.

### AWS ECS Fargate — the one with more knobs

```json
{
  "family": "medical-ai",
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "medical-ai:slim",
      "portMappings": [{ "containerPort": 8000 }],
      "healthCheck": {
        "command": ["CMD-SHELL",
          "python -c \"import urllib.request; urllib.request.urlopen('http://localhost:8000/health')\""],
        "interval": 30, "timeout": 5, "retries": 3, "startPeriod": 10
      },
      "logConfiguration": { "logDriver": "awslogs" },
      "secrets": [
        { "name": "GROQ_API_KEY", "valueFrom": "arn:aws:secretsmanager:…:groq-key" }
      ]
    }
  ]
}
```

- **No scale to zero.** The smallest always-on task (0.25 vCPU /
  0.5 GB) runs 24/7 and is a real monthly line item (on the order
  of $10–15/month). Stopping the service is the only $0 state,
  and a stopped service answers nothing.
- **The networking is yours to design** — VPC, subnets, security
  groups, ALB, target group health checks (a second health check,
  at the load balancer, on top of the container one). More control,
  more surface to get wrong: the ALB health check must use the
  same cheap `/health`, or the ALB will drain a healthy task.
- **Autoscaling is a policy, not a revision.** Target-tracking on
  CPU or ALB request count; a deploy is a new task definition and
  a service update, with `minimumHealthyPercent` deciding whether
  the old tasks stay up during the rollout.

### Comparison

| | Azure Container Apps | AWS ECS Fargate |
|---|---|---|
| Scale to zero | Yes — idle costs ~$0 | No — smallest task is a monthly line |
| Deploy unit | Revision, traffic-split built in | Task definition + service update |
| Ingress | Built in (external/internal) | ALB + target group (you build it) |
| Networking | Managed environment | Your VPC, subnets, security groups |
| Secrets | ACA secrets / Key Vault | Secrets Manager / SSM |
| Ops burden | Low — no cluster | Medium — the cluster is serverless, the network is not |
| When it wins | Bursty demo / side project; already on Azure | Already on AWS with a VPC + ALB; need the control |

### The decision rule

Pick Container Apps for the Medical AI demo: bursty traffic,
scale-to-zero, and the team is already on Azure for the rest of
the case. Pick ECS Fargate when the system already lives in an AWS
VPC with an ALB and the team wants the networking control — or
when the container needs something ACA does not offer (a sidecar
pattern beyond Dapr, a specific placement rule). Either way, the
image, the health endpoint and the secret rule do not change, so
moving between them later is a platform change, not a rewrite.


## 2026-11-28 — Part 5: CI/CD — build the image, scan it, push it, deploy it

Parts 1–4 made one image small and safe, gave it a local stack, and
showed it running on two platforms — all by hand. A pipeline makes
the sequence the same every time, and makes a bad image fail the
build instead of reaching users. This one is for the Medical AI
backend (FastAPI, the Part 2 Dockerfile). It runs on every push to
`main` and on every pull request, and deploys only from `main`.

### The stages, in order

1. **Build** the multi-stage image, with BuildKit layer caching so a code-only change does not rebuild the dependency layer.
2. **Test** inside a container built from that same image, so the tests run against exactly what will ship — not the laptop's Python.
3. **Scan** for known vulnerabilities, failing the build on any *critical* finding. High findings warn; critical ones block.
4. **Push** to the registry (Azure Container Registry), tagged twice: the commit SHA and `latest`.
5. **Deploy** to Azure Container Apps (the Part 4 decision). The ECS Fargate alternative is noted at the end — stages 1–4 do not change.

### The workflow

```yaml
# .github/workflows/deploy.yml
name: Build, test, scan and deploy
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
env:
  REGISTRY: medicalai.azurecr.io
  IMAGE_NAME: medical-ai
  RESOURCE_GROUP: medical-ai-rg
  CONTAINER_APP: medical-ai

jobs:
  build-test-scan:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Log in to ACR
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - name: Build image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: false
          load: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          # BuildKit cache: the pip-install layer is reused across
          # runs, so a code-only change rebuilds in seconds.
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Run tests inside the image
        run: |
          docker run --rm ${{ env.IMAGE_NAME }}:${{ github.sha }} \
            python -m pytest tests/ -q
      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          # Critical findings fail the build. Highs are shown but
          # do not block — tighten to HIGH once the base image is
          # clean enough that highs are rare.
          exit-code: '1'
          severity: 'CRITICAL'

  push-and-deploy:
    needs: build-test-scan
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Log in to ACR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - name: Build and push (SHA + latest)
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v2
        with:
          resourceGroup: ${{ env.RESOURCE_GROUP }}
          containerAppName: ${{ env.CONTAINER_APP }}
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

### Why it is shaped this way

- **Tests run inside the image, not on the runner.** `docker run …
  pytest` tests the artefact that will deploy, including the
  non-root user, the `/app/deps` layout and the pinned base image
  from Parts 1–2. Testing on the runner's Python would test a
  different environment and miss the packaging bugs the
  multi-stage build exists to catch.
- **Two tags, two jobs.** The SHA tag is the deployable unit:
  immutable, traceable to one commit, and what the deploy step
  references. `latest` is a convenience for humans pulling the
  image by hand — nothing deploys `latest`, because `latest` moves
  and a rollback to "the last good `latest`" is not a thing.
- **Scan before push, fail on CRITICAL only.** A vulnerable image
  never lands in the registry at all. The threshold is deliberately
  narrow at first: a pipeline that fails on every HIGH finding in a
  Python image full of transitive advisories gets disabled within a
  week. Start at CRITICAL, fix what it finds, tighten later.
- **No secrets in the workflow file.** The ACR login comes from
  `secrets.ACR_USERNAME` / `secrets.ACR_PASSWORD` — the Part 1 rule
  (no secrets in the image), extended to the pipeline. Prefer a
  workload identity federation credential over a long-lived ACR
  password when the subscription allows it.

### Deploying to ECS Fargate instead

Stages 1–4 are identical — same Dockerfile, same tests, same scan,
same tags — only the registry and the deploy step change: push to
ECR instead of ACR, then `aws ecs update-service
--force-new-deployment` with the new task definition. Two
differences: the deploy is a service update, not a revision, so
there is no built-in traffic split for a canary (Part 4); and the
deploy step needs AWS credentials, which should be an OIDC role,
not a stored access key.

### Rollback

A bad deploy is undone by redeploying the previous good SHA —
never by pushing a new `latest` over it, and never by editing the
container app in the portal:

```bash
# Last good SHA from the workflow run history, then:
az containerapp update --name medical-ai --resource-group medical-ai-rg \
  --image medicalai.azurecr.io/medical-ai:<previous-sha>
```

On Container Apps this creates a new revision, and traffic can be
split back to the old revision without a redeploy at all
(`az containerapp revision set-traffic`) — the faster rollback when
the old revision is still there. On ECS the equivalent is updating
the service back to the previous task definition revision. Either
way the rollback is one line because every deploy was a SHA. If a
deploy fails the health probe (`/health`, Parts 2–3), the platform
keeps the old revision serving.

### What I will add next in this file

- Next: the image size and scan results, measured — record the
  first real build's size, test time, and Trivy findings in the
  Part 2 table, and tighten the scan threshold to HIGH once the
  base image is clean.
