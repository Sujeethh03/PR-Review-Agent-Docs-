# 14 — Project Challenges

## Executive Summary

Seven real problems, told as **STAR stories** (Situation → Task → Action → Result). Every one is verifiable from the git history — these aren't invented.

**Challenge #1 (the critic agent) is the single most valuable story in this project.** It demonstrates measurement, judgement, and the willingness to delete your own work. Lead with it whenever you're asked about a hard decision, a failure, or a time you changed your mind.

| # | Challenge | Demonstrates | Priority |
|---|---|---|---|
| 1 | The critic agent rejected 87.5% of findings | Measurement-driven judgement, deleting your own work | 🔴 Lead with this |
| 2 | Parallel agents ran sequentially | Debugging silent performance bugs, async depth | 🔴 Critical |
| 3 | Works locally, fails in production (git binary) | DevOps, implicit dependencies | 🔴 Critical |
| 4 | The infinite webhook feedback loop | Event-driven system design | 🟠 High |
| 5 | Every confidence score was 1.0 | Prompt engineering, LLM calibration | 🟠 High |
| 6 | RAG retrieved the code being reviewed | RAG depth, subtle correctness | 🟠 High |
| 7 | One bad line number kills the whole review | API design, atomic failure | 🟡 Medium |

---

## Challenge 1 — The Critic Agent Rejected 87.5% of Findings

### Situation

The original architecture had **four** agents. Three specialists produced findings, and a fourth — the **critic** — reviewed those findings and decided which were worth showing a human. The reasoning was sound on paper: LLMs produce false positives, so add a validation layer.

Downstream of the critic sat a PostgreSQL findings store and a Next.js approval dashboard, where a human made the final call. Three components, one purpose: **don't show developers bad findings**.

### Task

Determine whether the critic actually improved output quality.

### Action

I ran the pipeline against test PRs with deliberately planted bugs and compared what the specialists found against what survived the critic.

**The result was stark: the critic rejected roughly 87.5% of findings — around seven out of every eight.** Worse, when I read the rejected ones, they included **real bugs I had personally planted**. Genuine null dereferences and hardcoded credentials, thrown out.

I looked at why. The critic was doing what LLMs do when you ask them to be a sceptical reviewer: **being sceptical is the easy, safe-looking behaviour.** With no ground truth and no consequence for rejecting a true positive, "I'm not certain this is a real issue" is always a defensible output. The critic had been optimised, by its prompt, into a rejection machine.

I also realised the architecture had a deeper problem. I'd built three components — critic, database, dashboard — to solve a problem I hadn't measured. I never established my false-positive rate *before* deciding I needed a filter for it.

### Result

I deleted all three in commit `a61bfd2`: the critic agent, the entire PostgreSQL layer (SQLAlchemy models, repository, migrations, API routes), and the whole Next.js frontend. Roughly a thousand lines of working code.

They were replaced by **one line**:

```python
CONFIDENCE_THRESHOLD = 0.70
to_post = [f for f in all_findings if f.confidence >= CONFIDENCE_THRESHOLD]
```

Findings above the threshold post directly to GitHub. The developer reviews them in GitHub's own UI, which was always better than my dashboard.

**Every dimension improved simultaneously:**

| Metric | Before | After |
|---|---|---|
| Findings surviving to the developer | ~12.5% | Everything ≥0.70 |
| Real bugs suppressed | Many | Far fewer |
| Components to maintain | 4 agents + DB + frontend | 3 agents |
| Latency | +1 LLM call, +human wait | Direct |
| Lines of code | ~1,000 more | Deleted |

### What I learned

> **Don't build a solution before measuring the problem.** I built a three-component filtering system for a false-positive rate I never measured. When I finally measured, the filter was doing more damage than the problem it was built for.
>
> **An LLM asked to be a sceptic will be a sceptic.** There was no asymmetry in the critic's incentives — rejecting a true positive cost it nothing, so it rejected freely. Any system where a component's failures are invisible to it will drift toward that failure.
>
> **Deleting working code is harder than writing it, and often more valuable.** I'd spent weeks on that dashboard. The sunk cost was real and I felt it. But keeping code alive because you're attached to the effort is exactly how systems rot.
>
> There's a residual honesty point: replacing a bad filter with a threshold is only *better*, not *good*. My 0.70 threshold is still a tuned guess, because I still don't have a labelled dataset. I traded a filter that was measurably harmful for one that's merely unmeasured. That's progress, not a solution.

