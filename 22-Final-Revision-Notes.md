# 22 — Final Revision Notes

## Contents

1. The 30-second explanation
2. The 2-minute explanation
3. The 5-minute explanation
4. The 10-minute explanation
5. Most likely questions
6. Difficult cross-questions
7. One-page revision sheet
8. Five-page revision sheet

---

## 1. The 30-Second Explanation

> PR Review Agent is an AI service that reviews GitHub pull requests automatically. A webhook hits my FastAPI backend, which acknowledges immediately and runs the pipeline in the background: fetch the diff, clone the repo, index it into a vector database, then run three specialist GPT-4o agents in parallel — one for bugs, one for security, one for codebase conventions. Each uses RAG to pull relevant context from the rest of the repository, so it reviews the change against the codebase rather than in isolation. Findings above a confidence threshold post as inline GitHub comments with one-click fixes.

**Use for:** "Tell me about yourself", recruiter screens, networking.

---

## 2. The 2-Minute Explanation

> PR Review Agent reviews GitHub pull requests automatically. When you open a PR, a webhook hits my FastAPI backend. I verify the HMAC signature, return 200 in about five milliseconds — that matters because GitHub times out at ten seconds and my pipeline takes ninety — and run the work as a background task.
>
> The pipeline does four things. **First**, fetch the diff and shallow-clone the repo at the head SHA. **Second**, build a semantic index: tree-sitter parses every file into functions and classes, OpenAI embeds those chunks, and they go into a per-repository ChromaDB collection. **Third** — the core — a LangGraph state machine fans out to three specialist GPT-4o agents running concurrently: bugs, security, and conventions. Each agent, for each diff hunk, does a vector search to pull the five most relevant chunks from elsewhere in the repo, so it's reviewing with whole-codebase context. **Fourth**, findings get deduplicated, filtered at 0.70 confidence, and posted as inline comments with GPT-generated fix suggestions and OWASP references.
>
> The decision I'm proudest of is a deletion. I originally had a fourth critic agent, plus PostgreSQL and a Next.js approval dashboard. When I measured the critic, it was rejecting about 87.5% of findings — including real bugs I'd planted. So I deleted all three, about a thousand lines, and replaced them with a single confidence threshold using GitHub's own review UI. The system got simpler, faster, and more accurate at once.
>
> The thing I'd fix first: my concurrency is unbounded. Three agents times N hunks means a fifty-hunk PR fires a hundred and fifty concurrent GPT-4o calls with nothing capping them. That's a rate-limit and cost problem, and the fix is a semaphore.

**Use for:** "Walk me through your project" — the default answer.

**Why it works:** flow not features, one quantified decision, one volunteered weakness.

---

## 3. The 5-Minute Explanation

Everything above, then:

> Let me go deeper on the three parts I find most interesting.
>
> **The RAG layer.** The reason this isn't just "paste your diff into ChatGPT" is context. An LLM reviewing a diff in isolation finds *local* problems — a null dereference, an off-by-one. It can't find *relational* problems: "this duplicates a helper in another module" or "every other query in this codebase uses the parameterised wrapper and this one doesn't." Those need whole-repo knowledge.
>
> So I chunk with tree-sitter rather than fixed line counts, because a chunk that starts mid-function encodes noise. The AST walk recurses unconditionally, so a class produces a chunk for itself *and* one per method — deliberate, because different queries want different granularity.
>
> The subtlest part is a filter I had to add after debugging. My pattern agent was finding nothing, and when I logged the retrieved chunks the top hit was always the **pre-change version of the same function**, still sitting in the index. Semantically, the nearest neighbour of a modified function *is* that function. It was comparing code to itself. So I exclude same-file chunks at overlapping line ranges. In RAG, what you exclude matters as much as what you retrieve.
>
> **Structured output.** I use forced tool calling. I define a JSON Schema for a `report_findings` function, and critically I set `tool_choice` to force that specific function — with `auto`, the model sometimes just replies conversationally. Then Pydantic validates on my side, because the schema constrains generation but doesn't guarantee conformance. `severity: Literal["high","medium","low"]` means a hallucinated "critical" raises a `ValidationError` instead of reaching my formatter.
>
> There's a related thing I'm pleased with: my security agent forces hardcoded secrets to high severity and confidence 1.0 **in Python**, even though the prompt already says to. Prompts express intent; code enforces invariants. Anything I can't afford to have wrong shouldn't depend on the model complying.
>
> **Parallelism, and the bug that broke it.** Everything was `async def`, LangGraph fanned out to three nodes, each used `asyncio.gather` over hunks. But I was using the *synchronous* OpenAI client, which blocks the event loop. My three "parallel" agents ran strictly sequentially. Nothing failed — the output was correct — the only symptom was that stage timings summed instead of overlapping. Switching to `AsyncOpenAI` fixed it, roughly 3× at the agent level and much more overall since each agent also parallelises across hunks.
>
> **Where I'm weakest** is measurement. I have no labelled dataset, so my 0.70 threshold is a tuned heuristic, not a statistically justified cutoff. I genuinely cannot tell you whether a prompt change makes the system better or worse. The fix is collecting developer responses — resolved, suggestion applied, dismissed — from the GitHub API as implicit labels, then calibrating properly. That's the biggest gap in the project and everything else is secondary to it.

