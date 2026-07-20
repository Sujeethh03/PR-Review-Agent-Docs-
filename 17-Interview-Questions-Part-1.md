# 17 — Interview Questions, Part 1

## How to Use This File

Fourteen core technical questions on project, architecture, stack, database, auth, API, and concurrency. Each uses this structure:

1. **Beginner answer** — what a junior would say
2. **Intermediate answer** — what a solid mid-level would say
3. **Advanced answer** — what gets you the offer
4. **What the interviewer is evaluating**
5. **Common mistakes**
6. **Follow-up questions**
7. **Best answer (2–5 min)** — the one to deliver
8. **Whiteboard**
9. **Production perspective**

**Part 2** ([18](18-Interview-Questions-Part-2.md)) covers security, DevOps, testing, and behavioural.

---

## Q1. "Walk me through your project."

**Beginner:** It's an AI tool that reviews GitHub pull requests and posts comments about bugs and security issues.

**Intermediate:** A FastAPI service receives GitHub webhooks when a PR opens. It fetches the diff, clones the repo, indexes the code into a vector database, then runs three GPT-4o agents in parallel to find bugs, security issues, and convention violations. Findings above a confidence threshold post as inline PR comments.

**Advanced:** *(see Best Answer)*

**Evaluating:** Can you compress a complex system without losing precision? Do you have judgement or just a feature list? Do you volunteer trade-offs?

**Common mistakes:** Listing technologies instead of explaining flow · going past 3 minutes · no trade-offs mentioned · saying "we" for solo work.

**Follow-ups:** "Why three agents?" · "Why did the critic fail?" · "What breaks at scale?" · "What would you change?"

### ⭐ Best answer (2–3 min)

> PR Review Agent is an AI service that reviews GitHub pull requests automatically. When you open a PR, a webhook hits my FastAPI backend. I verify the HMAC signature, return 200 immediately so GitHub doesn't time out, and run the actual work in a background task.
>
> That task does four things. First, it fetches the diff and shallow-clones the repo at the head SHA. Second, it builds a semantic index: tree-sitter parses every file into functions and classes, OpenAI embeds those chunks, and they go into a per-repository ChromaDB collection. Third — and this is the core — a LangGraph state machine fans out to three specialist GPT-4o agents running concurrently: bugs, security, and codebase conventions. Each agent, for each diff hunk, does a vector search to pull the five most relevant chunks from the rest of the repo, so it reviews with whole-codebase context rather than just the diff. Fourth, findings get deduplicated, filtered at a 0.70 confidence threshold, and posted as inline GitHub comments with one-click fix suggestions and OWASP references.
>
> The decision I'm proudest of is a deletion. I originally had a fourth critic agent, plus PostgreSQL and a Next.js approval dashboard. When I measured the critic, it was rejecting about 87.5% of findings — including real bugs I'd planted. So I deleted all three and replaced them with a single confidence threshold, using GitHub's own review UI. The system got simpler, faster, and more accurate simultaneously.
>
> The thing I'd fix first: my concurrency is unbounded. Three agents times N hunks means a fifty-hunk PR fires a hundred and fifty concurrent GPT-4o calls with nothing capping them. That's a rate-limit and cost problem, and the fix is a semaphore.

**Whiteboard:** GitHub → FastAPI → **200 in 5ms** (emphasise this arrow) → background task → diff/clone/ingest → ChromaDB cylinder → three stacked boxes (parallel) → dedup → filter ≥0.70 → back to GitHub.

**Production perspective:** In production I'd add a durable queue between the webhook and the pipeline, structured logging with correlation IDs, and a cost metric with alerting. Those three make it operable by someone who isn't me.

---

## Q2. "Explain your architecture."

**Beginner:** FastAPI backend that receives webhooks and calls OpenAI.

**Intermediate:** Event-driven pipeline. Webhook ingress, async background processing, a five-stage pipeline with parallel fan-out to three agents, and a vector store for retrieval.

**Advanced:** *(see Best Answer)*

**Evaluating:** Can you communicate a system visually? Do you know which parts matter and why?

**Common mistakes:** Drawing boxes without explaining arrows · not mentioning the async boundary · claiming patterns you didn't use.

