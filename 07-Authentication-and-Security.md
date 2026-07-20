# 07 — Authentication and Security

## Executive Summary

This project has **no user authentication** — no login, no passwords, no sessions, no bcrypt. It has **machine-to-machine authentication** in two directions:

| Direction | Mechanism | Purpose |
|---|---|---|
| **Inbound** (GitHub → us) | HMAC-SHA256 signature | Prove the webhook is genuinely from GitHub |
| **Outbound** (us → GitHub) | RS256 JWT → installation token | Prove we are the registered GitHub App |

Both are real, both are correctly implemented, and both are excellent interview material — **you have a genuine JWT implementation to discuss**, just not the user-session kind most candidates describe.

This file covers what's implemented, then the security theory (OAuth, bcrypt, CSRF, XSS, SQLi, rate limiting) you'll be asked about regardless.

---

# PART A — Inbound: HMAC Webhook Verification

## A1. The threat

`POST /webhook` is a **public internet endpoint**. Anyone can find it and POST to it. Without verification, an attacker could forge a `pull_request` event and make the service:

- Clone arbitrary repositories (data exfiltration attempt)
- Burn OpenAI credits (financial denial of service)
- Post comments on PRs (reputational damage)

## A2. The implementation

```python
WEBHOOK_SECRET = os.getenv("GITHUB_WEBHOOK_SECRET")
if not WEBHOOK_SECRET:
    raise RuntimeError("GITHUB_WEBHOOK_SECRET is not set")      # ① fail fast

def verify_signature(payload_body: bytes, signature_header: str) -> bool:
    if not signature_header:                                     # ②
        return False
    expected_signature = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode("utf-8"),                          # ③ key
        payload_body,                                            # ④ raw bytes
        hashlib.sha256,                                          # ⑤ algorithm
    ).hexdigest()
    return hmac.compare_digest(expected_signature, signature_header)   # ⑥
```

### Line by line

**① Fail fast at import.** If the secret isn't configured, the process refuses to start. The alternative — starting and silently accepting everything, or rejecting everything — is far worse. **Fail loudly at startup, never silently at runtime.**

**② Empty header → reject.** Defends against a request with no signature at all.

