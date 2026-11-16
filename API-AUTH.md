# API Authorization Issues

Beyond direct object references, APIs fail authorization in several other
ways. This file covers mass assignment, excessive data exposure, rate
limiting, and JWT header tricks — at concept level. All examples use generic
placeholders (`api.example.com`, `<TOKEN>`) and assume authorized testing
with your own accounts only.

## 1. Mass Assignment

When a `POST` or `PUT` accepts a JSON body, fields in the underlying data
model may be settable even if they are not in the official API docs:

```json
POST /api/v1/register
{
  "username": "testuser",
  "email": "test@example.com",
  "password": "<TEST_PASSWORD>",
  "role": "admin",
  "isAdmin": true,
  "verified": true,
  "creditBalance": 9999
}
```

Fields commonly worth testing:

```text
role, isAdmin, admin, verified, plan, tier, permissions, org, owner
```

### How to find hidden fields

1. Compare the fields in an admin "create user" request with a normal
   "register" request and diff them.
2. Read the API documentation for every field the model accepts.
3. Check client-side code (JavaScript bundles) for field names the UI sends.
4. Add candidate property names to a request and watch for `200` vs `400` —
   a field the server silently accepts is worth investigating.

If a hidden field changes your privilege, balance, or verification status,
you have a mass assignment finding — confirm it with the two-account loop
in [IDOR.md](IDOR.md) and report it with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md).

## 2. Excessive Data Exposure

An API often returns far more fields than the client needs. Compare the
fields in a response against the fields the UI actually displays:

```json
GET /api/v1/users/1234
{
  "id": 1234,
  "username": "testuser",
  "email": "test@example.com",
  "passwordHash": "<HASH>",
  "internalNotes": "VIP customer, escalate all tickets",
  "ssnLastFour": "1234"
}
```

Test for it by:

- Reading full JSON responses, not just what the UI renders.
- Checking whether other users' records expose the same extra fields
  (combine with BOLA: if you can read another user's record at all, the
  extra fields raise the impact).
- Checking list endpoints — a `GET /api/v1/users` list may leak fields that
  the single-record endpoint hides.

Remediation note: the API should return only the fields the client needs
for that call, and sensitive fields should never be serialized to the client
at all.

## 3. Rate Limiting

Rate limits protect login, password reset, OTP verification, and other
sensitive endpoints. Test whether they exist and whether they can be
bypassed — only against targets you are authorized to test, and without
disrupting the service.

### What to observe

- Does the endpoint return `429 Too Many Requests` after a burst of
  requests?
- Are the limits per-account, per-IP, or per-endpoint?
- Do limits apply equally to all HTTP methods and path variants?

### Common bypass families (concept level)

```text
X-Forwarded-For: 1.2.3.4
X-Real-IP: 5.6.7.8
Forwarded: for=9.9.9.9
User-Agent rotation
Path case / trailing-slash variants (e.g. /login vs /Login vs /login/)
```

### Batch abuse

A single request that carries many operations can bypass per-request limits:

- Arrays of login mutations in one GraphQL request.
- Bulk object fetches with many IDs in one call.
- Repeated password-reset or verification calls bundled into one request.

If a limit can be bypassed, the impact is whatever the limited endpoint
protects (e.g. credential stuffing, OTP brute force, enumeration). Report
the bypass and the endpoint it exposes.

## 4. JWT Header Tricks

JSON Web Tokens have a header, a payload, and a signature. The header
controls how the token is verified — and that is where many implementations
go wrong. Test at concept level; never forge tokens for an account you don't
own.

### 4.1 Token triage — what to inspect

Decode the header and payload (they are base64url, not encrypted) and note:

- `alg` — the signing algorithm
- `kid` — key identifier
- `jku` / `x5u` — URLs the server may fetch keys from
- Role, org, tenant, scope, or privilege claims in the payload
- `iss` (issuer) and `aud` (audience) — do they match the service?
- Whether mobile and web tokens are interchangeable across products

### 4.2 `alg: none`

Some servers accept a token with no signature at all:

```text
Header:  {"alg": "none", "typ": "JWT"}
Payload: {"sub": "1234", "role": "admin"}
Signature: (empty)
```

Test: send an unsigned token (header `alg: none`, empty signature) with
elevated claims and see if the server accepts it. A server that does is
trusting the client to choose the verification method.

### 4.3 Algorithm confusion (RS256 → HS256)

If the server verifies RS256 (asymmetric, public key) tokens but also
accepts HS256 (symmetric, shared secret), an attacker can sign an HS256
token using the server's **public key** as the HMAC secret — because the
public key is public. Test whether the server accepts an HS256 token signed
with its own public key.

### 4.4 `kid` trust

The `kid` header tells the server which key to verify with. Weak
implementations use it unsafely:

- **Path traversal**: `kid` values like `../../dev/null` or
  `../../etc/passwd` that the server opens as a file path.
- **Injection**: `kid` values passed into a database query or shell command.
- **Remote key fetch**: `jku` / `x5u` headers pointing at a URL the server
  fetches — if the server trusts an attacker-controlled URL, the attacker
  supplies their own key and signs any token they want.

Test whether the server validates `kid` against a fixed allowlist of known
key IDs, and whether `jku`/`x5u` URLs are restricted to trusted hosts.

### 4.5 Weak secrets

If a token uses HS256 with a weak or guessable secret, it can be cracked
offline with a targeted wordlist. Indicators: short secrets, secrets equal
to the app name, or secrets committed in public code. If you can crack the
secret of a test token on a target you own, rotate the secret and use a long
random one.

### 4.6 `typ` confusion

Some libraries treat the `typ` header as meaningful. A token with
`typ: JWT` vs a different `typ` value may be routed to different
verification logic. Note the `typ` value during triage and test whether
changing it alters acceptance — it is a low-probability but cheap check.

## 5. Putting It Together

A typical API authorization test follows this order:

1. **Map** the API surface and every identifier, field, and token in play.
2. **Run the A–B loop** from [IDOR.md](IDOR.md) for object and function
   access.
3. **Test mass assignment** on every create/update endpoint.
4. **Read full responses** for excessive data exposure.
5. **Probe rate limits** on sensitive endpoints.
6. **Triage tokens** — decode the JWT header and check `alg`, `kid`,
   `jku`/`x5u`, and claims.
7. **Report** confirmed findings with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md).
