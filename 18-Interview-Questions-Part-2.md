# 18 — Interview Questions, Part 2

## Coverage

Security · caching · DevOps · testing · logging · data formats · async · ORM · and the behavioural questions that decide most interviews.

**A note on the behavioural section (Q16–22):** these decide more offers than the technical ones at mid-level and above. Companies assume you can code; they're checking whether you can own work, communicate, and make judgement calls. Prepare them as carefully as the technical ones.

---

## Q1. "Tell me about security in your project."

### ⭐ Best answer

> Four things I did well, four gaps I know about.
>
> **Well:** HMAC-SHA256 webhook verification with `hmac.compare_digest` — constant-time, so an attacker can't forge a signature byte-by-byte by measuring response latency. Fail-fast at startup if the webhook secret is missing, because a service that starts and silently accepts unsigned webhooks is far worse than one that crashes. Short-lived credentials — JWTs at nine minutes, installation tokens at one hour. And a read-only trust model: the agent posts comments, never commits, with GitHub permissions scoped to exactly read-code and write-PR-comments.
>
> There's a fifth that's easy to overlook and matters a lot. My ingestion pipeline reads every file in a cloned repository and **sends the contents to OpenAI** for embedding. Without filtering, a user's committed `.env` file — with their database password — gets transmitted to a third party and stored in a vector database. So the walker has an explicit deny-list for `.env` files and `.pem`/`.key` extensions. Small piece of code, large blast radius if omitted.
>
> **Gaps:** No rate limiting — my endpoint is public and an attacker can flood it with invalid signatures, each costing a SHA-256. No replay protection — HMAC proves authenticity and integrity but not *freshness*, so a captured valid request replays forever; GitHub sends `X-GitHub-Delivery` as a unique UUID and I don't track it. Token leakage risk — I embed the installation token in the clone URL, so if GitPython raises and I print the exception, that token lands in my logs. And no dependency scanning: no Dependabot, no `pip-audit`, and mostly unpinned requirements.
>
> The one I'd prioritise is **egress** rate limiting, actually — capping concurrent OpenAI calls. Unlike the others it's not just a security issue, it's a cost and reliability issue that bites in normal operation rather than only under attack.

**Follow-ups:** "How would you rotate a leaked key?" (Revoke at GitHub immediately — assume it's compromised the moment it's exposed. Then generate a new one and redeploy. Critically, deleting it from git history doesn't help; it's in every clone and every fork.)

---

## Q2. "Explain SQL injection."

### ⭐ Best answer

> String-concatenated SQL lets user input change the query's *structure* rather than just its values.
>
> ```python
> query = f"SELECT * FROM users WHERE id = {user_id}"
> ```
> With `user_id = "1 OR 1=1"` you return every row. With `"1; DROP TABLE users--"` you destroy the table.
>
> The fix is **parameterised queries**: `cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))`. The reason this works is structural, not filtering — the driver sends the SQL *structure* and the *values* on separate channels. The database parses the query plan first, then binds values as pure data. Values can never be reinterpreted as syntax. It's not that dangerous characters are escaped; it's that there's no code path by which they could be executed.
>
> That's why escaping and blocklists fail. Unicode normalisation, alternate encodings, and second-order injection — where data is stored safely then used unsafely later — all defeat them. Parameterisation is a guarantee; filtering is a guess.
>
> Defence in depth beyond that: an ORM, which parameterises by default; input validation at the boundary; a least-privilege database user so your web app can't `DROP TABLE` even if injection succeeds; and a WAF as a backstop.
>
> **The connection to my project is that I detect it rather than being vulnerable to it.** I have no SQL database, but `sql_injection` is a first-class category in my security agent's prompt, and my `education.py` module maps it to OWASP A03:2021 with a written impact statement and a reference link, so the posted comment teaches rather than just flags.

---

## Q3. "Explain XSS."

### ⭐ Best answer