**Use for:** technical deep dives, senior interviews.

---

## 4. The 10-Minute Explanation

Everything above, plus these four sections:

### Architecture and its limits

> It's a single async Python process — one entry point, five pipeline stages, no database, no queue, no cache server. That minimalism is defensible at my scale, and I know exactly where each missing piece goes.
>
> The weakest link is `asyncio.create_task`. It schedules work in process memory, so a Railway restart mid-pipeline silently drops that review — no retry, no dead-letter queue, no visibility. There's also a subtle bug: I discard the task reference, and Python's docs are explicit that the event loop only holds weak references, so a running task can be garbage collected.
>
> The real fix is a broker. What I did get right is keeping that migration cheap: my pipeline is a pure function of five JSON-serialisable scalars with no closures over request state, so `asyncio.create_task(...)` becomes `review_pr.delay(...)` — one line at the call site. Designing for a migration you haven't made costs nothing if you just don't couple things unnecessarily.
>
> And the property I actually need from a queue isn't durability, it's **backpressure**. My unbounded fan-out is a direct consequence of having nothing to bound it.

### Security

> Two mechanisms, opposite directions. Inbound, GitHub signs webhooks with HMAC-SHA256 and I verify with `hmac.compare_digest` — constant-time, because `==` short-circuits at the first differing byte and that timing difference lets an attacker forge a signature byte by byte, reducing the search from 256^64 to about sixteen thousand attempts.
>
> Outbound, I sign a JWT with my RSA private key using RS256, exchange it for a one-hour installation token. RS256 rather than HS256 because GitHub must verify my tokens without being able to forge them.
>
> One security detail that's easy to overlook: my ingestion pipeline reads every file in a cloned repo and **sends the contents to OpenAI**. Without filtering, a user's committed `.env` — with their database password — gets embedded and stored in a vector database. So the walker has an explicit deny-list for `.env` files and `.pem`/`.key` extensions. Small code, large blast radius if omitted.
>
> Gaps: no rate limiting, no replay protection despite GitHub sending a unique delivery UUID, and the installation token is embedded in my clone URL where it could leak into logs via an exception message.

### Scaling

> The first thing to break isn't what people expect — it's **OpenAI rate limits, not my infrastructure.** At around ten concurrent PRs I'd be firing six hundred simultaneous GPT-4o calls. And the failure is silent: my retry is one attempt at a fixed one-second delay, and failed hunks get skipped, so the developer gets an incomplete review with no indication anything was dropped.
>
> At about a hundred times my current load the single process breaks and I need a queue with worker pools. At ten thousand repos the vector store breaks — roughly 630 GB — and I'd shard by repository using consistent hashing. That's an unusually clean sharding problem because every query is already scoped to one repo, so nothing ever spans shards.
>
> At the top tier the constraint stops being architectural and becomes economic. Twenty thousand reviews a day at maybe fifteen cents each is around ninety thousand dollars a month. That's where model routing, prompt caching, and eventually self-hosted inference matter more than any infrastructure change.

### What I'd do with three more months

> Reliability first, features last. Month one: the four correctness fixes — line-number validation, stale-chunk purging, the bot-comment filter, and a concurrency semaphore. About twenty-five lines total, highest impact per unit of effort.
>
> Month two: operability. Structured logging with correlation IDs, because right now two concurrent reviews interleave into unreadable output. A real test suite starting with the pure functions, which is most of my correctness risk with zero mocking. A durable queue. Cost metering with alerts.
>
> Month three: measurement — collecting developer feedback as implicit labels, building an evaluation suite with negative cases, and calibrating the threshold empirically. That's the one I'm most excited about, because until it exists everything else is faith-based.
>
> What I *wouldn't* build: more agents, a dashboard, or multi-provider support. All tempting, none of them address the fact that I can't measure whether the system works.