**Follow-ups:** "Why no message queue?" · "How do the agents share state?" · "What's the single point of failure?"

### ⭐ Best answer

> It's an event-driven pipeline with parallel fan-out, running as a single async Python process.
>
> Two things make it work. **First**, the ACK boundary. GitHub times out around ten seconds and my pipeline takes ninety, so the webhook handler does only three things: verify the HMAC, spawn a background task, return 200. That's five milliseconds. Everything else happens after the response.
>
> **Second**, nested parallelism. LangGraph fans out from `__start__` to three agent nodes with no dependencies between them, so they run concurrently. Inside each agent, `asyncio.gather` processes every diff hunk concurrently. Those multiply — three agents times twenty hunks is sixty simultaneous operations. Fully sequential this would take about fifteen minutes; it takes about fifteen seconds.
>
> That parallelism took a real bug fix to achieve, incidentally. My original code used the *synchronous* OpenAI client inside `async def` functions. Everything looked parallel — three graph nodes, `gather` everywhere — but every call blocked the single event loop in turn, so the three agents ran strictly sequentially. Nothing failed; the only symptom was that timings summed instead of overlapping. Switching to `AsyncOpenAI` fixed it.
>
> For state: the agents share a typed `AgentState` TypedDict, and each writes to a different key, so there's no write conflict.
>
> The weakest part is the queue, or rather the absence of one. `asyncio.create_task` schedules work in process memory. If Railway restarts mid-pipeline, that review is silently gone — no retry, no dead-letter queue, no visibility. The fix is a broker, and it's cheap because my pipeline is already a pure function of five JSON-serialisable scalars, so it's a one-line change at the call site.

**Whiteboard:** Draw the 200-OK arrow *second*, and say "this is the most important arrow on the board." Stack the three agents vertically to emphasise parallelism.

**Production perspective:** Single process is the SPOF. Two receivers behind a load balancer plus a queue and N workers removes it — and the design already supports that migration.

---

## Q3. "Why this tech stack?"

**Beginner:** Python is good for AI, FastAPI is fast, and OpenAI has the best models.

**Intermediate:** FastAPI because the workload is I/O-bound and it's async-native. Python because the AI ecosystem lives there. ChromaDB because it's embedded and free. GPT-4o because code review needs strong reasoning.

**Advanced:** *(see Best Answer)*

**Evaluating:** Did you choose deliberately or by default? Can you name what you'd choose differently?

**Common mistakes:** "It's popular" · not knowing alternatives · not naming a downside of any choice.

**Follow-ups:** "Why not Django?" · "Why not Pinecone?" · "Why not Claude or Gemini?" · "What would you change?"

### ⭐ Best answer

> Each choice traces to a constraint.
>
> **FastAPI** because the workload is almost entirely I/O — waiting on GitHub and OpenAI. It's ASGI-native, so one worker handles hundreds of concurrent awaits. Django would give me an ORM, admin, templates, and auth, all of which are dead weight here — I have no database, no HTML, and no users. Flask is WSGI, so one blocked request holds a worker.
>
> **Python** because the entire AI ecosystem is there — the OpenAI SDK, LangGraph, ChromaDB, and tree-sitter bindings all have first-class Python support.
>
> **ChromaDB embedded** because at single-instance scale a managed vector database is infrastructure with no corresponding problem. Zero cost, zero ops, no network hop. The trade is real though: it's ephemeral on Railway and can't be shared across instances. I kept `CHROMADB_PATH` configurable so moving to a persistent volume is config, not code.
>
> **GPT-4o** because false positives are the failure mode that kills adoption for a review tool, and smaller models produce noticeably weaker findings. I pay more per call to protect precision.
>
> **`text-embedding-3-small` over `-large`** because it's about six and a half times cheaper for a modest quality difference. Retrieval only needs to put *reasonably* relevant context in front of the model — the LLM does the reasoning.
>
> What I'd change: **LangGraph is arguably over-engineering now.** I chose it when the graph included a critic node downstream, where conditional edges mattered. After deleting the critic it's three parallel nodes with no dependencies, which is exactly `asyncio.gather`. If I started today with only this topology I'd use plain `gather`. I kept it because adding a fourth agent is where it pays off.
>
> And if I were rebuilding with an analytics requirement, I'd use **pgvector** instead of ChromaDB — one database for both vectors and relational data, with backups and replication ops teams already understand.

