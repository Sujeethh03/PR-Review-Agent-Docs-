# 11 — Testing

## Executive Summary

**Honest assessment: this is the weakest area of the project.** There are two files named `test_*.py`, but they are **not tests** — they are manual verification scripts that make real API calls, cost real money, print output for a human to read, and cannot run in CI.

That said, this file is genuinely valuable for interviews, because testing a **non-deterministic LLM system** is a hard and interesting problem that most candidates haven't thought about. Being able to explain *why* it's hard and *what you'd actually do* scores far better than claiming 90% coverage.

---

## 1. What Exists Today

### `test_ingestion.py`

Per CLAUDE.md: *"Runs a full ingest of `backend/` itself, queries ChromaDB, and runs an incremental ingest with a mock diff."*

### `test_specialist.py`

Per CLAUDE.md: *"Runs all three agents against deliberately broken/insecure sample diffs and asserts on expected findings (e.g. hardcoded secrets must be high severity, confidence 1.0). Requires `OPENAI_API_KEY` — makes real GPT-4o calls."*

### Why these are scripts, not tests

| Property | These scripts | Real tests |
|---|---|---|
| Run in CI | ❌ Need real API keys | ✅ |
| Cost | 💵 Real OpenAI spend | Free |
| Deterministic | ❌ LLM output varies | ✅ |
| Fast | ❌ Tens of seconds | Milliseconds |
| Isolated | ❌ Network, filesystem, API | ✅ |
| Assertion-based | ⚠️ Partially | ✅ |
| Discoverable by pytest | ❌ No test functions | ✅ |

**But they were genuinely valuable**, and you should say so:

> They let me iterate on prompts without opening a real PR. Before I had them, testing a prompt change meant pushing a commit to a demo repo and waiting for the webhook — a two-minute feedback loop with side effects on a real PR. These scripts made it a twenty-second local run. They were the right tool for the problem I had, which was prompt development, not regression prevention.
>
> The mistake was naming them `test_*.py`. That name implies pytest-discoverable, CI-runnable regression tests, and they're neither. `scripts/verify_ingestion.py` would have been honest about what they are.

---

## 2. The Testing Pyramid

```mermaid
graph TD
    A["E2E<br/>Few, slow, brittle, high confidence"] --> B["Integration<br/>Some, moderate speed"]
    B --> C["Unit<br/>Many, fast, cheap, isolated"]
    style A fill:#e74c3c,color:#fff
    style B fill:#f39c12,color:#000
    style C fill:#2ecc71,color:#000
```

| Level | Count | Speed | Tests |
|---|---|---|---|
| **Unit** | ~70% | ms | One function, no I/O |
| **Integration** | ~20% | 100ms–s | Modules together, mocked externals |
| **E2E** | ~10% | seconds+ | Whole system, real dependencies |

**The inverted pyramid anti-pattern** — mostly E2E tests — gives slow, flaky suites where a failure tells you *something* broke but not *what*.

---

## 3. What To Test First — And Why It's Easier Than It Looks

**The key insight for this project:** the highest-value logic is **pure and deterministic**. The LLM is non-deterministic, but everything around it isn't.

| Function | Pure? | Priority | Why |
|---|---|---|---|
| `parse_diff_hunks` | ✅ | 🔴 P0 | Line numbers drive comment placement — highest bug impact |
| `SpecialistResult.all_findings` | ✅ | 🔴 P0 | Dedup correctness |
| `_chunk_id` | ✅ | 🔴 P0 | Idempotency depends on it |
| `parse_diff_file_paths` | ✅ | 🟠 P1 | Drives incremental ingestion |
| `verify_signature` | ✅ | 🔴 P0 | Security-critical |
| `_deduplicate` (bug agent) | ✅ | 🟠 P1 | Interval overlap logic |
| `_enforce_secret_rules` | ✅ | 🟠 P1 | Security invariant |
| `format_finding_comment` | ✅ | 🟡 P2 | Output formatting |
| `get_educational_context` | ✅ | 🟡 P2 | Lookup with fallback |
| `walk_repo` | ⚠️ FS | 🟠 P1 | Secret-file exclusion is security-relevant |
| `parse_with_treesitter` | ⚠️ Lib | 🟠 P1 | Chunk boundaries |
| Agent orchestration | ⚠️ API | 🟠 P1 | Mockable |
| Prompt quality | ❌ LLM | 🟡 Eval suite | Different technique entirely |

