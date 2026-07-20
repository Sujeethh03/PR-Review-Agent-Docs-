# Whiteboard Guide

> **What to draw, in what order, with what words.** Whiteboarding is a performance, not a diagram. What you *say* while drawing matters more than what ends up on the board.

---

## Universal Rules

| Rule | Why |
|---|---|
| **Narrate while drawing** | Silence reads as uncertainty |
| **Draw left to right, top to bottom** | Matches how they read |
| **Label every arrow** | An unlabelled arrow is an unanswered question |
| **Leave whitespace** | You'll need to annotate during follow-ups |
| **Say what you're *not* drawing** | "I'm leaving out error handling for now, I'll come back to it" |
| **Pause and check** | "Does that level of detail work, or would you like me to go deeper anywhere?" |

**The single biggest mistake:** drawing in silence, then explaining. Draw and talk simultaneously.

---

## Drawing 1 — Project Architecture (the default)

**When:** "Walk me through your project" · "Explain your architecture"
**Time:** 3–4 minutes

### The sequence

```
STEP 1                      STEP 2                    STEP 3
┌────────┐                  ┌────────┐  POST+HMAC  ┌──────────┐
│ GitHub │                  │ GitHub │ ──────────► │ FastAPI  │
└────────┘                  └────────┘             │ /webhook │
                                                    └──────────┘

STEP 4 — THE KEY ARROW
┌────────┐  POST + HMAC   ┌──────────┐
│ GitHub │ ─────────────► │ FastAPI  │
│        │ ◄───────────── │ /webhook │
└────────┘   200 OK, 5ms  └──────────┘
                                │
                                ▼
                          ┌───────────┐
                          │ Background│
                          │   Task    │
                          └───────────┘
```

**Full board:**

```
┌────────┐  POST + HMAC   ┌──────────┐
│ GitHub │ ─────────────► │ FastAPI  │
│        │ ◄───────────── │ /webhook │
└───▲────┘  200 OK, 5ms   └────┬─────┘
    │                          ▼
    │                    ┌───────────┐
    │                    │ Background│
    │                    │   Task    │
    │                    └─────┬─────┘
    │                          ▼
    │        ┌────────┐  ┌────────┐  ┌─────────┐
    │        │  Diff  │─►│ Clone  │─►│ Ingest  │
    │        └────────┘  └────────┘  └────┬────┘
    │                                     ▼
    │                              ╔════════════╗
    │                              ║  ChromaDB  ║
    │                              ╚══════╤═════╝
    │                                     ▼
    │              ┌──────────────────────────────┐
    │              │  ┌────────────────────────┐  │
    │              │  │  🐛 Bug Agent          │  │
    │              │  ├────────────────────────┤  │   ┌──────────┐
    │              │  │  🔒 Security Agent     │──┼──►│  OpenAI  │
    │              │  ├────────────────────────┤  │   │  GPT-4o  │
    │              │  │  🎨 Pattern Agent      │  │   └──────────┘
    │              │  └────────────────────────┘  │
    │              │      ↑ ALL IN PARALLEL       │
    │              └──────────────┬───────────────┘
    │                             ▼
    │                    ┌─────────────────┐
    │                    │ Dedup + Filter  │
    │                    │  conf ≥ 0.70    │
    │                    └────────┬────────┘
    │                             ▼
    │                    ┌─────────────────┐
    └────────────────────│   PR Writer     │
      inline comments    └─────────────────┘
```

### The narration script

