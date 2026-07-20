# 00 — Table of Contents & How to Use This Knowledge Base

## Executive Summary

This is a complete, interview-oriented knowledge base for **PR Review Agent (CR-001)** — a multi-agent AI system that automatically reviews GitHub pull requests. It is written to teach, not merely to document. Every file is self-contained: you can open file 09 without reading 01–08 and still follow it.

**Read this page first.** It contains the single most important thing in the whole knowledge base: an honest statement of what this project *is* and what it *is not*, so you never get caught out in an interview.

---

## ⚠️ CRITICAL: What this project actually is

Many project-doc templates assume a "standard full-stack CRUD app": React frontend, PostgreSQL, JWT login, Redis cache, Docker on Render/Vercel. **This project is not that**, and pretending otherwise is the fastest way to fail an interview.

Here is the ground truth, verified against the source code:

| Commonly assumed | Reality in this repo | Where to read about it |
|---|---|---|
| Next.js / React frontend | **Does not exist.** A Next.js dashboard was built, then deliberately deleted in commit `a61bfd2`. The UI is now GitHub's own PR review interface. | [06-Frontend-Architecture.md](06-Frontend-Architecture.md) |
| PostgreSQL relational DB | **Does not exist.** A SQLAlchemy + PostgreSQL findings store was built, then deleted in `a61bfd2`. `backend/alembic/` is a dead leftover. | [04-Database-Design.md](04-Database-Design.md) |
| JWT / OAuth user login | **No human users exist.** There is JWT — but it is **RS256 GitHub App JWT for machine-to-machine auth**, not user sessions. No login, no passwords, no bcrypt. | [07-Authentication-and-Security.md](07-Authentication-and-Security.md) |
| Redis caching | **Not present.** Caching is done with in-process singletons and the ChromaDB vector store. | [12-Performance-Optimization.md](12-Performance-Optimization.md) |
| Docker + Render + Vercel | **Railway**, via `railpack.json` + `railway.toml`. No Dockerfile in the repo. | [10-Deployment-and-DevOps.md](10-Deployment-and-DevOps.md) |
| pytest test suite | Two **standalone scripts** (`test_ingestion.py`, `test_specialist.py`) that print and assert. Not pytest, not CI-runnable. | [11-Testing.md](11-Testing.md) |

### Why this is an *advantage*, not a weakness

Interviewers are far more impressed by **"we built it, measured it, and removed it"** than by feature lists. This project has exactly that story:

> We built a critic agent to filter findings. In testing it rejected roughly **87.5% of findings, including real bugs**. We measured that, concluded the critic was a net negative, and deleted it along with the database and dashboard it fed — replacing all three with a single confidence threshold and GitHub's native review UI.

That is a senior-engineer narrative: **measurement → judgement → deletion**. Deleting code you wrote is harder than writing it. Lead with this story. It is covered in depth in [14-Project-Challenges.md](14-Project-Challenges.md) and [15-Design-Decisions.md](15-Design-Decisions.md).

---

## The one-paragraph project description

> PR Review Agent is an event-driven AI service. A GitHub App webhook fires when a pull request is opened or updated. A FastAPI receiver validates the HMAC-SHA256 signature, returns `200` within milliseconds, and hands the work to a background task. That task fetches the PR diff, shallow-clones the repository at the head SHA, parses every file into semantic chunks with tree-sitter, embeds them with OpenAI `text-embedding-3-small`, and stores them in a per-repository ChromaDB collection. A LangGraph state machine then fans out to three specialist GPT-4o agents — bug, security, and pattern — which run concurrently, each retrieving relevant codebase context via vector search (RAG) for every diff hunk. Their findings are deduplicated, filtered at a 0.70 confidence threshold, and posted back as inline GitHub review comments with one-click fix suggestions and OWASP educational context.

Memorise the shape of that paragraph. Expanded versions (30-second, 2-minute, 5-minute, 10-minute) are in [22-Final-Revision-Notes.md](22-Final-Revision-Notes.md).

---

## Full file index

