# 10 — Deployment and DevOps

## Executive Summary

The backend deploys to **Railway** using platform-native build configuration — `railpack.json` and `railway.toml`. There is **no Dockerfile, no docker-compose, no CI/CD pipeline, and no monitoring stack.** Deployment is git-push-triggered.

The most instructive part of this file is a **real production failure**: the app worked locally and crashed in production because the container image lacked the `git` binary. Three commits document the fix. That's a genuine DevOps war story, and it teaches the exact lesson containers exist to teach.

---

## 1. Current Deployment

### The two config files

**`railway.toml`** — runtime behaviour:
```toml
[deploy]
startCommand = "uvicorn main:app --host 0.0.0.0 --port $PORT"
healthcheckPath = "/"
healthcheckTimeout = 120
restartPolicyType = "on_failure"
```

**`railpack.json`** — build-time system dependencies:
```json
{
  "aptPkgs": ["git"]
}
```

### Line-by-line

| Setting | Value | Why |
|---|---|---|
| `--host 0.0.0.0` | Bind all interfaces | `127.0.0.1` would only accept connections from *inside* the container — the platform's router could never reach it |
| `--port $PORT` | Platform-assigned | Railway injects `$PORT`. Hardcoding 8000 breaks routing |
| `healthcheckPath` | `/` | The endpoint Railway polls |
| `healthcheckTimeout` | 120s | Raised in commit `cc97f4e` |
| `restartPolicyType` | `on_failure` | Restart on crash, not on clean exit |

**The `0.0.0.0` detail is a classic interview question.** `--host 127.0.0.1` is the default for local dev and is *correct* there — you don't want your laptop's dev server exposed to the network. In a container it's fatal: the process listens only on the loopback interface, so the platform's health check and router get connection refused. The container starts, looks healthy from the inside, and is completely unreachable.

### Why the healthcheck timeout was raised to 120s

Commit `cc97f4e`: *"increase healthcheck timeout to 120s"*.

The app imports `chromadb`, `langgraph`, `openai`, and eight `tree-sitter-*` packages. That's a lot of module-loading, plus ChromaDB may initialise its persistence layer. Cold start exceeded the default timeout, so Railway killed the container as unhealthy — before it had finished starting.

**The deeper lesson:** this conflates **startup** with **liveness**. Kubernetes separates them:

| Probe | Question | On failure |
|---|---|---|
| **Startup** | Has it finished booting? | Wait longer; don't run other probes yet |
| **Liveness** | Is it alive? | Restart the container |
| **Readiness** | Can it serve traffic? | Remove from load balancer |

With only one probe, you must set the timeout to accommodate the *slowest* case (cold start), which means genuine hangs also take 120 seconds to detect. A startup probe would let you keep liveness aggressive.

**Say this in an interview:** *"I raised the healthcheck timeout to 120 seconds because heavy imports made cold start slow. That works, but it's the wrong tool — I'm using one probe for two questions. Kubernetes-style startup and liveness probes would let me be patient during boot and aggressive afterwards. With a single 120-second timeout, a real hang also takes two minutes to detect."*

---

## 2. The Production Failure — A Real DevOps Story

### The commit sequence

```
83552fa  fix Railway build — root requirements.txt for nixpacks auto-detection
ab5c9d3  fix Railway build — use nixpacks.toml for install phase
d980fd1  simplify railway.toml — rootDirectory handles backend path
1abc469  move railway.toml into backend/ to match rootDirectory setting
c66322a  add railpack.json to include git system dependency
d87d226  fix railpack.json — packages must be object not array
16124db  fix railpack.json — use aptPkgs for git system dependency
cc97f4e  increase healthcheck timeout to 120s
```

**Eight deployment commits.** This is what real deployment looks like.

### The root cause

```python
# repo_manager.py
import git                                    # GitPython
git.Repo.clone_from(clone_url, clone_path, depth=1)
```

**GitPython is not a git implementation.** It's a Python *wrapper* that shells out to the `git` executable. It requires `git` to be installed as a system binary.

- **Locally:** the developer's Mac has git. Everything works.
- **In the container:** Railway's Python build image includes Python and pip, but **not** git. `clone_from` fails with "git executable not found."