> *[Draw GitHub]* "Everything starts with GitHub — a PR is opened or updated."
>
> *[Draw FastAPI, arrow in]* "A webhook hits my FastAPI endpoint with an HMAC signature I verify before doing anything else."
>
> *[Draw the return arrow]* **"This is the most important arrow on the board.** I return 200 in about five milliseconds. GitHub times out webhook deliveries at around ten seconds and my pipeline takes ninety, so if I processed synchronously every delivery would fail and GitHub would eventually disable my webhook."
>
> *[Draw background task]* "So the handler validates, spawns a background task, and returns. Everything below this line happens after the response."
>
> *[Draw diff → clone → ingest chain]* "The pipeline fetches the diff, shallow-clones the repo at the head SHA, then indexes it — tree-sitter parses files into functions and classes, OpenAI embeds them."
>
> *[Draw ChromaDB]* "Into a per-repository ChromaDB collection. This is what lets me do retrieval later."
>
> *[Draw three agents stacked]* "Then three specialist agents — bugs, security, and codebase conventions. **These run concurrently, not sequentially.** That took a real bug fix to actually achieve — I'll come back to that if you're interested. Each one, for each diff hunk, does a vector search back into ChromaDB to pull relevant context from elsewhere in the repo."
>
> *[Draw dedup/filter]* "Findings get deduplicated — agents overlap on the same lines — then filtered at 0.70 confidence."
>
> *[Draw PR writer, arrow back to GitHub]* "And posted as inline comments on the diff with one-click fix suggestions."
>
> "Anywhere you'd like me to go deeper?"

**Timing:** ~3 minutes. Stop and check in.

---

## Drawing 2 — Async: Before vs After

**When:** "Tell me about a performance problem" · "Explain async/await"
**Time:** 2 minutes

Draw **two timelines side by side**. This is the clearest possible way to show the bug.

```
❌ BEFORE — synchronous OpenAI client

Time →  0s        15s       30s       45s
        │         │         │         │
Bug     ████████████
Security          ████████████
Pattern                     ████████████
                                      ▲
                             TOTAL: 45 seconds


✅ AFTER — AsyncOpenAI

Time →  0s        15s
        │         │
Bug     ████████████
Security████████████
Pattern ████████████
                  ▲
         TOTAL: ~15 seconds
```

### Narration

> "My three agents were supposed to run in parallel. LangGraph fanned out to three nodes, each used `asyncio.gather`, everything was `async def`.
>
> But I was using the **synchronous** OpenAI client — `OpenAI`, not `AsyncOpenAI`. A blocking call inside a coroutine doesn't yield; it holds the single event loop thread for the whole HTTP request. *[point at top timeline]* So each agent politely waited its turn.
>
> The insidious part is that **nothing failed.** No error, no warning. The output was completely correct. The only symptom was that my stage timings *summed* instead of overlapping — which is what tipped me off.
>
> *[point at bottom timeline]* Switching to `AsyncOpenAI` gave me roughly 3× at the agent level, and more overall because each agent also parallelises across hunks.
>
> The lesson: async is a **cooperative** model. One blocking call serialises everything. You audit the library, not the syntax — `async def` around a sync call is a lie the type checker won't catch."

---

## Drawing 3 — The Deletion Cascade

**When:** "Tell me about a hard decision" · "A time you changed your mind"
**Time:** 2 minutes

Draw as a **vertical chain**, top to bottom. The visual falling-domino effect does the work.

```
   ┌───────────────────────────────────┐
   │ Critic rejects ~87.5% of findings │
   │      — including REAL bugs        │
   └────────────────┬──────────────────┘
                    ▼
        ┌───────────────────────┐
        │    Critic deleted     │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │ Nothing to approve    │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  Dashboard deleted    │  ← Next.js, ~700 lines
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │ Nothing reads the DB  │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │ PostgreSQL deleted    │  ← SQLAlchemy + Alembic
        └───────────┬───────────┘
                    ▼
   ╔═══════════════════════════════════╗
   ║  CONFIDENCE_THRESHOLD = 0.70      ║
   ║       (one line)                  ║
   ╚═══════════════════════════════════╝
```

### Narration

> "I built a critic agent to filter findings before showing them to developers, plus PostgreSQL to store them and a Next.js dashboard for a human to approve them in. Three components, one goal: don't show developers bad findings.
>
> Then I measured. *[point at top]* The critic was rejecting about 87.5% of findings, and when I read the rejected ones they included real bugs I'd deliberately planted.
>
> The reason is structural. An LLM asked to be a sceptical reviewer, with no ground truth and no cost for rejecting a true positive, drifts into being a rejection machine. 'I'm not certain this is real' is always defensible.
>
> But the deeper mistake was mine: I'd built a three-component filtering system for a false-positive rate I **never measured**.
>
> *[trace down the chain]* So the critic went. Then there was nothing to approve, so the dashboard had no purpose. Then nothing read the database. About a thousand lines of working code, deleted.
>
> *[point at the bottom box]* Replaced by that. And every dimension improved at once — better findings, lower latency, less code, fewer failure modes.
>
> I'd add one honest caveat: replacing a measurably harmful filter with an unmeasured threshold is progress, not a solution. My 0.70 is still a tuned guess."