> Injecting script into a page that another user's browser executes in the context of your origin. Three types.
>
> **Stored** — the payload is saved server-side and served to every viewer. Worst, because it's persistent and hits everyone. **Reflected** — the payload is in the URL and echoed into the response; needs the victim to click a crafted link. **DOM-based** — never touches the server; client-side JavaScript writes untrusted data into the DOM.
>
> ```javascript
> el.innerHTML = userComment;    // ❌ executes script tags and event handlers
> el.textContent = userComment;  // ✅ treated as text
> ```
>
> React escapes by default — `<div>{userComment}</div>` is safe. The escape hatch is `dangerouslySetInnerHTML`, named that way deliberately.
>
> The important nuance is that **escaping is context-dependent**. HTML body, HTML attribute, JavaScript, CSS, and URL contexts each need different encoding. A value safe in a body is dangerous inside an `href` if it starts with `javascript:`. That's why "escape the input" is the wrong mental model — you encode on *output*, according to where it's going.
>
> Defence in depth: **Content-Security-Policy** headers restricting script sources, which turns XSS from exploitable into merely present. **`httpOnly` cookies**, so even successful XSS can't read session tokens. And **DOMPurify** when you genuinely must allow user HTML.
>
> **For my project:** I post LLM-generated markdown to GitHub, and GitHub sanitises rendered markdown, so this isn't exploitable. But the general risk is real and I'd think about it carefully if the dashboard returned — rendering LLM output as HTML in a system that didn't sanitise would be exactly the stored-XSS pattern, with the model as the injection vector.

---

## Q4. "Explain CSRF."

### ⭐ Best answer

> An attacker's site causes a victim's browser to make an authenticated request to your site, exploiting the fact that browsers attach cookies **automatically** to any request to a domain.
>
> ```html
> <!-- on evil.com; victim is logged into bank.com -->
> <form action="https://bank.com/transfer" method="POST">
>   <input name="to" value="attacker"><input name="amount" value="10000">
> </form><script>document.forms[0].submit()</script>
> ```
> The victim never sees it. Their cookies go along. The bank sees an authenticated request.
>
> Defences: a **CSRF token** — random per-session, embedded in the form; the attacker can't read it because the same-origin policy prevents reading your page. **`SameSite` cookies** — `Strict` or `Lax` tells the browser not to send cookies on cross-site requests; `Lax` is now the default in modern browsers, which has substantially reduced CSRF's prevalence. **Requiring a custom header**, since cross-origin custom headers trigger a preflight the attacker can't satisfy.
>
> **My project is structurally immune, and the reason is precise:** CSRF requires *automatic* credential attachment, which means cookies. I don't use cookies at all. Inbound authentication is an HMAC signature over the request body; outbound is an `Authorization` header. Browsers attach neither automatically. There's also no browser in the picture — my only client is GitHub's server.
>
> The general principle: **token-in-header APIs are structurally immune to CSRF; cookie-authenticated ones need explicit defences.** If I brought back the dashboard I'd keep the API token-in-header for exactly that reason.

---

## Q5. "Do you use Redis? Explain caching."

### ⭐ Best answer

> No Redis. I have three caching layers, all in-process or on-disk, and I can tell you exactly where Redis would go when I scale.
>
> **In-process singletons** for the OpenAI client, the ChromaDB client, and tree-sitter language objects. The ChromaDB one came from a real fix — I was creating a new client per hunk, so twenty hunks times three agents meant sixty client instantiations per PR, each opening the database.
>
> There's a subtle one I like: **negative caching** on tree-sitter languages. When an import fails for an unsupported language I cache the `None`. Without that, every `.rb` file in a Ruby repo retries a failing `importlib` call, and failed imports are expensive — Python stats every directory in `sys.path`.
>
> **ChromaDB on disk is the cache that saves real money** — embeddings paid for once. And **incremental ingestion** is the biggest optimisation in the system: after the first PR, I only re-embed changed files. Five files instead of five hundred.
>
> **Where Redis goes.** Three places, all triggered by the same thing: **every cache I have is in-process, so the moment I run two instances they diverge.**
>
> A **task queue** — `asyncio.create_task` doesn't survive restarts; Redis as a Celery broker gives durability, retries with backoff, and a dead-letter queue. **Distributed rate limiting** — in-process semaphores don't compose; five workers each holding a limit of twenty gives you a hundred concurrent, because each independently thinks it's compliant. And an **embedding cache keyed by content hash**, deduplicating identical code across repositories — right now ChromaDB gives me that per-repo but not globally.
>
> Why I didn't add it: Redis is a stateful service to deploy, monitor, and back up. At one instance it would have solved no problem I actually had. Adding infrastructure before you have its problem means maintaining something that returns nothing.

**Follow-ups:** "Cache invalidation strategy?" (TTL for tokens; content-hash keys for embeddings, which are immutable so they never need invalidating — the best kind of cache.) · "Cache stampede?" (Many requests missing simultaneously and all recomputing. Fix with a lock so one recomputes while others wait, or probabilistic early expiry.)