**Roughly 80% of correctness risk is in pure functions requiring zero mocking.** That's the argument for why this is achievable, not aspirational.

---

## 4. Unit Tests

### Testing the diff parser — the highest-value suite

```python
import pytest
from app.agents.specialist.diff_utils import parse_diff_hunks

def test_simple_addition():
    diff = """diff --git a/src/auth.py b/src/auth.py
--- a/src/auth.py
+++ b/src/auth.py
@@ -10,3 +10,5 @@ def login(user):
     if not user:
         return None
+    token = make_token(user)
+    log.info("issued")
     return token
"""
    hunks = parse_diff_hunks(diff)
    assert len(hunks) == 1
    assert hunks[0].file == "src/auth.py"          # "b/" stripped
    assert "token = make_token(user)" in hunks[0].added_code
    assert "if not user" not in hunks[0].added_code # context excluded


def test_deleted_lines_do_not_advance_counter():
    """THE critical invariant — deleted lines don't exist in the new file."""
    diff = """diff --git a/a.py b/a.py
--- a/a.py
+++ b/a.py
@@ -1,5 +1,3 @@
 line1
-deleted1
-deleted2
+added1
 line2
"""
    hunks = parse_diff_hunks(diff)
    # new file: line1(1), added1(2), line2(3)
    # added1 is at NEW line 2 — if deletions counted, we'd wrongly say 4
    assert hunks[0].line_start == 1
    assert hunks[0].line_end <= 3


def test_deleted_file_produces_no_hunk():
    diff = """diff --git a/gone.py b/gone.py
--- a/gone.py
+++ /dev/null
@@ -1,2 +0,0 @@
-a
-b
"""
    assert parse_diff_hunks(diff) == []


def test_pure_deletion_hunk_skipped():
    """No added lines → nothing to comment on."""
    diff = """diff --git a/a.py b/a.py
--- a/a.py
+++ b/a.py
@@ -1,3 +1,1 @@
 keep
-remove1
-remove2
"""
    assert parse_diff_hunks(diff) == []


@pytest.mark.parametrize("path_line,expected", [
    ("+++ b/src/auth.py",       "src/auth.py"),
    ("+++ b/deep/nested/f.py",  "deep/nested/f.py"),
    ("+++ src/no-prefix.py",    "src/no-prefix.py"),
])
def test_path_extraction(path_line, expected):
    diff = f"diff --git a/x b/x\n--- a/x\n{path_line}\n@@ -1,1 +1,2 @@\n+new\n"
    assert parse_diff_hunks(diff)[0].file == expected
```

**`test_deleted_lines_do_not_advance_counter` is the most valuable test in the project.** That invariant is subtle, easy to break in a refactor, and its failure mode is comments landing on the wrong lines — which looks like an LLM hallucination rather than a parser bug. Pinning it with a test is exactly what tests are for.

### Testing deduplication

```python
from app.models.findings import Finding, AgentOutput, SpecialistResult

def make_finding(file="a.py", line=10, category="bug", conf=0.8, agent="bug"):
    return Finding(file=file, line_start=line, line_end=line, severity="medium",
                   category=category, title="t", description="d",
                   suggestion="s", confidence=conf, agent=agent)

def test_keeps_highest_confidence_on_same_line():
    result = SpecialistResult(
        bug=AgentOutput(agent="bug", findings=[make_finding(conf=0.7)]),
        security=AgentOutput(agent="security",
            findings=[make_finding(category="sqli", conf=0.95, agent="security")]),
        pattern=AgentOutput(agent="pattern", findings=[]),
    )
    findings = result.all_findings()
    assert len(findings) == 1
    assert findings[0].confidence == 0.95


def test_different_lines_both_survive():
    result = SpecialistResult(
        bug=AgentOutput(agent="bug",
            findings=[make_finding(line=10), make_finding(line=20)]),
        security=AgentOutput(agent="security", findings=[]),
        pattern=AgentOutput(agent="pattern", findings=[]),
    )
    assert len(result.all_findings()) == 2


@pytest.mark.xfail(reason="KNOWN BUG: dedup keys on exact line_start, "
                          "so overlapping ranges both survive")
def test_overlapping_ranges_should_dedupe():
    f1 = Finding(file="a.py", line_start=40, line_end=45, severity="high",
                 category="bug", title="t", description="d", suggestion="s",
                 confidence=0.9, agent="bug")
    f2 = f1.model_copy(update={"line_start": 42, "line_end": 47, "confidence": 0.8})
    result = SpecialistResult(
        bug=AgentOutput(agent="bug", findings=[f1, f2]),
        security=AgentOutput(agent="security", findings=[]),
        pattern=AgentOutput(agent="pattern", findings=[]),
    )
    assert len(result.all_findings()) == 1     # currently returns 2
```

