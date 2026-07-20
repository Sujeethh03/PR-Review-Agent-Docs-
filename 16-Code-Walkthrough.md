# 16 — Code Walkthrough

## Executive Summary

A line-by-line reading of every important file. **~1,900 lines across 41 Python files.** This file is the reference you use when an interviewer says "show me the code" or "what does this function do."

**Reading order for maximum understanding:**
1. `main.py` — the spine
2. `github_client.py` — auth
3. `ingestion/agent.py` → `parser.py` → `store.py` — the index
4. `specialist/graph.py` → `bug/agent.py` → `context.py` — the analysis
5. `pr_writer/agent.py` → `formatter.py` — the output

---

## 1. The Request Lifecycle

The classic layered lifecycle, mapped to what this system actually does:

| Classic layer | This system | File |
|---|---|---|
| Client | GitHub webhook delivery | — |
| Middleware | *(none — a gap)* | — |
| Authentication | HMAC-SHA256 verification | `main.py:26` |
| Validation | `.get()` defaults + guard clauses | `main.py:144` |
| Controller | `webhook()` — routes and dispatches | `main.py:114` |
| Service | `_run_pipeline()` — orchestrates | `main.py:37` |
| Repository | `store.py` — ChromaDB access | `store.py` |
| Database | ChromaDB | `/tmp/chromadb` |
| Response | `{"status": "ok"}` — sent *before* work completes | `main.py:168` |

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant C as webhook() [controller]
    participant A as verify_signature() [auth]
    participant S as _run_pipeline() [service]
    participant R as store.py [repository]
    participant D as ChromaDB

    GH->>C: POST /webhook
    C->>A: verify_signature(raw_bytes, header)
    A-->>C: True / False
    alt invalid
        C-->>GH: 401
    end
    C->>C: route by X-GitHub-Event
    C->>S: asyncio.create_task(...)
    C-->>GH: 200 OK  ⏱️ ~5ms
    Note over S: everything below is AFTER the response
    S->>R: upsert_chunks(...)
    R->>D: collection.upsert(...)
    S->>GH: POST review comments
```

**The defining characteristic:** the response is sent **before** the service layer runs. In a conventional web app the response is the *last* step. Here it's the *second*.

---

## 2. `main.py` — The Spine (173 lines)

### Module setup

```python
load_dotenv(override=True)                                    # ①
app = FastAPI()
WEBHOOK_SECRET = os.getenv("GITHUB_WEBHOOK_SECRET")
if not WEBHOOK_SECRET:
    raise RuntimeError("GITHUB_WEBHOOK_SECRET is not set")    # ②
CONFIDENCE_THRESHOLD = 0.70                                   # ③
```

**① `override=True`** — makes `.env` authoritative over existing shell variables. Without it, a shell-exported `OPENAI_API_KEY` from another project silently wins and you debug the wrong key.

**② Fail fast at import.** No secret means the process refuses to start. The alternative — starting and rejecting every webhook, or worse, accepting all of them — is far more dangerous than a crash.

**③ The line that replaced the critic agent, the database, and the dashboard.**

### `verify_signature()` — lines 26–34

```python
def verify_signature(payload_body: bytes, signature_header: str) -> bool:
    if not signature_header:
        return False
    expected_signature = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode("utf-8"), payload_body, hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(expected_signature, signature_header)
```

Covered fully in [07-Authentication-and-Security.md](07-Authentication-and-Security.md). The two things to know: **raw bytes** (parsing would break the signature) and **`compare_digest`** (constant-time, defeats byte-by-byte timing attacks).

### `_run_pipeline()` — lines 37–103

```python
async def _run_pipeline(owner, repo_name, pr_number, installation_id, head_sha) -> None:
    repo_path = None
    try:
        diff = await get_pr_diff(installation_id, owner, repo_name, pr_number)
        repo_path = await clone_repo(installation_id, owner, repo_name, head_sha)
        ingestion_result = await run_ingestion(
            diff=diff, repo_path=repo_path, owner=owner,
            repo_name=repo_name, head_sha=head_sha)
    finally:
        if repo_path:
            shutil.rmtree(repo_path, ignore_errors=True)      # ①
