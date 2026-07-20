# 15 — Design Decisions

## Executive Summary

Sixteen significant decisions in **ADR format** (Architecture Decision Record): context, options considered, decision, consequences. Each includes the alternatives you'd be asked about and the trade-off you accepted.

**The meta-point for interviews:** a decision without a stated trade-off isn't a decision, it's a default. Every entry here names what was given up.

| # | Decision | Trade-off accepted |
|---|---|---|
| 1 | Webhooks over polling | Public endpoint + signature verification |
| 2 | Three specialised agents over one | 3× cost + dedup complexity |
| 3 | Remove the critic agent | No LLM-based validation layer |
| 4 | Confidence threshold over human approval | No human in the loop |
| 5 | GitHub UI over a custom dashboard | No custom analytics views |
| 6 | No database | No history, no analytics |
| 7 | ChromaDB embedded | Not shareable across instances |
| 8 | tree-sitter over fixed chunking | Complexity + 8 dependencies |
| 9 | GPT-4o for all agents | Higher cost than routing |
| 10 | `text-embedding-3-small` | Slightly lower retrieval quality |
| 11 | Forced tool calling | Locked to compatible providers |
| 12 | `temperature=0` | No output diversity |
| 13 | `asyncio.create_task` over a queue | **No durability, no retries** |
| 14 | LangGraph over `asyncio.gather` | Dependency for a simple graph |
| 15 | Read-only trust model | Can't auto-fix |
| 16 | `COMMENT` over `REQUEST_CHANGES` | Findings can be ignored |

---

## ADR-001 — Webhooks over Polling

**Context.** The system must detect new and updated PRs.

**Options.**

| Option | Latency | Wasted calls | Complexity |
|---|---|---|---|
| Poll the API every N minutes | Up to N min | Many | Low |
| GitHub webhooks | Near-instant | Zero | Public URL + HMAC |
| GitHub Actions | On push | Zero | Per-repo config |

**Decision.** Webhooks.

**Rationale.** Latency is user-facing — a developer waiting five minutes has already context-switched. Polling 1,000 repos every minute is 1,440,000 API calls/day, almost all returning nothing. GitHub Actions would require every user to add a workflow file, killing adoption.

**Consequences.**
- ✅ Sub-second detection; zero wasted calls
- ⚠️ Requires a publicly reachable HTTPS endpoint
- ⚠️ Requires HMAC verification — anyone can POST to a public URL
- ⚠️ Missed deliveries need GitHub's redelivery UI (no self-healing like polling)

---

## ADR-002 — Three Specialised Agents over One Generalist

**Context.** The system must find bugs, security issues, and convention violations.

**Decision.** Three agents with distinct system prompts, running in parallel.

**Rationale.**

| Factor | Explanation |
|---|---|
| Attention | One prompt asking for three things splits focus; specialist prompts concentrate it |
| Prompt budget | The security prompt alone lists 15 OWASP categories — three taxonomies won't fit usefully |
| Independent policy | Security forces confidence 1.0 on secrets; pattern abstains without context. These conflict in one prompt |
| Latency | max(t₁,t₂,t₃) instead of t₁+t₂+t₃ |

**Consequences.**
- ✅ Deeper analysis per domain; ~3× faster than sequential
- ✅ Add a fourth agent without touching the others
- ❌ **~3× the token cost**
- ❌ **Requires deduplication** precisely because agents overlap on the same lines
- ❌ Three prompts to maintain

**The honest counter:** one call would cost a third as much and produce naturally deduplicated output. I'm paying twice for specialisation — once in tokens, once in the dedup logic it forces. At very high volume the economics could flip, and the middle ground is routing: always run security, run bug and pattern only above a complexity threshold.

---

## ADR-003 — Remove the Critic Agent 🔴

**Context.** A fourth agent reviewed the specialists' findings and rejected weak ones.

**Measurement.** It rejected **~87.5% of findings, including real planted bugs.**

**Decision.** Delete it.

