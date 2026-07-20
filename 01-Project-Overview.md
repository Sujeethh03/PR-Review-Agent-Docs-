# 01 — Project Overview

## Executive Summary

**PR Review Agent (CR-001)** is an event-driven AI service that automatically reviews GitHub pull requests. When a PR is opened or updated, three specialist LLM agents analyse the diff in parallel — hunting bugs, security vulnerabilities, and convention violations — and post structured inline comments with one-click fix suggestions directly onto the diff.

It is a **RAG-powered, multi-agent, webhook-driven backend service** with no user interface of its own: it deliberately borrows GitHub's review UI instead of building one.

---

## 1. Problem Statement

### The human cost of code review

Code review is simultaneously the highest-value and most-neglected activity in software engineering.

**The evidence:**
- Defects caught in review cost roughly **1×** to fix. The same defect caught in production costs **30–100×** (IBM Systems Sciences Institute).
- Reviewers' defect-detection ability drops sharply after ~400 lines of code and after ~60 minutes of reviewing (SmartBear study of 2,500 reviews at Cisco).
- Median time-to-first-review in many organisations is measured in **hours to days**, during which the author has context-switched away and lost the mental model of their own change.

**The three specific failures this project targets:**

| Failure | Description | Consequence |
|---|---|---|
| **Latency** | Author waits hours for feedback | Context loss; expensive re-learning; PR pile-up |
| **Inconsistency** | Reviewer A blocks on a nit that Reviewer B ignores | Standards erode; reviews feel arbitrary and political |
| **Fatigue** | Reviewer 12 of the day skims and LGTMs | Real bugs ship; review becomes theatre |

### The specific gap: context-blind tooling

Existing automated tools mostly analyse the **diff in isolation**. But a huge class of real defects is only visible when you know the *rest of the codebase*:

```python
# The diff — looks perfectly fine on its own
def get_user(user_id):
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")
```

A diff-only linter may or may not flag the f-string. But it *cannot* tell you the thing that actually matters:

> "Every other query in this codebase uses the parameterised `db.query_safe(sql, params)` helper defined in `db/safe.py`. This one doesn't. It is both an SQL injection risk **and** a convention violation."

That second sentence requires knowledge of the whole repository. **Supplying that knowledge to an LLM is the core technical problem this project solves** — via retrieval-augmented generation over a per-repository vector index.

---

## 2. Existing Solutions & Why They Fall Short

| Solution | How it works | Strengths | Weaknesses |
|---|---|---|---|
| **Human review** | Peer reads the diff | Deep intent understanding; mentoring; accountability | Slow, inconsistent, fatigues, doesn't scale |
| **Linters** (ESLint, Pylint, RuboCop) | AST pattern matching against fixed rules | Fast, deterministic, zero cost, no false-positive drift | Only knows pre-written rules; no semantic understanding; cannot reason about intent |
| **SAST** (SonarQube, Semgrep, CodeQL) | Data-flow / taint analysis | Deep security analysis; low false negatives on known classes | Expensive to configure; noisy; whole-repo scans are slow; rules are static |
| **GitHub Copilot / Code Review** | LLM reads the diff | Semantic understanding; natural language output | Largely diff-scoped; limited whole-repo convention awareness |
| **This project** | RAG + specialised multi-agent LLM | Whole-repo context; role-specialised prompts; educational output; explicit confidence gating | LLM cost per PR; non-deterministic; needs vector index warm-up |

### The positioning statement

> Linters know the **rules**. SAST tools know the **data flow**. LLMs know **language**. None of them know **your codebase**. This project gives an LLM your codebase as retrievable memory, then splits the reviewing job across three specialists so each one does a narrow task well.

---

## 3. Why This Project Exists

Three motivations, in order of honesty:

1. **Technical**: Demonstrate a production-shaped, event-driven, multi-agent RAG system end-to-end — webhook ingress, async orchestration, vector retrieval, LLM fan-out, third-party API egress.
2. **Practical**: Give a solo developer or small team the "second reviewer" they don't have, within ~1–2 minutes of pushing.
3. **Educational**: Every comment the agent posts teaches. A finding isn't just "SQL injection here" — it carries the OWASP category, a plain-English impact statement, a reference link, and a one-click fix. The tool is designed to make the developer better, not just the code.

That third point is a genuine differentiator and worth mentioning in interviews. See `education.py` — a hand-curated map of 15 vulnerability categories to OWASP Top 10 references with written impact statements.

---

## 4. Features

### ✅ Implemented (verified in code)