---

## 5. Most Likely Questions

| # | Question | Where |
|---|---|---|
| 1 | Walk me through your project | Section 2 above |
| 2 | Why three agents instead of one? | [17](17-Interview-Questions-Part-1.md) Q1 |
| 3 | Why did you delete the database and frontend? | [15](15-Design-Decisions.md) ADR-003/005/006 |
| 4 | Explain RAG in your system | [09](09-AI-System.md) Q1 |
| 5 | How do you get structured output from an LLM? | [09](09-AI-System.md) Q2 |
| 6 | How do you prevent hallucinations? | [09](09-AI-System.md) Q3 |
| 7 | Explain your authentication | [07](07-Authentication-and-Security.md) |
| 8 | Why constant-time comparison? | [07](07-Authentication-and-Security.md) A3 |
| 9 | What breaks at 100× scale? | [13](13-Scalability.md) |
| 10 | Tell me about a hard problem | [14](14-Project-Challenges.md) #1 |
| 11 | What's your biggest weakness? | Measurement — no labelled data |
| 12 | How do you test a non-deterministic system? | [11](11-Testing.md) Q2 |
| 13 | Explain async/await | [12](12-Performance-Optimization.md) |
| 14 | What would you improve? | [20](20-Future-Improvements.md) |
| 15 | Explain normalization | [17](17-Interview-Questions-Part-1.md) Q5 |

---

## 6. Difficult Cross-Questions

These are the ones designed to unsettle you. Prepared answers below.

### "Isn't this just a wrapper around ChatGPT?"

> The LLM calls are maybe fifty lines of the nineteen hundred. The engineering is everything around them.
>
> Building a per-repository semantic index with tree-sitter chunking and vector retrieval — that's what lets an agent reason about a change *relative to* the codebase rather than in isolation, which is the whole differentiator. Orchestrating three agents in parallel with two nested levels of concurrency. Guaranteeing structured output through forced tool calling plus Pydantic validation. Deduplicating overlapping findings across agents. Mapping LLM line numbers onto GitHub's inline comment API. Handling webhook signature verification, GitHub App token exchange, and an event-driven pipeline that can't block.
>
> The honest version: a wrapper around ChatGPT is a prompt and a `fetch` call. What makes this different is that it knows your codebase. Take away the retrieval layer and the pattern agent becomes worthless and the other two get measurably weaker — I know because that's effectively what I had before I added the self-match filter and it wasn't finding anything.

### "Your critic agent failed. How do you know the threshold isn't also wrong?"

> I don't, and that's the honest answer. I replaced a filter that was *measurably* harmful with one that's merely *unmeasured*. That's progress, not a solution.
>
> What I can say is that the failure modes differ in a way that matters. The critic was actively rejecting known true positives — I could see specific planted bugs being discarded. The threshold at worst suppresses low-confidence findings, which is the behaviour I actually want, just possibly tuned wrong.
>
> But you're right that I can't defend 0.70 over 0.65 or 0.80 with data. To fix it I need ground truth, and the practical source is developer behaviour — did they resolve the comment, apply the suggestion, or dismiss it? Those are implicit labels available from the GitHub API. With a few thousand I'd sweep the threshold and pick the F1 maximum, or fit a calibration curve mapping raw scores to empirical probabilities.
>
> It's the top item on my roadmap precisely because everything else is guesswork until it exists.

### "Why should I trust an AI to review code?"

> You shouldn't trust it — and I designed it so you don't have to.
>
> Three deliberate constraints. It's **read-only**: it posts comments and never commits, never opens PRs, never modifies code. It posts with `event: "COMMENT"` rather than `REQUEST_CHANGES`, so it **cannot block a merge**. And fix suggestions use GitHub's suggestion blocks, so the *human* clicks apply — the developer stays the decision-maker.
>
> That's a deliberate asymmetry. A false positive in advisory mode costs thirty seconds of attention. A false positive in blocking mode costs a broken workflow and, very quickly, an uninstall. A tool that wrongly blocks a merge gets disabled within a week.
>
> The right framing is that it's a *first-pass* reviewer, not a replacement. It catches the mechanical things a tired human misses at 5pm on a Friday — the null dereference, the hardcoded key, the string comparison with `==` in Java. It cannot judge whether the change is the right thing to build. That's still the human's job.
>
> And I'd only move to blocking if I measured precision above some bar, scoped to high-severity security findings specifically. Never as a blanket policy, and never before measurement.