**Production perspective:** The stickiest choice is the embedding model. Switching it requires a full re-index — vectors from different models aren't comparable and the dimensions don't even match. Much stickier than the LLM choice.

---

## Q4. "Explain your database schema."

**Beginner:** I use ChromaDB, a vector database, to store code embeddings.

**Intermediate:** One ChromaDB collection per repository, named `owner__repo`. Each record has an ID, the source text, a 1536-dimension embedding, and metadata — file path, line range, language, chunk type, symbol name, and the commit SHA.

**Advanced:** *(see Best Answer)*

**Evaluating:** Do you understand data modelling, including for non-relational stores? Can you discuss the relational schema you deleted?

**Common mistakes:** Inventing a schema you don't have · not knowing why cosine · not being able to discuss the removed Postgres layer.

**Follow-ups:** "Why cosine and not Euclidean?" · "Why one collection per repo?" · "How do you handle updates?"

### ⭐ Best answer

> I don't have a relational schema — my only datastore is a vector database — and I can explain both that and the relational one I deleted.
>
> **ChromaDB** holds one collection per repository, named `owner__repo`. Each record is a code chunk: an ID that's the SHA-256 of file-plus-line-range, the raw source text, a 1536-dimension embedding from `text-embedding-3-small`, and metadata — file, start and end line, language, chunk type, symbol name, and the SHA it was ingested at. Index is HNSW with cosine distance.
>
> Three design points. **The ID is content-addressed**, so re-ingesting the same chunk upserts rather than duplicating — that's what makes ingestion idempotent. **Cosine rather than Euclidean** because embedding magnitude tracks text length, not meaning; I want a five-line and a fifty-line function doing the same thing to be similar, and cosine ignores magnitude. **Per-repo collections rather than one collection with a filter** because physical isolation makes cross-repo context leakage structurally impossible rather than dependent on remembering a `where` clause.
>
> There's a bug in that ID scheme I should name: it encodes line numbers, and line numbers change. When someone adds five lines above a function, the same function gets a new ID and the old row is never deleted. Over many PRs the index accumulates phantom chunks describing code that no longer exists — which particularly hurts my pattern agent, since it's retrieving "conventions" that were deleted. The fix is delete-before-insert on modified files rather than upsert.
>
> **The relational one I deleted** was `pr_reviews` one-to-many `findings`, third normal form, with a composite unique constraint on review-file-line-category for idempotency on re-runs. The Alembic migrations are still in the repo as dead code — one of them is literally called `drop_critic_columns`, which is the history of that decision written into the schema.

**Whiteboard:** Draw the ChromaDB record as a table with the four field groups (id / document / embedding / metadata). Then draw the deleted ER diagram alongside.

**Production perspective:** The ephemeral `/tmp` location is the real production issue — every deploy wipes the index and silently forces a full re-ingest, defeating my biggest cost optimisation.

---

## Q5. "Explain normalization."

**Beginner:** Organising tables to reduce duplicate data.

**Intermediate:** 1NF means atomic values, no repeating groups. 2NF adds no partial dependency on part of a composite key. 3NF adds no transitive dependency — non-key columns must depend only on the key.

**Advanced:** *(see Best Answer)*

**Evaluating:** Textbook knowledge *plus* judgement about when to break the rules.

**Common mistakes:** Reciting definitions without examples · claiming normalization is always right · not knowing when to denormalize.

**Follow-ups:** "When would you denormalize?" · "What's BCNF?" · "What anomalies does normalization prevent?"

### ⭐ Best answer

