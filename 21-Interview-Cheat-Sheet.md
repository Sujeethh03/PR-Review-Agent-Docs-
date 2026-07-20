# 21 — Interview Cheat Sheet

> **Read this in the 20 minutes before your interview.** Everything else is depth; this is recall.

---

## The 30-Second Pitch

> PR Review Agent is an AI service that reviews GitHub pull requests automatically. A webhook hits my FastAPI backend, which returns 200 immediately and runs the pipeline in the background: fetch the diff, clone the repo, index it into a vector database with tree-sitter and OpenAI embeddings, then fan out to three specialist GPT-4o agents — bugs, security, and codebase conventions — running in parallel. Each agent uses RAG to pull relevant context from the rest of the codebase, so it reviews the change *against* the repo rather than in isolation. Findings above a 0.70 confidence threshold post as inline GitHub comments with one-click fix suggestions.

---

## The Numbers (memorise these)

| Metric | Value |
|---|---|
| Specialist agents | **3** — bug, security, pattern |
| Confidence threshold to post | **0.70** |
| Per-agent internal floor | **0.6** |
| Critic rejection rate (why it was deleted) | **~87.5%** |
| Embedding model | `text-embedding-3-small` |
| Embedding dimensions | **1536** |
| Reasoning model | **GPT-4o**, `temperature=0` |
| Vector distance | **Cosine** (HNSW) |
| Context chunks retrieved | **5** (bug/security), **10** (pattern) |
| Clone depth | **1** (shallow) |
| Embedding batch size | **100** chunks |
| Line-window fallback | **60** lines, **15** overlap |
| Max file size ingested | **512 KB** |
| GitHub App JWT lifetime | **9 min** (max 10) |
| Installation token lifetime | **1 hour** |
| Languages via tree-sitter | **8** |
| Backend size | **~1,900 lines**, 41 files |
| End-to-end latency | **30–90 seconds** |
| Webhook ACK time | **~5 ms** |
| Parallelism speedup | **~60×** vs sequential |
| Concurrent calls (20-hunk PR) | **60** ⚠️ unbounded |

---

## The Stack, One Line Each

| Tech | Why |
|---|---|
| **FastAPI** | I/O-bound workload; ASGI-native async |
| **Python 3.13** | The AI ecosystem lives here |
| **GPT-4o** | Code review needs strong reasoning; false positives kill adoption |
| **text-embedding-3-small** | ~6.5× cheaper than large, modest quality gap |
| **ChromaDB** | Embedded, free, zero ops at single-instance scale |
| **tree-sitter** | Semantic chunks beat fixed-size for embedding quality |
| **LangGraph** | Typed state + declarative parallel fan-out |
| **Railway** | Git-push deploy, minimal config |
| **httpx** | Async HTTP client |
| **PyJWT** | RS256 GitHub App JWT |
| **GitPython** | Shallow clone (⚠️ needs the `git` binary) |

---

## What Does NOT Exist (say this confidently)

| Assumed | Reality |
|---|---|
| React/Next.js frontend | **Built, then deleted** in `a61bfd2`. GitHub's UI replaced it |
| PostgreSQL | **Built, then deleted.** `alembic/` is dead code |
| User login / OAuth | No users. JWT is **machine-to-machine** GitHub App auth |
| Redis | Not present. In-process singletons + ChromaDB |
| Docker | Railway buildpacks (`railpack.json`) |
| CI/CD | None — push to main deploys |
| pytest suite | Two verification **scripts**, not tests |

**The framing:** *"I built it, measured it, and deleted it"* is a stronger answer than any feature list.

---

## The Five Stories (know these cold)

### 1. The Critic Agent — 87.5% ⭐ LEAD WITH THIS
Built a fourth "critic" agent to filter findings, plus PostgreSQL and a Next.js approval dashboard. Measured it: rejecting ~87.5% of findings **including real planted bugs**. An LLM asked to be sceptical, with no cost for rejecting a true positive, becomes a rejection machine. Deleted all three — ~1,000 lines — replaced with one line: `CONFIDENCE_THRESHOLD = 0.70`. Simpler, faster, and more accurate simultaneously.
**Lesson:** don't build a solution before measuring the problem.