**③ The shared secret** is the HMAC key. GitHub holds the same value (set in the App's config).

**④ Raw bytes, not parsed JSON.** This is the crucial detail. HMAC is computed over the exact byte sequence GitHub sent. Parsing to a dict and re-serialising produces different whitespace and key ordering — and a completely different signature.

```python
payload_body = await request.body()      # ✅ raw bytes
# NOT: payload = await request.json()    # ❌ signature can never match
```

**⑤ SHA-256** — GitHub also sends a legacy SHA-1 header (`X-Hub-Signature`). Ignore it; SHA-1 is cryptographically broken.

**⑥ `hmac.compare_digest`, not `==`.**

## A3. Why constant-time comparison matters

Python's `==` on strings **short-circuits at the first differing byte**:

```python
"abcdef" == "xbcdef"   # fails at byte 0 — fast
"abcdef" == "abcdex"   # fails at byte 5 — measurably slower
```

That timing difference is an information leak. An attacker can forge a signature **byte by byte**:

```
Try signature "0...":  response in 10.001ms
Try signature "1...":  response in 10.001ms
...
Try signature "a...":  response in 10.014ms  ← slower! first byte is 'a'
Now brute-force byte 2 with 'a' fixed...
```

This reduces the search from 256^64 (infeasible) to 256 × 64 = 16,384 attempts (trivial).

`hmac.compare_digest` **always compares every byte** regardless of where the mismatch occurs, so response time carries no information.

```python
# Conceptually:
def compare_digest(a, b):
    result = 0
    for x, y in zip(a, b):
        result |= x ^ y      # XOR accumulates differences, never exits early
    return result == 0 and len(a) == len(b)
```

**This is one of the best security answers you can give in an interview** — it's specific, it demonstrates you understand side-channel attacks, and it's genuinely in your code.

## A4. HMAC vs plain hashing

```
❌ Plain hash:  signature = SHA256(payload)
   Anyone can compute this. Proves nothing.

✅ HMAC:        signature = HMAC-SHA256(secret, payload)
   Only someone holding the secret can compute it.
```

**HMAC's actual construction:**
```
HMAC(K, m) = H((K ⊕ opad) ‖ H((K ⊕ ipad) ‖ m))
```
The nested hashing with two different padded keys defends against **length-extension attacks**, which naive `H(secret ‖ message)` constructions are vulnerable to. Knowing this distinction is a strong signal.

**What HMAC proves:** authenticity (it came from someone with the secret) and integrity (the payload wasn't modified).
**What it does not prove:** freshness. See replay attacks below.

## A5. Known gaps

| Gap | Risk | Fix |
|---|---|---|
| **No replay protection** | A captured valid request can be re-sent indefinitely | Track `X-GitHub-Delivery` UUIDs in a TTL cache; reject duplicates |
| **No rate limiting** | Attacker floods with invalid signatures; each costs a SHA-256 | `slowapi` per-IP limit |
| **No request size limit** | 100 MB payload consumes memory before verification | Reverse-proxy `client_max_body_size` |
| **No IP allowlist** | Anyone can reach the endpoint | GitHub publishes hook IPs at `/meta` |

**On replay attacks specifically** — a good interview answer:
> HMAC proves authenticity and integrity but says nothing about *freshness*. If an attacker captures one valid request — say from a logging misconfiguration or a compromised proxy — they can replay it forever and I'd accept it every time, re-running the pipeline and re-burning API credits. GitHub sends `X-GitHub-Delivery`, a UUID unique per delivery. Caching seen IDs for, say, 24 hours in Redis and rejecting duplicates closes it. I haven't implemented it, and it's on my list.

---

# PART B — Outbound: GitHub App JWT

## B1. The three-token dance

```mermaid
sequenceDiagram
    autonumber
    participant App as Our Service
    participant GH as GitHub

    Note over App: 1. Sign a JWT with our RSA private key
    App->>App: JWT{iat, exp, iss: APP_ID}, RS256
    Note over App: Proves "I am App #123456"

    App->>GH: POST /app/installations/{id}/access_tokens<br/>Authorization: Bearer <JWT>
    GH->>GH: Verify with our stored PUBLIC key
    GH-->>App: 2. Installation token (1 hour, scoped)
    Note over GH: Proves "I may act for this installation"

    App->>GH: GET /repos/{o}/{r}/pulls/{n}<br/>Authorization: Bearer <install token>
    GH-->>App: 3. The resource
```

| Token | Lifetime | Signed with | Grants |
|---|---|---|---|
| **App JWT** | 9 min | Our RSA private key | Identity as the App only |
| **Installation token** | 1 hour | GitHub (opaque) | Scoped repo access |

**Why two steps rather than one?** Separation of identity from authorisation. The JWT says *who you are*; the installation token says *what you may touch*. An App installed on 500 organisations gets a different, independently-scoped token for each. If one leaks, its blast radius is one installation for one hour.

## B2. The implementation

```python
def _generate_jwt() -> str:
    app_id = os.getenv("GITHUB_APP_ID")

    raw = os.getenv("GITHUB_PRIVATE_KEY")           # ① production
    if raw:
        import base64
        private_key = base64.b64decode(raw).decode("utf-8")
    else:                                            # ② local dev
        key_path = os.getenv("GITHUB_PRIVATE_KEY_PATH")
        with open(key_path, "r") as f:
            private_key = f.read()

    now = int(time.time())
    payload = {
        "iat": now - 60,      # ③ 60s in the past — clock skew tolerance
        "exp": now + 540,     # ④ 9 minutes (GitHub's max is 10)
        "iss": app_id,        # ⑤ issuer = the App ID
    }
    return jwt.encode(payload, private_key, algorithm="RS256")   # ⑥
```

**① / ② Dual key loading.** Production supplies a **base64-encoded** key in an env var; local dev supplies a file path.

**Why base64?** A PEM file is multi-line:
```
-----BEGIN RSA PRIVATE KEY-----
MIIEow...
-----END RSA PRIVATE KEY-----
```
Most environment-variable systems mangle embedded newlines — some strip them, some escape them, some truncate at the first one. Base64 encodes the whole thing to a single safe line. This is a **real deployment problem with a real solution** and worth mentioning.

**③ `iat: now - 60`.** Backdating by 60 seconds tolerates clock skew between our server and GitHub's. Without it, if our clock is 5 seconds fast, GitHub sees a token "issued in the future" and rejects it. Small detail, painful bug.

**④ `exp: now + 540`.** Nine minutes against GitHub's ten-minute maximum. Short lifetime limits the damage window if a token leaks.

**⑤ `iss`** — the claim GitHub uses to look up which public key to verify with.

**⑥ RS256, not HS256.** This is the critical algorithm choice.

## B3. RS256 vs HS256 — the algorithm question

| | HS256 (symmetric) | RS256 (asymmetric) |
|---|---|---|
| Keys | One shared secret | Private (sign) + public (verify) |
| Who can sign | Anyone with the secret | Only the private key holder |
| Who can verify | Only secret holders | Anyone with the public key |
| Key distribution | Must share the secret | Public key is public |
| Use when | Single trusted party | **Multiple verifiers / third-party** |

**Why GitHub requires RS256:** GitHub needs to *verify* our tokens without being able to *forge* them. With HS256 we'd both hold the same secret, and GitHub could mint tokens as us. With RS256 we hold the private key and GitHub holds only the public key. GitHub can verify, and cannot impersonate.

**The `alg: none` attack — a classic interview question.** Early JWT libraries honoured the algorithm declared in the token's *header*. An attacker could take a valid token, change the header to `{"alg":"none"}`, strip the signature, and some libraries would accept it. Related: changing `RS256` to `HS256` and signing with the *public* key as the HMAC secret — since the public key is public, anyone could forge.

**The defence:** always specify expected algorithms explicitly at verification:
```python
jwt.decode(token, key, algorithms=["RS256"])   # never trust the header's alg
```
PyJWT requires the `algorithms` parameter for exactly this reason.

## B4. JWT structure

```
eyJhbGciOiJSUzI1NiJ9  .  eyJpYXQiOjE3MjE0...  .  Ky8xN2VmZ...
└──── HEADER ────────┘    └──── PAYLOAD ────┘    └─ SIGNATURE ─┘
   {"alg":"RS256",           {"iat":...,           RSA-SHA256 over
    "typ":"JWT"}              "exp":...,           header.payload
                              "iss":"123456"}
```

Each part is **base64url encoded, not encrypted**.

**The single most important JWT fact:** *anyone can read a JWT's payload.* Paste one into jwt.io and read every claim. The signature guarantees **integrity** (nobody modified it), not **confidentiality**.

**Therefore: never put secrets in a JWT.** No passwords, no PII, no API keys.

### Standard claims

| Claim | Name | Purpose |
|---|---|---|
| `iss` | Issuer | Who created it — used here |
| `sub` | Subject | Who it's about (usually user ID) |
| `aud` | Audience | Who it's for |
| `exp` | Expiration | Must reject after — used here |
| `nbf` | Not before | Reject before |
| `iat` | Issued at | Creation time — used here |
| `jti` | JWT ID | Unique ID, enables revocation lists |

## B5. Token handling in the clone URL

```python
token = await get_installation_token(installation_id)
clone_url = f"https://x-access-token:{token}@github.com/{owner}/{repo}.git"
git.Repo.clone_from(clone_url, clone_path, depth=1)
```

**This works but has a real risk:** credentials embedded in a URL can leak into process listings (`ps aux`), shell history, and — most commonly — **error messages**. If GitPython raises and the exception message includes the URL, the token lands in your logs.

```python
except Exception as e:
    print(f"Clone failed: {e}")     # ← may print the token
```

**Better approaches:**
```python
# Option A — credentials via environment, not URL
env = {"GIT_ASKPASS": "echo", "GIT_USERNAME": "x-access-token",
       "GIT_PASSWORD": token}
git.Repo.clone_from(url_without_token, path, depth=1, env=env)

# Option B — at minimum, redact before logging
def _redact(msg: str, token: str) -> str:
    return msg.replace(token, "***REDACTED***")
```

**Volunteering this is a strong security signal.** It shows you think about *credential lifecycle*, not just credential storage.

---

# PART C — Secrets Management

## C1. What's done correctly ✅

Verified with `git ls-files` — **no secrets are tracked in git.**

```gitignore
# .gitignore (root)
**/.env
*.pem

# backend/.gitignore
.env          # environment variables — never push this
*.pem         # your private key — never push this
```

| Secret | Storage (dev) | Storage (prod) |
|---|---|---|
| `GITHUB_APP_ID` | `.env` | Railway env var |
| `GITHUB_WEBHOOK_SECRET` | `.env` | Railway env var |
| `GITHUB_PRIVATE_KEY_PATH` | `.env` → `.pem` on disk | — |
| `GITHUB_PRIVATE_KEY` | — | Railway env var (base64) |
| `OPENAI_API_KEY` | `.env` | Railway env var |

**The `load_dotenv(override=True)` detail** is worth knowing:
```python
load_dotenv(override=True)
```
By default `python-dotenv` will **not** overwrite an existing shell environment variable. If your shell exports `OPENAI_API_KEY` for another project, `.env` would be silently ignored and you'd debug the wrong key for an hour. `override=True` makes `.env` authoritative. CLAUDE.md documents this explicitly — good practice.

## C2. Ingestion never indexes secrets ✅

```python
SKIP_FILENAMES = {".env", ".env.local", ".env.production", ".env.development"}
SKIP_EXTENSIONS = {".pem", ".key", ".lock", ".pyc", ".map"}
```

**This is important and easy to overlook.** The ingestion pipeline reads *every file* in a cloned repository and sends the contents to OpenAI for embedding. Without this skip list, a user's `.env` file — with their database passwords and API keys — would be transmitted to a third party and stored in a vector database.

**Say this in an interview:** *"My ingestion pipeline sends file contents to OpenAI. That means an unfiltered walk would exfiltrate any secrets committed to the user's repo. So the walker has an explicit deny-list for `.env` files and `.pem`/`.key` extensions. It's a small piece of code with a large blast radius if omitted."*

**Improvements worth naming:**
- Deny-list is fragile; `.env.staging` or `credentials.json` would slip through
- No entropy-based detection (high-randomness strings that look like keys)
- Doesn't respect `.gitignore`
- A better design: allow-list by extension (only index known source types) rather than deny-list

## C3. Production secret management

| Approach | Security | Rotation | Audit | Use when |
|---|---|---|---|---|
| `.env` file | ⚠️ Plaintext on disk | Manual | ❌ | Local dev only |
| Platform env vars (current) | ✅ Encrypted at rest | Manual redeploy | ⚠️ Limited | Small teams |
| **AWS Secrets Manager / Vault** | ✅✅ | ✅ Automatic | ✅ Full | Production teams |
| KMS-encrypted config | ✅✅ | Manual | ✅ | Regulated |

**What a proper setup adds:** automatic rotation on a schedule, audit logging of every access, fine-grained IAM per secret, and versioning with rollback.

---

# PART D — Security Theory

> 📚 **THEORY.** Mostly not applicable to this project (no users, no browser, no SQL). Know them; be honest about which you implemented.

## D1. OAuth 2.0

**OAuth is authoriSation, not authentiCation.** It answers "may this app access that resource?" not "who is this user?" (**OIDC** layers identity on top.)

### The four roles
| Role | Example |
|---|---|
| Resource Owner | The user |
| Client | The app requesting access |
| Authorization Server | Google/GitHub login |
| Resource Server | The API holding the data |

### Authorization Code flow with PKCE

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client App
    participant A as Auth Server
    participant R as Resource Server

    C->>C: generate code_verifier (random)
    C->>C: code_challenge = SHA256(verifier)
    C->>A: /authorize?...&code_challenge=X&method=S256
    A->>U: Login + consent screen
    U->>A: Approve
    A->>C: redirect with ?code=AUTH_CODE
    C->>A: POST /token {code, code_verifier}
    A->>A: SHA256(verifier) == stored challenge?
    A->>C: access_token + refresh_token
    C->>R: GET /data (Bearer access_token)
    R->>C: data
```

**Why PKCE (Proof Key for Code Exchange)?** On mobile and SPAs there's no way to keep a client secret secret. Without PKCE, an attacker intercepting the redirect (via a malicious app registering the same URL scheme) could exchange the code for a token. PKCE binds the code to a random verifier only the legitimate client knows. **PKCE is now recommended for all clients, including server-side ones.**

### Grant types
| Grant | Use | Status |
|---|---|---|
| Authorization Code + PKCE | Web/mobile with users | ✅ Recommended |
| Client Credentials | **Machine-to-machine** | ✅ Recommended |
| Implicit | Legacy SPA | ❌ Deprecated |
| Resource Owner Password | Legacy | ❌ Deprecated |

**Relevance here:** the GitHub App JWT flow is conceptually **Client Credentials** — a machine authenticating as itself with no user involved. If a user-facing dashboard returned, it would use Authorization Code + PKCE with "Login with GitHub."

## D2. Password hashing

> Not used — there are no passwords.

```python
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
bcrypt.checkpw(attempt.encode(), hashed)   # constant-time internally
```

**Why not SHA-256?** Because it's *fast* — and speed is the enemy. A GPU computes billions of SHA-256 hashes per second, so a leaked database is brute-forced quickly. bcrypt is **deliberately slow** and cost-tunable: `rounds=12` means 2^12 iterations, ~250ms per hash. That's imperceptible for one login and catastrophic for an attacker trying billions.

**Salt** — a unique random value per password, stored alongside the hash. It defeats rainbow tables and ensures two users with the same password get different hashes. bcrypt embeds the salt in its output string automatically.

**Modern recommendation: Argon2id** — winner of the Password Hashing Competition. It's *memory-hard* as well as slow, which resists GPU and ASIC attacks far better than bcrypt (GPUs have many cores but limited memory per core).

| Algorithm | Verdict |
|---|---|
| MD5, SHA-1 | ❌ Broken |
| SHA-256 (plain) | ❌ Too fast |
| PBKDF2 | ⚠️ Acceptable, FIPS-approved |
| bcrypt | ✅ Good |
| **Argon2id** | ✅✅ Best |

## D3. SQL Injection

> Not applicable — no SQL database. But the **security agent detects it**, which makes this a great topic for you.

```python
# ❌ VULNERABLE
query = f"SELECT * FROM users WHERE id = {user_id}"
# user_id = "1 OR 1=1"        → returns all users
# user_id = "1; DROP TABLE users--"  → destroys the table

# ✅ SAFE — parameterised
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

**Why parameterisation works:** the driver sends the SQL *structure* and the *values* separately. The database parses the query plan first, then binds values as pure data. Values can never be reinterpreted as SQL syntax — it's structurally impossible, not filtered.

**Defence in depth:** parameterised queries (primary) → ORM → input validation → least-privilege DB user → WAF.

**Never rely on escaping/blacklists.** Encoding tricks, unicode normalisation, and second-order injection (data stored safely, then used unsafely later) defeat them.

**The connection to this project:** `security/prompts.py` lists `sql_injection` as a detection category, and `education.py` maps it to OWASP A03:2021 with an impact statement. So you can say: *"I don't have SQL in my own code, but detecting SQL injection in other people's code is a first-class feature of my security agent, complete with OWASP references in the posted comment."*

## D4. XSS (Cross-Site Scripting)

| Type | Mechanism |
|---|---|
| **Stored** | Payload saved server-side, served to other users |
| **Reflected** | Payload in URL, echoed into the response |
| **DOM-based** | Client-side JS writes untrusted data into the DOM |

```javascript
// ❌ innerHTML executes injected script tags/handlers
el.innerHTML = userComment;
// ✅ textContent treats it as text
el.textContent = userComment;
// ✅ React escapes by default
<div>{userComment}</div>
// ❌ unless you opt out
<div dangerouslySetInnerHTML={{__html: userComment}} />
```

**Defences:** context-aware output encoding (HTML body vs attribute vs JS vs URL each need different escaping), Content-Security-Policy headers, `HttpOnly` cookies so stolen JS can't read session tokens, and sanitisation libraries (DOMPurify) when you must allow HTML.

**A relevant angle for this project:** the agent posts **LLM-generated markdown** to GitHub. GitHub sanitises rendered markdown, so this isn't exploitable — but the general risk of rendering LLM output as HTML in a system that *didn't* sanitise is real, and worth naming as something you'd think about if the dashboard returned.

## D5. CSRF (Cross-Site Request Forgery)

An attacker's site causes a victim's browser to make an authenticated request to your site, exploiting automatically-attached cookies.

```html
<!-- On evil.com; victim is logged into bank.com -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker"><input name="amount" value="10000">
</form><script>document.forms[0].submit()</script>
```

**Defences:**
| Defence | How |
|---|---|
| CSRF token | Random per-session token in form; attacker can't read it (same-origin policy) |
| `SameSite=Strict/Lax` cookies | Browser won't send cookies on cross-site requests — now default `Lax` in modern browsers |
| Custom header requirement | Cross-origin custom headers trigger preflight |
| Double-submit cookie | Token in both cookie and body; compare |

**Why this project is immune:** CSRF requires *automatic credential attachment* — cookies. This API uses `Authorization` headers and HMAC signatures, neither of which a browser attaches automatically. **Token-in-header APIs are structurally immune to CSRF.** That's the correct, precise answer.

## D6. CORS

**CORS relaxes the Same-Origin Policy; it does not protect your server.**

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # never ["*"] with credentials
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH"],
    allow_headers=["Authorization", "Content-Type"],
)
```

**The most misunderstood point:** CORS is enforced by the **browser**, not the server. `curl` ignores it entirely. It protects *users* from malicious sites reading responses on their behalf — it is **not** an access control mechanism. Never treat CORS as authorisation.

**Preflight:** for "non-simple" requests (PUT, PATCH, DELETE, or custom headers), the browser first sends `OPTIONS` to ask permission. Commit `952f068` fixed exactly this: `PATCH` wasn't in `allow_methods`, so the approve button's preflight failed.

**Why this project needs no CORS:** the only client is GitHub's server. Servers don't enforce CORS.

## D7. Rate Limiting

> ❌ **Not implemented. This is a real gap.**

| Algorithm | Behaviour |
|---|---|
| Fixed window | Simple; allows 2× burst at boundaries |
| Sliding window log | Accurate; memory-heavy |
| **Token bucket** | Allows bursts, enforces average — usually best |
| Leaky bucket | Smooths output to a constant rate |

```python
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address)