> Normalization eliminates redundancy so each fact is stored exactly once, which prevents three anomalies.
>
> **Update anomaly** — if a repo name is duplicated across a thousand finding rows, renaming it means updating a thousand rows, and a partial failure leaves inconsistent data. **Delete anomaly** — deleting the last finding for a PR loses the PR record entirely. **Insert anomaly** — you can't record a PR that has no findings yet.
>
> The forms build on each other. **1NF**: atomic values. Storing `"10,11,12,15"` in an `affected_lines` column violates it — I'd use `line_start` and `line_end`. **2NF**: 1NF plus no partial dependency on part of a composite key. If the key is review-plus-finding and `repo_name` depends only on review, split it out. **3NF**: no transitive dependency — if `agent_description` depends on `agent` which depends on the ID, that's transitive; extract an agents table. **BCNF** is stricter — every determinant must be a candidate key — but in practice 3NF handles almost everything.
>
> The judgement part is knowing when to break it. Normalization optimises for write consistency at the cost of read performance, because you need joins. On a read-heavy analytics workload I'd denormalize deliberately — store a `finding_count` on the reviews table instead of counting, or duplicate `repo_name` onto findings to avoid a join. The rule of thumb is **normalize until it hurts, denormalize until it works** — and critically, denormalize *deliberately* with a plan for keeping the duplicate consistent, usually a trigger or an application-level invariant.
>
> Worth noting the vector database I actually use is fully denormalised — one flat collection, metadata repeated on every chunk, no joins and no referential integrity. That's typical: vector stores optimise for one operation, similarity search with metadata filtering, rather than relational flexibility.

**Production perspective:** The composite unique constraint in my deleted schema was doing real work — enforcing idempotency on webhook redelivery. When I removed the database I didn't remove that need, I just moved it into Python via deterministic chunk IDs. The database was doing it more reliably.

---

## Q6. "Explain authentication in your project."

**Beginner:** I use JWT to authenticate with GitHub.

**Intermediate:** Two mechanisms. Inbound, GitHub signs webhooks with HMAC-SHA256 and I verify with a shared secret. Outbound, I sign a JWT with my RSA private key, exchange it for an installation token, and use that for API calls.

**Advanced:** *(see Best Answer)*

**Evaluating:** Do you understand *why* each mechanism was chosen, not just that you used it?

**Common mistakes:** Claiming user auth you don't have · not knowing HS256 vs RS256 · not knowing why constant-time comparison matters.

**Follow-ups:** "Why RS256 not HS256?" · "Why constant-time comparison?" · "What's the token lifetime and why?"

### ⭐ Best answer

> There's no user authentication — no login, no passwords, no sessions. It's machine-to-machine with GitHub as the only client. But there's authentication in both directions, using different mechanisms for different reasons.
>
> **Inbound**, GitHub proves it's GitHub with HMAC-SHA256. I read the raw request bytes — deliberately not the parsed JSON, because re-serialising changes whitespace and key order and the signature would never match. I compute HMAC with a shared secret and compare with `hmac.compare_digest`.
>
> That last part is the interesting bit. Python's `==` short-circuits at the first differing byte, so response timing leaks how many bytes were correct. An attacker could forge a signature byte by byte — measuring latency, fixing one byte at a time — which reduces the search from 256 to the 64th power down to about sixteen thousand attempts. `compare_digest` always compares every byte, so timing carries no information.
>
> **Outbound**, I prove I'm the App with a JWT. I sign it with my RSA private key using RS256, with `iss` set to my App ID, backdated sixty seconds for clock skew, expiring in nine minutes against GitHub's ten-minute maximum. I exchange that for an installation access token — one hour, scoped to that installation — and use it for actual API calls.
>
> The two-step matters: the JWT proves *identity*, the installation token grants *authorisation*. An App installed on five hundred organisations gets a separately-scoped token for each, so a leak has a blast radius of one installation for one hour.
>
> RS256 rather than HS256 because GitHub needs to verify my tokens without being able to forge them. With a symmetric secret we'd both hold the same key and GitHub could mint tokens as me.
>
> The gap I'd name: HMAC proves authenticity and integrity but not *freshness*. A captured valid request replays forever. GitHub sends `X-GitHub-Delivery` as a unique UUID — caching those and rejecting duplicates would close it.

**Whiteboard:** Two arrows, opposite directions. Inbound labelled "HMAC / symmetric / shared secret." Outbound labelled "JWT RS256 / asymmetric / private key" → "installation token / 1hr / scoped."

---

## Q7. "Explain JWT."

**Beginner:** A token with user info that proves who you are, signed so it can't be faked.