| # | Feature | Implementation | File |
|---|---|---|---|
| 1 | **GitHub webhook receiver** | FastAPI `POST /webhook`, HMAC-SHA256 verified | `main.py:114` |
| 2 | **Signature verification** | `hmac.compare_digest` — timing-attack safe | `main.py:26` |
| 3 | **Immediate 200 + background processing** | `asyncio.create_task` so GitHub never times out | `main.py:161` |
| 4 | **GitHub App authentication** | RS256 JWT → installation access token | `github_client.py:9` |
| 5 | **PR diff fetch** | `Accept: application/vnd.github.diff` | `github_client.py:46` |
| 6 | **Shallow repo clone** | `depth=1` at exact head SHA, auto-cleaned | `repo_manager.py:25` |
| 7 | **Semantic code chunking** | tree-sitter AST → functions/classes/methods | `parser.py:64` |
| 8 | **Line-window fallback** | 60-line windows, 15-line overlap, for unparseable files | `parser.py:106` |
| 9 | **Batched embedding with retry** | 100 chunks/call, 1 retry with backoff | `embedder.py:16` |
| 10 | **Per-repo vector store** | ChromaDB collection `{owner}__{repo}`, cosine | `store.py:17` |
| 11 | **Full vs incremental ingestion** | First PR = whole repo; later = changed files only | `agent.py:46` |
| 12 | **Deleted-file purging** | Vector rows removed when files are deleted | `store.py:73` |
| 13 | **Parallel multi-agent fan-out** | LangGraph `__start__` → 3 nodes → `END` | `graph.py:27` |
| 14 | **Per-hunk parallelism** | `asyncio.gather` across all hunks inside each agent | `bug/agent.py:108` |
| 15 | **RAG context retrieval** | Vector search per hunk, self-match filtered out | `context.py:25` |
| 16 | **Forced structured output** | OpenAI tool calling with `tool_choice` forced | `bug/agent.py:93` |
| 17 | **Hardcoded-secret override** | Deterministic rule beats LLM judgement | `security/agent.py:49` |
| 18 | **Two-pass deduplication** | By (file, line, category), then by (file, line) | `findings.py:29` |
| 19 | **Confidence gating** | Drop everything below 0.70 | `main.py:85` |
| 20 | **Inline GitHub review comments** | Multi-line ranges with `start_line`/`side` | `pr_writer/agent.py:66` |
| 21 | **One-click fix suggestions** | GPT-4o generates ` ```suggestion ` blocks | `suggester.py:25` |
| 22 | **OWASP educational context** | 15-category curated reference map | `education.py` |
| 23 | **Stale comment cleanup** | Old bot comments deleted before re-review | `pr_writer/agent.py:10` |
| 24 | **Graceful degradation** | Inline fails → summary body comment | `pr_writer/agent.py:81` |
| 25 | **Webhook feedback-loop guard** | Ignores its own review events | `main.py:125` |

### 🗑️ Removed (built, measured, deleted in `a61bfd2`)

| Feature | Why removed |
|---|---|
| **Critic agent** | Rejected ~87.5% of findings including true positives. Net negative. |
| **PostgreSQL findings store** | Only existed to feed the dashboard. No dashboard → no need. |
| **Next.js approval dashboard** | GitHub's native review UI is better and already where developers work. |

### 📚 Not built (be honest about these)

Redis caching · user accounts/login · rate limiting · multi-tenancy · billing · analytics dashboard · CI/CD pipeline · Dockerfile.

---

## 5. User Flow

### Primary flow — the only one that exists

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant GH as GitHub
    participant API as FastAPI /webhook
    participant BG as Background Task
    participant AI as AI Pipeline

    Dev->>GH: git push / open PR
    GH->>API: POST /webhook (pull_request.opened)
    API->>API: Verify HMAC-SHA256
    API-->>GH: 200 OK (in ~5ms)
    Note over API,GH: Critical: respond before GitHub's 10s timeout
    API->>BG: asyncio.create_task(pipeline)
    BG->>GH: GET PR diff
    BG->>GH: git clone --depth 1
    BG->>AI: Ingest → embed → store vectors
    BG->>AI: 3 agents fan out in parallel
    AI-->>BG: Findings
    BG->>BG: Dedup + filter (conf >= 0.70)
    BG->>GH: POST inline review comments
    GH-->>Dev: 🔔 Notification
    Dev->>GH: Read finding, click "Apply suggestion"
    Dev->>GH: git push (fix)
    GH->>API: POST /webhook (synchronize)
    Note over API,AI: Cycle repeats; stale comments deleted first
```

**Wall-clock experience:** developer pushes → notification arrives in roughly **30–90 seconds** depending on diff size and whether the repo needs a full first-time ingest.

### The zero-UI insight

