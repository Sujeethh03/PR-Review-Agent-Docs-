# 08 — API Documentation

## Executive Summary

This service exposes **two HTTP endpoints** and consumes **six external API operations** across GitHub and OpenAI. It is an **API consumer far more than an API provider** — the interesting surface is outbound, not inbound.

This file documents every endpoint and every external call in full, then covers REST design theory (idempotency, status codes, versioning, pagination) that interviewers ask regardless of your project's surface area.

---

# PART A — Endpoints We Expose

## A1. `POST /webhook`

The only functional endpoint.

### Request

| Field | Value |
|---|---|
| Method | `POST` |
| Path | `/webhook` |
| Content-Type | `application/json` |
| Auth | HMAC-SHA256 via `X-Hub-Signature-256` |

**Required headers:**

| Header | Example | Purpose |
|---|---|---|
| `X-Hub-Signature-256` | `sha256=a1b2c3...` | HMAC of body with shared secret |
| `X-GitHub-Event` | `pull_request` | Event type — drives routing |
| `X-GitHub-Delivery` | `72d3162e-cc78-...` | Unique delivery UUID (⚠️ currently unused) |

### Body (abridged `pull_request` event)

```json
{
  "action": "opened",
  "number": 7,
  "pull_request": {
    "number": 7,
    "title": "Add user authentication",
    "user": { "login": "Sujeethh03" },
    "head": { "sha": "a1b2c3d4e5f6...", "ref": "feature/auth" },
    "base": { "sha": "f6e5d4c3b2a1...", "ref": "main" }
  },
  "repository": { "full_name": "Sujeethh03/pr-review-agent-demo" },
  "installation": { "id": 12345678 }
}
```

### Fields actually consumed

| JSON path | Used for | Required? |
|---|---|---|
| `action` | Trigger only on `opened`/`synchronize` | ✅ |
| `pull_request.number` | PR to comment on | ✅ |
| `pull_request.head.sha` | Clone target + `commit_id` for review | ✅ |
| `repository.full_name` | Split into owner/repo | ✅ |
| `installation.id` | Token exchange | ✅ |
| `pull_request.title`, `user.login` | Logging only | ❌ |

### Responses

| Code | Body | When |
|---|---|---|
| `200` | `{"status":"ok","event":"pull_request"}` | Accepted (even if ignored) |
| `400` | `{"detail":"Unexpected repo format: ..."}` | `full_name` has no `/` |
| `401` | `{"detail":"Invalid signature"}` | HMAC mismatch |

### Event routing

```mermaid
graph TD
    A["X-GitHub-Event"] --> B{Type?}
    B -->|"pull_request_review<br/>pull_request_review_comment<br/>check_suite, status, ..."| C["200 — IGNORED<br/>(feedback-loop guard)"]
    B -->|push| D["200 — log only"]
    B -->|pull_request| E{Action?}
    E -->|"opened / synchronize"| F{"installation_id<br/>AND head_sha?"}
    E -->|"closed, labeled, ..."| G["200 — log only"]
    F -->|Yes| H["🚀 Spawn pipeline → 200"]
    F -->|No| G
    B -->|other| I["200 — 'Unhandled event'"]
    style C fill:#e67e22,color:#fff
    style H fill:#2ecc71,color:#000
```

### The ignore list — why it exists

```python
if event_type in (
    "security_advisory", "github_app_authorization",
    "check_suite", "check_run", "status",
    "pull_request_review", "pull_request_review_comment",
    "pull_request_review_thread",
):
    return {"status": "ok", "event": event_type}
```

The last three are the important ones. **Without them the system loops infinitely:**

```
Bot posts review → GitHub fires pull_request_review
                 → pipeline runs → bot posts review
                 → GitHub fires pull_request_review → ∞
```

Commit `021d87e` is titled *"silence review webhook feedback loop and fix cross-category dedup"* — this was hit in production and fixed.