**`@pytest.mark.xfail` is a great technique to demonstrate.** It documents a known bug *as an executable specification*. The suite stays green, but the moment someone fixes the bug the test "unexpectedly passes" (`XPASS`) and tells you to remove the marker. It's a TODO that can't rot.

### Testing security-critical functions

```python
import hmac, hashlib
from main import verify_signature

def test_valid_signature_accepted(monkeypatch):
    secret = b"test-secret"
    monkeypatch.setattr("main.WEBHOOK_SECRET", secret.decode())
    body = b'{"action":"opened"}'
    sig = "sha256=" + hmac.new(secret, body, hashlib.sha256).hexdigest()
    assert verify_signature(body, sig) is True

def test_tampered_body_rejected(monkeypatch):
    secret = b"test-secret"
    monkeypatch.setattr("main.WEBHOOK_SECRET", secret.decode())
    body = b'{"action":"opened"}'
    sig = "sha256=" + hmac.new(secret, body, hashlib.sha256).hexdigest()
    assert verify_signature(b'{"action":"closed"}', sig) is False

def test_empty_signature_rejected():
    assert verify_signature(b"anything", "") is False
```

### Testing the secret-file exclusion

```python
def test_walker_excludes_secret_files(tmp_path):
    """Security-critical: ingestion sends file contents to OpenAI."""
    (tmp_path / "app.py").write_text("print('hello')")
    (tmp_path / ".env").write_text("DB_PASSWORD=hunter2")
    (tmp_path / "key.pem").write_text("-----BEGIN RSA PRIVATE KEY-----")

    files = walk_repo(str(tmp_path), IngestionConfig())
    names = {os.path.basename(f) for f in files}

    assert "app.py" in names
    assert ".env" not in names       # would exfiltrate the user's DB password
    assert "key.pem" not in names
```

**`tmp_path` is a pytest built-in fixture** giving a unique temporary directory per test, auto-cleaned. No mocking needed for filesystem code.

---

## 5. Testing LLM Agents with Mocks

**The key realisation: the agent code is deterministic. Only the LLM isn't.** Mock the LLM and everything else becomes ordinary testing.