**Interview framing:** Use this for "tell me about a difficult decision", "a time you failed", "a time you changed your mind", "how do you handle sunk cost", and "your proudest technical decision."

---

## Challenge 2 — The Parallel Agents Ran Sequentially

### Situation

The architecture was built for parallelism: LangGraph fanning out to three agents, each using `asyncio.gather` across all diff hunks. Everything was `async def`. Reviews were taking noticeably longer than expected — 45+ seconds for work I estimated at 15.

### Task

Find out why a parallel system was performing like a sequential one.

### Action

I instrumented each stage with timings. The three agents' durations **summed rather than overlapped** — bug finished at 15s, security at 30s, pattern at 45s. That pattern is only possible if they ran one after another.

But the code was unambiguously parallel. LangGraph had three edges from `__start__`. Each agent gathered over hunks. Every function was a coroutine.

The problem was one import:

```python
from openai import OpenAI          # ← synchronous client
_client = OpenAI(api_key=...)

async def _process_hunk(hunk, collection_name):
    response = _client.chat.completions.create(...)    # blocks the event loop
```

**`OpenAI` is the blocking client. `AsyncOpenAI` is the async one.** A blocking call inside a coroutine doesn't yield — it holds the single event loop thread for the entire HTTP request. Every "concurrent" agent politely waited its turn.

### Result

Switched to `AsyncOpenAI` everywhere — agents, suggester, and context retrieval. Commit `a61bfd2` records it: *"Switch all specialist agents and pr_writer to AsyncOpenAI so GPT-4o calls are non-blocking and the parallel fan-out actually runs concurrently."*

**Roughly 3× faster at the agent level.** And because each agent also gathers over N hunks, the real improvement was larger — with 20 hunks, "before" was effectively 60 sequential calls.

I also handled the one genuinely blocking dependency honestly. ChromaDB's client has no async variant, so instead of wrapping it in `async def` and pretending, I wrote ingestion as a plain sync function pushed to a worker thread:

```python
return await asyncio.to_thread(_run_ingestion_sync, ...)
```

### What I learned

> **`async` is not a performance feature — it's a cooperative concurrency model.** A coroutine only yields at an `await` on something that genuinely suspends. One blocking call is enough to serialise the entire loop.
>
> **The insidious part was that nothing failed.** No error, no warning, no exception. The output was completely correct. The only symptom was a wall-clock number that didn't match my mental model. Silent performance bugs are the hardest kind because there's nothing to grep for.
>
> **In async code you audit the library, not the syntax.** `async def` wrapped around a sync call is a lie that the type checker won't catch. Now the first question I ask about any SDK is whether there's a separate async client — `AsyncOpenAI` vs `OpenAI`, `httpx.AsyncClient` vs `httpx.Client`.
>
> Python's `asyncio` debug mode would have caught this — it warns when a callback blocks the loop longer than `slow_callback_duration`. I should have had it on in development.

**Interview framing:** "Tell me about a performance problem", "a bug that was hard to find", "explain async/await."

---

## Challenge 3 — Worked Locally, Failed in Production

### Situation

The pipeline ran perfectly on my machine. Deployed to Railway, every repository clone failed.

### Task

Diagnose a failure that only manifested in the production environment.

### Action

The error pointed at GitPython's clone call. The cause turned out to be a category of dependency I hadn't thought about.

```python
import git                                    # GitPython
git.Repo.clone_from(clone_url, clone_path, depth=1)
```

**GitPython is not a git implementation.** It's a Python wrapper that shells out to the `git` **binary**. `pip install gitpython` installs the Python package — and nothing else. It requires `git` to be present on the system.

My Mac has git. Railway's Python build image doesn't.

Fixing it took three attempts, because Railway distinguishes language packages from system packages:

```json
{"packages": ["git"]}        // c66322a — wrong key
{"packages": {"git": "*"}}   // d87d226 — right key, wrong type
{"aptPkgs": ["git"]}         // 16124db — correct
```

### Result

Clones worked. But the sequence of eight deployment commits told me something more useful than the fix.

### What I learned

