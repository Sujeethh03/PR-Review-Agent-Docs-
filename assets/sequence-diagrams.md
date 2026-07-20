# Sequence Diagrams — Detailed Flows

Detailed sequence diagrams for every major flow, each with a walkthrough of the critical steps.

---

## 1. Complete Happy Path

The full pipeline, end to end.

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub
    participant FA as FastAPI
    participant BG as Background Task
    participant GC as GitHubClient
    participant RM as RepoManager
    participant IN as Ingestion
    participant CD as ChromaDB
    participant OA as OpenAI
    participant LG as LangGraph
    participant PW as PRWriter

    GH->>FA: POST /webhook (pull_request.opened)
    FA->>FA: payload_body = await request.body()
    Note right of FA: raw bytes — parsing<br/>would break the HMAC
    FA->>FA: hmac.compare_digest()
    FA->>FA: route on X-GitHub-Event
    FA->>BG: asyncio.create_task(_safe_run_pipeline)
    FA-->>GH: 200 OK ⏱️ ~5ms

    rect rgb(240, 248, 255)
    Note over BG,OA: STAGE 1 — Fetch
    BG->>GC: get_pr_diff()
    GC->>GC: _generate_jwt() RS256, 9 min
    GC->>GH: POST /app/installations/{id}/access_tokens
    GH-->>GC: installation token (1 hr)
    GC->>GH: GET /pulls/{n}<br/>Accept: application/vnd.github.diff
    GH-->>GC: unified diff text
    end

    rect rgb(255, 250, 240)
    Note over BG,RM: STAGE 2 — Clone
    BG->>RM: clone_repo()
    RM->>RM: purge older clones of this repo
    RM->>GH: git clone --depth 1 (token in URL)
    GH-->>RM: /tmp/repos/{owner}-{repo}-{sha}
    end

    rect rgb(245, 255, 245)
    Note over BG,CD: STAGE 3 — Index
    BG->>IN: run_ingestion()
    Note right of IN: asyncio.to_thread —<br/>ChromaDB client is blocking
    IN->>CD: get_or_create_collection("owner__repo")
    alt Collection empty
        IN->>IN: walk_repo() — every file
        Note right of IN: skip .env, .pem, binaries, >512KB
    else Collection has data
        IN->>IN: parse_diff_file_paths() — changed only
        IN->>CD: delete chunks of deleted files
    end
    loop per file
        IN->>IN: tree-sitter AST → chunks
        IN->>OA: embeddings.create(batch of 100)
        OA-->>IN: 1536-dim vectors
        IN->>CD: upsert(ids, docs, embeddings, meta)
    end
    IN-->>BG: IngestionResult
    BG->>BG: shutil.rmtree(repo) 🗑️
    Note right of BG: finally block — always runs,<br/>and BEFORE the expensive LLM phase
    end

    rect rgb(255, 245, 250)
    Note over BG,OA: STAGE 4 — Analyse
    BG->>LG: run_specialist_agents()
    par Bug Agent
        LG->>OA: embed hunk
        LG->>CD: query top-5
        LG->>OA: GPT-4o forced tool call
    and Security Agent
        LG->>OA: embed hunk
        LG->>CD: query top-5
        LG->>OA: GPT-4o forced tool call
    and Pattern Agent
        LG->>OA: embed hunk
        LG->>CD: query top-10
        alt no context returned
            LG->>LG: skip hunk entirely
        else
            LG->>OA: GPT-4o forced tool call
        end
    end
    LG-->>BG: SpecialistResult
    end

    rect rgb(250, 245, 255)
    Note over BG,GH: STAGE 5 — Publish
    BG->>BG: all_findings() — 2-pass dedup
    BG->>BG: filter confidence >= 0.70
    BG->>PW: run_pr_writer()
    PW->>GH: GET existing review comments
    PW->>GH: DELETE stale bot comments (parallel)
    PW->>OA: generate_suggestion() × N (parallel)
    PW->>GH: POST /pulls/{n}/reviews (batched)
    GH-->>PW: 201 Created
    end

    GH-->>GH: 🔔 notify developer