### Foundation — read these first
| File | What it covers | Priority |
|---|---|---|
| [01-Project-Overview.md](01-Project-Overview.md) | Problem statement, existing solutions, features, user flow, functional & non-functional requirements, constraints, assumptions | 🔴 Critical |
| [02-System-Architecture.md](02-System-Architecture.md) | High-level architecture, 9 Mermaid diagrams, every component and arrow explained | 🔴 Critical |
| [03-Low-Level-Design.md](03-Low-Level-Design.md) | Class diagrams, module boundaries, data structures, algorithms (diff parsing, chunking, dedup) | 🟠 High |

### The stack, layer by layer
| File | What it covers | Priority |
|---|---|---|
| [04-Database-Design.md](04-Database-Design.md) | **Vector DB (ChromaDB) as the real datastore** + full relational/SQL theory: ER diagrams, normalization, indexes, transactions, isolation levels, locks, connection pooling, N+1 — taught properly because interviewers *will* ask | 🔴 Critical |
| [05-Backend-Architecture.md](05-Backend-Architecture.md) | FastAPI, folder structure, DI, service layer, Pydantic, validation, exception handling, middleware, background tasks, async | 🔴 Critical |
| [06-Frontend-Architecture.md](06-Frontend-Architecture.md) | The deleted Next.js dashboard, why GitHub's UI replaced it, plus complete React/Next.js interview theory | 🟡 Medium |
| [07-Authentication-and-Security.md](07-Authentication-and-Security.md) | GitHub App RS256 JWT, HMAC webhook verification, installation tokens, secrets management, plus full OAuth/bcrypt/CSRF/XSS/SQLi theory | 🔴 Critical |
| [08-API-Documentation.md](08-API-Documentation.md) | Every endpoint, every GitHub API call, request/response shapes, REST design theory | 🟠 High |
| [09-AI-System.md](09-AI-System.md) | **The heart of the project.** LangGraph, agents, tool calling, RAG, embeddings, chunking, hallucination control, cost optimisation | 🔴 Critical |

### Operations
| File | What it covers | Priority |
|---|---|---|
| [10-Deployment-and-DevOps.md](10-Deployment-and-DevOps.md) | Railway, railpack/railway.toml, env vars, CI/CD, monitoring, plus complete Docker theory | 🟠 High |
| [11-Testing.md](11-Testing.md) | Current test scripts (honest assessment), pytest, mocking, coverage, the testing pyramid, how to test non-deterministic LLMs | 🟠 High |
| [12-Performance-Optimization.md](12-Performance-Optimization.md) | The async parallelism fix, singletons, caching, profiling, benchmarking, N+1, connection pooling | 🔴 Critical |
| [13-Scalability.md](13-Scalability.md) | 100 → 1M users: what breaks first at each tier and how to fix it. Queues, sharding, load balancers, microservices | 🔴 Critical |

### Narrative & judgement — what separates senior from junior
| File | What it covers | Priority |
|---|---|---|
| [14-Project-Challenges.md](14-Project-Challenges.md) | The 7 hard problems, told as STAR stories: the critic failure, false async parallelism, line-number drift, Railway git dependency | 🔴 Critical |
| [15-Design-Decisions.md](15-Design-Decisions.md) | Every major decision with reasoning, alternatives, and trade-offs in ADR format | 🔴 Critical |
| [16-Code-Walkthrough.md](16-Code-Walkthrough.md) | Line-by-line walkthrough of every important file, class, function, and the full request lifecycle | 🔴 Critical |

### Interview preparation
| File | What it covers | Priority |
|---|---|---|
| [17-Interview-Questions-Part-1.md](17-Interview-Questions-Part-1.md) | Project, architecture, tech stack, database, auth, API, async — 9-part answer format each | 🔴 Critical |
| [18-Interview-Questions-Part-2.md](18-Interview-Questions-Part-2.md) | Security, performance, testing, CI/CD, behavioural, HR — 9-part answer format each | 🔴 Critical |
| [19-System-Design-Expansion.md](19-System-Design-Expansion.md) | "Design a code review system for 10,000 repos" — the full system design interview | 🟠 High |
| [20-Future-Improvements.md](20-Future-Improvements.md) | Known bugs (6 real ones found in audit), roadmap, what you'd do with 3 more months | 🟠 High |
| [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) | One-page rapid recall: numbers, names, one-liners | 🔴 Critical |
| [22-Final-Revision-Notes.md](22-Final-Revision-Notes.md) | 30-sec / 2-min / 5-min / 10-min pitches, hardest cross-questions, 1-page & 5-page revision sheets | 🔴 Critical |