**Intermediate:** Three base64url parts — header, payload, signature — separated by dots. Header declares the algorithm, payload holds claims like `iss`, `exp`, `sub`, and the signature covers header-plus-payload. Stateless: the server validates without a database lookup.

**Advanced:** *(see Best Answer)*

**Evaluating:** Do you know JWTs are readable, not encrypted? Do you know the `alg:none` attack? Do you understand revocation?

**Common mistakes:** Saying JWTs are encrypted · putting secrets in the payload · not knowing revocation is the hard problem.

**Follow-ups:** "How do you revoke a JWT?" · "What's the `alg:none` attack?" · "Where do you store it client-side?"

### ⭐ Best answer

> Three base64url-encoded parts joined by dots: header, payload, signature. The header declares the algorithm, the payload holds claims — `iss`, `exp`, `iat`, `sub` — and the signature covers the first two parts.
>
> **The single most important fact: a JWT is encoded, not encrypted.** Anyone can paste one into jwt.io and read every claim. The signature guarantees integrity — nobody tampered with it — not confidentiality. So you never put secrets, passwords, or PII in the payload. My tokens carry only `iat`, `exp`, and my App ID, all non-sensitive.
>
> **On algorithms:** I use RS256, asymmetric, because GitHub must verify without being able to forge. HS256 is symmetric — fine when one party both signs and verifies, dangerous when you're handing verification to a third party.
>
> **The classic attack is `alg:none`.** Early libraries trusted the algorithm declared in the token's own header. An attacker could take a valid token, set the header to `{"alg":"none"}`, strip the signature, and some libraries accepted it. There's a related variant: change `RS256` to `HS256` and sign with the *public* key as the HMAC secret — and since the public key is public, anyone can forge. The defence is always specifying expected algorithms at verification: `jwt.decode(token, key, algorithms=["RS256"])`. PyJWT requires that parameter for exactly this reason.
>
> **The hard problem with JWTs is revocation.** Statelessness is the selling point, but it means you can't invalidate a token before it expires — there's nothing to delete. The mitigations are short expiry plus refresh tokens, a `jti` claim checked against a denylist (which reintroduces state, defeating the point), or accepting the window. My tokens expire in nine minutes, which makes revocation largely moot — the window is smaller than the time it'd take to notice a compromise.
>
> For a user-facing app, storing the token client-side is its own trade-off: `localStorage` is vulnerable to XSS, `httpOnly` cookies are immune to XSS but need CSRF protection. I'd take httpOnly cookies with `SameSite=Lax`, because XSS is the more common and more damaging vector.

---

## Q8. "Explain OAuth."

**Beginner:** It lets you log in with Google or GitHub without giving them your password.

**Intermediate:** An authorisation framework with four roles — resource owner, client, authorization server, resource server. Authorization Code flow: redirect to the auth server, user consents, you get a code, exchange it server-side for an access token.

**Advanced:** *(see Best Answer)*

**Evaluating:** Do you know OAuth is authorisation not authentication? Do you know PKCE?

**Common mistakes:** Calling OAuth "login" · not knowing PKCE · not knowing why Implicit flow was deprecated.

**Follow-ups:** "What's PKCE for?" · "OAuth vs OIDC?" · "Which grant would you use for a mobile app?"

### ⭐ Best answer

> First distinction: **OAuth is authoriSation, not authentiCation.** It answers "may this app access that resource on the user's behalf?" not "who is this user?" People conflate them because "Login with Google" is built on OAuth — but the identity layer is **OpenID Connect**, which adds an `id_token` on top. Using a raw OAuth access token as proof of identity is a known anti-pattern.
>
> Four roles: the **resource owner** (the user), the **client** (the app), the **authorization server** (the login provider), and the **resource server** (the API holding the data).
>
> **Authorization Code flow with PKCE** is the current recommendation for everything user-facing. The client generates a random `code_verifier`, hashes it into a `code_challenge`, and sends the challenge with the authorize request. The user consents, the client gets a code, then exchanges it along with the original verifier. The auth server hashes the verifier and checks it matches the stored challenge.
>
> **PKCE exists because mobile apps and SPAs can't keep a secret.** Anything shipped to a device is extractable. Without PKCE, an attacker who intercepts the redirect — say a malicious app registering the same custom URL scheme — could exchange the code for a token. PKCE binds the code to a random value only the legitimate client holds. It started as mobile-specific and is now recommended for all clients including server-side ones.
>
> **Implicit flow is deprecated** — it returned the access token directly in the URL fragment, where it lands in browser history, referrer headers, and server logs. PKCE made it unnecessary.
>
> **Relevant to my project:** my GitHub App flow is conceptually the **Client Credentials** grant — a machine authenticating as itself, no user involved, no redirect. If I brought back a user-facing dashboard, that's where Authorization Code plus PKCE would come in, with "Login with GitHub."