```

### Critical steps explained

| Step | Why it matters |
|---|---|
| `request.body()` before parsing | HMAC must cover the exact bytes GitHub sent |
| 200 OK before any work | GitHub times out at ~10s; pipeline takes ~90s |
| `Accept: vnd.github.diff` | Content negotiation — same URI, different representation |
| `--depth 1` | Only current file contents matter; seconds vs minutes |
| `asyncio.to_thread` | ChromaDB is blocking; offload instead of blocking the loop |
| `rmtree` before the LLM phase | Disk held for ~10s of ingestion, not ~60s of agent work |
| Pattern agent skips empty context | No evidence, no judgement — reduces false positives |
| Filter *before* suggestions | Don't pay for suggestions on findings you'll drop |

---

## 2. Authentication — Both Directions

```mermaid
sequenceDiagram
    autonumber
    participant App as Our Service
    participant PK as Private Key (PEM)
    participant GH as GitHub

    rect rgb(255, 245, 245)
    Note over App,GH: INBOUND — is this really GitHub?
    GH->>App: POST /webhook<br/>X-Hub-Signature-256: sha256=abc...
    App->>App: body = await request.body()
    App->>App: expected = "sha256=" + HMAC-SHA256(secret, body)
    App->>App: hmac.compare_digest(expected, received)
    Note right of App: constant-time —<br/>== would leak byte position<br/>via response latency
    alt match
        App-->>GH: 200 OK
    else mismatch
        App-->>GH: 401 Unauthorized
    end
    end

    rect rgb(245, 245, 255)
    Note over App,GH: OUTBOUND — are we really the App?
    App->>PK: read key
    alt production
        PK-->>App: base64 env var → decode
        Note right of App: PEM is multi-line;<br/>env vars mangle newlines
    else local
        PK-->>App: read .pem file
    end
    App->>App: payload = {iat: now-60, exp: now+540, iss: APP_ID}
    Note right of App: iat backdated for clock skew<br/>exp = 9 min (max 10)
    App->>App: jwt.encode(payload, key, algorithm="RS256")
    Note right of App: RS256 asymmetric — GitHub verifies<br/>with the public key, cannot forge
    App->>GH: POST /app/installations/{id}/access_tokens<br/>Authorization: Bearer <JWT>
    GH->>GH: verify signature with stored public key
    GH-->>App: {token: "ghs_...", expires_at, permissions}
    App->>GH: any API call<br/>Authorization: Bearer <installation token>
    end
```

**The key distinction:** the JWT proves **identity** ("I am App #123456"); the installation token grants **authorisation** ("I may act on this installation's repos"). An App installed on 500 orgs gets 500 independently-scoped tokens, so a leak has a blast radius of one installation for one hour.

---

## 3. RAG Retrieval for a Single Hunk

```mermaid
sequenceDiagram
    autonumber
    participant AG as Agent
    participant CTX as context.py
    participant OA as OpenAI
    participant CD as ChromaDB

    AG->>CTX: get_context_for_hunk(hunk, collection, n=5)
    CTX->>CTX: if not hunk.added_code.strip(): return []
    Note right of CTX: deletion-only hunk —<br/>nothing to query with

    CTX->>CD: get_collection(name)
    alt collection missing
        CD-->>CTX: raises
        CTX-->>AG: [] — degrade, don't fail
        Note right of CTX: agents still work,<br/>just without codebase context
    end

    CTX->>OA: embeddings.create([hunk.added_code])
    Note right of OA: SAME model as indexing —<br/>mandatory, vectors from different<br/>models are incomparable
    OA-->>CTX: 1536-dim query vector

    CTX->>CD: query(embeddings, n_results, include=[docs, metadatas])
    Note right of CD: excludes embeddings —<br/>1536 floats × 5 unused
    CD-->>CTX: top-N by cosine similarity

    loop each result
        CTX->>CTX: same file AND overlapping lines?
        alt yes
            CTX->>CTX: skip — this IS the code under review
        else no
            CTX->>CTX: keep
        end
    end
    CTX-->>AG: filtered context chunks
    AG->>AG: format_context_chunks() → prompt