---

## Drawing 4 — System Design at Scale

**When:** "Design this for 10,000 repositories"
**Time:** 8–10 minutes

**Draw only after** clarifying requirements and doing estimation out loud.

```
┌────────┐   ┌─────┐   ┌────┐   ┌──────────────┐
│ GitHub │──►│ WAF │──►│ LB │──►│  Webhook     │
│ 10k    │   └─────┘   └────┘   │  Receivers   │
│ repos  │                      │  (stateless) │
└───▲────┘                      └──────┬───────┘
    │                                  │
    │                            ┌─────▼──────┐
    │                            │   Redis    │ idempotency
    │                            │ delivery_id│
    │                            └─────┬──────┘
    │                                  ▼
    │              ╔═══════════════════════════════════╗
    │              ║   QUEUE (partitioned by org)      ║ ◄── THE key change
    │              ╚════┬═══════════┬═══════════┬══════╝
    │                   ▼           ▼           ▼
    │            ┌──────────┐ ┌──────────┐ ┌──────────┐
    │            │ Ingestion│ │ Analysis │ │Publisher │
    │            │ CPU+mem  │ │ I/O bound│ │ GH limits│
    │            └────┬─────┘ └────┬─────┘ └────┬─────┘
    │                 │            │            │
    │                 ▼            ▼            │
    │          ╔════════════╗ ┌─────────┐      │
    │          ║ Vector     ║ │   LLM   │      │
    │          ║ Store      ║ │ Gateway │──► OpenAI
    │          ║ SHARDED    ║ │ routing │──► Fallback
    │          ║ ▓ ▓ ▓      ║ │ budget  │
    │          ╚════════════╝ └─────────┘
    │                 │
    │          ┌──────▼──────┐
    │          │ PostgreSQL  │ metadata + FEEDBACK LABELS
    │          │ + replicas  │
    │          └─────────────┘
    └──────────────────────────────────────────────┘
                    inline comments
```

### Narration

> *[After stating your estimates]* "About 110 concurrent reviews at peak, 1.8 million API calls a day, roughly 630 gigabytes of vector data, and — worth naming — around $90,000 a month in LLM costs. That last number drives more design decisions than anything else.
>
> *[Draw receivers]* "Webhook receivers are stateless — verify the signature, dedupe on delivery ID, enqueue. They scale on requests per second."
>
> *[Draw the queue large and emphasised]* **"This is the pivotal change from what I actually built.** I use `asyncio.create_task`, which lives in process memory — a restart drops in-flight reviews silently. A queue gives me four things: durability, retries with a dead-letter queue, backpressure through a bounded worker pool, and independent scaling.
>
> "And the property I actually need most isn't durability — it's **backpressure**. My current unbounded fan-out is a direct consequence of having nothing to bound it."
>
> *[Draw three worker pools]* "Three worker types because they have genuinely different resource profiles. Ingestion is CPU and memory heavy — tree-sitter parsing. Analysis is I/O bound on external APIs. Publishing is bound by GitHub's rate limits. Different scaling signals, so they scale independently."
>
> *[Draw sharded vector store]* "630 gigabytes doesn't fit on one node. I shard by repository using consistent hashing — and this is an unusually clean sharding problem, because every query is already scoped to one repo. Nothing ever spans shards. Consistent hashing rather than modulo, because going from ten to eleven shards with modulo remaps 91% of keys; consistent hashing remaps about a ninth."
>
> *[Draw LLM gateway]* "And this is the other component that doesn't exist in my version. At $90k a month, cost governance needs a single home — model routing so trivial hunks go to a cheaper model, per-org budget enforcement, distributed rate limiting, provider failover, and metering. Without a gateway those decisions scatter across every call site and drift."
>
> *[Draw Postgres]* "Metadata and analytics — and critically, **feedback labels**. Whether developers resolved, applied, or dismissed each finding. That's the ground truth my current system completely lacks, and without it I can't calibrate my confidence threshold or measure whether a prompt change helped."