> **I had an undeclared implicit system dependency.** My `requirements.txt` described my Python dependencies completely and my system dependencies not at all. That's a gap the tooling doesn't warn you about — pip has no concept of "this package needs a binary on PATH."
>
> **This is precisely the class of bug containers exist to prevent, and I'd undermined that.** By letting the platform's buildpack infer my image, I gave up the one thing containers offer: an explicit, reproducible environment. A Dockerfile with `RUN apt-get install -y git` sitting next to `pip install` would have made the dependency visible in code review, versioned in git, and identical everywhere.
>
> **The general principle: explicit beats inferred.** Buildpacks are wonderful until your dependencies aren't the ones they guessed. The convenience is real, and so is the cost of debugging when the guess is wrong. For anything I'd hand to a team, I'd own the Dockerfile.
>
> I'd also have caught this immediately by running the container locally before deploying. My local environment and my production environment differed in a way I never tested.

**Interview framing:** "Tell me about a production issue", "explain Docker's value", "works on my machine."

---

## Challenge 4 — The Infinite Webhook Feedback Loop

### Situation

The PR writer agent posts review comments to GitHub. Shortly after deploying it, the system started reviewing the same PR repeatedly.

### Task

Stop the runaway loop and understand why an event-driven system did this.

### Action

The cycle was immediate once I traced it:

```
Bot posts a review comment
  → GitHub fires pull_request_review_comment
    → My webhook receives it
      → Pipeline runs, posts another comment
        → GitHub fires another event → ∞
```

I'd built a system that **reacted to events it generated itself**. Each iteration cost a full pipeline run — clone, embed, three agents' worth of GPT-4o calls.

The fix was an explicit ignore list:

```python
if event_type in (
    "security_advisory", "github_app_authorization",
    "check_suite", "check_run", "status",
    "pull_request_review",              # ← the loop
    "pull_request_review_comment",      # ← the loop
    "pull_request_review_thread",       # ← the loop
):
    return {"status": "ok", "event": event_type}
```

### Result

Loop broken. Commit `021d87e`: *"silence review webhook feedback loop and fix cross-category dedup."*

### What I learned

> **In event-driven systems, always ask what events your own actions produce.** This is the fundamental hazard of the pattern: your outputs become inputs. A synchronous request/response system can't do this to you.
>
> **My fix works but it's blunt.** I ignore the event *type*, which means I also ignore review comments from **humans**. If a developer replies to one of my findings asking a question, I can't see it. That closes off a genuinely useful feature — the bot responding to questions about its own comments.
>
> The better fix filters on **actor identity** rather than event category:
> ```python
> if payload.get("sender", {}).get("login") == f"{APP_NAME}[bot]":
>     return {"status": "ok", "event": "self-generated"}
> ```
> That ignores my own events while still seeing humans'.
>
> Interestingly, **this is the same mistake I made in `_delete_bot_comments`** — filtering on `user.type == "Bot"` instead of on my specific bot's login, which means I delete every other bot's comments too. Same conceptual error twice: **filtering by category when I meant identity.** Noticing the pattern across two different bugs was more useful than either individual fix.
>
> The cost-control angle also matters: a runaway loop in an LLM system burns money continuously. A circuit breaker on "reviews per PR per hour" would be a sensible backstop independent of the event filtering.

**Interview framing:** "Tell me about a bug in a distributed system", "event-driven design pitfalls."

---

## Challenge 5 — Every Confidence Score Was 1.0

### Situation

The whole quality strategy depends on confidence scores — agents filter at 0.6, the global gate is 0.70. In early runs, essentially every finding came back with confidence **1.0**.

### Task

Get usable calibration out of the model, or find a different quality signal.

### Action

The threshold was filtering nothing. If everything is 1.0, `>= 0.70` passes everything, and the confidence architecture is decorative.

The cause is a known LLM property: **models are poorly calibrated on self-reported confidence.** Asked "how confident are you", they produce whatever token pattern matches confident-sounding output in their training distribution. That skews high.

I rewrote the confidence section of every system prompt with three distinct mechanisms:

```
Confidence scoring guide:
  1.0 — certain: present and exploitable as written
  0.8 — likely: strong evidence, minor uncertainty about execution context
  0.6 — possible: plausible but depends on caller or framework configuration
  Below 0.6 — do not include in output

Do NOT default to 1.0. Reserve 1.0 only for hardcoded_secret findings and for
vulnerabilities that are unambiguously exploitable with no caveats. Most findings
should be 0.6 or 0.8.
```

1. **Anchored examples** — a description for each level so the model has reference points rather than a bare number line.
2. **An explicit prohibition** — "Do NOT default to 1.0."
3. **An expected distribution** — "Most findings should be 0.6 or 0.8."