```python
import pytest, json
from unittest.mock import AsyncMock, MagicMock, patch

def make_openai_response(findings: list[dict]):
    """Build a fake OpenAI tool-call response."""
    tool_call = MagicMock()
    tool_call.function.arguments = json.dumps({"findings": findings})
    message = MagicMock()
    message.tool_calls = [tool_call]
    choice = MagicMock()
    choice.message = message
    response = MagicMock()
    response.choices = [choice]
    return response


FINDING = {
    "file": "a.py", "line_start": 10, "line_end": 12,
    "severity": "high", "category": "null_reference",
    "title": "Null dereference", "description": "d",
    "suggestion": "s", "confidence": 0.9,
}


@pytest.mark.asyncio
async def test_low_confidence_findings_dropped():
    low = {**FINDING, "confidence": 0.5}       # below the 0.6 floor
    mock_client = MagicMock()
    mock_client.chat.completions.create = AsyncMock(
        return_value=make_openai_response([FINDING, low]))

    with patch("app.agents.specialist.bug.agent._get_client", return_value=mock_client), \
         patch("app.agents.specialist.bug.agent.get_context_for_hunk",
               new=AsyncMock(return_value=[])):
        result = await run_bug_agent({
            "diff": SAMPLE_DIFF, "collection_name": "test",
            "owner": "o", "repo_name": "r",
            "bug_output": None, "security_output": None, "pattern_output": None,
        })

    findings = result["bug_output"].findings
    assert len(findings) == 1
    assert findings[0].confidence == 0.9


@pytest.mark.asyncio
async def test_invalid_severity_raises():
    """Pydantic must reject hallucinated enum values."""
    bad = {**FINDING, "severity": "critical"}      # not in the Literal
    mock_client = MagicMock()
    mock_client.chat.completions.create = AsyncMock(
        return_value=make_openai_response([bad]))

    with patch("app.agents.specialist.bug.agent._get_client", return_value=mock_client), \
         patch("app.agents.specialist.bug.agent.get_context_for_hunk",
               new=AsyncMock(return_value=[])):
        result = await run_bug_agent({...})

    # The ValidationError is caught by the agent's try/except
    assert result["bug_output"].error is not None


@pytest.mark.asyncio
async def test_one_failing_hunk_does_not_kill_others():
    """return_exceptions=True must isolate failures."""
    mock_client = MagicMock()
    mock_client.chat.completions.create = AsyncMock(
        side_effect=[make_openai_response([FINDING]),
                     Exception("API timeout"),
                     make_openai_response([FINDING])])
    ...
    assert len(result["bug_output"].findings) == 2      # 2 of 3 survived


def test_secret_rule_override():
    """Pure function — no mocking needed at all."""
    f = Finding(file="a.py", line_start=1, line_end=1, severity="low",
                category="hardcoded_secret", title="t", description="d",
                suggestion="s", confidence=0.3, agent="security")
    out = _enforce_secret_rules([f])
    assert out[0].severity == "high"
    assert out[0].confidence == 1.0
    assert f.severity == "low"          # original unmutated (model_copy)
```

**Note that last assertion** — it verifies `model_copy` returns a new object rather than mutating. Testing immutability guarantees is a nice touch.

---

## 6. Integration Tests

```python
from fastapi.testclient import TestClient
import hmac, hashlib, json

SECRET = "test-secret"

def sign(body: bytes) -> str:
    return "sha256=" + hmac.new(SECRET.encode(), body, hashlib.sha256).hexdigest()

def test_valid_webhook_returns_200(monkeypatch):
    monkeypatch.setattr("main.WEBHOOK_SECRET", SECRET)
    from main import app
    client = TestClient(app)

    body = json.dumps({
        "action": "opened", "number": 7,
        "pull_request": {"number": 7, "head": {"sha": "abc123"},
                         "user": {"login": "dev"}, "title": "t"},
        "repository": {"full_name": "owner/repo"},
        "installation": {"id": 123},
    }).encode()

    with patch("main.asyncio.create_task") as mock_task:     # don't run the pipeline
        r = client.post("/webhook", content=body,
                        headers={"X-Hub-Signature-256": sign(body),
                                 "X-GitHub-Event": "pull_request"})
    assert r.status_code == 200
    mock_task.assert_called_once()                            # pipeline WAS spawned


def test_invalid_signature_returns_401(monkeypatch):
    monkeypatch.setattr("main.WEBHOOK_SECRET", SECRET)
    client = TestClient(app)
    r = client.post("/webhook", content=b'{"action":"opened"}',
                    headers={"X-Hub-Signature-256": "sha256=deadbeef",
                             "X-GitHub-Event": "pull_request"})
    assert r.status_code == 401


def test_self_generated_event_does_not_spawn_pipeline(monkeypatch):
    """The feedback-loop guard — a real bug that was fixed."""
    monkeypatch.setattr("main.WEBHOOK_SECRET", SECRET)
    body = json.dumps({"action": "submitted"}).encode()
    with patch("main.asyncio.create_task") as mock_task:
        r = client.post("/webhook", content=body,
                        headers={"X-Hub-Signature-256": sign(body),
                                 "X-GitHub-Event": "pull_request_review"})
    assert r.status_code == 200
    mock_task.assert_not_called()          # ← the actual assertion that matters
```

**That last test is a regression test for a real production bug** (commit `021d87e`, the infinite feedback loop). Regression tests for bugs you actually hit are the highest-value tests you can write.