```

**① The `finally` block is well-placed.** It runs whether ingestion succeeded or threw, so a failed pipeline never leaves gigabytes on disk. `ignore_errors=True` means cleanup failure can't mask the original exception.

**The scoping subtlety:** `diff` and `ingestion_result` are assigned inside `try` but used after `finally`. Is that a bug? No — if any assignment throws, the exception propagates past the `finally` and the function exits. The code after is unreachable in that case. It's correct, though a reader has to think about it.

**Note the ordering choice:** the repo is deleted *before* the expensive LLM phase. Disk is held only for ingestion, not for the 30–60 seconds of agent work. That's deliberate resource hygiene.

```python
    specialist_result = await run_specialist_agents(
        diff=diff, collection_name=ingestion_result.collection_name,
        owner=owner, repo_name=repo_name)
    all_findings = specialist_result.all_findings()           # ② dedup happens here

    to_post = [f for f in all_findings if f.confidence >= CONFIDENCE_THRESHOLD]
    dropped = len(all_findings) - len(to_post)                # ③

    if to_post:                                               # ④
        await run_pr_writer(findings=to_post, ..., diff=diff)
```

**② `all_findings()` is a method, not a property** — it performs the two-pass deduplication described in [03](03-Low-Level-Design.md). Calling it looks like a simple accessor but does real work.

**③ Counting the dropped findings** is good observability instinct — you can see the filter working, not just its output.

**④ Guard against posting nothing.** `run_pr_writer` also returns early on empty, so this is belt-and-braces — but it avoids a pointless installation-token fetch.

### `_safe_run_pipeline()` — lines 106–111

```python
async def _safe_run_pipeline(*args, **kwargs) -> None:
    try:
        await _run_pipeline(*args, **kwargs)
    except Exception as e:
        print(f"Pipeline error: {e}")
```

**Why this wrapper exists:** an unhandled exception in a detached `asyncio` task doesn't crash the process — it produces a "Task exception was never retrieved" warning, often at garbage-collection time, sometimes never. This wrapper guarantees the error is at least printed.

**Weakness:** `print(e)` gives the message but not the traceback. `logger.exception()` would give both.

### `webhook()` — lines 114–168

```python
payload_body = await request.body()                          # raw bytes — required
signature_header = request.headers.get("X-Hub-Signature-256", "")
if not verify_signature(payload_body, signature_header):
    raise HTTPException(status_code=401, detail="Invalid signature")
payload = json.loads(payload_body)                            # parse AFTER verify
```

**The ordering is the security property:** nothing is parsed, logged, or acted on before authenticity is proven.

```python
if event_type in ("security_advisory", ..., "pull_request_review",
                  "pull_request_review_comment", "pull_request_review_thread"):
    return {"status": "ok", "event": event_type}
```

**The feedback-loop guard.** Without the last three, posting a comment triggers an event that triggers a review that posts a comment — infinite, and expensive. Commit `021d87e`.

```python
repo = payload.get("repository", {}).get("full_name", "unknown")
if "/" not in repo:
    raise HTTPException(status_code=400, detail=f"Unexpected repo format: {repo}")
owner, repo_name = repo.split("/", 1)                         # maxsplit=1
```

Chained `.get()` with defaults — no `KeyError` on a malformed payload. `maxsplit=1` guards against a theoretical extra slash producing a three-tuple unpacking error.

```python
if action in ("opened", "synchronize") and installation_id and head_sha:
    asyncio.create_task(_safe_run_pipeline(owner, repo_name, number,
                                            installation_id, head_sha))
```

**Three conditions:** the right action, plus the two fields the pipeline can't run without.

⚠️ **The GC bug:** the return value is discarded. Python's docs state the event loop keeps only weak references to tasks, so a running task can be collected. Fix:
```python
_background_tasks: set[asyncio.Task] = set()
task = asyncio.create_task(...)
_background_tasks.add(task)
task.add_done_callback(_background_tasks.discard)
```

---

## 3. `github_client.py` — Authentication (57 lines)

```python
def _generate_jwt() -> str:
    raw = os.getenv("GITHUB_PRIVATE_KEY")
    if raw:
        private_key = base64.b64decode(raw).decode("utf-8")   # ① production
    else:
        with open(os.getenv("GITHUB_PRIVATE_KEY_PATH")) as f:
            private_key = f.read()                             # ② local

    now = int(time.time())
    payload = {"iat": now - 60, "exp": now + 540, "iss": app_id}   # ③ ④
    return jwt.encode(payload, private_key, algorithm="RS256")     # ⑤