**A more robust alternative:** instead of ignoring the event type entirely, check whether the actor is your own bot:
```python
if payload.get("sender", {}).get("login") == f"{APP_NAME}[bot]":
    return {"status": "ok", "event": "self-generated"}
```
This lets you react to *humans'* review comments (e.g. replying to a question about a finding) while still ignoring your own. Worth mentioning as a future improvement.

### ⚠️ Idempotency gap

GitHub retries failed deliveries and offers manual redelivery. There is **no deduplication** — the same `X-GitHub-Delivery` UUID processed twice runs the pipeline twice, posting duplicate comments and doubling cost.

**The fix:**
```python
delivery_id = request.headers.get("X-GitHub-Delivery")
if delivery_id and await seen_recently(delivery_id):     # Redis SETNX with TTL
    return {"status": "ok", "event": "duplicate"}
```

---

## A2. `GET /` — health check

```python
@app.get("/")
async def root():
    return {"status": "code review agent is running"}
```

Referenced by `railway.toml` as `healthcheckPath = "/"` with a 120-second timeout.

**Limitation:** this is a **liveness** probe (is the process running?), not a **readiness** probe (can it do work?). It returns 200 with an invalid `OPENAI_API_KEY`, a corrupted ChromaDB, or an unreachable GitHub.

| Probe | Question | Failure action |
|---|---|---|
| Liveness | Is the process alive? | Restart the container |
| Readiness | Can it serve traffic? | Remove from load balancer |
| Startup | Has it finished booting? | Delay other probes |

**A proper readiness endpoint:**
```python
@app.get("/health/ready")
async def ready():
    checks = {
        "github_config": bool(os.getenv("GITHUB_APP_ID")),
        "openai_config": bool(os.getenv("OPENAI_API_KEY")),
        "chromadb":      _chroma_reachable(),
        "in_flight":     len(_background_tasks),
    }
    healthy = all(v for k, v in checks.items() if k != "in_flight")
    return JSONResponse(
        {"status": "ready" if healthy else "degraded", "checks": checks},
        status_code=200 if healthy else 503,
    )
```

---

## A3. 🗑️ The deleted API

Removed in `a61bfd2` (`backend/app/api/findings.py`, 120 lines):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/findings` | List findings (with filters, pagination) |
| `GET` | `/api/findings/{id}` | Single finding detail |
| `PATCH` | `/api/findings/{id}` | Approve / dismiss |
| `GET` | `/api/reviews` | List PR reviews |

`PATCH` is what triggered the CORS bug in commit `952f098` — `PATCH` wasn't in `allow_methods`, so the browser's preflight failed.

**Good REST design detail worth noting:** `PATCH` was the correct verb. The dashboard changed only `status` from `pending` to `approved` — a **partial** update. `PUT` would semantically mean "replace the entire finding", requiring the client to send every field.

---

# PART B — External APIs We Consume

```mermaid
graph LR
    A["Our Service"] -->|"1. POST access_tokens"| B["GitHub API"]
    A -->|"2. GET pulls/{n} (diff)"| B
    A -->|"3. git clone"| C["GitHub Git"]
    A -->|"4. GET pulls/{n}/comments"| B
    A -->|"5. DELETE comments/{id}"| B
    A -->|"6. POST pulls/{n}/reviews"| B
    A -->|"7. POST issues/{n}/comments"| B
    A -->|"8. POST embeddings"| D["OpenAI"]
    A -->|"9. POST chat/completions"| D
    style B fill:#24292e,color:#fff
    style D fill:#10a37f,color:#fff
```

## B1. GitHub — Create installation access token

```http
POST https://api.github.com/app/installations/{installation_id}/access_tokens
Authorization: Bearer <APP_JWT>
Accept: application/vnd.github+json
```

**Response:**
```json
{
  "token": "ghs_16C7e42F292c6912E7710c838347Ae178B4a",
  "expires_at": "2026-07-20T11:15:32Z",
  "permissions": { "contents": "read", "pull_requests": "write" }
}
```

**Implementation** (`github_client.py:32`):
```python
async def get_installation_token(installation_id: int) -> str:
    token = _generate_jwt()
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{GITHUB_API}/app/installations/{installation_id}/access_tokens",
            headers={"Authorization": f"Bearer {token}",
                     "Accept": "application/vnd.github+json"},
        )
        response.raise_for_status()
        return response.json()["token"]