### "You have no tests. Why should I believe this works?"

> Fair, and I won't argue the premise — my testing is the weakest part of the project.
>
> What I'd offer instead of a claim: it's running in production on public GitHub repositories, and the README links to specific PRs with real posted comments you can read. PR #7 has ten findings across two Java files — an infinite loop where a counter is never incremented, a Java string comparison using `==` instead of `.equals()`, a missing return statement. Those are real defects, correctly identified, with working one-click fixes.
>
> That's evidence, not proof. It doesn't tell you the false-positive rate, and I've been clear that I can't measure that.
>
> On the tests themselves — what I have is two verification scripts that make real API calls. They were genuinely useful for the problem I had, which was iterating on prompts fast, but they're not tests: they cost money, they're non-deterministic, and they can't run in CI. Naming them `test_*.py` was a mistake.
>
> The encouraging thing is that most of my correctness risk is in **pure functions**. The diff parser, the deduplication, the chunk ID generation — no I/O, no mocking, no cost. That's maybe eighty percent of the risk coverable with straightforward table-driven tests. The blocker is that my module-level singletons make the agent code awkward to mock, so making dependencies injectable is step one.

### "Three agents seems wasteful. Convince me."

> It might be, and I'll give you the counter-argument first because it's genuinely strong: one call would cost a third as much and produce naturally deduplicated output. My design costs 3× the tokens *and* forces me to write a deduplication pass, precisely because the agents overlap and all flag the same line. I'm paying twice for the specialisation.
>
> The case for it: attention dilution is real. A prompt asking for bugs, security issues, *and* convention violations splits the model's focus three ways. My security prompt alone lists fifteen OWASP categories with definitions and a calibration guide — three taxonomies in one prompt would blur the boundaries and blow past useful prompt length.
>
> More concretely, the agents have *conflicting* policies. Security forces confidence 1.0 for hardcoded secrets. Pattern abstains entirely when retrieval returns no context. I can't tell one prompt to both always-flag-at-maximum-confidence and stay-silent-without-evidence.
>
> And latency: three concurrent calls take the max, not the sum.
>
> Where I'd concede: at very high volume the economics could flip. The sensible middle ground is routing — always run security, and run bug and pattern only on hunks above some complexity threshold. That preserves depth where it matters and cuts cost where it doesn't.

### "What if I told you the pattern agent adds no value?"

> I'd want to see the data, and I'd have no way to contradict you — which is itself the problem.
>
> I can't measure per-agent contribution. I don't track which agent produced the findings developers actually act on. So your hypothesis is untestable with what I currently collect, and that's a gap rather than a defence.
>
> What I'd say for it: the pattern agent is the only one that genuinely requires the RAG layer, and it's the reason I built RAG at all. Bugs and security vulnerabilities are largely findable from the diff alone. "This deviates from your codebase's conventions" is only answerable with retrieved context — it's the capability that distinguishes this from a diff-scoped linter.
>
> What I'd say against it: it's also the agent most likely to produce noise, because "convention" is fuzzy in a way that "null dereference" isn't. And it's the one I had to add the most guardrails to — twice the retrieved context, plus abstaining entirely when retrieval returns nothing.
>
> The experiment I'd run: A/B it. Ship pattern findings to half the repositories, track suggestion-acceptance and dismissal rates by agent, and let the data decide. If it's genuinely additive noise, I'd delete it — I have form for deleting things that don't earn their place.

---

## 7. One-Page Revision Sheet