```

**① Base64** because PEM files are multi-line and env vars mangle newlines differently on every platform.
**③ `iat: now - 60`** — backdated for clock skew. Without it, a server clock a few seconds fast produces a token GitHub sees as "issued in the future."
**④ `exp: now + 540`** — 9 minutes against GitHub's 10-minute maximum.
**⑤ RS256** — asymmetric, so GitHub can *verify* without being able to *forge*.

```python
async def get_pr_diff(installation_id, owner, repo, pr_number) -> str:
    token = await get_installation_token(installation_id)
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{GITHUB_API}/repos/{owner}/{repo}/pulls/{pr_number}",
            headers={"Authorization": f"Bearer {token}",
                     "Accept": "application/vnd.github.diff"})   # ← content negotiation
        response.raise_for_status()
        return response.text
```

**The `Accept` header is the whole trick** — same URL returns JSON metadata or a unified diff depending on what you ask for. Textbook REST content negotiation.

⚠️ **Two inefficiencies:** `get_installation_token` is called three times per pipeline (tokens last an hour — cacheable), and a new `httpx.AsyncClient` is created per call rather than reusing a pooled one.

---

## 4. `repo_manager.py` — Cloning (27 lines)

```python
async def clone_repo(installation_id, owner, repo, head_sha) -> str:
    clone_path = os.path.join(CLONE_BASE, f"{owner}-{repo}-{head_sha}")
    if os.path.exists(clone_path):
        return clone_path                                      # ① idempotent

    os.makedirs(CLONE_BASE, exist_ok=True)
    prefix = f"{owner}-{repo}-"
    for entry in os.listdir(CLONE_BASE):
        if entry.startswith(prefix):
            shutil.rmtree(os.path.join(CLONE_BASE, entry), ignore_errors=True)   # ②

    token = await get_installation_token(installation_id)
    clone_url = f"https://x-access-token:{token}@github.com/{owner}/{repo}.git"  # ③
    git.Repo.clone_from(clone_url, clone_path, depth=1)        # ④
    return clone_path
```

**① Idempotency** — same SHA, reuse the existing clone.
**② Purge older commits of the same repo** before cloning a new one, bounding disk usage.
**③ Token in the URL** ⚠️ — risks leaking into logs via exception messages.
**④ `depth=1`** — shallow clone. Only the tip tree, not the history. Seconds instead of minutes on large repos.

⚠️ **`clone_from` is a blocking call inside an `async def`.** Same class of bug as the original `OpenAI` client issue — it should go through `asyncio.to_thread`. It's less severe because clones are a few seconds and there's typically only one, but it's the same category.

---

## 5. Ingestion Package

### `agent.py` — the orchestrator

```python
async def run_ingestion(...) -> IngestionResult:
    return await asyncio.to_thread(_run_ingestion_sync, ...)   # ← honest offload
```

**The correct handling of a blocking dependency.** ChromaDB's client is synchronous; rather than wrapping it in `async def` and lying to the event loop, the whole module is written as a sync function and explicitly pushed to a worker thread.

```python
if collection_has_data(collection):
    mode = "incremental"
    modified_paths, deleted_paths = parse_diff_file_paths(diff)
    if deleted_paths:
        delete_chunks_for_files(collection, deleted_paths)     # ⚠️ only DELETED
    files_to_ingest = [os.path.join(repo_path, p) for p in modified_paths
                       if os.path.isfile(os.path.join(repo_path, p))]
else:
    mode = "full"
    files_to_ingest = walk_repo(repo_path, config)
```

⚠️ **The stale-chunk bug lives here.** Only *deleted* files are purged. *Modified* files are upserted over — but chunk IDs encode line numbers, so when line ranges shift, the old IDs are orphaned forever. The fix is one line: also `delete_chunks_for_files(collection, modified_paths)` before re-ingesting. See [03-Low-Level-Design.md](03-Low-Level-Design.md#7-algorithm-deep-dive-4).

**`if os.path.isfile(...)`** guards against a diff listing a file that no longer exists on disk.

### `walker.py` — file discovery

```python
SKIP_FILENAMES  = {".env", ".env.local", ".env.production", ".env.development"}
SKIP_EXTENSIONS = {".pem", ".key", ".lock", ".pyc", ".map"}