That third one turned out to matter most. Nudging the *distribution* worked better than instructing on individual scores.

I also added a deterministic override for the one case where I genuinely want 1.0:

```python
def _enforce_secret_rules(findings):
    return [f.model_copy(update={"severity": "high", "confidence": 1.0})
            if f.category == "hardcoded_secret" else f for f in findings]
```

### Result

Usable spread across 0.6, 0.8, and 1.0. Commit `a61bfd2`: *"Tighten confidence scoring prompts to prevent all-1.0 outputs."*

### What I learned

> **LLM confidence is not calibrated probability.** When a model says 0.8, that does not mean it's right 80% of the time. It's a token, not a posterior. Treating it as probability is a category error — but it's still *useful* as a relative ranking signal, which is all my threshold needs.
>
> **Prompting the distribution beats prompting the value.** "Most findings should be 0.6 or 0.8" shifted output more effectively than any instruction about individual scores. That's a genuinely transferable prompt engineering technique.
>
> **Never rely on a prompt for an invariant you can enforce in code.** The prompt already said hardcoded secrets must be 1.0. I enforce it in Python anyway, because prompts are suggestions and models drift across versions. Anything I can't afford to have wrong shouldn't depend on the model complying.
>
> **And the honest limitation:** spread is not calibration. I got a usable distribution, but I still can't tell you my precision at 0.70 versus 0.80, because I have no labelled dataset. The threshold is a tuned heuristic. To fix it properly I'd need ground truth — and the practical source is developer behaviour: did they resolve the comment, apply the suggestion, or dismiss it? That's an implicit label I could collect from the GitHub API and currently don't.

**Interview framing:** "Tell me about prompt engineering", "how do you evaluate LLM output", "working with non-deterministic systems."

---

## Challenge 6 — RAG Retrieved the Code Being Reviewed

### Situation

The pattern agent — which compares new code against codebase conventions — was producing weak, generic findings. It rarely identified genuine inconsistencies.

### Task

Work out why an agent with vector search over the whole repository wasn't finding anything to compare against.

### Action

I logged the chunks actually retrieved for each hunk. The top hits were almost always **the pre-change version of the same function**, sitting in the index from the previous ingestion.

Obvious in hindsight: the most semantically similar code to a modified function *is* that function. The vector search was working perfectly and returning exactly the wrong thing.

So the agent was being asked "does this code follow the codebase's conventions?" while being handed **the code itself** as its comparison set. Of course it found nothing.

The fix — an interval-overlap filter on retrieval results:

```python
for doc, meta in zip(results["documents"][0], results["metadatas"][0]):
    if (meta.get("file") == hunk.file
        and meta.get("start_line", 0) <= hunk.line_end
        and meta.get("end_line", 0) >= hunk.line_start):
        continue        # skip — this chunk IS the code under review
    chunks.append({...})
```

I also gave the pattern agent twice the context (`n_results=10` vs 5) and made it **skip hunks entirely** when retrieval returns nothing:

```python
context_chunks = await get_context_for_hunk(hunk, collection_name, n_results=10)
if not context_chunks:
    return []      # no basis for comparison → stay silent
```

### Result

The pattern agent started producing genuine findings — actual deviations from established conventions rather than generic observations.

### What I learned

> **In RAG, what you exclude matters as much as what you retrieve.** The naive implementation is "embed the query, take the top-k." The top-k is frequently dominated by near-duplicates of the query itself, and near-duplicates carry zero information.
>
> **The abstention design is the part I'm most pleased with.** Making the pattern agent stay silent when it has no context is the opposite of what an LLM naturally does — models want to answer. Most false positives in LLM systems come from models producing output when they should decline. Building "no evidence, no judgement" into the control flow rather than the prompt makes it a guarantee rather than a hope.
>
> **A residual issue I'd fix:** the filter runs *after* retrieval, so if three of five results are self-matches I'm left with two chunks of context rather than five. I should over-fetch — request ten, filter, take the top five — or push the exclusion into ChromaDB's `where` clause so the index does the work.

**Interview framing:** "Explain RAG", "how do you improve retrieval quality", "a subtle bug you found."

---

## Challenge 7 — One Bad Line Number Kills the Whole Review

### Situation

Occasionally a review would post as a single flat summary comment instead of ten inline comments on the diff — dramatically less useful, since the developer loses all code context.