@app.post("/webhook")
@limiter.limit("100/minute")
async def webhook(request: Request): ...
```

**Two distinct needs here:**
1. **Ingress** — cap requests to `/webhook` per IP, so signature-verification floods can't exhaust CPU.
2. **Egress** — cap concurrent OpenAI calls. This is the more urgent one: a 50-hunk PR fires 150 concurrent GPT-4o calls with nothing bounding them. An `asyncio.Semaphore(10)` around the API call is a five-line fix.

---

## D8. Security Posture Summary

| Control | Status | Notes |
|---|---|---|
| Webhook signature verification | ✅ | HMAC-SHA256, constant-time |
| Fail-fast on missing secret | ✅ | Refuses to start |
| Secrets not in git | ✅ | Verified with `git ls-files` |
| Secret files excluded from ingestion | ✅ | `.env`, `.pem`, `.key` |
| Short-lived tokens | ✅ | JWT 9 min, install token 1 hr |
| Least-privilege GitHub permissions | ✅ | Read code, write PR comments |
| Asymmetric signing (RS256) | ✅ | Correct for third-party verification |
| Read-only trust model | ✅ | Never modifies user code |
| Repo cleanup after use | ✅ | `finally: rmtree` |
| Input validation on LLM output | ✅ | Pydantic + JSON Schema |
| **Rate limiting** | ❌ | Real gap |
| **Replay protection** | ❌ | `X-GitHub-Delivery` unused |
| **Token redaction in logs** | ❌ | Clone URL contains token |
| **Confidence bounds** | ❌ | `Field(ge=0, le=1)` missing |
| **Structured audit logging** | ❌ | `print()` only |
| **Dependency scanning** | ❌ | No Dependabot/`pip-audit` |
| Request size limits | ❌ | No cap before verification |

---

## D9. Interview Questions

### Q1. "Explain authentication in your project."

**Best answer:**
> There's no user authentication — no login, no passwords, no sessions. It's a machine-to-machine system with GitHub as the only client. But there is authentication in both directions, using two different mechanisms for two different reasons.
>
> **Inbound**, GitHub proves it's GitHub with an HMAC-SHA256 signature. I read the raw request bytes — deliberately not the parsed JSON, because re-serialising would change whitespace and ordering and break the signature — compute HMAC with a shared secret, and compare with `hmac.compare_digest`. That last part matters: Python's `==` short-circuits at the first differing byte, so response timing would leak how many bytes were correct. An attacker could forge a signature byte by byte, reducing the search from astronomically infeasible to about sixteen thousand attempts. Constant-time comparison always checks every byte.
>
> **Outbound**, I prove I'm the App with a JWT. I sign a token with my RSA private key using RS256, with `iss` set to my App ID, backdated sixty seconds for clock skew and expiring in nine minutes against GitHub's ten-minute maximum. I exchange that for an installation access token — good for an hour, scoped to that specific installation — and use it for actual API calls.
>
> The two-step matters: the JWT proves *identity*, the installation token grants *authorisation*. An App installed on five hundred organisations gets a separately-scoped token for each, so a leak has a blast radius of one installation for one hour.
>
> RS256 rather than HS256 because GitHub needs to verify my tokens without being able to forge them. With a symmetric secret we'd both hold the same key and GitHub could mint tokens as me.

---

### Q2. "Why constant-time comparison?"

*(Covered in A3 above — this is your single strongest security answer. Know the byte-by-byte attack narrative cold.)*

**Follow-ups:**
- *"Where else do timing attacks apply?"* — password comparison (bcrypt's `checkpw` is constant-time internally), API key checks, and any equality test on a secret.
- *"Is this a practical attack over a network?"* — Harder, because network jitter adds noise. But it's demonstrably feasible with enough samples and statistical averaging, and it costs nothing to defend against, so there's no reason to accept the risk.

---

### Q3. "What are your security weaknesses?"

**Best answer — name four with fixes:**
> **No rate limiting.** `/webhook` is public. An attacker can flood it with invalid signatures and each one costs me a SHA-256 computation. Worse, on the egress side I have no cap on concurrent OpenAI calls — a fifty-hunk PR fires a hundred and fifty at once. Fixes: `slowapi` on ingress, `asyncio.Semaphore` on egress.
>
> **No replay protection.** HMAC proves authenticity and integrity but not freshness. A captured valid request replays forever. GitHub sends `X-GitHub-Delivery` as a unique UUID per delivery — caching those and rejecting duplicates closes it.
>
> **Token leakage risk in logs.** I embed the installation token in the clone URL. If GitPython raises and I print the exception, that token could land in my logs. It's short-lived, one hour, but it's a real exposure. Fix is passing credentials via environment variables instead of the URL, or at minimum redacting before logging.
>
> **No dependency scanning.** I have no Dependabot, no `pip-audit`, and several unpinned dependencies in `requirements.txt`. A supply-chain vulnerability would go unnoticed. That's the cheapest one to fix — it's a config file.
>
> The one I'd prioritise is egress rate limiting, because unlike the others it's not just a security issue — it's a cost and reliability issue that will bite in normal operation, not just under attack.

**Why this scores well:** four specific gaps with specific fixes, and a prioritisation with reasoning.

---

### Q4. "Is your system vulnerable to CSRF?"

**Best answer:**
> No, and structurally rather than by accident. CSRF depends on the browser *automatically* attaching credentials — cookies — to a cross-origin request. My API doesn't use cookies at all. Inbound authentication is an HMAC signature over the request body, and outbound uses `Authorization` headers. Browsers don't automatically attach either.
>
> There's also no browser in the picture: my only client is GitHub's server making a POST. CSRF is an attack on a *user's* browser session, and there is no user session.
>
> If I brought back the dashboard, CSRF would become relevant. Then I'd use `SameSite=Lax` cookies at minimum, or — better — keep the API token-in-header, which stays immune by design. That's the general principle: token-in-header APIs are structurally immune to CSRF, cookie-authenticated ones need explicit defences.

**What's evaluated:** Do you understand *why* an attack works, or just its name? Saying "we're immune because we don't use cookies" shows mechanism-level understanding.

---

## D10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `==` on signatures/tokens | Timing attack | `hmac.compare_digest` |
| Verifying parsed JSON, not raw bytes | Signature never matches | Read `request.body()` first |
| Secrets in git | Permanent — history survives deletion | `.gitignore` + rotate immediately |
| Long-lived tokens | Large breach window | Short expiry + refresh |
| Trusting JWT `alg` header | `alg:none` forgery | `algorithms=["RS256"]` explicitly |
| Secrets in JWT payload | It's base64, not encrypted | Only non-sensitive claims |
| CORS as access control | It's a browser policy | Real authorisation server-side |
| String-concatenated SQL | Injection | Parameterised queries |
| `innerHTML` with user data | XSS | `textContent` / framework escaping |
| Fast hash for passwords | GPU brute force | bcrypt / Argon2id |
| Credentials in URLs | Leak into logs and `ps` | Env vars or credential helpers |
| Ingesting `.env` files | Exfiltrating user secrets | Explicit skip list ✅ done |

---

## D11. Cross-References

| Topic | File |
|---|---|
| Webhook handler flow | [05-Backend-Architecture.md](05-Backend-Architecture.md) |
| GitHub API details | [08-API-Documentation.md](08-API-Documentation.md) |
| The security agent's detection categories | [09-AI-System.md](09-AI-System.md) |
| Secret env vars in deployment | [10-Deployment-and-DevOps.md](10-Deployment-and-DevOps.md) |
| Rate limiting at scale | [13-Scalability.md](13-Scalability.md) |
| Security fixes on the roadmap | [20-Future-Improvements.md](20-Future-Improvements.md) |

**Next:** [08-API-Documentation.md](08-API-Documentation.md)