for dirpath, dirnames, filenames in os.walk(repo_path):
    dirnames[:] = [d for d in dirnames if not should_skip_dir(d, config.skip_dirs)]  # ①
```

**① `dirnames[:] = ...` is the important line.** Mutating the list **in place** tells `os.walk` not to descend into those directories. Reassigning (`dirnames = [...]`) would rebind a local name and `os.walk` would still walk `node_modules`. A subtle, well-known Python gotcha, handled correctly.

**The skip list is security-critical.** Ingestion sends file contents to OpenAI. Without this, a user's committed `.env` — with their database password — gets embedded and stored in a vector database.

```python
def is_binary(file_path: str, probe_bytes: int = 8_192) -> bool:
    with open(file_path, "rb") as f:
        return b"\x00" in f.read(probe_bytes)
```

**Null-byte heuristic** — imperfect (UTF-16 text contains nulls; some binaries don't have them in the first 8 KB) but fast and good enough. Reading only the first 8 KB avoids loading a 100 MB binary to classify it.

### `parser.py` — chunking

Covered in depth in [03-Low-Level-Design.md](03-Low-Level-Design.md#5-algorithm-deep-dive-2). Key lines:

```python
def visit(node):
    chunk_type = NODE_TYPE_MAP.get(node.type)
    if chunk_type:
        start_line = node.start_point[0] + 1      # tree-sitter is 0-indexed
        end_line   = node.end_point[0] + 1
        chunks.append(Chunk(...))
    for child in node.children:
        visit(child)                               # ← recurses ALWAYS
```

The unconditional recursion produces overlapping class-and-method chunks — deliberate, for multi-granularity retrieval.

```python
except Exception:
    _language_cache[language] = None               # ← negative caching
    return None
```

Caching the *failure* prevents retrying an expensive failed import for every file of an unsupported language.

### `embedder.py`

```python
for i in range(0, len(chunks), batch_size):        # batches of 100
    for attempt in range(2):                        # ⚠️ only 2 attempts
        try:
            response = client.embeddings.create(input=texts, model=model)
            results.extend(zip(batch, vectors))     # ⚠️ relies on response order
            break
        except Exception:                            # ⚠️ retries non-transient errors
            if attempt == 1: raise
            time.sleep(1)                            # ⚠️ no exponential backoff
```

Four weaknesses flagged. `zip(batch, vectors)` assumes OpenAI returns embeddings in input order — it does, and there's an `index` field for safety, but sorting by it would be more defensive.

### `store.py` — the repository

```python
def _chunk_id(chunk: Chunk) -> str:
    key = f"{chunk.file}:{chunk.start_line}:{chunk.end_line}"
    return hashlib.sha256(key.encode()).hexdigest()
```

Content-addressed, deterministic → idempotent upserts. Also the source of the stale-chunk bug (line numbers in the key).

```python
def delete_chunks_for_files(collection, file_rel_paths):
    for file_path in file_rel_paths:
        collection.delete(where={"file": file_path})    # ⚠️ N+1
```

**A real N+1** — should be `where={"file": {"$in": file_rel_paths}}` in one call.

---

## 6. Specialist Package

### `graph.py` — 35 lines defining the topology

```python
graph.add_edge("__start__", "bug")
graph.add_edge("__start__", "security")
graph.add_edge("__start__", "pattern")
graph.add_edge("bug", END)
graph.add_edge("security", END)
graph.add_edge("pattern", END)
```

Three edges from `__start__` = concurrent execution. Each node writes a **different** state key (`bug_output`, `security_output`, `pattern_output`), so there's no write conflict and no reducer needed.

### `diff_utils.py` — `parse_diff_hunks()`

The most correctness-critical pure function in the codebase. Full walkthrough in [03-Low-Level-Design.md](03-Low-Level-Design.md#4-algorithm-deep-dive-1). The invariant to remember:

```python
if line.startswith("+"):   current_new_line += 1
elif line.startswith("-"): pass          # deleted lines don't exist in the new file
else:                      current_new_line += 1
```

### `context.py` — RAG retrieval

```python
if (meta.get("file") == hunk.file
    and meta.get("start_line", 0) <= hunk.line_end
    and meta.get("end_line", 0) >= hunk.line_start):
    continue        # skip — this chunk IS the code under review