```
╔═══════════════════════════════════════════════════════════════════╗
║  PR REVIEW AGENT — ONE PAGE                                       ║
╠═══════════════════════════════════════════════════════════════════╣
║  WHAT: AI reviews GitHub PRs. Webhook → RAG index → 3 parallel    ║
║        GPT-4o agents → dedup → filter 0.70 → inline comments      ║
║                                                                    ║
║  STACK: FastAPI · Python 3.13 · GPT-4o · text-embedding-3-small   ║
║         ChromaDB · tree-sitter · LangGraph · Railway              ║
║                                                                    ║
║  NOT PRESENT: frontend (deleted) · PostgreSQL (deleted) · Redis   ║
║               · user auth · Docker · CI/CD · real tests           ║
║                                                                    ║
║  NUMBERS: 3 agents · 0.70 threshold · 0.6 agent floor             ║
║           1536 dims · cosine · 5/10 context chunks                ║
║           87.5% critic rejection · 9min JWT · 1hr token           ║
║           ~1,900 LOC · 30–90s latency · 5ms ACK · ~60× speedup    ║
║                                                                    ║
║  PIPELINE: webhook → HMAC → 200 OK → bg task → diff → clone       ║
║            → tree-sitter chunk → embed → ChromaDB                 ║
║            → LangGraph fan-out ×3 → RAG per hunk → GPT-4o         ║
║            → dedup 2-pass → filter ≥0.70 → suggest → post         ║
║                                                                    ║
║  5 STORIES:                                                        ║
║   1. Critic rejected 87.5% incl. real bugs → deleted 3 components ║
║   2. Sync OpenAI client → "parallel" agents ran sequentially      ║
║   3. GitPython needs git BINARY → prod-only failure, 8 commits    ║
║   4. Bot comment → webhook → review → comment → ∞ loop            ║
║   5. RAG top hit was the code being reviewed → self-match filter  ║
║                                                                    ║
║  TOP BUGS: deletes ALL bots' comments · stale chunks accumulate   ║
║            · LLM line numbers unvalidated (422) · unbounded       ║
║            concurrency · hunk embedded 3× (60 calls → 1)          ║
║                                                                    ║
║  BIGGEST WEAKNESS: no labelled data → can't measure precision     ║
║                    → 0.70 is a tuned guess                        ║
║                                                                    ║
║  BREAKS FIRST AT SCALE: OpenAI rate limits (not CPU/DB)           ║
║                                                                    ║
║  KEY LINES:                                                        ║
║   "Prompts express intent; code enforces invariants."             ║
║   "async is cooperative — one blocking call serialises all."      ║
║   "A datastore with no reader is a liability."                    ║
║   "In RAG, what you exclude matters as much as what you retrieve."║
║   "A tool that wrongly blocks a merge gets disabled in a week."   ║
╚═══════════════════════════════════════════════════════════════════╝
```

---

## 8. Five-Page Revision Sheet

### Page 1 — System

**Flow:** GitHub webhook → HMAC verify (`compare_digest`) → **200 in 5ms** → `asyncio.create_task` → fetch diff → shallow clone (`depth=1`) → walk files (skip `.env`/`.pem`/binaries/>512KB) → tree-sitter chunk (fallback: 60-line windows, 15 overlap) → embed in batches of 100 → ChromaDB `owner__repo` → LangGraph fan-out ×3 → per-hunk `asyncio.gather` → RAG retrieval (5 or 10 chunks, self-match filtered) → GPT-4o forced tool call, `temperature=0` → filter ≥0.6 → 2-pass dedup → filter ≥0.70 → GPT-4o suggestions → format with OWASP → delete stale bot comments → POST review.

**Two levels of parallelism:** 3 agents × N hunks = 3N concurrent. 20-hunk PR = 60 calls. ~60× faster than sequential. ⚠️ Unbounded.

**Error handling, 4 levels:** hunk (`return_exceptions=True`) → agent (`AgentOutput(error=...)`) → post (422 → summary fallback) → pipeline (`_safe_run_pipeline`). Never 500 to GitHub.

### Page 2 — AI

**Chunking:** tree-sitter AST → function/class/method. Recursion is unconditional, so a class emits a chunk for itself *and* each method — different retrieval granularities. Fallback triggers: unknown extension, missing package, or empty parse.

**RAG:** embed hunk → cosine search → **exclude same-file overlapping chunks** (critical — otherwise the top hit is the pre-change version of the same function) → format into prompt with file paths and line numbers.

**Structured output:** JSON Schema + `tool_choice` forced (not `auto`) + `additionalProperties: false` + `required` on all + `enum` on severity. Then Pydantic `Literal` validation as backstop.

**Hallucination control, 7 layers:** `temperature=0` → schema → forced tool call → Pydantic → confidence ≥0.6 → deterministic secret override → confidence ≥0.70. **Missing: line-number validation.**