The user flow contains **no screens built by this project**. Everything the developer sees is GitHub. This was deliberate:

> Developers already live in the GitHub PR page. A separate dashboard means a second tab, a second login, a second notification channel, and a second place to forget about. Meeting users where they already are beat building our own UI.

This is a strong product-thinking answer. See [06-Frontend-Architecture.md](06-Frontend-Architecture.md).

---

## 6. Functional Requirements

| ID | Requirement | Priority | Status | Verification |
|---|---|---|---|---|
| FR-01 | Receive GitHub webhooks for PR events | Must | ✅ | `main.py:114` |
| FR-02 | Reject webhooks with invalid signatures (401) | Must | ✅ | `main.py:119` |
| FR-03 | Respond 200 before processing completes | Must | ✅ | `main.py:161` |
| FR-04 | Authenticate as a GitHub App | Must | ✅ | `github_client.py` |
| FR-05 | Fetch the unified diff of a PR | Must | ✅ | `github_client.py:46` |
| FR-06 | Clone the repo at the exact head SHA | Must | ✅ | `repo_manager.py` |
| FR-07 | Chunk source semantically across ≥5 languages | Must | ✅ (8 languages) | `parser.py` |
| FR-08 | Embed and store chunks per repository | Must | ✅ | `store.py` |
| FR-09 | Re-ingest incrementally on subsequent PRs | Should | ✅ | `agent.py:46` |
| FR-10 | Detect bugs (null refs, logic, leaks) | Must | ✅ | `bug/` |
| FR-11 | Detect security vulns (OWASP Top 10) | Must | ✅ | `security/` |
| FR-12 | Detect convention violations vs codebase | Should | ✅ | `pattern/` |
| FR-13 | Run all three agents concurrently | Must | ✅ | `graph.py` |
| FR-14 | Deduplicate overlapping findings | Must | ✅ | `findings.py:29` |
| FR-15 | Suppress low-confidence findings | Must | ✅ | `main.py:85` |
| FR-16 | Post findings as inline diff comments | Must | ✅ | `pr_writer/agent.py` |
| FR-17 | Include applicable one-click fixes | Should | ✅ | `suggester.py` |
| FR-18 | Include educational OWASP context | Could | ✅ | `education.py` |
| FR-19 | Remove stale comments on re-review | Should | ✅ | `pr_writer/agent.py:10` |
| FR-20 | Never modify user code | Must | ✅ | Read-only by design |
| FR-21 | Human approval queue before posting | Was Must | 🗑️ Removed | Replaced by threshold |

**Note FR-21.** Requirements changing mid-project is a classic interview question ("tell me about a time requirements changed"). This is your real example — see [18-Interview-Questions-Part-2.md](18-Interview-Questions-Part-2.md).

---

## 7. Non-Functional Requirements

| ID | Category | Requirement | Achieved? | How |
|---|---|---|---|---|
| NFR-01 | **Latency** | Webhook ACK < 1s | ✅ ~5ms | Background task offload |
| NFR-02 | **Latency** | Review posted < 3 min | ✅ ~30–90s | Full parallelism |
| NFR-03 | **Security** | Reject forged webhooks | ✅ | HMAC + `compare_digest` |
| NFR-04 | **Security** | No secrets in repo | ✅ | `.gitignore`, env vars, base64 key |
| NFR-05 | **Security** | Never ingest secret files | ✅ | `.env`/`.pem`/`.key` skip list |
| NFR-06 | **Security** | Least-privilege GitHub perms | ✅ | Read code, write PR comments only |
| NFR-07 | **Reliability** | One agent failure ≠ total failure | ✅ | Per-agent try/except + `return_exceptions=True` |
| NFR-08 | **Reliability** | Pipeline failure never 500s the webhook | ✅ | `_safe_run_pipeline` wrapper |
| NFR-09 | **Reliability** | Clean up disk after clone | ✅ | `finally: shutil.rmtree` |
| NFR-10 | **Accuracy** | Minimise false positives | ⚠️ Partial | Dual threshold (0.6 → 0.70); no ground-truth measurement |
| NFR-11 | **Determinism** | Same diff → same review | ⚠️ Partial | `temperature=0`, but LLMs aren't bit-deterministic |
| NFR-12 | **Cost** | Bounded spend per PR | ❌ Not enforced | No token budget or concurrency cap — known gap |
| NFR-13 | **Maintainability** | Add an agent without touching others | ✅ | Uniform agent interface + LangGraph node |
| NFR-14 | **Observability** | Trace a PR through the pipeline | ⚠️ Weak | `print()` only, no structured logging |
| NFR-15 | **Scalability** | Handle concurrent PRs | ⚠️ Limited | Single process, in-memory tasks — see [13](13-Scalability.md) |