```

⚠️ **Called 3× per pipeline run** — once in `get_pr_diff`, once in `clone_repo`, once in `run_pr_writer`. Each generates a fresh JWT and makes a round trip.

**Why it's inefficient:** the token is valid for an hour. Three calls for one pipeline is two wasted round trips (~200–400ms) plus three RSA signing operations.

**The fix — a TTL cache:**
```python
_token_cache: dict[int, tuple[str, float]] = {}

async def get_installation_token(installation_id: int) -> str:
    cached = _token_cache.get(installation_id)
    if cached and cached[1] > time.time() + 300:      # 5-min safety margin
        return cached[0]
    ... fetch ...
    _token_cache[installation_id] = (token, expires_at_epoch)
    return token
```

**This is a genuine, easy optimisation and good to volunteer.** The 5-minute margin matters — you don't want a token expiring mid-pipeline.

## B2. GitHub — Fetch PR diff

```http
GET https://api.github.com/repos/{owner}/{repo}/pulls/{n}
Authorization: Bearer <installation_token>
Accept: application/vnd.github.diff        ← content negotiation
```

**The `Accept` header is the whole trick.** Same URL, different representations:

| Accept | Returns |
|---|---|
| `application/vnd.github+json` | PR metadata as JSON |
| `application/vnd.github.diff` | **Unified diff text** ← used |
| `application/vnd.github.patch` | Git patch format (with commit messages) |

**This is textbook REST content negotiation** — one resource identified by one URI, multiple representations selected by the client. Worth naming explicitly in an interview.

**Limitation:** GitHub truncates diffs over ~20,000 lines or 1 MB. Very large PRs would silently produce a partial review. Not handled — a real gap.

## B3. GitHub — Clone repository

```python
clone_url = f"https://x-access-token:{token}@github.com/{owner}/{repo}.git"
git.Repo.clone_from(clone_url, clone_path, depth=1)
```

**`depth=1`** — shallow clone. Fetches only the tip commit's tree, not the full history.

| | Full clone | Shallow (`depth=1`) |
|---|---|---|
| Linux kernel | ~4.5 GB, minutes | ~250 MB, seconds |
| History available | Complete | Tip only |
| Needed here? | ❌ | ✅ Only current file contents matter |

**`x-access-token`** is GitHub's required username placeholder when authenticating with an installation token over HTTPS.

⚠️ **Security note:** the token is embedded in the URL, which risks leaking into logs via exception messages. See [07-Authentication-and-Security.md](07-Authentication-and-Security.md).

## B4. GitHub — List and delete review comments

```http
GET  /repos/{owner}/{repo}/pulls/{n}/comments?per_page=100
DELETE /repos/{owner}/{repo}/pulls/comments/{comment_id}
```

```python
tasks = [
    client.delete(f"{GITHUB_API}/repos/{owner}/{repo_name}/pulls/comments/{c['id']}",
                  headers=headers)
    for c in response.json()
    if c.get("user", {}).get("type") == "Bot"          # ⚠️ THE BUG
]
await asyncio.gather(*tasks, return_exceptions=True)
```

### ⚠️ Confirmed bug — this deletes *every* bot's comments

The filter is `user.type == "Bot"`, which matches **all** GitHub Apps — Dependabot, Codecov, Sonar, Renovate, and any other review bot on that PR. On every push, this service silently destroys other tools' comments.

**The fix:**
```python
BOT_LOGIN = f"{os.getenv('GITHUB_APP_NAME')}[bot]"
... if c.get("user", {}).get("login") == BOT_LOGIN
```

**Two other issues in the same function:**

1. **No pagination.** `per_page=100` with no `Link` header following. A PR with 150 comments leaves 50 stale.
2. **Unbounded delete concurrency.** 100 simultaneous DELETEs can trip GitHub's secondary rate limits.

**Why this makes a great interview story:**
> I found this auditing my own code. My stale-comment cleanup filters on `user.type == "Bot"` — but that matches *every* GitHub App, not just mine. On a repo with Dependabot or Codecov, every push silently deletes their comments too. It's a one-line fix — filter on the bot's login rather than its type — but the failure mode is nasty because it's invisible: nobody gets an error, comments just quietly disappear. It's a good lesson in how "filter for my own resources" needs an identity check, not a category check.

## B5. GitHub — Post a review with inline comments

```http
POST /repos/{owner}/{repo}/pulls/{n}/reviews
```

```json
{
  "commit_id": "a1b2c3d4e5f6...",
  "event": "COMMENT",
  "comments": [
    {
      "path": "src/auth.py",
      "line": 45,
      "side": "RIGHT",
      "start_line": 42,
      "start_side": "RIGHT",
      "body": "## 🚨 HIGH — SQL Injection\n\n..."
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `commit_id` | Head SHA — anchors comments to a specific commit |
| `event` | `COMMENT` (advisory), `APPROVE`, or `REQUEST_CHANGES` |
| `path` | Repo-relative file path (no `b/` prefix!) |
| `line` | **End** line of the range |
| `side` | `RIGHT` = new version, `LEFT` = old |
| `start_line` | Start of a multi-line range (omit for single-line) |

**Why `event: "COMMENT"` and not `REQUEST_CHANGES`?** Because `REQUEST_CHANGES` **blocks the merge** until dismissed. That's a deliberate trust decision:

> My precision is unmeasured — I have no labelled dataset, so I genuinely don't know my false-positive rate. Blocking merges on an unmeasured signal would be irresponsible. A tool that wrongly blocks a merge gets disabled within a week. `COMMENT` makes it advisory: helpful when right, ignorable when wrong. If I ever measured precision above some bar, `REQUEST_CHANGES` for high-severity security findings would be defensible — but only then.

### The 422 failure mode — a real architectural weakness

```python
if response.status_code in (200, 201):
    return True
if response.status_code == 422:
    return False              # → fall back to a flat summary comment
response.raise_for_status()
```

**GitHub validates the entire review atomically.** If *any single comment* targets a line not present in the diff, the whole request returns `422 Unprocessable Entity` and **zero comments post**.

Since line numbers come from GPT-4o, hallucination is a live risk. One bad line number degrades all ten findings to a single flat summary comment with no inline context.

```mermaid
graph TD
    A["10 findings"] --> B["POST review with 10 comments"]
    B --> C{Response}
    C -->|201| D["✅ 10 inline comments"]
    C -->|422| E["❌ ZERO posted"]
    E --> F["Fallback: 1 summary body comment"]
    F --> G["⚠️ All inline context lost"]
    style E fill:#e74c3c,color:#fff
    style G fill:#e67e22,color:#fff
```

**Three fixes, in order of preference:**

1. **Validate before posting** — check each finding's line against `parse_diff_hunks()` output; drop or clamp invalid ones. Cheapest and best.
2. **Post individually** — one API call per comment, so one failure loses one comment. More calls, better isolation.
3. **Retry with bisection** — on 422, split the batch and retry halves to isolate the offender. Clever but complex.

**Say this in an interview:**
> The 422 behaviour is the design flaw I'd fix first. GitHub validates reviews atomically, so one hallucinated line number costs me all ten inline comments — they collapse into a single summary with no code context, which is dramatically less useful. The irony is I already have the data to prevent it: `parse_diff_hunks` gives me exactly which lines are in the diff. I just never cross-check the LLM's output against it before posting. It's maybe fifteen lines of validation.

## B6. GitHub — Fallback body comment

```http
POST /repos/{owner}/{repo}/issues/{n}/comments
```

**Note the path is `/issues/`, not `/pulls/`.** In GitHub's data model **every PR is an issue** — PRs are issues with an associated branch and diff. General (non-inline) comments live on the issue; inline diff comments live on the pull request. Knowing this quirk is a nice detail.

## B7. OpenAI — Embeddings

```python
response = client.embeddings.create(
    input=texts,                        # up to 100 chunk texts
    model="text-embedding-3-small",
)
vectors = [item.embedding for item in response.data]   # 1536 floats each
```

| Model | Dims | Relative cost | Quality |
|---|---|---|---|
| `text-embedding-3-small` | 1536 | **1×** ← used | Good |
| `text-embedding-3-large` | 3072 | ~6.5× | Better |
| `text-embedding-ada-002` | 1536 | ~1.3× | Legacy |

**Why `small`:** roughly 6.5× cheaper than `large` for a modest quality difference on code retrieval. Since retrieval only needs to surface *reasonably* relevant context — not perfectly ranked context — the extra quality doesn't change agent output much. **Good cost engineering with an articulable rationale.**

**Batching:** 100 texts per request instead of one. 1,000 chunks = 10 requests instead of 1,000. Saves ~990 round trips of network latency and HTTP overhead.

**Retry:** one retry with a 1-second sleep, then raise. Weak — see [09-AI-System.md](09-AI-System.md).

## B8. OpenAI — Chat completions with forced tool calling

```python
response = await client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": user_prompt},
    ],
    tools=[{
        "type": "function",
        "function": {
            "name": "report_findings",
            "description": "Report all bug findings from the code review",
            "parameters": FINDINGS_SCHEMA,
        },
    }],
    tool_choice={"type": "function", "function": {"name": "report_findings"}},
    temperature=0,
)
```

**`tool_choice` forced** — the model *must* call `report_findings`. It cannot reply with prose. This guarantees parseable structured output. Full treatment in [09-AI-System.md](09-AI-System.md).

## B9. API call budget per PR

For a PR with **H** hunks and **F** findings above threshold:

| Call | Count | Notes |
|---|---|---|
| Installation token | **3** | ⚠️ Should be 1 (cacheable) |
| Get diff | 1 | |
| Git clone | 1 | |
| Embeddings (ingest) | ⌈chunks/100⌉ | Full ingest only on first PR |
| Embeddings (retrieval) | **3 × H** | ⚠️ Same hunk embedded 3× |
| GPT-4o (agents) | **3 × H** | Minus pattern's skipped hunks |
| GPT-4o (suggestions) | F | |
| List comments | 1 | |
| Delete comments | C | One per stale bot comment |
| Post review | 1 (+1 on fallback) | |

**Worked example — 20-hunk PR, 10 findings, warm index:**
```
3 tokens + 1 diff + 1 clone + 60 embeddings + 60 GPT-4o + 10 suggestions
+ 1 list + ~10 deletes + 1 post  ≈  147 API calls
```

### The two visible inefficiencies

**1. Triple embedding.** Each agent independently calls `get_context_for_hunk`, which embeds the same `hunk.added_code`. Three identical embedding calls per hunk.

```python
# Fix: embed once, share across agents
async def precompute_hunk_embeddings(hunks) -> dict[str, list[float]]:
    texts = [h.added_code for h in hunks]
    response = await client.embeddings.create(input=texts, model="...")  # ONE batched call
    return {h.file + str(h.line_start): v.embedding
            for h, v in zip(hunks, response.data)}
```
This turns 60 individual calls into **1 batched call** — a ~98% reduction on that line item.

**2. Triple token fetch.** As covered in B1.

**These two fixes together are the highest-leverage cost optimisation available**, and being able to quantify them ("60 calls become 1") is exactly the kind of specificity interviewers reward.

---

# PART C — REST API Design Theory

## C1. HTTP methods and their properties

| Method | Purpose | Safe? | Idempotent? |
|---|---|---|---|
| `GET` | Read | ✅ | ✅ |
| `POST` | Create / process | ❌ | ❌ |
| `PUT` | Full replace | ❌ | ✅ |
| `PATCH` | Partial update | ❌ | ⚠️ Usually not |
| `DELETE` | Remove | ❌ | ✅ |

- **Safe** — no side effects. Crawlers may call freely.
- **Idempotent** — N identical calls have the same effect as one.

**Why `DELETE` is idempotent:** deleting an already-deleted resource leaves the same end state. The response code may differ (404 vs 204), but the *state* doesn't.

**Why `POST` isn't:** two POSTs create two resources. **This is precisely the webhook's problem** — GitHub redelivering the same event runs the pipeline twice.

## C2. Idempotency in practice

**The pattern:**
```python
@app.post("/webhook")
async def webhook(request: Request):
    delivery_id = request.headers.get("X-GitHub-Delivery")
    if await redis.set(f"delivery:{delivery_id}", "1", nx=True, ex=86400) is None:
        return {"status": "ok", "note": "duplicate"}     # already processed
    ...
```

`SET NX EX` is atomic: set-if-not-exists with expiry. Returns `None` if the key existed — meaning we've seen this delivery.

**Why the TTL matters:** without expiry the key set grows forever. 24 hours comfortably exceeds GitHub's retry window.

## C3. Status codes

| Code | Meaning | Use here |
|---|---|---|
| `200 OK` | Success with body | ✅ Webhook ACK |
| `201 Created` | Resource created | GitHub returns for review creation |
| `202 Accepted` | **Accepted, processing async** | ⚠️ *Arguably more correct than 200 for the webhook* |
| `204 No Content` | Success, no body | DELETE responses |
| `400 Bad Request` | Malformed | ✅ Bad repo format |
| `401 Unauthorized` | Not authenticated | ✅ Bad HMAC |
| `403 Forbidden` | Authenticated, not permitted | GitHub perms |
| `404 Not Found` | Doesn't exist | |
| `409 Conflict` | State conflict | Duplicate delivery |
| `422 Unprocessable` | Well-formed but semantically invalid | ✅ GitHub's response to bad line numbers |
| `429 Too Many Requests` | Rate limited | Should implement |
| `500 Internal Error` | Server bug | ⚠️ Never sent — deliberate |
| `503 Unavailable` | Temporarily down | Readiness failures |

**`401` vs `403`** — a classic interview question. `401` means *"I don't know who you are"* (authentication failed — retry with credentials). `403` means *"I know who you are and you may not"* (authorisation failed — retrying won't help).

**Why the webhook could arguably return `202 Accepted`:** it acknowledges receipt without having completed processing. `200` implies the work is done. In practice GitHub treats any 2xx as success, so it's semantic purity rather than function — but it's a good detail to raise unprompted.

## C4. Versioning

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URI path | `/v1/findings` | Visible, cacheable | URL churn |
| Header | `Accept: application/vnd.api.v1+json` | Clean URLs | Less discoverable |
| Query param | `/findings?version=1` | Easy | Pollutes params |
| **Date-based** | `X-GitHub-Api-Version: 2022-11-28` | Precise, granular | More versions to track |

**GitHub uses date-based**, and this project sends it:
```python
headers = {..., "X-GitHub-Api-Version": "2022-11-28"}
```

**Why pinning a version matters:** without it you get whatever the API's default is, which changes over time. Your code breaks on someone else's deploy schedule. Pinning means breakage happens when *you* choose to upgrade.

## C5. Pagination

| Strategy | Query | Pros | Cons |
|---|---|---|---|
| Offset | `?page=2&per_page=50` | Simple, jump to any page | **O(n) at depth**; drift on inserts |
| **Cursor/keyset** | `?after=eyJpZCI6MTAwfQ` | O(1), stable | No random access |
| Link header | GitHub's approach | Self-describing | Must parse |

**Why offset degrades:** `OFFSET 100000` makes the database scan and discard 100,000 rows before returning any. Keyset pagination (`WHERE id > last_seen_id LIMIT 50`) uses the index and is constant-time regardless of depth.

**GitHub's `Link` header:**
```
Link: <https://api.github.com/...?page=2>; rel="next",
      <https://api.github.com/...?page=5>; rel="last"
```

⚠️ **This project doesn't follow it** — `_delete_bot_comments` fetches `per_page=100` and stops. PRs with more than 100 review comments leave stale ones behind. A real (if minor) bug.

## C6. REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Transport | HTTP/1.1+ | HTTP POST | HTTP/2 |
| Format | JSON | JSON | Protobuf (binary) |
| Over/under-fetching | Common | Solved | N/A |
| Caching | ✅ HTTP native | ⚠️ Hard | ⚠️ Custom |
| Streaming | ⚠️ SSE | Subscriptions | ✅ Native bidirectional |
| Best for | Public APIs, CRUD | Varied client needs | Internal microservices |

**GitHub offers both REST and GraphQL.** This project uses REST because the calls are simple resource fetches with no over-fetching problem — and because the `Accept: application/vnd.github.diff` content negotiation for raw diffs is a REST-specific capability that GraphQL doesn't offer.

**Where GraphQL would genuinely help:** fetching PR metadata, files, and existing comments in **one** round trip instead of three. If I were optimising latency, that's where I'd look.

---

## C7. Interview Questions

### Q1. "Walk me through your API design."

**Best answer:**
> My service is much more an API *consumer* than a provider. I expose two endpoints — a webhook receiver and a health check — and consume about nine distinct GitHub and OpenAI operations.
>
> The webhook is `POST /webhook`. It's authenticated by HMAC signature rather than a token, because that's the webhook pattern: the provider signs the payload rather than presenting credentials. It always returns 200, even for events it ignores, because GitHub disables webhooks that persistently fail. Failures are logged, never surfaced as 5xx.
>
> One design point I'd defend: it arguably should return `202 Accepted` rather than `200 OK`, since it acknowledges receipt without having done the work. GitHub treats any 2xx the same so it's semantics rather than function, but `202` is the more honest code.
>
> On the consuming side, the piece I find most interesting is content negotiation. To fetch a diff I `GET` the pull request URL with `Accept: application/vnd.github.diff` instead of the usual JSON accept header. Same URI, different representation — that's REST content negotiation working exactly as designed, and it's why I don't need a separate diff endpoint.
>
> The gap I'd name is idempotency. GitHub sends `X-GitHub-Delivery`, a unique UUID per delivery, and retries on failure. I don't track it, so a redelivery re-runs the whole pipeline and posts duplicate comments. The fix is a Redis `SET NX EX` on the delivery ID — atomic set-if-not-exists with a 24-hour TTL, which comfortably covers GitHub's retry window.

---

### Q2. "How do you handle third-party API failures?"

**Best answer:**
> Differently at each layer, and with one genuine weakness.
>
> **GitHub calls** use `raise_for_status()`, so any 4xx or 5xx raises. That propagates to `_safe_run_pipeline`, which catches, logs, and stops. The webhook already returned 200, so GitHub sees success regardless.
>
> **OpenAI embedding calls** get one retry with a one-second sleep, then raise.
>
> **OpenAI agent calls** are wrapped in `asyncio.gather(..., return_exceptions=True)`, so a single hunk failing returns an exception object in the results list and gets skipped. Nineteen successful hunks still produce findings. That's the layer I'm happiest with.
>
> **Posting** has an explicit fallback: if GitHub returns 422 for inline comments, I catch it specifically and post a summary body comment instead. Degraded but not silent.
>
> The weakness is retry strategy. One retry at a fixed one second is barely a retry — it handles a transient blip and nothing else. There's no exponential backoff, no jitter, and no special handling for 429, which is the failure I'm most likely to actually hit given my unbounded concurrency. And there's no circuit breaker, so during an OpenAI outage every incoming PR would hammer a dead API and fail slowly.
>
> What I'd add: `tenacity` with exponential backoff and jitter, respecting the `Retry-After` header on 429s, plus a circuit breaker that trips after N consecutive failures. And most importantly, making failure *visible* — right now a fully failed review is silent. The developer sees nothing and assumes the bot found no problems, which is worse than an error.

**Why this scores well:** it distinguishes four layers, names the strongest and weakest, and identifies "silent failure" as the real problem rather than just "no retries".

---

### Q3. "Your review post is atomic — one bad comment loses all. How would you fix it?"

**Best answer:**
> Three options, and I'd pick the cheapest one first.
>
> **Validate before posting.** I already parse the diff into hunks with exact line ranges — that's how I build the agent prompts. So I have ground truth for which lines are commentable. Before posting, I cross-check every finding's `line_end` against those ranges and either drop the finding or clamp it to the nearest valid line. That's about fifteen lines of code and prevents the 422 entirely rather than recovering from it. This is clearly the right first move, and honestly it's a little embarrassing that I have the data and don't use it.
>
> **Post individually.** One API call per comment instead of one batched review. Then a bad line loses exactly one comment. The costs are more API calls, more rate-limit pressure, and losing the grouped "review" presentation on GitHub — the comments appear as separate events rather than one review.
>
> **Bisect on failure.** On a 422, split the batch in half and retry each. Logarithmic isolation of the offender. Clever, but it's added complexity for a problem that validation prevents outright, so I'd only reach for it if validation somehow proved insufficient.
>
> I'd do validation, and keep the summary fallback as a genuine last resort rather than the primary error path it currently is.

---

### Q4. "Why does `_delete_bot_comments` have a bug?"

**Best answer:** *(See B4 — the `user.type == "Bot"` story.)*

Then extend:
> The general lesson is that filtering by *category* when you mean *identity* is a whole class of bug. "Delete all bot comments" reads naturally and passes review, but "bot" is a category that includes every other tool on the PR. What I actually meant was "delete comments authored by *me*", which needs an identity check against my own bot login.
>
> The failure mode is what makes it serious: it's completely silent. No error, no log line, nothing fails. Another team's Dependabot comments just quietly stop existing, and the cause is invisible unless someone thinks to correlate it with my bot's activity. Silent destructive operations are the worst kind of bug because they don't announce themselves.

**Why this is a strong answer:** it generalises from a specific bug to a class of bug, and identifies silence as the aggravating factor.

---

## C8. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| 5xx from a webhook | Provider disables the hook | Always 2xx; log internally |
| No idempotency key | Duplicate processing on retry | Track delivery ID |
| `POST` for updates | Breaks idempotency expectations | `PUT`/`PATCH` |
| Ignoring pagination | Silent data truncation | Follow `Link` headers |
| No API version pin | Breaks on provider's schedule | `X-GitHub-Api-Version` ✅ done |
| Credentials in URLs | Leak into logs | Headers or env |
| Batching without failure isolation | One bad item loses all | Validate first, or post individually |
| Re-fetching cacheable tokens | Wasted latency | TTL cache with safety margin |
| Filtering by type not identity | Affects others' resources | Check the specific login/ID |

---

## C9. Cross-References

| Topic | File |
|---|---|
| Auth for these calls | [07-Authentication-and-Security.md](07-Authentication-and-Security.md) |
| OpenAI call details | [09-AI-System.md](09-AI-System.md) |
| Backend request handling | [05-Backend-Architecture.md](05-Backend-Architecture.md) |
| Reducing API call volume | [12-Performance-Optimization.md](12-Performance-Optimization.md) |
| Rate limits at scale | [13-Scalability.md](13-Scalability.md) |
| The bugs listed here | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [09-AI-System.md](09-AI-System.md)