```

**The self-match filter is the most important step.** The index holds the *pre-change* version of the file being reviewed, and semantically that's the nearest neighbour of the modified code. Without the filter, the agent compares the code to itself.

⚠️ **Refinement:** the filter runs after retrieval, so 3 self-matches out of 5 leaves only 2 chunks. Over-fetching (n=10, filter, take 5) would be more robust.

---

## 4. Error Handling and Degradation

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub
    participant FA as FastAPI
    participant BG as Pipeline
    participant AG as Agents
    participant PW as PRWriter

    GH->>FA: POST /webhook
    FA-->>GH: 200 OK

    FA->>BG: _safe_run_pipeline()

    rect rgb(255, 250, 240)
    Note over BG: LEVEL 4 — resource cleanup
    BG->>BG: try: clone + ingest
    BG->>BG: finally: rmtree(ignore_errors=True)
    Note right of BG: runs whether or not<br/>ingestion succeeded
    end

    rect rgb(255, 245, 245)
    Note over AG: LEVEL 3 — per-hunk isolation
    BG->>AG: run_bug_agent()
    AG->>AG: gather(*hunks, return_exceptions=True)
    Note right of AG: one hunk failing returns an<br/>exception OBJECT, not a raise
    AG->>AG: [f for r in results if isinstance(r, list)]
    Note right of AG: ⚠️ exceptions silently dropped —<br/>should be logged
    end

    rect rgb(245, 245, 255)
    Note over AG: LEVEL 2 — per-agent isolation
    alt agent throws
        AG-->>BG: AgentOutput(findings=[], error="...")
        Note right of AG: other 2 agents unaffected
    end
    end

    rect rgb(245, 255, 245)
    Note over PW,GH: LEVEL 1 — posting fallback
    BG->>PW: run_pr_writer()
    PW->>GH: POST /pulls/{n}/reviews
    alt 422 Unprocessable
        GH-->>PW: 422
        Note right of PW: GitHub validates the batch ATOMICALLY —<br/>one bad line number rejects ALL comments
        PW->>GH: POST /issues/{n}/comments (summary)
        Note right of PW: degraded but not silent
    else 201
        GH-->>PW: 201 Created
    end
    end

    rect rgb(250, 240, 240)
    Note over BG: LEVEL 0 — pipeline catch-all
    alt anything else raises
        BG->>BG: print("Pipeline error: ...")
        Note right of BG: webhook already 200'd;<br/>never 5xx to GitHub
    end
    end
```

**The five-level philosophy:** partial output beats no output at every level. But note the weakness — a *fully* failed review is completely silent. The developer sees nothing and concludes the bot found no problems, which is worse than an error message.

---

## 5. The Infinite Feedback Loop (the bug, and the fix)

```mermaid
sequenceDiagram
    autonumber
    participant BOT as PR Writer
    participant GH as GitHub
    participant FA as FastAPI
    participant PIPE as Pipeline

    rect rgb(255, 235, 235)
    Note over BOT,PIPE: ❌ BEFORE — infinite loop
    BOT->>GH: POST review comment
    GH->>FA: webhook: pull_request_review_comment
    FA->>PIPE: spawn pipeline 💸
    PIPE->>BOT: post findings
    BOT->>GH: POST review comment
    GH->>FA: webhook: pull_request_review_comment
    FA->>PIPE: spawn pipeline 💸
    Note over BOT,PIPE: ∞ — each cycle is a full<br/>clone + embed + 3 agents
    end

    rect rgb(235, 255, 235)
    Note over BOT,PIPE: ✅ AFTER — ignore list
    BOT->>GH: POST review comment
    GH->>FA: webhook: pull_request_review_comment
    FA->>FA: event_type in IGNORE_LIST?
    FA-->>GH: 200 OK — no pipeline spawned
    end
```

**The current fix is blunt:** it ignores the event *type*, which also ignores review comments from **humans**. So the bot can't see when a developer replies to a finding with a question.

**The better fix filters on actor identity:**
```python
if payload.get("sender", {}).get("login") == f"{APP_NAME}[bot]":
    return {"status": "ok", "event": "self-generated"}
```

**Note this is the same conceptual error** as `_delete_bot_comments` filtering on `user.type == "Bot"` — **filtering by category when identity was meant.**

---

## 6. Incremental vs Full Ingestion

```mermaid
sequenceDiagram
    autonumber
    participant IN as Ingestion
    participant CD as ChromaDB
    participant FS as Filesystem
    participant OA as OpenAI

    IN->>CD: get_or_create_collection("owner__repo")
    IN->>CD: collection.count() > 0?

    alt FIRST PR — full ingest
        CD-->>IN: count == 0
        IN->>FS: walk_repo() — every file
        Note right of FS: skip .env, .pem, .key,<br/>binaries, >512KB, node_modules
        FS-->>IN: ~500 files
        loop each file
            IN->>IN: tree-sitter → chunks
            IN->>OA: embed (batch 100)
            IN->>CD: upsert
        end
        Note over IN,OA: minutes, dollars
    else SUBSEQUENT PR — incremental
        CD-->>IN: count > 0
        IN->>IN: parse_diff_file_paths(diff)
        IN->>CD: delete chunks of DELETED files
        Note right of CD: ⚠️ MODIFIED files are NOT purged —<br/>stale chunks accumulate forever
        IN->>FS: only changed files
        FS-->>IN: ~5 files
        loop each file
            IN->>OA: embed
            IN->>CD: upsert (new IDs if lines shifted)
        end
        Note over IN,OA: seconds, cents
    end
```