**Being able to say "NFR-12 and NFR-14 are not met, and here's exactly what I'd do" is worth more than claiming everything is green.**

---

## 8. Constraints

### Technical constraints

| Constraint | Impact | Mitigation chosen |
|---|---|---|
| GitHub webhook timeout (~10s) | Cannot process synchronously | Immediate 200 + background task |
| GitHub App JWT max life 10 min | Must regenerate constantly | 9-min expiry, generated per call |
| Installation token life 1 hour | Cannot cache long-term | Fetched fresh per operation |
| OpenAI rate limits (RPM/TPM) | Unbounded fan-out will 429 | ⚠️ Not yet mitigated — known gap |
| LLM context window | Cannot send whole repo | RAG: retrieve top-5/10 chunks only |
| GitHub inline comments must target diff lines | Hallucinated line → 422 | Fallback to summary comment |
| Python 3.13 | `tree-sitter-languages` incompatible | Individual `tree-sitter-*` packages |
| Railway ephemeral filesystem | `/tmp/chromadb` wiped on deploy | ⚠️ Accepted; volume would fix |
| Railway build image lacks git | GitPython clone fails | `railpack.json` → `aptPkgs: ["git"]` |

### Business / resource constraints

- **Solo developer** — every hour spent on a dashboard is an hour not spent on review quality. Directly motivated deleting the frontend.
- **Personal OpenAI budget** — motivated `temperature=0`, batching, the small embedding model, and the confidence filter (fewer suggester calls).
- **No labelled dataset** — no ground-truth corpus of "real bugs", so precision/recall could not be measured numerically. This is the honest reason NFR-10 is only partially met.

---

## 9. Assumptions

Listing assumptions explicitly is a strong senior-engineer signal — it shows you know where the system's edges are.

| # | Assumption | If false, what happens | Risk |
|---|---|---|---|
| A1 | Repos fit in memory/disk for shallow clone | Timeouts on huge monorepos | Medium |
| A2 | PRs are reasonably sized (<1000 lines) | Cost & rate limits explode | **High** |
| A3 | OpenAI API is available | Pipeline fails; no queue/retry | **High** |
| A4 | GPT-4o line numbers are approximately correct | 422 → all inline comments degrade to one summary | **High** |
| A5 | ChromaDB index survives between PRs | Silent full re-ingest; extra cost | Medium (fails on Railway) |
| A6 | One installation ≈ one trusted tenant | No isolation between installations | Medium |
| A7 | Developers want automated comments | Tool gets muted/uninstalled | Medium |
| A8 | Vector similarity ≈ semantic relevance | Poor context → weaker findings | Medium |
| A9 | Codebase conventions are consistent enough to learn | Pattern agent produces noise | Medium |
| A10 | Only *this* bot comments on the PR | Other bots' comments get deleted (real bug) | **High** |

Assumptions **A4** and **A10** correspond to confirmed defects — see [20-Future-Improvements.md](20-Future-Improvements.md).

---

## 10. Interview Questions

### Q1. "Tell me about your project in two minutes."

**Best answer:**
> PR Review Agent is an AI service that reviews GitHub pull requests automatically. When you open a PR, a GitHub webhook hits my FastAPI backend. I verify the HMAC signature, return 200 immediately so GitHub doesn't time out, and run the actual work in a background task.
>
> That task does four things. First, it fetches the diff and shallow-clones the repo at the head SHA. Second, it builds a semantic index: tree-sitter parses every file into functions and classes, OpenAI embeds those chunks, and they go into a per-repository ChromaDB collection. Third — and this is the core — a LangGraph state machine fans out to three specialist GPT-4o agents that run concurrently: one for bugs, one for security, one for codebase conventions. Each agent, for each diff hunk, does a vector search to pull the five most relevant chunks from the rest of the repo, so it's reviewing with whole-codebase context rather than just the diff. Fourth, the findings get deduplicated, filtered at a 0.70 confidence threshold, and posted as inline GitHub comments with one-click fix suggestions and OWASP references.
>
> The decision I'm proudest of is a deletion. I originally had a fourth "critic" agent plus a PostgreSQL store and a Next.js approval dashboard. When I measured the critic, it was rejecting about 87.5% of findings — including real bugs. So I deleted all three and replaced them with a single confidence threshold, using GitHub's own review UI. The system got simpler, faster, and more accurate at the same time.

**What's being evaluated:** Can you compress a complex system without losing precision? Do you have judgement, or just feature lists?

**Common mistakes:** Listing technologies instead of explaining flow. Going over 3 minutes. Not mentioning trade-offs.

