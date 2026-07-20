# PR Review Agent — Complete Knowledge Base

Interview-oriented technical documentation for **[PR Review Agent (CR-001)](https://github.com/Sujeethh03/pr-review-agent-demo)** — a multi-agent AI system that automatically reviews GitHub pull requests.

**26 documents · ~13,700 lines · 24 Mermaid diagrams**

This is written to *teach*, not merely to document. Every file is self-contained.

---

## ⚠️ Read this first

This knowledge base documents the system **as it actually exists**, including components that were built and then deliberately deleted.

| Commonly assumed | Reality |
|---|---|
| Next.js / React frontend | **Built, then deleted** in commit `a61bfd2`. The UI is GitHub's own PR review interface |
| PostgreSQL database | **Built, then deleted.** `backend/alembic/` is dead code |
| JWT / OAuth user login | No human users. JWT is **RS256 GitHub App machine-to-machine auth** |
| Redis | Not present. In-process singletons + ChromaDB |
| Docker + Render + Vercel | **Railway**, via `railpack.json` + `railway.toml` |
| pytest suite | Two standalone verification **scripts**, not CI-runnable tests |

**Why the deletions are the best part of this project:** a critic agent was built to filter findings. Measurement showed it rejecting **~87.5% of findings, including real bugs**. It was deleted along with the database and dashboard it fed, and replaced with a single confidence threshold. That's measurement → judgement → deletion, which is a stronger engineering narrative than any feature list.

---

## What the system does

> A GitHub webhook fires when a PR opens. A FastAPI receiver validates the HMAC-SHA256 signature, returns `200` in ~5ms, and hands the work to a background task. That task fetches the diff, shallow-clones the repo at the head SHA, parses every file into semantic chunks with tree-sitter, embeds them with OpenAI `text-embedding-3-small`, and stores them in a per-repository ChromaDB collection. A LangGraph state machine then fans out to three specialist GPT-4o agents — bug, security, and pattern — running concurrently, each retrieving relevant codebase context via vector search (RAG) for every diff hunk. Findings are deduplicated, filtered at a 0.70 confidence threshold, and posted as inline GitHub review comments with one-click fix suggestions and OWASP educational context.

---

## Contents

### Start here
| File | Contents |
|---|---|
| [00 — Table of Contents](00-Table-of-Contents.md) | How to use this, study plans, key numbers |
| [01 — Project Overview](01-Project-Overview.md) | Problem, existing solutions, features, requirements, constraints, assumptions |
| [02 — System Architecture](02-System-Architecture.md) | 9 Mermaid diagrams, every component and arrow explained |
| [03 — Low-Level Design](03-Low-Level-Design.md) | Class diagrams, data structures, 4 algorithm deep-dives |

### The stack
| File | Contents |
|---|---|
| [04 — Database Design](04-Database-Design.md) | ChromaDB/HNSW + full relational theory (normalization, indexes, ACID, isolation, pooling, N+1) |
| [05 — Backend Architecture](05-Backend-Architecture.md) | FastAPI, DI, layering, validation, exceptions, background tasks |
| [06 — Frontend Architecture](06-Frontend-Architecture.md) | The deleted dashboard + React/Next.js theory |
| [07 — Authentication & Security](07-Authentication-and-Security.md) | HMAC, RS256 JWT, secrets + OAuth/bcrypt/XSS/CSRF/SQLi theory |
| [08 — API Documentation](08-API-Documentation.md) | Every endpoint and external call + REST design theory |
| [09 — AI System](09-AI-System.md) | ⭐ RAG, agents, tool calling, prompts, hallucination control, cost |

### Operations
| File | Contents |
|---|---|
| [10 — Deployment & DevOps](10-Deployment-and-DevOps.md) | Railway, env vars, CI/CD + full Docker theory |
| [11 — Testing](11-Testing.md) | Honest assessment, pytest, mocking, **evaluating non-deterministic LLMs** |
| [12 — Performance](12-Performance-Optimization.md) | The false-parallelism bug, caching, profiling |
| [13 — Scalability](13-Scalability.md) | 100 → 1M users: what breaks first at each tier |

### Judgement
| File | Contents |
|---|---|
| [14 — Project Challenges](14-Project-Challenges.md) | ⭐ 7 STAR stories, all verifiable in git history |
| [15 — Design Decisions](15-Design-Decisions.md) | ⭐ 16 ADRs with alternatives and trade-offs |
| [16 — Code Walkthrough](16-Code-Walkthrough.md) | Line-by-line + a register of 16 real bugs |

### Interview prep
| File | Contents |
|---|---|
| [17 — Interview Questions, Part 1](17-Interview-Questions-Part-1.md) | Project, architecture, stack, DB, auth, API, concurrency |
| [18 — Interview Questions, Part 2](18-Interview-Questions-Part-2.md) | Security, DevOps, testing + behavioural |
| [19 — System Design Expansion](19-System-Design-Expansion.md) | "Design this for 10,000 repos" |
| [20 — Future Improvements](20-Future-Improvements.md) | Prioritised bug register + roadmap |
| [21 — Interview Cheat Sheet](21-Interview-Cheat-Sheet.md) | ⭐ Read 20 min before an interview |
| [22 — Final Revision Notes](22-Final-Revision-Notes.md) | ⭐ 30-sec / 2-min / 5-min / 10-min pitches, hard cross-questions |

### Assets
| File | Contents |
|---|---|
| [Mermaid Diagrams](assets/mermaid-diagrams.md) | All 24 diagrams, copy-paste ready |
| [Sequence Diagrams](assets/sequence-diagrams.md) | 8 detailed flows with walkthroughs |
| [Whiteboard Guide](assets/whiteboard-guide.md) | What to draw, in what order, with what words |

---

## Key numbers

| Metric | Value |
|---|---|
| Specialist agents | 3 — bug, security, pattern |
| Confidence threshold | 0.70 (agent floor 0.6) |
| Critic rejection rate (why it was deleted) | ~87.5% |
| Embedding model / dims | `text-embedding-3-small` / 1536 |
| Reasoning model | GPT-4o, `temperature=0` |
| Vector distance | Cosine (HNSW) |
| Context chunks | 5 (bug/security), 10 (pattern) |
| Backend size | ~1,900 lines, 41 files |
| End-to-end latency | 30–90s |
| Webhook ACK | ~5ms |
| Parallelism speedup | ~60× vs sequential |

---

## Study plans

**1 hour** → [21 Cheat Sheet](21-Interview-Cheat-Sheet.md) · [22 pitches](22-Final-Revision-Notes.md) · [14 critic story](14-Project-Challenges.md)

**1 day** → [00](00-Table-of-Contents.md) → [01](01-Project-Overview.md), [02](02-System-Architecture.md), [09](09-AI-System.md) → [14](14-Project-Challenges.md), [15](15-Design-Decisions.md) → [17](17-Interview-Questions-Part-1.md), [18](18-Interview-Questions-Part-2.md) → [21](21-Interview-Cheat-Sheet.md), [22](22-Final-Revision-Notes.md)

**1 week** → Everything in order. Then **run the project**, open a test PR, and watch the logs.

---

## Labels used throughout

- **✅ IN THE CODE** — verified present in the repository
- **🗑️ REMOVED** — built, then deleted. Discussed in past tense with reasoning
- **📚 THEORY / NOT BUILT** — general knowledge for interviews, *not* part of this project

If an interviewer asks "did you use Redis?", the correct answer is *"No — here's what I used instead and why, and here's where Redis would go if we scaled."* That scores higher than a bluff every time.