---

## Q6. "Explain your Git workflow."

### ⭐ Best answer

> Feature branches merged into main, with the history showing genuine feature work rather than a stream of "fix" commits.
>
> Looking at my log, there are named feature branches — `feature/pr-writer`, `feature/educational-layer`, `feature/weekly-digest` — merged via pull requests. Which is fitting for a PR review tool: it reviewed its own PRs.
>
> **What I do well:** descriptive commit messages that explain *why*, not just what. My largest commit — the one deleting the critic, the database, and the frontend — has a full body listing every change with reasoning. Six months later that message is the only reason I can reconstruct the decision. That's the real value of commit messages: they're the only documentation that can't drift out of sync with the code.
>
> **What I'd do differently:** I have eight consecutive deployment-fix commits — `fix railpack.json`, `fix railpack.json again`, `move railway.toml`. Each was a genuine deploy-and-see cycle against a platform I couldn't test locally, so I understand why they exist. But squashing them into one "configure Railway deployment" commit would leave a much more readable history. On a team I'd squash-merge feature branches for exactly that reason: the branch keeps the messy iteration, main keeps the clean narrative.
>
> **On a team** I'd add branch protection requiring review and green CI, conventional commit prefixes so changelogs generate automatically, and a rule that main is always deployable.
>
> One thing I feel strongly about: **never rewrite history on a shared branch.** Force-pushing to main destroys other people's work. Rebase your own feature branch before merging — that's fine and keeps history linear — but once something's shared, it's append-only.

---

## Q7. "Explain your deployment lifecycle."

### ⭐ Best answer

> Push to main, Railway builds and deploys. No gate — which is fine solo and would be irresponsible with a team.
>
> Two config files. `railway.toml` sets the start command, health check path, a 120-second health check timeout, and an on-failure restart policy. `railpack.json` declares system-level apt packages.
>
> The start command is `uvicorn main:app --host 0.0.0.0 --port $PORT`. Both flags matter. `0.0.0.0` because `127.0.0.1` — the local dev default — means the process only accepts connections from *inside* the container, so the platform's router gets connection refused while the container looks fine internally. And `$PORT` because Railway assigns it dynamically.
>
> **The interesting part is why `railpack.json` exists.** I had a bug that only appeared in production. My repo cloning uses GitPython — which isn't a git implementation, it's a wrapper that shells out to the `git` binary. So `pip install gitpython` gives you the Python package but not the executable it needs. My Mac has git; Railway's Python build image doesn't. Every clone failed in production and worked locally.
>
> Took three attempts to fix, because Railway distinguishes language packages from system packages and I had to land on `aptPkgs`.
>
> The real lesson is that I had an **undeclared implicit system dependency**. My `requirements.txt` described my Python dependencies completely and my system dependencies not at all — and pip has no concept of "this package needs a binary on PATH." That's exactly the class of bug containers exist to prevent, and I'd undermined that by letting a buildpack infer my image instead of declaring it. With an explicit Dockerfile, `RUN apt-get install -y git` would sit next to `pip install`, visible in review and identical everywhere.
>
> **What I'd add for a team:** a CI gate — lint, type check, tests, dependency audit — before deploy. Pinned dependencies, because unpinned means two deploys of identical code can install different versions. A staging environment. And most importantly a **durable queue**, because right now a deploy mid-pipeline silently drops that review.

---

## Q8. "How do you manage environment variables?"

### ⭐ Best answer

> Twelve-factor: config in the environment, never in code. Locally a `.env` file; in production Railway environment variables. Nothing sensitive is in git — `.gitignore` covers `**/.env` and `*.pem`, and I've verified with `git ls-files` that neither is tracked.
>
> Two details worth mentioning because both came from real problems.
>
> **`load_dotenv(override=True)`.** By default `python-dotenv` will *not* overwrite an existing shell variable. If your shell exports `OPENAI_API_KEY` for another project, your `.env` is silently ignored and you spend an hour debugging why the wrong key is being used. `override=True` makes `.env` authoritative.
>
> **The GitHub private key is base64-encoded in production.** A PEM file is multi-line, and environment variables are fundamentally single-line — different platforms mangle embedded newlines differently. Some strip them, some escape them as literal `\n`, some truncate at the first one. You get an unparseable key and a cryptic RSA error. Base64 collapses it to one safe line, decoded at read time. My code handles both: base64 env var in production, file path locally.
>
> **What I'd add for production:** a proper secret manager — AWS Secrets Manager or Vault. Platform env vars are encrypted at rest, which is fine, but they give you no automatic rotation, no audit log of who accessed what, no fine-grained IAM per secret, and no versioning for rollback. Rotating a key today means editing a variable and redeploying, with no record that it happened.