```

**The single most important line in the RAG layer.** Without it the top hit is the pre-change version of the same function, and the pattern agent has nothing to compare against. Story in [14-Project-Challenges.md](14-Project-Challenges.md#challenge-6).

⚠️ `collection.query()` is a blocking call inside an `async` function — should be offloaded.

### `bug/agent.py` — representative of all three

```python
async def _process_hunk(hunk, collection_name) -> list[Finding]:
    context_chunks = await get_context_for_hunk(hunk, collection_name, n_results=5)
    user_prompt = USER_PROMPT_TEMPLATE.format(...)
    response = await _get_client().chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "system", "content": SYSTEM_PROMPT},
                  {"role": "user",   "content": user_prompt}],
        tools=[{"type": "function", "function": {
            "name": "report_findings", "parameters": FINDINGS_SCHEMA}}],
        tool_choice={"type": "function",
                     "function": {"name": "report_findings"}},   # ← forced
        temperature=0)
    raw = json.loads(response.choices[0].message.tool_calls[0].function.arguments)
    return [Finding(**item, agent="bug")
            for item in raw.get("findings", [])
            if item.get("confidence", 0) >= 0.6]                 # ← agent-level floor
```

```python
async def run_bug_agent(state: AgentState) -> dict:
    try:
        hunks = parse_diff_hunks(state["diff"])
        results = await asyncio.gather(
            *[_process_hunk(h, state["collection_name"]) for h in hunks],
            return_exceptions=True)                              # ← failure isolation
        all_findings = [f for r in results if isinstance(r, list) for f in r]
        return {"bug_output": AgentOutput(agent="bug",
                                          findings=_deduplicate(all_findings))}
    except Exception as e:
        return {"bug_output": AgentOutput(agent="bug", findings=[], error=str(e))}
```

**`return_exceptions=True`** is the key primitive — one failed hunk returns an exception object in the results list instead of destroying the whole gather.

⚠️ **Exceptions are silently dropped** by `isinstance(r, list)`. They should at least be counted and logged.

### `security/agent.py` — the one meaningful difference

```python
def _enforce_secret_rules(findings: list[Finding]) -> list[Finding]:
    return [f.model_copy(update={"severity": "high", "confidence": 1.0})
            if f.category == "hardcoded_secret" else f for f in findings]
```

**The best AI design decision in the project.** The prompt already asks for this; the code enforces it. **Prompts express intent; code enforces invariants.**

`model_copy` returns a new object rather than mutating — traceable and concurrency-safe.

---

## 7. PR Writer Package

### `agent.py`

```python
async def _delete_bot_comments(headers, owner, repo_name, pr_number):
    response = await client.get(f".../pulls/{pr_number}/comments",
                                params={"per_page": 100})        # ⚠️ no pagination
    tasks = [client.delete(f".../pulls/comments/{c['id']}", headers=headers)
             for c in response.json()
             if c.get("user", {}).get("type") == "Bot"]           # 🔴 THE BUG
    await asyncio.gather(*tasks, return_exceptions=True)
```

🔴 **`user.type == "Bot"` matches every GitHub App** — Dependabot, Codecov, Renovate. This silently deletes other tools' comments on every push. Should filter on the specific bot login.

```python
    comment = {"path": f.file, "line": f.line_end, "side": "RIGHT", "body": ...}
    if f.line_start != f.line_end:
        comment["start_line"] = f.line_start
        comment["start_side"] = "RIGHT"
```

**Conditional multi-line range** — GitHub rejects `start_line` equal to `line`, so single-line findings must omit it.

```python
        if response.status_code == 422:
            return False        # → fall back to a summary body comment
```

⚠️ **GitHub validates reviews atomically** — one bad line number rejects all comments. See [14](14-Project-Challenges.md#challenge-7).

### `suggester.py`

```python
if not suggested or suggested == flagged_code.strip():
    return None                                    # ① no-op fix
if len(suggested) > len(flagged_code) * 4:
    return None                                    # ② runaway output
```

**Two sanity checks on LLM output.** ① If the "fix" is identical to the input, it's not a fix. ② If it's 4× longer, the model has probably started explaining rather than fixing. Cheap heuristics guarding against a common LLM failure mode — good instinct.

### `formatter.py` and `education.py`

```python
_SEVERITY_LABEL = {"high": "🚨 HIGH", "medium": "⚠️ MEDIUM", "low": "💡 LOW"}
```

Colour *and* text — accessible, not colour-alone.

```python
def get_educational_context(finding: Finding) -> str:
    entry = _OWASP.get(finding.category)
    if entry:
        return f"### Why this matters\n{entry['impact']}\n\n📚 **Reference:** [{entry['ref']}]({entry['url']})"
    fallback = _SEVERITY_FALLBACK.get(finding.severity, "")
    return f"### Why this matters\n{fallback}" if fallback else ""