---

## Q9. "Explain your API design."

*(Full treatment in [08-API-Documentation.md](08-API-Documentation.md).)*

**Evaluating:** REST fundamentals, idempotency awareness, status code precision.

### ⭐ Best answer

> My service is much more an API *consumer* than a provider — two endpoints exposed, about nine external operations consumed.
>
> The webhook is `POST /webhook`, authenticated by HMAC signature rather than a bearer token, because that's the webhook pattern: the provider signs the payload rather than presenting credentials. It always returns 200, even for events it ignores, because GitHub disables webhooks that persistently fail. Failures go to logs, never to 5xx.
>
> One point I'd concede: it arguably should return **202 Accepted** rather than 200, since it acknowledges receipt without having done the work. GitHub treats any 2xx identically so it's semantics rather than function, but 202 is the honest code.
>
> On the consuming side, the piece I find most interesting is **content negotiation**. To get a diff I `GET` the pull request URL with `Accept: application/vnd.github.diff` instead of the JSON accept header. Same URI, different representation — that's REST working as designed, and it's why GitHub doesn't need a separate diff endpoint.
>
> **The gap is idempotency.** GitHub sends `X-GitHub-Delivery`, a unique UUID per delivery, and retries on failure. I don't track it, so a redelivery re-runs the pipeline and posts duplicate comments. The fix is a Redis `SET NX EX` on the delivery ID — atomic set-if-not-exists with a 24-hour TTL covering GitHub's retry window.
>
> There's also a design flaw in how I post. GitHub validates a review's inline comments **atomically** — if any one targets a line not in the diff, the whole request 422s and zero comments post. Since line numbers come from GPT-4o, one hallucination collapses ten findings into a single flat summary. The irony is I already parse the diff into exact line ranges to build the prompts; I just never validate the output against them. About fifteen lines of code.

**Production perspective:** Idempotency and output validation are both "boring" reliability work that matters more than features. Neither is hard; both were deprioritised, and both would bite in production.

---

## Q10. "Explain state management."

> ⚠️ Frontend question. Be honest: you built it, then deleted it. Full treatment in [06-Frontend-Architecture.md](06-Frontend-Architecture.md).

### ⭐ Best answer

> My current project has no frontend — I built a Next.js dashboard and deleted it when the approval workflow it served was removed. So I'll answer from that experience and say what I'd do differently.
>
> **The key distinction is server state versus client state**, and conflating them is the most common mistake — it's the one I made. Findings fetched from an API are *server* state: they have caching, staleness, refetch, and error semantics. Client state is genuinely local — is the drawer open, what's in this input.
>
> I hand-rolled it: `useState` for the findings array, `useEffect` to fetch on mount, another `useState` for a pending flag per button. That works, but I wrote by hand what React Query gives free — loading and error states, request deduplication, cache invalidation, and refetch after mutation. It's also *why* I had a duplicate-submit bug I had to fix manually: without a pending state, users clicked Approve repeatedly. React Query's `isPending` handles that.
>
> What I'd build now: **React Query for all server data**, keyed by resource. **`useState` for genuinely local UI state.** **Context only for rarely-changing globals** like theme or current user — Context re-renders every consumer on any change, so it's wrong for frequently-updating data. **No Redux**, because there's no complex client-side state machine here; it'd be boilerplate around what is fundamentally a cache.
>
> The decision tree I'd use: is it server data? React Query. Is it shared across distant components and rarely changes? Context. Is it complex client state with many transitions? Zustand or Redux Toolkit. Otherwise `useState` and lift it as needed.

---

## Q11. "What's your performance bottleneck?"