**Then say:** "Let me call out the three things here that don't exist in what I built: the queue, the LLM gateway, and sharding. Everything else scales by adding instances."

---

## Drawing 5 — RAG and the Self-Match Problem

**When:** "Explain RAG" · "A subtle bug you found"
**Time:** 2 minutes

```
INDEXING
  repo ──► tree-sitter ──► chunks ──► embed ──► ╔═══════════╗
                                                ║ ChromaDB  ║
RETRIEVAL                                       ║ owner__   ║
  diff hunk ──► embed ──► similarity search ──► ║   repo    ║
                                                ╚═════╤═════╝
                                                      │
                                              top 5 results
                                                      │
                              ┌───────────────────────▼──────────────────────┐
                              │ 1. src/auth.py:38-52  ◄── ❌ THE OLD VERSION │
                              │ 2. src/auth.py:40-55  ◄── ❌ SAME REGION     │
                              │ 3. src/session.py     ◄── ✅ actually useful │
                              │ 4. src/token.py       ◄── ✅                 │
                              │ 5. src/middleware.py  ◄── ✅                 │
                              └──────────────────────────────────────────────┘
                                        │
                                 SELF-MATCH FILTER
                                 same file + overlapping lines → drop
                                        │
                                        ▼
                              3 chunks of genuine context
```

### Narration

> "My pattern agent — the one comparing new code against codebase conventions — was finding almost nothing. So I logged what was actually being retrieved.
>
> *[point at results 1 and 2]* "The top hits were the **pre-change version of the same function**, still sitting in the index from the previous ingestion.
>
> Obvious in hindsight: semantically, the code most similar to a modified function *is* that function. The vector search was working perfectly and returning exactly the wrong thing. The agent was being asked 'does this follow the codebase's conventions?' while being handed the code itself as its comparison set.
>
> *[draw the filter]* "So I exclude chunks from the same file at overlapping line ranges. Standard interval-overlap test.
>
> The general lesson: **in RAG, what you exclude matters as much as what you retrieve.** The naive implementation is 'embed the query, take top-k,' and top-k is frequently dominated by near-duplicates of the query — which carry zero information.
>
> One refinement I'd still make: the filter runs *after* retrieval, so if three of five hits are self-matches I'm down to two chunks. I should over-fetch ten, filter, then take five."

---

## Recovery Phrases

When you're stuck or need time:

| Situation | Say |
|---|---|
| Need thinking time | "Let me think about that for a second." *(Silence is fine. Filler isn't.)* |
| Don't know | "I don't know. Here's how I'd find out —" |
| Made a mistake | "Actually, let me correct that." *(Then fix it. Owning errors reads as confidence.)* |
| Question is ambiguous | "Do you mean X or Y?" |
| Going too deep | "I'm going deeper than you might need — want me to move on?" |
| Ran out of board | "Let me erase this section, we're past it." |
| Asked about something you don't have | "I don't have that. Here's what I did instead and why, and here's where it'd go." |

**The single best recovery move:** turn "I don't know" into "I don't know, but here's how I'd figure it out." That converts a gap into a demonstration of method.

---

## Pre-Interview Practice

Draw each of these **from memory**, out loud, timed:

- [ ] Drawing 1 — Architecture (3 min)
- [ ] Drawing 2 — Async before/after (2 min)
- [ ] Drawing 3 — Deletion cascade (2 min)
- [ ] Drawing 4 — Scale design (8 min)
- [ ] Drawing 5 — RAG self-match (2 min)

**Record yourself once.** You'll immediately hear the filler words and the places you go silent while drawing. Those are the two things worth fixing.

---

**See also:** [mermaid-diagrams.md](mermaid-diagrams.md) · [../21-Interview-Cheat-Sheet.md](../21-Interview-Cheat-Sheet.md)