### Task

Understand the intermittent degradation.

### Action

GitHub's review API accepts a batch of inline comments and **validates them atomically**. If any single comment targets a line not present in the diff, the entire request returns `422 Unprocessable Entity` and **zero comments post**.

Line numbers come from GPT-4o. Occasionally it reports a line just outside the hunk — off by a few, or referencing a line it inferred from context. One such finding rejects all ten.

My existing fallback catches the 422 and posts a summary body comment instead:

```python
if response.status_code == 422:
    return False            # → _post_body_comment()
```

So it degrades rather than failing — but it degrades *everything*, not just the offending finding.

### Result

Currently mitigated, not fixed. It's the top item on my roadmap, and the reason is that **the data to prevent it entirely already exists**.

`parse_diff_hunks()` returns exactly which lines are in the diff — I use it to *build* the prompts. I simply never cross-check the LLM's output against it before posting:

```python
def validate_finding_lines(findings: list[Finding], diff: str) -> list[Finding]:
    valid = [(h.file, h.line_start, h.line_end) for h in parse_diff_hunks(diff)]
    kept = []
    for f in findings:
        if any(f.file == file and start <= f.line_end <= end for file, start, end in valid):
            kept.append(f)
        else:
            logger.warning("finding.line_out_of_diff", file=f.file, line=f.line_end)
    return kept
```

### What I learned

> **Atomic batch APIs turn one bad item into total failure.** That's a general hazard worth recognising: whenever an API validates a batch as a unit, your error isolation is only as good as your input validation.
>
> **I had the validation data and didn't use it.** That's the part that stings. I parse the diff into precise line ranges to build the prompts, and then trust the model's line numbers blindly on the way out. Validating at the boundary where untrusted data leaves my system is a principle I applied on the *input* side — HMAC verification, Pydantic on LLM output — and forgot on the output side.
>
> **Graceful degradation can hide the need for a real fix.** My fallback worked well enough that the problem stayed low-priority for a while. A degraded path that's *too* comfortable removes the pressure to fix the underlying cause. I'd rather have loud failure and a real fix than quiet degradation.

**Interview framing:** "A limitation you know about", "how do you handle third-party API constraints", "technical debt."

---

## How to Deploy These Stories

| Question | Use |
|---|---|
| "Tell me about a challenge" | #1 (critic) |
| "Tell me about a failure" | #1 — you built the wrong thing and deleted it |
| "A time you changed your mind" | #1 |
| "A difficult technical decision" | #1 |
| "A performance problem" | #2 (false parallelism) |
| "A hard-to-find bug" | #2 or #6 |
| "Explain async/await" | #2 |
| "A production issue" | #3 (git binary) |
| "Why containers?" | #3 |
| "Distributed systems pitfalls" | #4 (feedback loop) |
| "Prompt engineering" | #5 (confidence calibration) |
| "Evaluating LLM output" | #5 |
| "Explain RAG deeply" | #6 (self-match) |
| "Known limitations" | #7 (line numbers) |
| "Technical debt" | #7 |

### The universal STAR structure

1. **Situation** — one sentence of context
2. **Task** — what you had to determine or fix
3. **Action** — what you *specifically* did (use "I", not "we")
4. **Result** — quantified where possible
5. **Learning** — the transferable principle

**Keep it to 2–3 minutes.** Interviewers will ask follow-ups if they want depth; the mistake is front-loading everything.

### Why these stories work

Every one has:
- ✅ A **number** (87.5%, 3×, 8 commits, 1.0 confidence)
- ✅ A **verifiable commit** in git history
- ✅ A **transferable principle**, not just a fix
- ✅ An **honest residual limitation** you volunteer

That last point is what distinguishes senior candidates. Every story here ends with something still imperfect, named without prompting.

---

## Cross-References

| Topic | File |
|---|---|
| The decisions behind these problems | [15-Design-Decisions.md](15-Design-Decisions.md) |
| Async and parallelism detail | [12-Performance-Optimization.md](12-Performance-Optimization.md) |
| RAG internals | [09-AI-System.md](09-AI-System.md) |
| Deployment specifics | [10-Deployment-and-DevOps.md](10-Deployment-and-DevOps.md) |
| Behavioural question bank | [18-Interview-Questions-Part-2.md](18-Interview-Questions-Part-2.md) |
| Fixes for these issues | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [15-Design-Decisions.md](15-Design-Decisions.md)