**Rationale.** An LLM asked to be a sceptical reviewer, with no ground truth and no cost for rejecting a true positive, optimises into a rejection machine. "I'm not certain this is real" is always defensible. The critic wasn't filtering noise — it was suppressing signal.

More fundamentally: I built a filter for a false-positive rate I never measured.

**Consequences.**
- ✅ Real bugs stopped being suppressed
- ✅ Removed an LLM call from the critical path
- ✅ Cascaded into deleting the database and dashboard (ADR-005, ADR-006)
- ❌ No LLM-based validation layer at all now
- ⚠️ Replaced by a threshold that is *also* unvalidated — this is progress, not a solution

**Full story:** [14-Project-Challenges.md](14-Project-Challenges.md#challenge-1).

---

## ADR-004 — Confidence Threshold over Human Approval

**Context.** With the critic gone, something must gate what reaches developers.

**Options.**

| Option | Precision | Latency | Scales? |
|---|---|---|---|
| Post everything | Lowest | Instant | ✅ |
| **Threshold at 0.70** | Medium | Instant | ✅ |
| Human approval queue | Highest | Hours | ❌ |
| Second LLM critic | Measured: worse | +1 call | ✅ |

**Decision.** `CONFIDENCE_THRESHOLD = 0.70`, applied after a per-agent floor of 0.6.

**Rationale.** A human queue defeats the purpose — the value proposition is *fast* feedback. If a human must approve every finding, you've rebuilt code review with extra steps. The threshold is one line, zero latency, and tunable.

**Why two stages?** The 0.6 agent-level floor is the *model* self-censoring; the 0.70 global gate is *my* policy. Separating them means I can tune posting policy without touching prompts.

**Consequences.**
- ✅ Zero added latency, trivially tunable
- ❌ **The threshold is a tuned guess, not statistically derived** — no labelled dataset
- ❌ Some false positives reach developers
- ❌ Some true positives are suppressed below 0.70

**What would validate it:** collect developer responses (resolved / suggestion applied / dismissed) as implicit labels, sweep the threshold, pick the F1 maximum.

---

## ADR-005 — GitHub's UI over a Custom Dashboard

**Context.** A Next.js approval dashboard existed. With approval removed, it had no purpose.

**Decision.** Delete it; use GitHub's native PR review UI.

**Rationale.** Every capability comparison favours GitHub: inline display on the actual diff, syntax highlighting, threaded replies, resolve/unresolve, notifications through channels developers already configured, permissions inherited from the repo, and — decisively — **`suggestion` blocks with one-click apply**, which cannot be replicated outside GitHub.

Plus the adoption argument: a separate dashboard is a second tab, a second login, and a second notification channel to ignore.

**Consequences.**
- ✅ Better UX than I could build; zero frontend maintenance; zero auth to build
- ✅ One-click fix application became possible
- ❌ **No custom analytics views** — can't show "most common category across repos"
- ❌ No cross-repo dashboard

**The principle:** build UI where your users already are, not where it's convenient for you to build.

---

## ADR-006 — No Database

**Context.** PostgreSQL + SQLAlchemy + Alembic + a repository layer existed to store findings for the dashboard.

**Decision.** Delete it. GitHub comments are the persistence layer.

**Rationale.** The database had exactly one consumer — the dashboard. When the dashboard went, nothing read from it. **A datastore with no reader is pure liability:** migrations to maintain, connections to pool, backups to run, and a new failure mode in the critical path.

GitHub already stores every finding durably, versioned, searchable, permissioned, and in the place developers look.

**Consequences.**
- ✅ Removed an entire operational surface
- ✅ No migrations, no connection pool, no backup strategy
- ❌ **No historical analytics** — can't measure trends or false-positive rate over time
- ❌ **Lost atomicity** — if the pipeline crashes after posting five of ten comments, there's no rollback. GitHub's API has no transaction. This is a real regression I accepted knowingly
- ⚠️ `backend/alembic/` remains as dead code — should be deleted

**If it returned:** as an **analytics sink** written asynchronously *off* the hot path, not as a blocking approval store. Same technology, entirely different architectural role — and a failed write would degrade analytics, never reviews.

---

## ADR-007 — ChromaDB Embedded over a Managed Vector DB

**Context.** Need vector storage for code embeddings.

**Options.**

| Option | Cost | Ops | Scales |
|---|---|---|---|
| **ChromaDB embedded** | Free | None | Single instance |
| Pinecone | Paid | Managed | ✅ |
| Weaviate self-hosted | Infra cost | High | ✅ |
| pgvector | Postgres cost | Medium | ✅ |
| FAISS | Free | Manual persistence | Library only |

**Decision.** ChromaDB embedded at `/tmp/chromadb`.

**Rationale.** Zero cost, zero operations, no network hop, and a simple API. At single-instance scale a managed vector database is infrastructure with no corresponding problem.

**Consequences.**
- ✅ Free, fast, no ops
- ❌ **Ephemeral on Railway** — wiped on every deploy, silently forcing full re-ingests
- ❌ **Not shareable** — two instances would maintain divergent indexes
- ❌ Blocking client — needs `asyncio.to_thread`

**Mitigation already in place:** `CHROMADB_PATH` is an environment variable with a default, so moving to a persistent volume is a config change rather than a code change.

**If I rebuilt with analytics requirements:** **pgvector**. One database serving both vectors and relational data, with transactions, backups, and replication that operations teams already know how to run.

---

## ADR-008 — tree-sitter over Fixed-Size Chunking

**Context.** Source files must be split into embeddable chunks.

**Decision.** tree-sitter AST chunking, with 60-line/15-overlap windows as fallback.

**Rationale.** Fixed-size chunks split functions mid-body. A chunk starting mid-loop and ending mid-conditional encodes noise, and its embedding will never be a good retrieval match. Semantic chunks are coherent units of meaning.

**A specific sub-decision:** the AST walk recurses unconditionally, so a class emits a chunk for *itself* and for each method. That's deliberate — different queries want different zoom levels. A validation-logic query should match a method; a "what's our service structure" query should match a class. Cost: ~2× storage and embedding. Benefit: retrieval quality at multiple granularities.

**Consequences.**
- ✅ Coherent chunks; symbol names available as metadata
- ✅ Graceful fallback for 3 independent failure modes
- ❌ Eight `tree-sitter-*` dependencies (slow cold start)
- ❌ Only 8 languages; everything else falls back to windows
- ❌ ~2× embedding cost from the overlapping granularity

**Note:** `tree-sitter-languages` (the convenience bundle) doesn't support Python 3.13, forcing individual packages. A real constraint that shaped the dependency list.

---

## ADR-009 — GPT-4o for All Agents

**Decision.** GPT-4o everywhere — three agents plus the fix suggester.

**Rationale.** Code review requires strong reasoning about control flow, data flow, and security semantics. Smaller models produce noticeably weaker findings, and **false positives are the failure mode that kills adoption**. Paying more per call to protect precision is the right trade for an advisory tool.

**Consequences.**
- ✅ Strong reasoning; reliable tool calling; good instruction following
- ❌ **Most expensive option**, on every hunk regardless of triviality
- ❌ Vendor lock-in to OpenAI's tool-calling format

**The optimisation I haven't done:** model routing. A markdown change or a config edit doesn't need the flagship model.
```python
if hunk.file.endswith((".md", ".json", ".yaml")) or len(hunk.added_code) < 150:
    return "gpt-4o-mini"
```
50–80% saving on those hunks. It's on the roadmap.

---

## ADR-010 — `text-embedding-3-small` over `-large`

**Decision.** `text-embedding-3-small`, 1536 dimensions.

**Rationale.** Roughly **6.5× cheaper** than `-large` for a modest quality difference on code retrieval. Retrieval only needs to surface *reasonably* relevant context, not perfectly ranked context — the LLM does the reasoning, retrieval just has to put useful material in front of it. The marginal ranking improvement from `-large` doesn't measurably change agent output.

**Consequences.**
- ✅ ~6.5× cost saving on the highest-volume API call
- ✅ Half the storage (1536 vs 3072 dims)
- ❌ Slightly lower retrieval precision
- ⚠️ **Switching models later requires a full re-index** — vectors from different models are incomparable, and the dimensions don't even match

That last point is a genuine lock-in worth naming: the embedding model choice is much stickier than the LLM choice.

---

## ADR-011 — Forced Tool Calling for Structured Output

**Options.**

| Approach | Reliability |
|---|---|
| "Return JSON" in the prompt | ~85% — prose wrappers, markdown fences |
| JSON mode | ~99% valid JSON, but any shape |
| **Forced tool calling** | Valid JSON conforming to *my* schema |
| Structured Outputs (`strict: true`) | Constrained decoding — guaranteed |

**Decision.** Forced tool calling with `tool_choice` pinned to the function.

**Rationale.** `tool_choice="auto"` lets the model *decide* whether to call the function — and sometimes it just replies conversationally. Forcing it removes the prose escape hatch entirely.

Combined with `additionalProperties: false` (no invented fields), `required` on every field (no omissions), and `enum` on severity (no hallucinated values), the output shape is constrained at generation time. Pydantic then validates at construction as a backstop.

**Consequences.**
- ✅ Near-guaranteed parseable output; self-documenting schema
- ❌ Locked to providers supporting OpenAI-style tool calling
- ⚠️ **Should upgrade to `strict: true`** — constrained decoding makes conformance a guarantee rather than high compliance. One-key change.

---

## ADR-012 — `temperature=0`

**Decision.** Zero temperature on every call.

**Rationale.** The same diff should produce the same review. Creative variation is a bug in code review, not a feature — a developer who re-runs the bot and gets different findings loses trust in both runs.

**Consequences.**
- ✅ Maximum reproducibility
- ❌ No diversity — the model won't explore alternative interpretations
- ⚠️ **Not bit-deterministic.** Floating-point non-associativity in GPU kernels, batching effects, and backend model updates all introduce variation. This is why exact-match regression tests on LLM output don't work, and why [11-Testing.md](11-Testing.md) argues for property-based assertions.

---

## ADR-013 — `asyncio.create_task` over a Task Queue 🔴

**Context.** GitHub times out at ~10s; the pipeline takes 30–90s.

**Options.**

| Option | Durable | Retries | Backpressure | Infra |
|---|---|---|---|---|
| Synchronous processing | — | — | — | ❌ Times out |
| **`asyncio.create_task`** | ❌ | ❌ | ❌ | ✅ None |
| FastAPI `BackgroundTasks` | ❌ | ❌ | ❌ | ✅ None |
| Celery + Redis | ✅ | ✅ | ✅ | ❌ Broker + workers |
| SQS + workers | ✅ | ✅ | ✅ | ❌ Cloud setup |

**Decision.** `asyncio.create_task`.

**Rationale.** At one-instance scale, a broker is stateful infrastructure to deploy, monitor, and back up — solving problems I didn't yet have. The failure mode was "one review is missed and the developer pushes again," which was acceptable.

**Consequences — this is the weakest decision in the system, and I'd say so first:**
- ✅ Zero infrastructure
- ❌ **No durability** — restart loses in-flight reviews silently
- ❌ **No retries** — a transient failure is permanent
- ❌ **No backpressure** — unbounded fan-out is a direct consequence
- ❌ **GC bug** — the task reference is discarded; Python's docs warn the event loop only holds weak references, so a running task can be collected

**The three-line partial fix:**
```python
_background_tasks: set[asyncio.Task] = set()
task = asyncio.create_task(_safe_run_pipeline(...))
_background_tasks.add(task)
task.add_done_callback(_background_tasks.discard)
```

**Why the migration is cheap:** the pipeline is a pure function of five JSON-serialisable scalars — no closures over request state. `asyncio.create_task(...)` → `review_pr.delay(...)` is one line. That boundary was kept clean deliberately.

---

## ADR-014 — LangGraph over Plain `asyncio.gather`

**Decision.** LangGraph `StateGraph` for the agent fan-out.

**Rationale at the time.** Typed shared state via `AgentState` TypedDict; declarative topology; and the original graph was more complex — it included a critic node downstream of the specialists, where conditional edges and merge semantics genuinely mattered.

**Consequences.**
- ✅ Typed state; topology visible in 35 lines; easy to add a fourth agent
- ❌ A dependency for what is now a very simple graph
- ⚠️ **Honest concession:** after deleting the critic, the graph is three parallel nodes with no dependencies — which is exactly `asyncio.gather`. If I were starting today with only this topology, plain `gather` would be the more honest choice. I kept it because the extension path (a test-coverage agent, a performance agent) is where it pays off.

**Being willing to say "this is arguably over-engineered now" is worth more than defending it absolutely.**

---

## ADR-015 — Read-Only Trust Model

**Decision.** The agent posts comments. It never commits, never opens PRs, never modifies code.

**Rationale.** Trust must be earned before it's spent. An automated system that writes code needs a much higher precision bar than one that suggests. My precision is unmeasured, so writing would be irresponsible.

GitHub's `suggestion` blocks give the best of both: the agent proposes the exact fix, and the **human clicks apply**. The developer stays the decision-maker while getting one-click convenience.

**Consequences.**
- ✅ Cannot break a codebase; minimal GitHub permissions; easy to trust and adopt
- ❌ Can't auto-fix trivial issues (formatting, imports) where the risk genuinely is low
- ❌ Every fix needs a human click

**The permission model reflects it:** read code, write PR comments. Nothing else.

---

## ADR-016 — `event: "COMMENT"` over `REQUEST_CHANGES`

**Decision.** Post reviews as advisory comments, not blocking change requests.

**Rationale.** `REQUEST_CHANGES` **blocks the merge** until explicitly dismissed. Blocking a merge on an unmeasured signal is irresponsible — and practically, **a tool that wrongly blocks a merge gets disabled within a week.**

The asymmetry matters: a false positive in advisory mode costs thirty seconds of a developer's attention. A false positive in blocking mode costs a broken workflow and, very quickly, an uninstall.

**Consequences.**
- ✅ Non-intrusive; safe to adopt; wrong findings are ignorable
- ❌ Findings can be ignored entirely — including correct critical ones
- ⚠️ No enforcement

**When I'd revisit:** if I measured precision above some bar — say 95% on high-severity security findings — then `REQUEST_CHANGES` scoped *only* to that category would be defensible. Never as a blanket policy, and never before measurement.

---

## Interview Questions

### Q1. "What's the decision you're proudest of?"

**Best answer:**
> Deleting the critic agent, the database, and the frontend — about a thousand lines of working code, in one commit.
>
> I'd built a critic agent to filter findings before showing them to developers, plus a PostgreSQL store and a Next.js approval dashboard downstream of it. Three components, all working, all serving the goal of not showing developers bad findings.
>
> Then I measured. The critic was rejecting about 87.5% of findings — and when I read the rejected ones, they included real bugs I'd deliberately planted. Genuine null dereferences and hardcoded credentials, thrown out.
>
> The reason is structural. An LLM asked to be a sceptical reviewer, with no ground truth and no cost for rejecting a true positive, drifts into being a rejection machine. "I'm not certain this is a real issue" is always defensible. There was no feedback loop punishing it for being wrong in that direction.
>
> But the deeper mistake was mine: I'd built a three-component filtering system for a false-positive rate I never measured. I assumed the problem, designed for it, and only checked afterwards.
>
> So I deleted all three and replaced them with one line — a 0.70 confidence threshold — posting directly to GitHub's own review UI. Every dimension improved at once: better findings, lower latency, less code, fewer failure modes.
>
> The reason I'm proud of it isn't the outcome, it's that it was hard. I'd spent weeks on that dashboard and the sunk cost was real. Keeping code alive because you're attached to the effort is how systems rot.
>
> And I'd add one honest caveat: replacing a measurably harmful filter with an unmeasured threshold is progress, not a solution. My 0.70 is still a tuned guess, because I still don't have labelled data.

---

### Q2. "What decision would you change?"

**Best answer:**
> `asyncio.create_task` for background processing. It's the weakest thing in the system.
>
> When the webhook arrives I validate it, spawn a task, and return 200. That solves the real constraint — GitHub times out around ten seconds, my pipeline takes ninety. But the task lives in process memory, which means three problems.
>
> No durability: if Railway restarts the container mid-pipeline, that review vanishes silently. No retries: a transient OpenAI failure is permanent. And no backpressure, which is the one that actually bites — the unbounded fan-out that fires 150 concurrent GPT-4o calls on a large PR is a *direct consequence* of having no queue to bound it.
>
> There's also a subtle bug I'd fix regardless: I discard the return value of `create_task`. Python's documentation is explicit that the event loop only holds weak references to tasks, so a running task can be garbage collected. It's rare, but the failure mode is the worst kind — silent, non-deterministic disappearance. Three lines fix it: keep a module-level set and add a done-callback to discard.
>
> The real answer is a broker — Redis with Celery, or SQS with workers.
>
> The one thing I did get right is keeping the migration cheap. My pipeline is a pure function of five JSON-serialisable scalars with no closures over request state, so swapping `asyncio.create_task(...)` for `review_pr.delay(...)` is genuinely one line at the call site. I kept that boundary clean deliberately, because designing for a migration you haven't made yet costs nothing if you just don't couple things unnecessarily.
>
> Why I didn't build it: at my scale the failure mode was "one review is missed and the developer pushes again." That was acceptable. With a team it isn't.

---

### Q3. "How do you make technical decisions?"

**Best answer:**
> Three things, and the third is the one I got wrong early and now weight most heavily.
>
> **Start from the constraint, not the technology.** GitHub times out at ten seconds and my pipeline takes ninety — that constraint alone determines that processing must be asynchronous. The technology choice comes after.
>
> **Prefer the option with fewer moving parts until you have the problem the complex one solves.** I didn't add Redis because nothing needed shared state across requests. I didn't add microservices because 1,900 lines doesn't need distribution. Adding infrastructure before you have its problem means maintaining something that returns nothing.
>
> **Measure before you build the fix.** This is the one I learned the hard way. I built a critic agent, a database, and a dashboard to filter false positives — and never measured my false-positive rate first. When I finally measured, the filter was suppressing 87.5% of findings including real bugs. Three components built for an assumed problem.
>
> The other thing I try to do is name what I'm giving up. A decision without a stated trade-off isn't a decision, it's a default. When I chose embedded ChromaDB I wrote down that it wouldn't work across instances — so when that becomes true, I already know what breaks and why. And I try to keep migration paths cheap, which usually just means not coupling things unnecessarily.

---

## Common Mistakes

| Mistake | Better |
|---|---|
| Choosing technology before understanding the constraint | Constraint → requirements → technology |
| Adding infrastructure "for scale" prematurely | Add it when you have its problem |
| Building a solution before measuring the problem | Measure first (the critic lesson) |
| Keeping code out of sunk-cost attachment | Delete what has no purpose |
| Not stating trade-offs | Every decision gives something up |
| Defending a decision absolutely | Concede when it's arguably wrong now (LangGraph) |
| Coupling to a choice you may reverse | Keep boundaries clean and migrations cheap |

---

## Cross-References

| Topic | File |
|---|---|
| The stories behind these decisions | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Architecture these produced | [02-System-Architecture.md](02-System-Architecture.md) |
| AI-specific decisions | [09-AI-System.md](09-AI-System.md) |
| Where decisions break at scale | [13-Scalability.md](13-Scalability.md) |
| Decisions to revisit | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [16-Code-Walkthrough.md](16-Code-Walkthrough.md)