### Assets
| File | What it covers |
|---|---|
| [assets/mermaid-diagrams.md](assets/mermaid-diagrams.md) | Every Mermaid diagram in one place, copy-paste ready |
| [assets/sequence-diagrams.md](assets/sequence-diagrams.md) | Detailed sequence diagrams for all major flows |
| [assets/whiteboard-guide.md](assets/whiteboard-guide.md) | Exactly what to draw on a whiteboard, in what order, with what words |

---

## Study plans

### If you have 1 hour before an interview
1. [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) — 20 min
2. [22-Final-Revision-Notes.md](22-Final-Revision-Notes.md) — the 2-min and 5-min pitches — 20 min
3. [14-Project-Challenges.md](14-Project-Challenges.md) — the critic agent story — 20 min

### If you have 1 day
1. This file — 15 min
2. [01](01-Project-Overview.md), [02](02-System-Architecture.md), [09](09-AI-System.md) — 3 hours
3. [14](14-Project-Challenges.md), [15](15-Design-Decisions.md) — 2 hours
4. [17](17-Interview-Questions-Part-1.md), [18](18-Interview-Questions-Part-2.md) — 3 hours
5. [21](21-Interview-Cheat-Sheet.md), [22](22-Final-Revision-Notes.md) — 1 hour

### If you have 1 week
Read every file in order. Then, critically: **run the project**, open a test PR, and watch the logs. Nothing beats having actually seen your own system work when you describe it.

---

## The numbers you must memorise

These come up constantly. Know them cold.

| Metric | Value | Why it matters |
|---|---|---|
| Specialist agents | **3** (bug, security, pattern) | Core architecture |
| Confidence threshold to post | **0.70** (`main.py:23`) | The filter that replaced the critic |
| Per-agent internal floor | **0.6** | Findings below this are dropped inside each agent |
| Embedding model | `text-embedding-3-small` | 1536 dimensions |
| Embedding dimensions | **1536** | Vector size in ChromaDB |
| Reasoning model | **GPT-4o**, `temperature=0` | Determinism for reproducible reviews |
| Vector distance metric | **Cosine** (`hnsw:space`) | Set in `store.py:22` |
| Critic rejection rate | **~87.5%** | The measurement that killed the critic |
| Clone depth | **1** (shallow) | Speed — seconds not minutes |
| Context chunks retrieved | **5** (bug/security), **10** (pattern) | RAG `n_results` |
| Max file size ingested | **512 KB** | `IngestionConfig.max_file_bytes` |
| Embedding batch size | **100** chunks | Per OpenAI API call |
| Line window fallback | **60** lines, **15** overlap | When tree-sitter can't parse |
| GitHub App JWT lifetime | **9 min** (max 10) | `github_client.py:27` |
| Languages parsed | **8** via tree-sitter | py, js, ts, java, go, rust, c, cpp |
| Total backend code | **~1,900 lines** across 41 Python files | Small, focused |

---

## How every file is structured

Per your specification, each file contains: Executive Summary · Beginner→Advanced concepts · Why this design · Alternatives · Advantages · Disadvantages · Production Best Practices · Common Mistakes · Interview Questions · Follow-ups · Real-world Example · Code Examples · Line-by-line explanation · Diagrams · Mermaid · Tables · Cross-references.

---

## A note on intellectual honesty

Throughout this knowledge base, three labels are used strictly:

- **✅ IN THE CODE** — verified present in the repository right now.
- **🗑️ REMOVED** — was built, then deleted. Discuss in past tense, with the reasoning.
- **📚 THEORY / NOT BUILT** — general knowledge you need for interviews, but *not* part of this project. **Never claim you built these.**

If an interviewer asks "did you use Redis?", the correct answer is *"No — here's what I used instead and why, and here's where Redis would go if we scaled."* That answer scores higher than a bluff every single time.

**Next:** [01-Project-Overview.md](01-Project-Overview.md)