---

## Q9. "How do you test?"

*(Full treatment in [11-Testing.md](11-Testing.md).)*

### ⭐ Best answer

> Poorly, and I'd rather tell you exactly why than dress it up.
>
> I have two files named `test_ingestion.py` and `test_specialist.py`, but they're not tests — they're manual verification scripts. Real OpenAI calls, real cost, output printed for me to read, can't run in CI. Naming them `test_*` was a mistake; `scripts/verify_*.py` would have been honest about what they are.
>
> They *were* genuinely useful for the problem I had, which was iterating on prompts. Before them, testing a prompt change meant pushing a commit and waiting for a webhook — a two-minute loop with side effects on a real PR. These made it twenty seconds locally.
>
> **What I'd build, and the encouraging part is that most of the risk is in pure functions.** `parse_diff_hunks` is pure — table-driven tests over fixture diffs, no mocking, no cost. The single most valuable test asserts that deleted lines don't advance the new-file line counter, because that invariant is subtle, easy to break in a refactor, and its failure looks like an LLM hallucination rather than a parser bug. Same for dedup, chunk ID determinism, and HMAC verification. Roughly 80% of my correctness risk, zero mocking.
>
> Then agents with a mocked OpenAI client — the code around the LLM is fully deterministic, so I can assert that low-confidence findings drop, that a hallucinated severity raises `ValidationError`, and that one failing hunk doesn't kill the others.
>
> **And then the part specific to AI systems: an evaluation suite.** Unit tests can't verify prompt quality — I could make a prompt dramatically worse and every test would still pass. So you need fixed diffs with known planted issues, run nightly because they cost money, asserting on *properties* rather than exact text: "at least one finding has category `hardcoded_secret` with severity high." And critically, **negative cases** — a clean refactor should produce zero findings. Without those you optimise purely for recall and your false-positive rate silently explodes.
>
> The blocker on all of it is my module-level singletons. Making the client injectable with a default is the one change that unlocks everything else.

---

## Q10. "How do you handle logging?"

### ⭐ Best answer

> `print()` statements, which is the weakest part of the backend.
>
> What's actually good is the *content* — I log entry and exit of each stage, counts at each transition, and a per-finding summary with severity and confidence. The observability instinct is right.
>
> What's wrong is the mechanism. No levels, so I can't filter debug from error in production. No structure, so I can't query "all failures for repo X." No timestamps, so I can't measure stage durations. No tracebacks — `print(e)` gives the message and loses the stack. And stdout only, so no aggregation and no alerting.
>
> **The serious one is concurrency.** Two PRs processing simultaneously interleave their output with no way to tell which line belongs to which review — and that's precisely the situation where I'd most need the logs.
>
> The fix is `structlog` with a bound context per pipeline run, emitting JSON. Every line from one review carries the same PR number and SHA, so `repo="owner/demo" AND event="pipeline.failed"` becomes a query. That's maybe two hours of work and it's the first thing I'd do before putting this in front of a team.
>
> Beyond logs, the other two pillars: **metrics** — Prometheus counters for reviews completed, findings by severity, and most importantly **estimated LLM spend**, because unbounded cost is my biggest operational risk and I currently have no visibility into it. And **traces** with OpenTelemetry once there are multiple services.
>
> One principle I'd apply: **alert on symptoms users feel, not causes.** "Reviews aren't posting" is actionable. "CPU at 80%" might be completely fine.

---

## Q11. "JSON vs XML?"

### ⭐ Best answer