---

## 7. Evaluation Suites — Testing the AI Itself

**This is the part most candidates have never considered, and it's what separates a good answer from a great one.**

Unit tests verify code. They cannot verify **prompt quality**. You could make a prompt dramatically worse and every unit test would still pass.

### The problem with asserting on LLM output

```python
# ❌ Will fail constantly — wording varies run to run
assert finding.title == "SQL Injection vulnerability in user query"
```

### The solution: property-based assertions

```python
# ✅ Assert on PROPERTIES, not exact text
@pytest.mark.eval          # separate marker — not run in normal CI
@pytest.mark.asyncio
async def test_detects_hardcoded_secret():
    diff = """diff --git a/cfg.py b/cfg.py
--- a/cfg.py
+++ b/cfg.py
@@ -1,1 +1,2 @@
 import os
+API_KEY = "sk-proj-abc123realLookingKeyMaterial"
"""
    result = await run_security_agent({"diff": diff, "collection_name": "empty", ...})
    findings = result["security_output"].findings

    assert len(findings) >= 1                                    # found something
    secret = [f for f in findings if f.category == "hardcoded_secret"]
    assert len(secret) >= 1                                      # right category
    assert secret[0].severity == "high"                          # forced rule
    assert secret[0].confidence == 1.0                           # forced rule
    assert 1 <= secret[0].line_start <= 2                        # plausible line
```

### A proper evaluation harness

```python
EVAL_CASES = [
    {"name": "hardcoded_api_key", "diff": ..., "expect_category": "hardcoded_secret",
     "expect_severity": "high", "must_detect": True},
    {"name": "sql_injection_fstring", "diff": ..., "expect_category": "sql_injection",
     "must_detect": True},
    {"name": "infinite_loop", "diff": ..., "expect_agent": "bug", "must_detect": True},
    # NEGATIVE cases matter just as much:
    {"name": "clean_refactor", "diff": ..., "must_detect": False},
    {"name": "added_docstring", "diff": ..., "must_detect": False},
]

async def run_eval() -> dict:
    tp = fp = fn = 0
    for case in EVAL_CASES:
        findings = await run_all_agents(case["diff"])
        detected = any(f.category == case.get("expect_category") for f in findings)
        if case["must_detect"] and detected:      tp += 1
        elif case["must_detect"] and not detected: fn += 1
        elif not case["must_detect"] and findings: fp += len(findings)

    precision = tp / (tp + fp) if (tp + fp) else 0
    recall    = tp / (tp + fn) if (tp + fn) else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) else 0
    return {"precision": precision, "recall": recall, "f1": f1}
```

**Negative cases are the ones people forget.** A clean refactor or an added docstring should produce *zero* findings. Without negative cases you optimise purely for recall and your false-positive rate silently explodes — which is exactly the failure that kills adoption.

**Run these nightly, not per-commit** — they cost real money and take real time. Track precision/recall/F1 over time. A prompt change that improves recall by 5% while dropping precision by 20% is a regression, and only an eval suite tells you that.

### Confidence threshold tuning

```python
def sweep_threshold(labelled_findings: list[tuple[Finding, bool]]):
    """Find the threshold that maximises F1 against ground truth."""
    for t in [0.5, 0.6, 0.65, 0.70, 0.75, 0.8, 0.85, 0.9]:
        posted = [(f, is_real) for f, is_real in labelled_findings if f.confidence >= t]
        tp = sum(1 for _, real in posted if real)
        fp = sum(1 for _, real in posted if not real)
        fn = sum(1 for f, real in labelled_findings if real and f.confidence < t)
        print(f"{t}: P={tp/(tp+fp):.2f} R={tp/(tp+fn):.2f}")
```

**This is what would justify the 0.70 threshold empirically.** Right now it's a tuned guess. Naming that honestly is worth a lot.

---

## 8. Coverage

```bash
pytest --cov=app --cov-report=html --cov-report=term-missing
```

| Metric | Measures | Weakness |
|---|---|---|
| Line | Lines executed | Ignores branches |
| **Branch** | Both sides of each `if` | Better default |
| Function | Functions called | Coarse |