**Follow-ups:** "Why three agents and not one?" · "Why did the critic fail?" · "What breaks at scale?"

---

### Q2. "What problem does this solve that a linter doesn't?"

**Beginner:** Linters only catch pre-written rule violations; my system uses an LLM so it understands meaning.

**Intermediate:** Linters do AST pattern matching against a fixed ruleset — extremely good at "this variable is unused", useless at "this function duplicates logic in another module". Mine embeds the whole repository and retrieves relevant context, so it can reason about the change *relative to the codebase*.

**Advanced:** They occupy different points on the precision/recall–generality curve. A linter is high precision, zero generality: it catches exactly what it was programmed to catch, deterministically, at zero marginal cost. An LLM reviewer is lower precision but unbounded generality — it can flag issues nobody wrote a rule for. They're complementary, not competing: you'd run the linter in CI as a blocking gate and the LLM as advisory commentary. My confidence threshold exists precisely to manage the precision problem — I'm trading recall for precision because a noisy reviewer gets ignored, and an ignored reviewer has zero value regardless of its recall.

**Follow-ups:** "How do you measure precision?" (Honest answer: I don't — no labelled dataset. That's NFR-10 and it's a real gap.) · "Would you block merges on this?" (No — advisory only, because precision isn't measured.)

---

### Q3. "Why did you delete the dashboard? Isn't a UI valuable?"

**Best answer:**
> The dashboard existed to serve a workflow — human approval of every finding before posting — that I removed. Once findings post automatically above a confidence threshold, there's nothing left to approve, so the dashboard had no job.
>
> Even if I'd kept human approval, GitHub's PR review UI is better than anything I'd build: it shows comments inline on the diff with full syntax highlighting, threads replies, supports one-click suggestion application, and it's already where developers work. Building my own meant a second tab, a second auth system, and a second notification channel to ignore.
>
> The deeper principle: I deleted a component because its purpose disappeared, not because it was badly built. Keeping code alive out of attachment to the effort spent is sunk-cost reasoning.

**What's being evaluated:** Product judgement and non-attachment to your own code.

---

### Q4. "What are the biggest risks in this system?"

**Best answer:** Name three real ones with mitigations:

1. **False positives eroding trust.** One confidently wrong comment costs more credibility than ten correct ones earn. Mitigated by dual confidence thresholds (0.6 internal, 0.70 to post) and by making the tool advisory rather than blocking. Not fully solved — I have no labelled dataset, so precision is unmeasured.
2. **Unbounded cost and rate limits.** A 50-hunk PR fires 150 concurrent GPT-4o calls with no semaphore. That will 429 and it will be expensive. The fix is a bounded worker pool plus per-PR token budget — that's the top item on my roadmap.
3. **Line-number drift causing total degradation.** GitHub rejects the entire review with 422 if any single comment targets a line outside the diff. So one hallucinated line number collapses all inline comments into a single flat summary. The fix is validating each finding's line against the parsed hunks before posting.

**Why this scores well:** Volunteering specific, quantified weaknesses with named fixes demonstrates ownership. Candidates who claim no weaknesses signal they haven't looked.

---

## 11. Real-World Analogy

Think of a hospital emergency department:

| Hospital | This system |
|---|---|
| Patient arrives at reception | Webhook hits `/webhook` |
| Reception checks insurance card | HMAC signature verification |
| "Take a seat, you're admitted" | Immediate 200 response |
| Patient's medical history retrieved | Repo cloned & vector-indexed |
| Three specialists examine in parallel | Bug / Security / Pattern agents |
| Cardiologist, neurologist, radiologist | Each has narrow deep expertise |
| Consultants compare notes, merge duplicates | Deduplication pass |
| Only confident diagnoses go on the chart | 0.70 confidence threshold |
| Chart written in the patient's own file | Inline comments on the PR |
| Patient gets a leaflet explaining the condition | OWASP educational context |

The analogy also explains **why three agents beat one**: you don't want a generalist skimming for everything at once. Specialisation raises depth per unit of attention — which is exactly what a focused system prompt does for an LLM.

---

## 12. Cross-References

| To learn | Read |
|---|---|
| How components connect | [02-System-Architecture.md](02-System-Architecture.md) |
| The AI/RAG internals | [09-AI-System.md](09-AI-System.md) |
| Why each decision was made | [15-Design-Decisions.md](15-Design-Decisions.md) |
| The hard problems and how they were solved | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Line-by-line code | [16-Code-Walkthrough.md](16-Code-Walkthrough.md) |
| Rapid revision | [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) |

**Next:** [02-System-Architecture.md](02-System-Architecture.md)