```

**Lookup with graceful fallback.** 15 curated OWASP categories with written impact statements; unknown categories degrade to a severity-based message rather than showing nothing. **This is why `category` is `str` and not `Literal`** — it's an open set by design, and the fallback handles the tail.

---

## 8. Cross-Cutting Patterns

### The singleton, repeated in five modules

```python
_client: AsyncOpenAI | None = None
def _get_client() -> AsyncOpenAI:
    global _client
    if _client is None:
        _client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    return _client
```

Lazy — module import stays cheap; the client is built on first use. **The cost is testability**: mocking requires monkey-patching module globals.

### Graceful degradation, at four levels

| Level | Code | Effect |
|---|---|---|
| Hunk | `return_exceptions=True` | One hunk fails, others proceed |
| Agent | `except → AgentOutput(error=...)` | One agent fails, two succeed |
| Retrieval | `except → return []` | No index → review without context |
| Posting | `422 → body comment` | Inline fails → summary |
| Pipeline | `_safe_run_pipeline` | Log; webhook already 200'd |

---

## 9. The Complete Bug Register

Every defect found auditing this codebase, with location and fix:

| # | Bug | Location | Severity | Fix |
|---|---|---|---|---|
| 1 | Deletes **all** bots' comments | `pr_writer/agent.py:30` | 🔴 High | Filter on bot login, not `type` |
| 2 | Stale chunks accumulate | `ingestion/agent.py:51` | 🔴 High | Purge modified files before upsert |
| 3 | LLM line numbers unvalidated | `pr_writer/agent.py:79` | 🔴 High | Cross-check against `parse_diff_hunks` |
| 4 | Unbounded concurrency | All agents | 🔴 High | `asyncio.Semaphore` |
| 5 | `create_task` reference discarded | `main.py:161` | 🟠 Med | Module-level set + done-callback |
| 6 | Dedup on exact `line_start` | `findings.py:33` | 🟠 Med | Use interval overlap (already exists in bug agent) |
| 7 | N+1 deletes | `store.py:77` | 🟠 Med | `$in` operator |
| 8 | Token fetched 3× per pipeline | `github_client.py` | 🟠 Med | TTL cache |
| 9 | Hunk embedded 3× | `context.py` | 🟠 Med | Precompute and share |
| 10 | No pagination on comments | `pr_writer/agent.py:18` | 🟡 Low | Follow `Link` headers |
| 11 | Blocking calls in async | `context.py:44`, `repo_manager.py:25` | 🟡 Low | `asyncio.to_thread` |
| 12 | `confidence` unbounded | `findings.py:14` | 🟡 Low | `Field(ge=0.0, le=1.0)` |
| 13 | Exceptions silently dropped | All agents | 🟡 Low | Log the exception objects |
| 14 | Token in clone URL | `repo_manager.py:23` | 🟡 Low | Env-based credentials |
| 15 | Duplicated `FINDINGS_SCHEMA` | 3 agent files | 🟡 Low | Base class |
| 16 | Dead `alembic/` directory | `backend/alembic/` | 🟢 Cosmetic | Delete |

**Being able to produce this table from memory is a genuine differentiator.** Most candidates cannot name a single bug in their own project. Naming sixteen, ranked, with fixes, demonstrates that you've actually read your own code critically.

---

## 10. Interview Questions

### Q1. "Walk me through your most important file."

**Best answer:**
> `main.py`, 173 lines, and the structure tells you what the system is.
>
> Three things happen at module level: `load_dotenv` with `override=True` so `.env` beats any shell variable — that saved me an hour of debugging the wrong API key once. Then a fail-fast check that refuses to start without a webhook secret, because a service that starts and silently accepts unsigned webhooks is far more dangerous than one that crashes. And `CONFIDENCE_THRESHOLD = 0.70`, which is the one line that replaced a critic agent, a PostgreSQL database, and a Next.js dashboard.
>
> Then the webhook handler. Read raw bytes — deliberately not parsed JSON, because HMAC has to be over exactly what GitHub sent. Verify with `compare_digest`, constant-time. Only then parse and route.
>
> Routing has an explicit ignore list for events my own bot generates, because without it posting a comment triggers an event that triggers a review that posts a comment — I hit that infinite loop in production.
>
> If it's a `pull_request` event with action `opened` or `synchronize`, and I have both an installation ID and a head SHA, I spawn a background task and return 200. That's about five milliseconds, and it's the most important thing in the file — GitHub times out around ten seconds and my pipeline takes ninety.
>
> The pipeline itself is a sequence: fetch diff, clone, ingest, three agents, dedupe, filter, post. Two details I'd point out. The clone cleanup is in a `finally`, and it runs *before* the LLM phase — so I hold disk for the ten seconds of ingestion, not the sixty seconds of agent work. And the whole thing is wrapped in `_safe_run_pipeline`, because an unhandled exception in a detached asyncio task produces a warning at garbage-collection time that's easy to never see.
>
> One bug in that file I'd fix: I discard the return value of `create_task`. Python's docs say the event loop only holds weak references, so a running task can be garbage collected mid-execution. Three lines fix it.

---

### Q2. "What's the worst code in your project?"

**Best answer:**
> The three specialist agents. They're about ninety percent identical — the `FINDINGS_SCHEMA` constant is byte-for-byte duplicated across all three files, twenty-seven lines each.
>
> Each agent has exactly one genuinely distinct behaviour. The bug agent does interval-overlap deduplication. The security agent forces hardcoded secrets to high severity. The pattern agent skips hunks with no retrieved context. Everything else — parse hunks, gather over them, forced tool call, filter at 0.6, wrap in try/except — is identical.
>
> The right shape is a `BaseSpecialistAgent` holding the skeleton, with subclasses supplying a system prompt, an agent name, and an optional `post_process` hook. That's about two hundred lines deleted.
>
> Why it exists: I built the agents in sequence, and copy-paste was faster than designing an abstraction before I knew what would vary. I'd defend that as a way to *start* — premature abstraction is its own problem, and you can't factor out variation you haven't seen yet. But I've now seen it. So leaving it duplicated has crossed from pragmatism into technical debt.
>
> There's a concrete cost, not just an aesthetic one: only the bug agent deduplicates internally. Security and pattern can emit overlapping findings from adjacent hunks, and nobody noticed because the logic lives in one file instead of a shared base. Duplication doesn't just cost lines — it lets behaviour drift.

---

### Q3. "Show me a bug in your own code."

**Best answer:**
> I'll give you the one with the nastiest failure mode.
>
> My stale-comment cleanup deletes old bot comments before posting new ones, so re-reviewing a PR doesn't pile up duplicates. The filter is `user.type == "Bot"`.
>
> But `type == "Bot"` matches *every* GitHub App — Dependabot, Codecov, Renovate, any other review tool. So on a repo with other automation, every push silently deletes their comments too.
>
> What makes it serious isn't the severity, it's the silence. Nothing errors. Nothing logs. Another team's Dependabot alerts just quietly stop existing, and the cause is completely invisible unless someone thinks to correlate it with my bot's activity. Silent destructive operations are the worst kind of bug.
>
> The fix is one line — filter on the bot's login rather than its type.
>
> The general lesson is what I find more interesting: I filtered by **category** when I meant **identity**. "Delete all bot comments" reads naturally and would pass code review. What I actually meant was "delete comments authored by me."
>
> And I made the same mistake twice. My webhook feedback-loop guard ignores `pull_request_review` events by *type*, which also means I ignore review comments from humans — so I can't see when a developer replies to one of my findings asking a question. Same conceptual error: category where I meant identity. Spotting the pattern across two unrelated bugs was more valuable than either individual fix.

---

## 11. Cross-References

| Topic | File |
|---|---|
| Algorithms in detail | [03-Low-Level-Design.md](03-Low-Level-Design.md) |
| FastAPI patterns | [05-Backend-Architecture.md](05-Backend-Architecture.md) |
| Auth code | [07-Authentication-and-Security.md](07-Authentication-and-Security.md) |
| AI/agent code | [09-AI-System.md](09-AI-System.md) |
| The stories behind the bugs | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Prioritised fixes | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [17-Interview-Questions-Part-1.md](17-Interview-Questions-Part-1.md)