**Coverage is necessary, not sufficient.** This test gives 100% coverage of `parse_diff_hunks` and asserts almost nothing:

```python
def test_useless():
    parse_diff_hunks(SOME_DIFF)     # no assertion — 100% covered, 0% verified
```

**Realistic targets:** 80% overall; ~95% for pure business logic; low coverage acceptable for thin adapter code that's mostly third-party calls.

**Mutation testing** (`mutmut`) is the real check: it mutates your code (flips `<` to `<=`, changes constants) and verifies your tests *fail*. Surviving mutants mean your tests cover the line but don't verify its behaviour.

---

## 9. Test Doubles

| Double | Purpose | Example |
|---|---|---|
| **Dummy** | Fills a parameter, never used | `None` for an unused config |
| **Stub** | Returns canned data | Fake OpenAI response |
| **Spy** | Records calls | `mock_task.assert_called_once()` |
| **Mock** | Pre-programmed with expectations | `AsyncMock(side_effect=[...])` |
| **Fake** | Working lightweight implementation | In-memory ChromaDB |

**What to mock here:**

| Dependency | Mock? | Why |
|---|---|---|
| OpenAI | ✅ Always | Cost, latency, non-determinism |
| GitHub API | ✅ Always | Side effects on real PRs |
| ChromaDB | ⚠️ Use a temp dir | It's embedded — a real instance is cheap |
| Filesystem | ❌ Use `tmp_path` | Real FS is fast and more faithful |
| tree-sitter | ❌ Real | Deterministic, fast, and the thing under test |

**The rule: mock what's slow, costly, or has side effects. Don't mock what's fast and deterministic** — mocking tree-sitter would test your mock, not your parser.

---

## 10. The Testability Blocker

**Module-level singletons make mocking awkward:**

```python
_client: AsyncOpenAI | None = None

def _get_client() -> AsyncOpenAI:
    global _client
    if _client is None:
        _client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    return _client
```

To mock this you patch a private module-level function, which is fragile and leaks state between tests (the singleton persists).

**The refactor that unlocks testing:**

```python
async def _process_hunk(hunk, collection_name, client: AsyncOpenAI | None = None):
    client = client or _get_client()      # injectable, defaults to singleton
    ...
```

Backwards compatible, zero production change, and tests can now pass a fake directly. **This is the single change that makes everything else testable** — worth naming as the first step in any testing improvement plan.

---

## 11. Interview Questions

### Q1. "How do you test your project?"

**Best answer — lead with honesty:**
> Poorly, and I can tell you exactly why and what I'd fix.
>
> What I have is two files named `test_ingestion.py` and `test_specialist.py`, but they're not tests — they're manual verification scripts. They make real OpenAI calls, cost real money, print output for me to read, and can't run in CI. Naming them `test_*` was a mistake; `scripts/verify_*.py` would have been honest.
>
> They *were* genuinely useful, though, for the problem I actually had: iterating on prompts. Before them, testing a prompt change meant pushing a commit and waiting for a webhook — a two-minute loop with side effects on a real PR. These made it twenty seconds locally.
>
> What I'd build, and the encouraging part is that most of the risk is in pure functions. `parse_diff_hunks` is pure — table-driven tests over fixture diffs, no mocking, no cost. The single most valuable test I'd write asserts that deleted lines don't advance the new-file line counter, because that invariant is subtle, easy to break in a refactor, and its failure looks like an LLM hallucination rather than a parser bug. Same for the dedup logic, chunk ID determinism, and HMAC verification. That's maybe 80% of my correctness risk with zero mocking.
>
> Then agents with a mocked OpenAI client — the agent code around the LLM is completely deterministic, so I can assert that low-confidence findings get dropped, that a hallucinated severity raises `ValidationError`, and that one failing hunk doesn't kill the others.
>
> Then integration tests with FastAPI's `TestClient` for the webhook — signed payload returns 200, unsigned returns 401, and a `pull_request_review` event doesn't spawn a pipeline. That last one is a regression test for a real infinite-loop bug I hit.
>
> The blocker on all of it is my module-level singletons. Making the client injectable with a default is the one change that unlocks everything else.

---

### Q2. "How do you test something non-deterministic?"