**Incremental ingestion is the single biggest cost optimisation in the system** — 5 files instead of 500 per PR.

**Two problems:**
1. **The stale-chunk bug** — modified files are upserted without purging, and since chunk IDs encode line numbers, shifted code leaves orphaned vectors forever.
2. **Ephemeral storage** — Railway wipes `/tmp/chromadb` on every deploy, so the collection empties and the next PR silently pays for a full re-ingest.

---

## 7. Scaled Architecture — Queue Flow

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub
    participant W as Receiver
    participant R as Redis
    participant Q as Queue
    participant WK as Worker
    participant DLQ as Dead Letter Queue

    GH->>W: POST /webhook
    W->>W: verify HMAC
    W->>R: SET NX delivery_id EX 86400
    alt already seen
        R-->>W: nil
        W-->>GH: 200 "duplicate"
        Note right of W: idempotency — GitHub<br/>retries and redelivers
    end
    W->>Q: enqueue {owner, repo, pr, sha, installation_id}
    Note right of Q: 5 JSON-serialisable scalars —<br/>why the migration is one line
    W-->>GH: 200 OK ⏱️ ~5ms

    Q->>WK: dequeue (visibility timeout)
    WK->>WK: run pipeline

    alt success
        WK->>Q: ack (delete message)
    else transient failure
        WK->>Q: nack
        Q->>Q: retry with exponential backoff
        Note right of Q: attempt 1: 1s<br/>attempt 2: 4s<br/>attempt 3: 16s
    else exhausted retries
        Q->>DLQ: move message
        Note right of DLQ: manual inspection<br/>and replay
    end
```

**Four properties the queue buys:** durability (survives restarts), retries with a DLQ, **backpressure** (a bounded worker pool caps concurrency), and independent scaling.

**The one that matters most here is backpressure** — the current unbounded fan-out that fires 150 concurrent LLM calls is a direct consequence of having nothing to bound it.

---

## 8. Async: The False Parallelism Bug

```mermaid
sequenceDiagram
    participant EL as Event Loop
    participant B as Bug Agent
    participant S as Security Agent
    participant P as Pattern Agent

    Note over EL,P: ❌ BEFORE — synchronous OpenAI client
    EL->>B: schedule
    B->>B: _client.chat.completions.create() 🔒
    Note over EL,P: EVENT LOOP BLOCKED — 15s<br/>nothing else can execute
    B-->>EL: return
    EL->>S: schedule
    S->>S: _client.chat.completions.create() 🔒
    Note over EL,P: BLOCKED — 15s
    S-->>EL: return
    EL->>P: schedule
    P->>P: _client.chat.completions.create() 🔒
    Note over EL,P: BLOCKED — 15s
    P-->>EL: return
    Note over EL: TOTAL: 45 seconds
```

```mermaid
sequenceDiagram
    participant EL as Event Loop
    participant B as Bug Agent
    participant S as Security Agent
    participant P as Pattern Agent

    Note over EL,P: ✅ AFTER — AsyncOpenAI
    EL->>B: schedule
    B->>B: await create() ⏸️ yields
    EL->>S: schedule
    S->>S: await create() ⏸️ yields
    EL->>P: schedule
    P->>P: await create() ⏸️ yields
    Note over EL,P: all three HTTP requests in flight
    B-->>EL: response (15s)
    S-->>EL: response (15s)
    P-->>EL: response (15s)
    Note over EL: TOTAL: ~15 seconds
```

**Why it was hard to spot:** nothing failed. No error, no warning, no exception. Output was completely correct. The only symptom was that stage timings **summed** instead of overlapping.

**The lesson:** `async` is a *cooperative* model. A coroutine only yields at an `await` on something that genuinely suspends. One blocking call serialises everything — and `async def` wrapped around a sync call is a lie the type checker won't catch.

---

**See also:** [mermaid-diagrams.md](mermaid-diagrams.md) · [whiteboard-guide.md](whiteboard-guide.md)