```mermaid
graph TD
    A["pip install gitpython"] --> B["Python package installed ✅"]
    B --> C["import git works ✅"]
    C --> D["git.Repo.clone_from()"]
    D --> E{"git binary<br/>on PATH?"}
    E -->|"Local: yes"| F["✅ Works"]
    E -->|"Container: no"| G["❌ GitCommandNotFound"]
    style G fill:#e74c3c,color:#fff
    style F fill:#2ecc71,color:#000
```

### The fix, and the two failed attempts

```json
// Attempt 1 (c66322a) — wrong key
{"packages": ["git"]}

// Attempt 2 (d87d226) — right key, wrong type
{"packages": {"git": "*"}}

// Attempt 3 (16124db) — correct: apt packages are distinct from language packages
{"aptPkgs": ["git"]}
```

The distinction: `packages` means *language runtime* packages (Python version, Node version). `aptPkgs` means **system-level Debian packages**. Git is a system binary, not a language runtime.

### The interview answer

> I had a bug that only appeared in production, and it's exactly the class of bug containers are supposed to prevent.
>
> My repo cloning uses GitPython. What I didn't fully register is that GitPython isn't a git implementation — it's a wrapper that shells out to the `git` binary. So `pip install gitpython` gives you the Python package but not the executable it depends on. My Mac has git installed, so everything worked locally. Railway's Python build image doesn't, so every clone failed in production.
>
> It took me three attempts to fix because Railway's build config distinguishes language packages from system packages, and I had to land on `aptPkgs` specifically.
>
> The real lesson is that I had an **undeclared implicit system dependency**. My `requirements.txt` described my Python dependencies completely and my system dependencies not at all. Containers exist to make environments reproducible, and I'd undermined that by letting the platform infer my image instead of declaring it.
>
> With an explicit Dockerfile, `RUN apt-get install -y git` would have sat right next to `pip install`, visible in review, versioned in git, and identical locally and in production. I'd have caught it the first time I ran the container locally. That's the argument for owning your Dockerfile rather than relying on buildpack auto-detection: buildpacks are convenient until your dependencies aren't the ones they guessed.

**Why this scores extremely well:** it's specific, it identifies a *category* of bug, and it draws a general architectural conclusion (explicit > inferred) rather than just "I fixed a config file."

---

## 3. Environment Variables

| Variable | Local | Production | Purpose |
|---|---|---|---|
| `GITHUB_APP_ID` | `.env` | Railway var | App identifier (JWT `iss`) |
| `GITHUB_WEBHOOK_SECRET` | `.env` | Railway var | HMAC key |
| `GITHUB_PRIVATE_KEY_PATH` | `.env` → `.pem` | — | Local key file path |
| `GITHUB_PRIVATE_KEY` | — | Railway var (**base64**) | Production key |
| `OPENAI_API_KEY` | `.env` | Railway var | OpenAI auth |
| `CHROMADB_PATH` | default `/tmp/chromadb` | default | Vector store location |
| `PORT` | 8000 | **Injected by Railway** | Bind port |

### The base64 private key trick

```python
raw = os.getenv("GITHUB_PRIVATE_KEY")
if raw:
    private_key = base64.b64decode(raw).decode("utf-8")   # production
else:
    with open(os.getenv("GITHUB_PRIVATE_KEY_PATH")) as f:
        private_key = f.read()                             # local
```

**The problem:** a PEM file is multi-line.
```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA...
-----END RSA PRIVATE KEY-----
```

Environment variables are fundamentally single-line strings. Different platforms mangle embedded newlines differently — some strip them, some escape them as literal `\n`, some truncate at the first one. The result is an unparseable key and a cryptic RSA error.

**Base64 collapses it to one line**, safe under any transport. Decode at read time.

**This is a small, real, production-tested solution** — mention it, it's the kind of detail that signals you've actually shipped something.

### The `load_dotenv(override=True)` detail

```python
load_dotenv(override=True)
```

By default `python-dotenv` will **not** overwrite an existing shell environment variable. If your shell exports `OPENAI_API_KEY` for another project, `.env` is silently ignored — and you spend an hour debugging why the wrong API key is being used. `override=True` makes `.env` authoritative.

### The 12-Factor principle

> *"Store config in the environment."*