> JSON, essentially always, for web APIs.
>
> | | JSON | XML |
> |---|---|---|
> | Verbosity | Compact | ~2× larger |
> | Parsing | Native in JS, fast everywhere | Slower, heavier |
> | Schema | JSON Schema | XSD, more mature |
> | Comments | ❌ | ✅ |
> | Namespaces | ❌ | ✅ |
> | Attributes vs elements | One model | Two — a persistent source of ambiguity |
>
> XML still wins in specific places: SOAP-based enterprise integration, document formats where mixed content matters (XML was designed for documents with markup *around* text), and anywhere XSD validation and XSLT transformation are entrenched.
>
> The real reason JSON won for APIs is that it maps directly onto the data structures programming languages already have — objects and arrays. XML's attributes-versus-elements distinction has no natural equivalent, so every parser has to make arbitrary decisions about how to represent it.
>
> **In my project** everything is JSON: GitHub webhook payloads, GitHub API responses, OpenAI request and response bodies, and my tool-calling schema — which is JSON Schema specifically. The one non-JSON payload is the PR diff, which I fetch as plain text with `Accept: application/vnd.github.diff`, because a unified diff is a line-oriented format that would be actively worse wrapped in JSON.
>
> Worth knowing what's newer: **Protocol Buffers** for internal service-to-service — binary, schema-first, much smaller and faster, at the cost of not being human-readable. **MessagePack** as a binary JSON. For a public API, JSON's debuggability usually outweighs the efficiency gain; for high-volume internal RPC, protobuf wins.

---

## Q12. "Explain async processing in your project."

*(Overlaps [17](17-Interview-Questions-Part-1.md) Q12 — emphasise the *why* here rather than the mechanics.)*

### ⭐ Best answer

> Async exists in my system because of one hard constraint: **GitHub times out webhook deliveries at around ten seconds, and my pipeline takes ninety.**
>
> That constraint alone determines the architecture. Synchronous processing is impossible — GitHub would mark every delivery failed and eventually disable my webhook. So the handler validates, spawns a background task, and returns 200 in about five milliseconds.
>
> The second reason is that the workload is **I/O-bound, not CPU-bound**. Almost all wall-clock time is waiting on GitHub and OpenAI. That makes async the right concurrency model rather than threads or processes: coroutines cost kilobytes each versus megabytes for threads, there's no GIL contention because nothing is CPU-bound, and no context-switch overhead.
>
> The caveat I learned the hard way: **async only helps if every call in the path is non-blocking.** I used the synchronous OpenAI client inside async functions, and my three "parallel" agents ran strictly sequentially. Nothing failed — the output was correct, it was just three times slower than it should have been.
>
> Where I opt out deliberately: ChromaDB's client is blocking with no async variant, so ingestion is written as a plain sync function and pushed to a worker thread with `asyncio.to_thread`. That's honest — the alternative is wrapping blocking code in `async def`, which is a lie the type system won't catch.
>
> **What async doesn't give me is durability**, and that's the real limitation. `asyncio.create_task` schedules work in process memory. A restart drops it silently. Async solved my latency constraint; it didn't solve reliability. That needs a queue, and the migration is one line because my pipeline is already a pure function of five JSON-serialisable scalars.

---

## Q13. "ORM vs raw SQL?"

### ⭐ Best answer

> ORM by default, raw SQL where it earns its place.
>
> **ORM wins on:** parameterisation by default, which eliminates the most common injection vector without developers having to think about it. Type safety and IDE completion. Migrations as versioned code. And database portability, though that matters less in practice than people claim — most teams never switch.
>
> **Raw SQL wins on:** complex analytical queries with window functions and CTEs, which ORMs express awkwardly if at all. Bulk operations, where an ORM's per-row object instantiation is genuinely slow. And precise query-plan control when you're optimising.
>
> The pragmatic answer is both, in the same codebase. SQLAlchemy Core sits between them — you get parameterisation and composability without the full ORM object layer.
>
> **The ORM trap is the N+1 query**, and it's insidious because the code looks innocent:
> ```python
> reviews = session.query(PRReview).all()      # 1 query
> for r in reviews: print(r.findings)          # N queries, one per review
> ```
> Lazy loading fires a query on attribute access. A hundred reviews become a hundred and one queries. The fix is eager loading — `selectinload` for to-many relations, `joinedload` for to-one — but you have to know to look.
>
> **I have a real N+1 in my own code**, in the vector store rather than an ORM. My chunk deletion loops one call per file:
> ```python
> for file_path in file_rel_paths:
>     collection.delete(where={"file": file_path})       # N round trips
> ```
> Should be `where={"file": {"$in": file_rel_paths}}` — one call. A PR deleting fifty files currently makes fifty round trips.
>
> Which is the point: **N+1 isn't an ORM problem, it's a "looping over a remote call" problem.** The ORM just makes it easy to do accidentally.

---

## Q14. "Explain CI/CD."

### ⭐ Best answer

