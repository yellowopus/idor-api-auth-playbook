# API Authorization Testing Checklist

Copy-paste checklist for testing IDOR / BOLA, BFLA, mass assignment,
excessive data exposure, rate limiting, and JWT issues on an authorized
target. Work through it with two test accounts you control (User A and
User B).

## Setup

- [ ] Target is in scope and you have written authorization
- [ ] Created two test accounts: User A (owner) and User B (attacker)
- [ ] Captured User A's full flows: create, read, update, delete
- [ ] Listed every object ID User A creates or accesses

## Object-Level Authorization (BOLA / IDOR)

- [ ] Replayed User A's requests with User B's token — can User B read
      User A's data?
- [ ] Tested sequential IDs (±1, ±10) from your own IDs
- [ ] Tested UUIDs harvested from your own account's responses
- [ ] Tested IDs in every location: URL path, query string, JSON body,
      headers, cookies, GraphQL arguments
- [ ] Tested sub-resources directly (attachments, comments, invoices)
      without going through the parent endpoint
- [ ] Tested nested resources (`/api/orgs/1/users/2` — changed both IDs)
- [ ] Tested batch/bulk endpoints with other users' IDs included
- [ ] Tested export/download endpoints with another user's ID
- [ ] Tested linked-object swaps (e.g. `order.address_id` set to another
      user's address)

## Write Access (Write IDOR)

- [ ] PUT / PATCH / DELETE with another user's object ID
- [ ] Tested all HTTP verbs on each endpoint (GET, POST, PUT, PATCH,
      DELETE) — a blocked GET does not mean a blocked PUT
- [ ] Tested parameter pollution (`id[]=...`, duplicate `id` params)
- [ ] Tested type confusion (string vs integer ID, array in JSON)
- [ ] Tested state/status field changes (skip order states, cancel another
      user's order)

## Function-Level Authorization (BFLA)

- [ ] Called admin endpoints directly as a normal user
      (`/api/v1/admin/users`, `/api/v1/admin/payments/all`, ...)
- [ ] Tested bulk operations (`/api/v1/users/bulk-delete`,
      `/api/v1/export/all-data`)
- [ ] Searched JS bundles and API docs for hidden admin/internal routes
- [ ] Enumerated common prefixes: `/admin/`, `/manage/`, `/internal/`

## Mass Assignment

- [ ] Added candidate fields to every create/update body:
      `role`, `isAdmin`, `admin`, `verified`, `plan`, `tier`,
      `permissions`, `org`, `owner`
- [ ] Diffed admin "create user" fields against normal "register" fields
- [ ] Checked whether an accepted field changed privilege, balance, or
      verification status

## Excessive Data Exposure

- [ ] Read full JSON responses — compared returned fields against what the
      UI displays
- [ ] Checked list endpoints for fields the single-record endpoint hides
- [ ] Checked whether other users' records (via BOLA) expose extra fields

## Rate Limiting

- [ ] Sent a burst of requests to login / password-reset / OTP endpoints —
      did a `429` appear?
- [ ] Tested whether limits are per-account, per-IP, or per-endpoint
- [ ] Tested common bypass headers (`X-Forwarded-For`, `X-Real-IP`,
      `Forwarded`)
- [ ] Tested path variants (case, trailing slash) against the same endpoint
- [ ] Tested batch abuse (many operations bundled into one request)

## JWT / Token Issues

- [ ] Decoded the JWT header: noted `alg`, `kid`, `jku`, `x5u`, `typ`
- [ ] Noted role / org / tenant / scope claims in the payload
- [ ] Checked `iss` and `aud` match the service
- [ ] Tested `alg: none` acceptance (unsigned token)
- [ ] Tested RS256 → HS256 confusion (public key used as HMAC secret)
- [ ] Tested `kid` handling (path traversal, injection, unvalidated values)
- [ ] Tested `jku` / `x5u` trust (attacker-controlled key URL)
- [ ] Checked whether tokens are reusable across products (mobile vs web)

## Reporting

- [ ] Each confirmed finding has: owner account, attacker account, request,
      response, and impact
- [ ] Finding written up with [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md)
- [ ] No other user's data was accessed, modified, or stored
- [ ] Testing stayed in scope; findings reported through the proper channel