Config that varies between deploys (credentials, hostnames, resource handles) lives in environment variables, not in code. This gives you: no secrets in git, the same artifact deployable to any environment, and config changes without a rebuild.

---

## 4. 📚 Docker Theory

> **Not used in this project.** Railway builds images via buildpacks. But Docker is asked about constantly, and the git-binary story gives you a genuine reason to discuss it.

### VM vs container

```
Virtual Machines                    Containers
┌──────────┬──────────┐            ┌──────────┬──────────┐
│  App A   │  App B   │            │  App A   │  App B   │
│  Bins    │  Bins    │            │  Bins    │  Bins    │
│ Guest OS │ Guest OS │  ~GBs      ├──────────┴──────────┤  ~MBs
├──────────┴──────────┤            │   Docker Engine     │
│     Hypervisor      │            ├─────────────────────┤
├─────────────────────┤            │      Host OS        │
│      Host OS        │            └─────────────────────┘
└─────────────────────┘
Boot: minutes                       Boot: milliseconds
```

Containers share the host kernel and isolate via **namespaces** (process, network, mount, user isolation) and **cgroups** (CPU, memory, I/O limits). No guest OS means orders of magnitude less overhead.

### The Dockerfile this project should have

```dockerfile
# ---- Build stage ----
FROM python:3.13-slim AS builder

WORKDIR /app
# Copy requirements FIRST — this layer caches independently of source changes
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ---- Runtime stage ----
FROM python:3.13-slim

# THE FIX for the production bug — explicit system dependency
RUN apt-get update && \
    apt-get install -y --no-install-recommends git && \
    rm -rf /var/lib/apt/lists/*

# Run as non-root
RUN useradd --create-home --shell /bin/bash appuser
WORKDIR /app

COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/')"

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Every decision explained

| Line | Why |
|---|---|
| `python:3.13-slim` | ~150 MB vs ~1 GB for the full image. Smaller = faster pulls, smaller attack surface |
| Multi-stage build | Build tools stay in the builder stage; runtime image ships only installed packages |
| `COPY requirements.txt` **before** source | Layer caching — source changes don't invalidate the pip install layer |
| `--no-cache-dir` | pip's cache is useless in an image; it's just size |
| `apt-get install git` | **The production bug fix, declared explicitly** |
| `rm -rf /var/lib/apt/lists/*` | Removes ~40 MB of apt metadata in the *same* layer (a separate `RUN` wouldn't shrink it) |
| `useradd` + `USER appuser` | **Never run as root.** A container escape as root is far worse |
| `PYTHONUNBUFFERED=1` | Without it, Python buffers stdout and logs appear late or vanish on crash |
| `HEALTHCHECK --start-period` | The Docker equivalent of the 120s timeout — grace period during boot |

### Layer caching — why order matters

```dockerfile
# ❌ Every source change reinstalls all dependencies (~2 min)
COPY . .
RUN pip install -r requirements.txt

# ✅ Source changes only rebuild the last layer (~2 sec)
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

Docker caches each layer by the checksum of its inputs. Changing any file invalidates that layer **and every layer after it**. Since source changes far more often than dependencies, dependencies must be installed first.

### docker-compose for local development

```yaml
version: "3.9"
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    volumes:
      - ./backend:/app          # live reload
      - chroma_data:/data/chromadb
    environment:
      - CHROMADB_PATH=/data/chromadb
    depends_on:
      redis: { condition: service_healthy }

  redis:                        # for the future task queue
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5
    volumes: [redis_data:/data]

volumes:
  chroma_data:
  redis_data:
```

**Named volumes vs bind mounts:**

| | Bind mount (`./src:/app`) | Named volume (`data:/var/lib`) |
|---|---|---|
| Location | Specific host path | Docker-managed |
| Use for | **Source code, live reload** | **Persistent data** |
| Portable | ❌ Host-dependent | ✅ |

**This directly solves the ephemeral-ChromaDB problem.** A named volume survives container recreation, so the vector index persists across deploys.

### Docker networking

| Mode | Behaviour |
|---|---|
| `bridge` (default) | Private network; containers reach each other **by service name** |
| `host` | Shares host network stack; no isolation |
| `none` | No networking |

In compose, `redis` resolves as a hostname from the `api` container via Docker's embedded DNS. That's why you write `redis://redis:6379` rather than an IP.

---

## 5. 📚 CI/CD

> **Not implemented.** No `.github/workflows/` exists. Deployment is git-push-triggered by Railway with no gate.

### The pipeline this project needs

```mermaid
graph LR
    A["git push"] --> B["Lint<br/>ruff"]
    B --> C["Type check<br/>mypy"]
    C --> D["Unit tests<br/>pytest, mocked"]
    D --> E["Security scan<br/>pip-audit, bandit"]
    E --> F["Build image"]
    F --> G["Deploy staging"]
    G --> H["Smoke test"]
    H --> I["Deploy production"]
    style D fill:#2ecc71,color:#000
    style I fill:#3498db,color:#fff
```

### GitHub Actions workflow

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
          cache: pip                      # cache deps between runs

      - run: pip install -r backend/requirements.txt -r backend/requirements-dev.txt

      - name: Lint
        run: ruff check backend/

      - name: Type check
        run: mypy backend/app/ --ignore-missing-imports

      - name: Unit tests
        run: pytest backend/tests/unit -v --cov=app --cov-report=xml
        env:
          OPENAI_API_KEY: sk-test-fake     # mocked — never a real key
          GITHUB_WEBHOOK_SECRET: test-secret

      - name: Security scan
        run: |
          pip install pip-audit bandit
          pip-audit -r backend/requirements.txt
          bandit -r backend/app/ -ll

      - uses: codecov/codecov-action@v4
```

### Pin your dependencies

```txt
# ❌ Current requirements.txt — mostly unpinned
fastapi
uvicorn
httpx
openai>=1.0.0

# ✅ Pinned
fastapi==0.115.0
uvicorn[standard]==0.32.0
httpx==0.27.2
openai==1.54.3
```

**Why this matters:** unpinned dependencies mean two deploys of *identical code* can install different versions and behave differently. Builds stop being reproducible, and you get breakage on someone else's release schedule.

Better still: `pip-tools` or `uv` with a lockfile, capturing transitive dependencies with hashes.

**Honest self-assessment:** *"My `requirements.txt` is mostly unpinned, which means my builds aren't reproducible. Two deploys of identical code could install different FastAPI versions. That's the cheapest thing on my list to fix — it's a lockfile."*

---

## 6. 📚 Monitoring and Observability

> **Not implemented.** `print()` to stdout only.

### The three pillars

| Pillar | Question | Tool |
|---|---|---|
| **Logs** | What happened? | structlog → Loki / CloudWatch |
| **Metrics** | How much / how fast? | Prometheus + Grafana |
| **Traces** | Where did time go? | OpenTelemetry + Jaeger |

### Structured logging

```python
import structlog
logger = structlog.get_logger()

log = logger.bind(pr=pr_number, repo=f"{owner}/{repo_name}", sha=head_sha[:7])
log.info("pipeline.started")
log.info("ingestion.complete", chunks=result.chunks_stored, mode=result.mode)
log.info("findings.filtered", posted=len(to_post), dropped=dropped)
log.error("pipeline.failed", exc_info=True)
```

```json
{"event":"findings.filtered","pr":7,"repo":"owner/demo","sha":"a1b2c3d",
 "posted":10,"dropped":1,"timestamp":"2026-07-20T10:15:32Z","level":"info"}
```

**The killer feature is the bound context.** Every log line from one pipeline run carries the same `pr` and `sha`, so concurrent reviews stay separable. With `print()`, two simultaneous PRs interleave into unreadable output — exactly when you'd most need the logs.

### Metrics worth tracking

```python
from prometheus_client import Counter, Histogram

reviews_total = Counter("reviews_total", "Reviews completed", ["status"])
pipeline_duration = Histogram("pipeline_duration_seconds", "Duration", ["stage"])
findings_posted = Counter("findings_posted_total", "Findings", ["severity", "agent"])
llm_tokens = Counter("llm_tokens_total", "Tokens", ["model", "type"])
llm_cost = Counter("llm_cost_usd_total", "Estimated cost")
```

**The one that matters most here: `llm_cost_usd_total`.** This system's dominant operational risk is unbounded spend from unbounded concurrency. A cost metric with an alert is the difference between noticing in an hour and noticing on the monthly invoice.

### The four golden signals (Google SRE)

| Signal | This system |
|---|---|
| **Latency** | Time from webhook to comments posted |
| **Traffic** | Webhooks per minute |
| **Errors** | Failed pipelines / total |
| **Saturation** | Concurrent in-flight reviews; OpenAI rate-limit headroom |

**Saturation is the one to watch** — with unbounded `asyncio.gather`, saturation is invisible until 429s start.

### Alerting

| Alert | Threshold | Severity |
|---|---|---|
| Pipeline error rate | >10% over 15 min | 🔴 Page |
| Daily LLM spend | >$X | 🔴 Page |
| OpenAI 429 rate | >5% | 🟠 Warn |
| p95 latency | >5 min | 🟠 Warn |
| Health check failing | 2 consecutive | 🔴 Page |

**Alert on symptoms users feel, not causes.** "Reviews aren't posting" is actionable; "CPU is at 80%" might be completely fine.

---

## 7. Deployment Comparison

| Platform | Model | Pros | Cons | Fit here |
|---|---|---|---|---|
| **Railway** ← used | PaaS | Zero config, git-push deploy, cheap | Ephemeral FS, limited control | ✅ Good for solo |
| Render | PaaS | Similar; persistent disks | Slower cold starts | ✅ Would work |
| Fly.io | Edge containers | Global, real volumes | More config | ✅ Good |
| Heroku | PaaS | Mature | Expensive; ephemeral FS | ⚠️ |
| AWS ECS/Fargate | Managed containers | Full control, EFS volumes | Complex, IAM overhead | ⚠️ Overkill now |
| Kubernetes | Orchestration | Ultimate control, HPA | Massive operational burden | ❌ Way overkill |
| Vercel/Lambda | Serverless | Scale to zero | **Execution time limits kill it** | ❌ Incompatible |

**Why serverless doesn't work here — a good interview point:**

> Lambda and Vercel functions cap execution time — typically 15 minutes on Lambda, 10 seconds to 5 minutes on Vercel depending on plan. My pipeline runs 30–90 seconds *after* the HTTP response is already sent, as a detached background task. Serverless platforms freeze or terminate the execution environment once the response returns, so my background task would be killed mid-flight.
>
> You *can* make it work, but only by restructuring: the webhook function returns immediately and enqueues to SQS, then a separate worker function consumes the queue. That's actually a better architecture than what I have — it gives durability and retries — but it's a genuine redesign, not a deployment change. Which is the point: serverless isn't a deployment target you can swap in, it's an architectural constraint you design around.

---

## 8. Interview Questions

### Q1. "Walk me through your deployment."

**Best answer:**
> Railway, triggered by git push to main. Two config files: `railway.toml` sets the start command, health check path, a 120-second health check timeout, and an on-failure restart policy. `railpack.json` declares system-level apt packages.
>
> The start command is `uvicorn main:app --host 0.0.0.0 --port $PORT`. Both flags matter. `0.0.0.0` because binding to `127.0.0.1` — the local dev default — means the process only accepts connections from inside the container, so the platform's router gets connection refused and the container looks dead while appearing fine internally. And `$PORT` because Railway assigns it dynamically; hardcoding 8000 breaks routing.
>
> Secrets are Railway environment variables. The GitHub App private key is base64-encoded, because a PEM is multi-line and environment variables mangle newlines differently on every platform. My code handles both paths — base64 env var in production, file path locally.
>
> What I don't have: no Dockerfile, no CI/CD, no monitoring beyond stdout, and mostly-unpinned dependencies. Push to main deploys straight to production with no test gate. That's fine for a solo project and would be irresponsible with a team.

---

### Q2. "Tell me about a production bug."

**Best answer:** *(The git binary story from section 2 — know this cold. It's your strongest DevOps answer.)*

**Follow-ups:**
- *"How would you have caught it earlier?"* — Running the actual container locally before deploying. Or a CI step that builds the image and runs a smoke test. The failure was that my local environment and my production environment differed in an undeclared way, and I never tested the production one.
- *"Why not just use a Dockerfile from the start?"* — Convenience. Buildpacks auto-detect and get you deployed in minutes. The trade is that they guess your dependencies, and they guessed wrong. For anything I'd hand to a team, I'd own the Dockerfile.

---

### Q3. "How would you set up CI/CD?"

**Best answer:**
> Six stages, ordered by how fast they fail — cheapest checks first so feedback is quick.
>
> **Lint** with ruff — seconds. **Type check** with mypy — catches a real class of bug given how much of my code passes dicts around. **Unit tests** with pytest against mocked OpenAI, so they're fast, free, and deterministic. **Security scanning** with `pip-audit` for known CVEs in dependencies and `bandit` for common Python security anti-patterns. **Build** the image. **Deploy to staging**, run a smoke test, then production.
>
> The gate that matters most for me specifically is dependency pinning. My `requirements.txt` is mostly unpinned right now, so two deploys of identical code can install different library versions. Before any of the rest, I'd generate a lockfile with hashes — reproducible builds are the foundation everything else sits on.
>
> For deployment strategy I'd want blue-green or canary rather than the rolling replace I have now. Though honestly for a webhook consumer, the more valuable thing is the durable queue: right now a deploy mid-pipeline silently drops that review, because `asyncio.create_task` doesn't survive a restart. A queue makes deploys safe regardless of strategy, because in-flight work is in the broker rather than in process memory.
>
> One thing I'd be careful about: secrets in CI. Tests must never use a real `OPENAI_API_KEY`. Mocked clients with an obviously-fake key, so a leaked CI log costs nothing.

---

### Q4. "Your data is on ephemeral storage. Is that a problem?"

**Best answer:**
> Yes, though it's a cost and correctness problem rather than a data-loss one.
>
> ChromaDB lives at `/tmp/chromadb`, and Railway's filesystem is ephemeral — every deploy and every restart wipes it. Nothing *breaks*: the next PR finds an empty collection and does a full ingest instead of an incremental one. The system is self-healing by design, which is why I didn't notice immediately.
>
> But it defeats my single biggest cost optimisation. Incremental ingestion means embedding five changed files instead of five hundred. When the index is wiped, I pay full embedding cost again for a repo I've already indexed — and at unpredictable intervals, tied to my deploy schedule rather than anything meaningful.
>
> There's a second-order effect too: a fresh index means the pattern agent has no historical context on the first PR after a deploy, so its findings are weaker until the index rebuilds.
>
> The fix is a Railway persistent volume mounted somewhere durable, with `CHROMADB_PATH` pointed at it. I deliberately made that path an environment variable with a default rather than hardcoding it, so it's a config change and not a code change.
>
> Longer term, if I ran more than one instance, an embedded database stops working entirely — each instance would have its own divergent index. At that point I'd move to ChromaDB in server mode, or to pgvector so I get one database serving both vectors and any relational data I add.

**Why this scores well:** it distinguishes severity ("cost, not data loss"), explains *why* it went unnoticed, names a second-order effect, gives the immediate fix, and identifies the scale point where the whole approach breaks.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `--host 127.0.0.1` in a container | Unreachable; health checks fail | `0.0.0.0` |
| Hardcoding the port | Platform routing breaks | `$PORT` |
| Undeclared system dependencies | Works locally, fails in prod | Explicit Dockerfile / `aptPkgs` |
| Unpinned dependencies | Non-reproducible builds | Lockfile with hashes |
| `COPY . .` before `pip install` | Cache invalidated on every change | Requirements first |
| Running as root | Container escape is catastrophic | `USER appuser` |
| Secrets in the image | Baked into layers permanently | Runtime env vars |
| No `PYTHONUNBUFFERED` | Logs delayed or lost on crash | Set it to 1 |
| Ephemeral volumes for state | Silent data loss | Named volumes |
| One probe for startup and liveness | Slow hang detection | Separate probes |
| No CI gate before deploy | Broken code reaches production | Test job on the branch |
| Real API keys in CI | Leak via logs | Mocked clients |

---

## 10. Cross-References

| Topic | File |
|---|---|
| Env vars and secrets | [07-Authentication-and-Security.md](07-Authentication-and-Security.md) |
| What CI would run | [11-Testing.md](11-Testing.md) |
| Cold start and startup cost | [12-Performance-Optimization.md](12-Performance-Optimization.md) |
| Multi-instance deployment | [13-Scalability.md](13-Scalability.md) |
| The git binary story as a challenge | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Persistent storage on the roadmap | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [11-Testing.md](11-Testing.md)