> Continuous Integration is merging to a shared main frequently with automated verification on every merge. Continuous Delivery means main is always deployable; Continuous Deployment means it deploys automatically.
>
> **I have deployment without integration** — push to main and Railway deploys, with no test gate. That's Continuous Deployment without the CI that makes it safe.
>
> What I'd build, ordered by how fast each stage fails so feedback is quick: **lint** with ruff, seconds. **Type check** with mypy — genuinely valuable for me because a lot of my code passes dicts around. **Unit tests** against a mocked OpenAI, so they're fast, free, and deterministic. **Security scan** — `pip-audit` for known CVEs and `bandit` for Python security anti-patterns. **Build** the image. **Deploy to staging**, smoke test, then production.
>
> The gate that matters most for me specifically is **dependency pinning**. My `requirements.txt` is mostly unpinned, so two deploys of identical code can install different library versions and behave differently. Builds aren't reproducible, which undermines everything else — there's no point testing a build that isn't the one you ship. That's the first thing I'd fix and it's just a lockfile.
>
> For deployment strategy I'd want blue-green or canary rather than the rolling replace I have. Though honestly the more valuable thing for a webhook consumer is the **durable queue** — right now a deploy mid-pipeline silently drops that review, and no deployment strategy fixes that, because the work is in process memory rather than in a broker.
>
> One thing I'd be careful about: **never real API keys in CI.** Mocked clients with an obviously fake key, so a leaked CI log costs nothing.

---

## Q15. "How do you handle changing requirements?"

### ⭐ Best answer

> I have a concrete example where a core requirement was deleted mid-project.
>
> The original design had a hard requirement: **every finding must be approved by a human before posting.** That drove three components — a critic agent to pre-filter, a PostgreSQL store to hold pending findings, and a Next.js dashboard for the human to approve in.
>
> Then I measured the critic. It was rejecting about 87.5% of findings, including real bugs I'd planted. And in investigating that, I realised the requirement itself was wrong. Human approval was defeating the product's core value — the point is *fast* feedback, and if a human has to approve every finding, you've rebuilt code review with extra steps.
>
> So I removed the requirement and everything downstream of it. About a thousand lines deleted, replaced by one confidence threshold.
>
> **What I'd do differently:** I should have validated the requirement earlier and more cheaply. I built three components before testing whether the underlying assumption — that false positives were frequent enough to need a filter — was even true. I never measured my false-positive rate before designing a system to reduce it.
>
> The cheap validation would have been: post findings directly to a test PR, look at them, count the bad ones. An afternoon's work that would have told me the filter was unnecessary before I built it.
>
> **On a team**, the process answer is: get the requirement's *intent* in writing, not just the requirement. "Every finding must be approved" is a solution; "developers must not be flooded with false positives" is the actual goal. If I'd written down the goal I'd have noticed sooner that a confidence threshold serves it better than an approval queue. And I'd want a cheap experiment defined before committing to a large build.

**Evaluating:** Adaptability, and whether you distinguish requirements from goals.

---

## Q16. "Tell me about technical debt in your project."

### ⭐ Best answer

> I'll name it in three categories, because they need different responses.
>
> **Debt I took deliberately, with a plan.** `asyncio.create_task` instead of a real queue. I knew it meant no durability and no retries; at my scale the failure mode was "one review is missed and the developer pushes again," which was acceptable. The plan is a broker, and I kept the migration cheap — my pipeline is a pure function of five JSON-serialisable scalars, so it's one line at the call site. That's *deliberate prudent* debt in Fowler's terms: you know what you're borrowing and how you'll repay it.
>
> **Debt I accumulated carelessly.** My three specialist agents are about ninety percent duplicated — the `FINDINGS_SCHEMA` constant is byte-for-byte identical across three files. I built them in sequence and copy-paste was faster than designing an abstraction before I knew what varied. I'd defend that as a way to *start* — you can't factor out variation you haven't seen — but I've now seen it, so it's crossed from pragmatism into debt.
>
> And it has a concrete cost, not just an aesthetic one: only the bug agent deduplicates internally. Security and pattern can emit overlapping findings, and nobody noticed because the logic lives in one file instead of a shared base. **Duplication doesn't just cost lines, it lets behaviour drift.**
>
> **Dead code.** `backend/alembic/` is a leftover from the deleted database layer. Nothing imports it. It should just be deleted — dead code is worse than no code because it makes future readers wonder whether it matters.
>
> **How I'd manage it on a team:** track it visibly rather than in people's heads — issues tagged as debt, so it competes for prioritisation like anything else. Pay it down opportunistically when touching adjacent code. And distinguish the categories: deliberate debt with a plan is fine and often correct, careless debt should be fixed when noticed, and dead code should be deleted immediately since it costs nothing to remove.