**Evaluating:** Have you measured? Do you know the difference between I/O-bound and CPU-bound?

### ⭐ Best answer

> External API latency, specifically OpenAI. About 60% of wall-clock time is GPT-4o agent calls and another 15% is fix-suggestion generation. Ingestion embeddings are 12%. Everything I actually control — HMAC, parsing, dedup, formatting — is well under 1%.
>
> That shapes every optimisation decision: **optimising my Python is close to pointless.** What matters is how many API calls I make and how concurrently.
>
> On concurrency I've done the work. Three agents run in parallel via LangGraph, and inside each agent `asyncio.gather` processes all hunks concurrently. Those multiply to sixty simultaneous operations on a typical PR — roughly sixty times faster than sequential.
>
> Getting there required fixing a bug worth mentioning: I originally used the *synchronous* OpenAI client inside async functions. The code looked parallel but every call blocked the event loop, so the agents ran sequentially. Nothing failed — the only symptom was timings summing instead of overlapping.
>
> **My biggest remaining inefficiency is duplicated work.** Each agent independently embeds the same diff hunk to run its vector search. Twenty hunks times three agents is sixty embedding calls where twenty distinct embeddings exist — and since embeddings batch, one call would do. Sixty API calls become one. It exists because I built the agents to be fully independent, which is architecturally valuable but means shared work gets duplicated. The fix hoists the computation to the orchestrator without coupling the agents.
>
> Second place: I fetch a GitHub installation token three times per pipeline, and they're valid for an hour.
>
> And the risk I'd name unprompted: my concurrency has **no ceiling**. Sixty is what twenty hunks produce; two hundred hunks produce six hundred. That's a rate-limit and cost problem, and a semaphore is a five-line fix.

**Production perspective:** I'd measure with `py-spy`, not `cProfile` — `cProfile` measures CPU and would tell me my hottest function is in the HTTP client's polling loop, which is true and useless. And I'd report p50/p95/p99, not averages, because users experience the tail.

---

## Q12. "Explain concurrency in your project."

### ⭐ Best answer

> Three levels, plus one deliberate exception.
>
> **Level one: the ACK boundary.** The webhook handler spawns a background task with `asyncio.create_task` and returns immediately. GitHub gets 200 in five milliseconds while the pipeline runs for ninety seconds.
>
> **Level two: agent fan-out.** LangGraph has three edges from `__start__` to three agent nodes with no dependencies between them, so they execute concurrently. They share a typed `AgentState` TypedDict and each writes a different key, so there's no write conflict.
>
> **Level three: hunk-level.** Inside each agent, `asyncio.gather` fires all hunks concurrently with `return_exceptions=True`, so one failing hunk returns an exception object rather than destroying the whole gather.
>
> Levels two and three multiply: three agents times twenty hunks is sixty concurrent operations.
>
> **The deliberate exception** is ingestion. ChromaDB's client is synchronous with no async variant. Rather than wrapping it in `async def` and lying to the event loop, I wrote it as a plain sync function and call it through `asyncio.to_thread`. It runs on a worker thread; the loop stays free. I think that honesty matters — `async def` around blocking code is a lie the type system won't catch.
>
> Which is exactly the bug I had. I used the synchronous `OpenAI` client inside async functions, so my three "parallel" agents ran strictly sequentially. The lesson: **async is a cooperative model, and one blocking call serialises everything.** In async code you audit the library, not the syntax.
>
> **The gap:** no bound on concurrency. A semaphore of ten would turn sixty concurrent calls into six waves — slower, about ninety seconds instead of fifteen, but bounded. I'd take that trade: a review that arrives reliably in ninety seconds beats one that arrives in fifteen and sometimes drops a third of its findings to rate limits.

**Whiteboard:** Draw two sequence diagrams side by side — sync client (three 15s blocks in series, 45s total) versus async (three overlapping, 15s total).

---

## Q13. "How do you handle validation?"

### ⭐ Best answer