### 2. False Parallelism — 3×
Three agents were supposed to run in parallel. LangGraph fan-out ✅, `asyncio.gather` ✅, everything `async def` ✅ — but I used the **synchronous** `OpenAI` client, which blocks the event loop. They ran strictly sequentially. Nothing failed; the only symptom was timings *summing* instead of overlapping. Fixed with `AsyncOpenAI`.
**Lesson:** async is cooperative — one blocking call serialises everything. Audit the library, not the syntax.

### 3. The Git Binary — 8 commits
Clones worked locally, failed in production. GitPython isn't a git implementation — it **shells out to the `git` binary**. `pip install` gives you the Python package, not the executable. My Mac had git; Railway's build image didn't. Three attempts to fix (`packages` → wrong type → `aptPkgs`).
**Lesson:** undeclared implicit system dependency. Exactly what containers prevent — and I'd undermined that by letting a buildpack infer my image.

### 4. The Infinite Loop
Bot posts a comment → GitHub fires an event → pipeline runs → bot posts a comment → ∞. Each iteration a full LLM pipeline. Fixed with an event-type ignore list.
**Lesson:** in event-driven systems, always ask what events your own actions produce. My fix filters by *type* when I meant *identity* — the same mistake as my comment-deletion bug.

### 5. RAG Retrieved Itself
The pattern agent found nothing. Logged the retrieved chunks: the top hit was always **the pre-change version of the same function**. It was comparing code to itself. Fixed with an interval-overlap filter excluding same-file chunks.
**Lesson:** in RAG, what you *exclude* matters as much as what you retrieve.

---

## The Bugs You'd Volunteer

| Bug | Fix |
|---|---|
| Deletes **all** bots' comments (`user.type == "Bot"`) | Filter on bot login |
| Stale chunks accumulate (IDs encode line numbers) | Delete before insert on modified files |
| LLM line numbers unvalidated → 422 kills all comments | Cross-check against `parse_diff_hunks` |
| Unbounded concurrency (150+ calls) | `asyncio.Semaphore(10)` |
| `create_task` reference discarded — GC risk | Module set + done-callback |
| Same hunk embedded 3× | Precompute and share (**60 calls → 1**) |
| Token fetched 3× per pipeline | TTL cache |
| N+1 deletes | `$in` operator |

**The four 🔴 fixes total ~25 lines.**

---

## Killer One-Liners

**On the critic:**
> "An LLM asked to be a sceptic, with no cost for rejecting a true positive, optimises into a rejection machine."

**On deletion:**
> "A datastore with no reader is a liability, not an asset."

**On the frontend:**
> "Build UI where your users already are, not where it's convenient for you to build."

**On async:**
> "`async` is not a performance feature — it's a cooperative model. One blocking call serialises everything."

**On prompts vs code:**
> "Prompts express intent; code enforces invariants. Anything I can't afford to have wrong shouldn't depend on the model complying."

**On confidence:**
> "LLM confidence is not calibrated probability. When it says 0.8 that doesn't mean it's right 80% of the time — it's a token, not a posterior."

**On RAG:**
> "The nearest neighbour of a modified function is its own previous version. In RAG, what you exclude matters as much as what you retrieve."

**On the security override:**
> "The prompt already says secrets must be high severity. I enforce it in Python anyway, because prompts are suggestions and models drift."

**On the queue:**
> "The property I need most isn't durability — it's backpressure."

**On measurement:**
> "I can't tell you whether a prompt change made the system better or worse. That's the biggest gap in the project."

**On blocking merges:**
> "A tool that wrongly blocks a merge gets disabled within a week."

**On silent failure:**
> "A visibly broken system gets fixed; a silently degraded one erodes trust invisibly."

---

## Rapid Q&A