---

## Q17. "Describe your individual contribution."

### ⭐ Best answer

> This was a solo project, so I'll be precise: I designed and built all of it — webhook handling, GitHub App authentication, the ingestion pipeline, the three agents, the RAG layer, the PR writer, and the deployment.
>
> But I'd rather tell you about the two contributions that required judgement rather than typing.
>
> **The first was deleting a thousand lines of working code.** I'd built a critic agent, a PostgreSQL layer, and a Next.js dashboard. Measurement showed the critic rejecting 87.5% of findings including real bugs, which meant the entire downstream chain existed to serve a broken premise. Deleting work you spent weeks on is genuinely harder than writing it, and I think recognising when to do it is a more valuable skill than the building was.
>
> **The second was diagnosing a bug nothing reported.** My three agents were supposed to run in parallel — LangGraph fan-out, `asyncio.gather`, everything `async def`. But I was using the *synchronous* OpenAI client, so every call blocked the event loop and they ran strictly sequentially. Nothing failed. The output was completely correct. The only symptom was that stage timings summed instead of overlapping, and I only caught it because I was looking at the numbers rather than the results.
>
> **On a team**, what I'd bring from this: I'm comfortable owning something end to end, including the unglamorous parts — I debugged an eight-commit deployment saga against a platform I couldn't reproduce locally. And I've learned to write down *why* rather than what; my largest commit message explains the full reasoning for deleting three components, and six months later that message is the only reason I can reconstruct the decision.
>
> What I'd want from a team is the thing I most obviously lacked: someone to push back before I built three components for a problem I hadn't measured.

**Note on honesty:** never say "we" for solo work. If asked directly, say it was solo — inflating team size is trivially detectable and disqualifying.

---

## Q18. "How do you collaborate with a team?"

### ⭐ Best answer

> This was solo, so let me answer from what the project taught me about collaboration rather than claiming experience I don't have.
>
> **The thing I did that would transfer directly is writing decisions down.** My commit messages explain reasoning, not just changes. The commit deleting the critic, the database, and the dashboard has a full body listing every change and why. That's the only documentation that can't drift out of sync with the code, and on a team it's how someone six months later understands why the obvious-looking thing wasn't done.
>
> **What I most obviously lacked was review.** I built a critic agent, a database, and a dashboard for a false-positive rate I never measured. A code review or a design discussion would very likely have surfaced "have you checked whether this is actually a problem?" before I spent weeks on it. The value of review isn't catching typos — it's catching bad premises early.
>
> **In a review I give**, I'd want to distinguish blocking concerns from preferences. "This has an N+1 query" is blocking. "I'd have named this differently" is a comment, not a gate. A reviewer who blocks on preferences makes people avoid review.
>
> **In a review I receive**, I try to separate the code from myself. Deleting a thousand lines of my own work was good practice for that — if I can delete something I spent weeks on because the data said to, taking a review comment isn't hard.
>
> **On disagreement:** I'd want the disagreement to be about evidence rather than opinion. My whole critic story is an argument for that — I had a strong intuition that filtering was necessary, and the measurement said otherwise. If I disagree with someone I'd rather propose the cheap experiment that settles it than argue from first principles.

---

## Q19. "How do you work under resource constraints?"

### ⭐ Best answer

> Three constraints shaped this project directly, and each produced a decision I'd defend.
>
> **Solo developer, limited time.** Every hour on a dashboard was an hour not on review quality. That directly motivated deleting the frontend and using GitHub's UI — which turned out better anyway, since GitHub's suggestion blocks give one-click fix application that I couldn't have built.
>
> **Personal budget for API calls.** That drove `text-embedding-3-small` instead of `-large`, about six and a half times cheaper for a modest quality difference. Batching a hundred chunks per call. Incremental ingestion, which is the single biggest cost saving — five files instead of five hundred after the first PR. And generating fix suggestions only *after* the confidence filter, so I don't pay for suggestions on findings I'm going to drop.
>
> **No labelled dataset.** This is the constraint I couldn't engineer around, and it's the honest limitation of the project. Without ground truth I can't measure precision or recall, so my 0.70 confidence threshold is a tuned heuristic rather than a statistically justified cutoff. What I did instead was choose a *conservative* design — advisory comments rather than blocking merges — so that being wrong is cheap. That's what you do when you can't measure: reduce the cost of being wrong rather than pretending you're right.
>
> **The general approach:** constraints usually force you toward the simpler design, and the simpler design is often better. I didn't add Redis, microservices, or Kubernetes — not because I couldn't, but because none of them solved a problem I had. Scarcity is a decent forcing function for scope discipline.
>
> Where I'd push back: constraints shouldn't compromise correctness or security. I still did HMAC verification properly, still excluded `.env` files from ingestion, still used constant-time comparison. Those are cheap and non-negotiable.