> Validation happens at every trust boundary, and I have two: GitHub's webhook and GPT-4o's output.
>
> **At the webhook** there's a deliberate inversion. This is my only real endpoint and I *bypass* FastAPI's automatic Pydantic validation on it — I read raw bytes rather than a parsed model, because HMAC must be computed over exactly what GitHub sent. Parsing and re-serialising changes whitespace and key ordering and the signature would never match. So the order is: verify authenticity first, parse second. After that it's defensive parsing — chained `.get()` with defaults so a malformed payload can't `KeyError` — plus guard clauses for the fields the pipeline can't run without.
>
> **At the LLM boundary** there are two layers. First, the OpenAI tool schema constrains generation: `required` on every field, `additionalProperties: false` so it can't invent fields, and an `enum` on severity. Second, Pydantic validates at construction — `severity: Literal["high","medium","low"]` means a hallucinated "critical" raises `ValidationError` at the boundary instead of reaching my formatter and producing a broken comment.
>
> Why both? The schema constrains generation but the API doesn't *guarantee* conformance. Pydantic is the backstop.
>
> There's a deliberate asymmetry worth explaining: `severity` is a `Literal` but `category` is a plain `str`. Severity drives formatting logic and must be one of three values. Category is an open set — my prompts list fifteen but new ones are useful, and my OWASP lookup degrades gracefully to a severity-based fallback for unknown ones. Constraining it would reject useful output.
>
> **Two gaps I'd fix.** `confidence: float` is unbounded — the model could return 1.5 and Pydantic accepts it. One line: `Field(ge=0.0, le=1.0)`. And more seriously, I never validate line numbers against the actual diff, which is what causes GitHub to 422 the entire review when the model hallucinates one.
>
> The general principle: **Pydantic at trust boundaries, dataclasses internally.** My `Chunk` objects are dataclasses because my own parser constructs thousands in a hot loop and there's no adversary. In the same file, `IngestionResult` is Pydantic because it crosses a module boundary and is constructed once.

---

## Q14. "How do you work with third-party APIs?"

### ⭐ Best answer

> Four principles, and one place I fall short.
>
> **Isolate them behind adapters.** `github_client.py` and `store.py` are the only modules that know their respective APIs. Nothing else imports `httpx` for GitHub calls or `chromadb` directly. Swapping ChromaDB for pgvector means rewriting one file — that's the Repository pattern doing its job.
>
> **Pin the API version.** I send `X-GitHub-Api-Version: 2022-11-28`. Without it you get whatever the default is, which changes over time — your code breaks on someone else's deploy schedule. Pinning means breakage happens when *you* choose to upgrade.
>
> **Degrade rather than fail.** Every external call has a defined failure behaviour. If ChromaDB has no collection, retrieval returns empty and agents review without context. If a single GPT-4o call fails, `return_exceptions=True` skips that hunk. If inline comments 422, I fall back to a summary body comment. The principle is that partial output beats no output.
>
> **Respect their constraints.** GitHub times out at ten seconds, so I ACK before processing. GitHub App JWTs max at ten minutes, so I use nine. Installation tokens last an hour.
>
> **Where I fall short is retry strategy.** My embedding retry is one attempt after a fixed one-second sleep, and it catches all exceptions — so it retries a 401 auth error pointlessly, wasting time before failing anyway. There's no exponential backoff, no jitter, no respect for `Retry-After` on 429s, and no circuit breaker. During an OpenAI outage I'd hammer a dead API for every incoming PR.
>
> What I'd add: `tenacity` with exponential backoff and jitter, `retry_if_exception_type` so only transient failures retry, and a circuit breaker that trips after N consecutive failures. But the most important thing is making failure **visible** — right now a fully failed review is silent. The developer sees nothing and concludes the bot found no problems, which is worse than an error.

**Production perspective:** The silent-failure problem is the one that matters most. A visibly broken system gets fixed; a silently degraded one erodes trust invisibly.

---

## Cross-References

| Topic | File |
|---|---|
| Security, DevOps, behavioural questions | [18-Interview-Questions-Part-2.md](18-Interview-Questions-Part-2.md) |
| System design interview | [19-System-Design-Expansion.md](19-System-Design-Expansion.md) |
| Rapid recall | [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) |
| Timed pitches | [22-Final-Revision-Notes.md](22-Final-Revision-Notes.md) |

**Next:** [18-Interview-Questions-Part-2.md](18-Interview-Questions-Part-2.md)