| Question | Answer |
|---|---|
| Why FastAPI? | I/O-bound; ASGI-native async; Django's ORM/admin/auth would be dead weight |
| Why three agents? | Attention, prompt budget, independent calibration, parallelism. Costs 3× and forces dedup |
| Why no database? | Its only consumer was the deleted dashboard. GitHub is the persistence layer |
| Why no frontend? | GitHub's PR UI is better and it's where developers already are |
| Why no Redis? | Nothing needs shared state at one instance. Would go: queue, rate limits, embedding cache |
| Why cosine not Euclidean? | Embedding magnitude tracks text length, not meaning |
| Why one collection per repo? | Physical isolation — a forgotten `where` clause can't leak cross-repo |
| Why `temperature=0`? | Reproducible reviews. Creative variation is a bug here |
| Why constant-time compare? | `==` short-circuits → timing leaks byte position → forge signature in ~16k attempts not 256^64 |
| Why RS256 not HS256? | GitHub must verify without being able to forge |
| Why raw bytes for HMAC? | Parsing and re-serialising changes whitespace/ordering → signature never matches |
| Why `COMMENT` not `REQUEST_CHANGES`? | Precision is unmeasured; blocking on an unmeasured signal is irresponsible |
| Why shallow clone? | Only current file contents matter; seconds vs minutes |
| Why `to_thread` for ingestion? | ChromaDB is blocking; honest offload beats lying to the event loop |
| What breaks first at scale? | **OpenAI rate limits** — not CPU, not the database |
| Biggest weakness? | Can't measure quality — no labelled dataset |
| Biggest inefficiency? | Same hunk embedded 3×; 60 API calls where 1 would do |

---

## Whiteboard — Draw In This Order

1. **GitHub** (left)
2. **FastAPI /webhook** ← arrow labelled `POST + HMAC`
3. **→ 200 OK, 5ms** ← *"most important arrow on the board"*
4. **Background task** (below)
5. **Diff → Clone → Ingest** (chain right)
6. **ChromaDB** (cylinder, under Ingest)
7. **Three boxes stacked vertically** — Bug / Security / Pattern ← *emphasise parallel*
8. **Dedup → Filter ≥0.70**
9. **PR Writer → arrow back to GitHub**
10. **OpenAI cloud** (right) with dotted arrows from Ingest and all three agents

**Narration:** *"Two things make this work. This arrow — 200 in five milliseconds, because GitHub times out at ten seconds and my pipeline takes ninety. And these three boxes running concurrently, which took a real bug fix to actually achieve."*

---

## Red Flags — Never Say These

| ❌ Don't say | ✅ Say instead |
|---|---|
| "We built…" (solo project) | "I built…" |
| "I used PostgreSQL" | "I built it, then deleted it because…" |
| "It has a React frontend" | "I built one and removed it — GitHub's UI is better" |
| "I used JWT for user login" | "JWT for machine-to-machine GitHub App auth" |
| "It's fully tested" | "Testing is my weakest area — here's the plan" |
| "There are no weaknesses" | Name four, with fixes |
| "I used Redis for caching" | "No Redis. Here's where it'd go when I scale" |
| "It's production-ready" | "It works; here's what's missing for a team" |

**The meta-rule:** every honest gap you volunteer buys credibility for every claim you make.

---

## The Closing Question

**"Do you have questions for us?"** — always have three.

- "How do you handle code review on this team — what's the balance between automated tooling and human review?"
- "What does your path to production look like — CI gates, staging, deployment strategy?"
- "What's the biggest technical challenge the team is facing right now?"

---

## Final Checklist

Before you walk in:

- [ ] Can recite the 30-second pitch cold
- [ ] Know the 87.5% critic story in 2 minutes
- [ ] Know the false-parallelism story
- [ ] Can name four bugs with fixes
- [ ] Can draw the architecture in the right order
- [ ] Know cosine vs Euclidean, RS256 vs HS256, constant-time comparison
- [ ] Ready to say "I don't have that, here's what I'd do" without flinching
- [ ] Have three questions for them

**Next:** [22-Final-Revision-Notes.md](22-Final-Revision-Notes.md)