---

## Q20. "How extensible is your system?"

### ⭐ Best answer

> Extensible along the axis I designed for, less so along others — and I can be specific about both.
>
> **Adding a fourth agent is easy**, maybe fifty lines. Write a system prompt, write an agent function matching the same signature, register it as a LangGraph node with an edge from `__start__`, and add a key to `AgentState`. Nothing else changes, because the agents share no state and communicate only through the graph. A test-coverage agent or a performance agent would drop straight in. That's what LangGraph buys me and it's the main reason I've kept it despite the graph being simple now.
>
> **Adding a language is trivial** — one line in `EXTENSION_LANGUAGE_MAP` plus the `tree-sitter-*` package. And if the package is missing, the line-window fallback handles it gracefully rather than failing.
>
> **Swapping the vector store is one file.** Nothing outside `store.py` imports `chromadb`. Moving to pgvector means rewriting that module, not touching callers. That's the Repository pattern earning its keep.
>
> **Moving to a task queue is one line at the call site**, because the pipeline is a pure function of five JSON-serialisable scalars with no closures over request state. `asyncio.create_task(...)` becomes `review_pr.delay(...)`. I kept that boundary clean deliberately — designing for a migration you haven't made costs nothing if you just don't couple things unnecessarily.
>
> **Where it's rigid:** swapping the LLM provider would touch all three agents, because I'm locked to OpenAI's tool-calling format and each agent duplicates the schema. A base class with a provider abstraction would fix it, and it's the same refactor that fixes the duplication.
>
> And **swapping the embedding model requires a full re-index** — vectors from different models aren't comparable and the dimensions don't even match. That's genuine, unavoidable lock-in and it's much stickier than the LLM choice. Worth knowing which of your dependencies are cheap to change and which aren't.

---

## Q21. "What would you do with three more months?"

### ⭐ Best answer

> Reliability first, features last — because the features I have are useless if the system silently drops reviews.
>
> **Month one: correctness and safety.** Validate LLM line numbers against the parsed diff, which prevents the 422 that currently collapses ten inline comments into one flat summary. Fix the stale-chunk bug — delete before insert on modified files, so my index stops filling with phantom code. Fix the comment cleanup that deletes *every* bot's comments rather than mine. And add a semaphore to bound concurrency. All small, all high-impact.
>
> **Month two: operability.** Structured logging with correlation IDs, so concurrent reviews stop interleaving. A real test suite starting with the pure functions, which is most of my correctness risk with zero mocking. A durable queue, which fixes lost work, gives retries, and provides backpressure naturally. And cost metering with alerting, because unbounded spend is my biggest operational risk and I currently have no visibility into it.
>
> **Month three: measurement.** This is the one I'd be most excited about. I'd collect developer responses to findings — resolved, suggestion applied, or dismissed — from the GitHub API as implicit labels. That gives me the ground truth I don't have, which means I could finally measure precision and recall, sweep my confidence threshold properly instead of guessing at 0.70, and build a real evaluation suite for prompt changes.
>
> Right now I can't tell you whether a prompt change made the system better or worse. That's the biggest gap in the project, and everything else is secondary to fixing it.
>
> **What I wouldn't build:** more agents, a dashboard, or multi-provider support. All tempting, none of them address the fact that I can't measure whether the system works.

**Why this scores well:** it prioritises unglamorous reliability work over features, and identifies measurement as the root constraint.

---

## Cross-References

| Topic | File |
|---|---|
| Technical questions | [17-Interview-Questions-Part-1.md](17-Interview-Questions-Part-1.md) |
| System design | [19-System-Design-Expansion.md](19-System-Design-Expansion.md) |
| The stories behind the answers | [14-Project-Challenges.md](14-Project-Challenges.md) |
| Rapid recall | [21-Interview-Cheat-Sheet.md](21-Interview-Cheat-Sheet.md) |

**Next:** [19-System-Design-Expansion.md](19-System-Design-Expansion.md)