**Best answer — this is the differentiating question:**
> You separate the two problems, because they need completely different techniques.
>
> **The code around the LLM is deterministic.** Parsing, validation, filtering, deduplication, formatting — none of that varies. I mock the OpenAI client with a canned tool-call response and test it like any other code. That covers most of the system.
>
> **The LLM itself needs evaluation, not testing.** You can't assert on exact output — the wording varies every run, so `assert title == "SQL Injection in query"` fails constantly. Instead you assert on *properties*: given a diff with a hardcoded API key, at least one finding has category `hardcoded_secret`, severity high, and a line number inside the diff. Those hold across runs even though the prose doesn't.
>
> Then you build an evaluation suite — a fixed set of diffs with known planted issues, run nightly rather than per-commit because it costs real money, tracking precision, recall, and F1 over time. That's regression testing for *prompts*, which unit tests fundamentally cannot do. I could make a prompt dramatically worse and every unit test would still pass.
>
> The part people forget is **negative cases**. A clean refactor or an added docstring should produce zero findings. Without those in the suite you optimise purely for recall, and your false-positive rate silently explodes — which is exactly the failure mode that gets a review bot muted.
>
> And that connects to something I genuinely lack: I have no labelled dataset, so I can't tell you my precision. My 0.70 confidence threshold is a tuned heuristic, not a statistically justified cutoff. With ground truth I'd sweep the threshold and pick the F1 maximum. The practical source of labels is developer behaviour — did they resolve the comment, apply the suggestion, or dismiss it? That's an implicit label I could collect from the GitHub API and I'm not collecting it.

**Why this scores extremely well:** it demonstrates a conceptual distinction most candidates don't have (testing vs evaluation), names a specific commonly-missed detail (negative cases), and connects it to an honest measurement gap with a concrete plan.

---

### Q3. "What's your test coverage?"

**Best answer:**
> Effectively zero for automated tests, and I'd rather say that than quote a number.
>
> But I'd push back gently on coverage as the metric. Coverage measures which lines executed, not whether behaviour was verified. I could call `parse_diff_hunks` with no assertions and report 100% coverage of it while verifying nothing.
>
> If I were setting targets I'd aim for 80% overall with roughly 95% on pure business logic — the diff parser, dedup, chunk IDs — and accept low coverage on thin adapter code that's mostly third-party calls, where the tests would mostly assert that mocks were called.
>
> The metric I'd actually trust is mutation testing. `mutmut` flips comparison operators and changes constants, then checks whether my tests fail. Surviving mutants mean the line is covered but its behaviour isn't verified. That measures test *quality* rather than test *presence*.
>
> For this project specifically, the metric I'd care most about isn't code coverage at all — it's precision and recall on my evaluation suite. A 95%-covered codebase that posts wrong comments is worse than a 60%-covered one that posts right ones.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Calling real APIs in tests | Slow, costly, flaky, can't run in CI | Mock external services |
| Asserting exact LLM text | Fails constantly | Assert properties |
| No negative eval cases | False-positive rate explodes unmeasured | Include clean-diff cases |
| Coverage as the only metric | High coverage, low verification | Mutation testing |
| Inverted pyramid (mostly E2E) | Slow, flaky, poor failure localisation | Mostly unit |
| Mocking what you're testing | Tests the mock | Only mock slow/costly/side-effecting |
| Shared state between tests | Order-dependent flakiness | Fresh fixtures; reset singletons |
| Naming scripts `test_*.py` | Implies CI-runnable regression tests | Name them `scripts/verify_*.py` |
| No regression test after a bugfix | Same bug returns | Test the bug first, then fix |

---

## 13. Cross-References

| Topic | File |
|---|---|
| Functions worth testing | [03-Low-Level-Design.md](03-Low-Level-Design.md) |
| DI as the testability blocker | [05-Backend-Architecture.md](05-Backend-Architecture.md) |
| Why LLM output varies | [09-AI-System.md](09-AI-System.md) |
| Where tests fit in CI | [10-Deployment-and-DevOps.md](10-Deployment-and-DevOps.md) |
| Testing on the roadmap | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [12-Performance-Optimization.md](12-Performance-Optimization.md)