**Prompt engineering wins:** anchored confidence examples · explicit "do NOT default to 1.0" · stated expected distribution ("most should be 0.6 or 0.8") · explicit permission to return empty.

**Cost:** `-small` over `-large` (~6.5×) · batching 100 · incremental ingestion (biggest win) · suggestions only after filtering.

### Page 3 — Security & Infra

**Inbound:** HMAC-SHA256 over **raw bytes** (parsing breaks the signature), `hmac.compare_digest` (constant-time — `==` leaks byte position, reducing forgery from 256^64 to ~16k attempts).

**Outbound:** RS256 JWT (`iat: now-60` for clock skew, `exp: now+540`, `iss: app_id`) → installation token (1 hour, scoped). RS256 not HS256 because GitHub must verify without forging.

**Secrets:** `.gitignore` covers `**/.env` and `*.pem`, verified untracked. Production key is **base64** because PEMs are multi-line and env vars mangle newlines. `load_dotenv(override=True)` so `.env` beats shell vars.

**Ingestion deny-list** (`.env`, `.pem`, `.key`) — critical because ingestion sends file contents to OpenAI.

**Gaps:** no rate limiting · no replay protection (`X-GitHub-Delivery` unused) · token in clone URL · no dependency scanning.

**Deploy:** Railway. `railway.toml` (start command, healthcheck 120s, restart on failure). `railpack.json` → `aptPkgs: ["git"]` because GitPython shells out to the git binary. `--host 0.0.0.0` (127.0.0.1 = unreachable in a container), `--port $PORT`.

### Page 4 — Decisions & Trade-offs

| Decision | Gave up |
|---|---|
| Webhooks over polling | Public endpoint + HMAC |
| 3 agents over 1 | 3× cost + dedup complexity |
| Deleted critic | No LLM validation layer |
| Threshold over human approval | No human in the loop |
| GitHub UI over dashboard | No custom analytics |
| No database | No history, **no atomicity** |
| ChromaDB embedded | Ephemeral, not shareable |
| tree-sitter | 8 deps, slow cold start |
| GPT-4o everywhere | Most expensive option |
| `-small` embeddings | Re-index required to change |
| `asyncio.create_task` | **No durability/retries/backpressure** |
| LangGraph | Arguably over-engineered now |
| Read-only | Can't auto-fix |
| `COMMENT` not `REQUEST_CHANGES` | Ignorable |

**Scaling breaking points:** 200 PRs/day → OpenAI rate limits (tuning) · 2,000 → single process (queue) · 20,000 → vector store 630GB + noisy neighbours (shard by repo, consistent hashing) · 200,000 → unit economics ~$90k/month (model routing, prompt caching, self-hosting).

### Page 5 — Weaknesses & Answers

**Bug register (16 total, 4 critical, ~25 lines to fix):**
1. `user.type == "Bot"` deletes every bot's comments → filter on login
2. Stale chunks (IDs encode line numbers) → delete before insert
3. LLM line numbers unvalidated → 422 kills all comments → cross-check `parse_diff_hunks`
4. Unbounded concurrency → `asyncio.Semaphore(10)`
5. `create_task` ref discarded → GC risk → module set + callback
6. Dedup on exact `line_start` → use interval overlap
7. N+1 deletes → `$in`
8. Token fetched 3× → TTL cache
9. Hunk embedded 3× → precompute (**60 → 1**)

**The honest framing for weaknesses:**
> "Testing is my weakest area — here's the plan." · "I can't measure precision — no labelled data." · "No Redis; here's where it'd go." · "This decision is arguably wrong now."

**Never say:** "we" for solo work · "it has a React frontend" · "I used PostgreSQL" · "it's fully tested" · "there are no weaknesses."

**The meta-rule:** every honest gap you volunteer buys credibility for every claim you make.

---

## Cross-References

| Need | File |
|---|---|
| Rapid recall before walking in | [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) |
| The stories in full | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Question banks | [17](17-Interview-Questions-Part-1.md) · [18](18-Interview-Questions-Part-2.md) |
| System design | [19-System-Design-Expansion.md](19-System-Design-Expansion.md) |
| Start over | [00-Table-of-Contents.md](00-Table-of-Contents.md) |

---

**Good luck. You built something real, you measured it, and you deleted the parts that didn't work. That's the story — tell it honestly and it's a strong one.**
