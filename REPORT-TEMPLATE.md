# Report Template — IDOR / API Authorization Finding

Use this template for each confirmed finding. Replace every `<PLACEHOLDER>`
with your own values. Use generic placeholders (`api.example.com`,
`<USER_A_ID>`) — never include real credentials, tokens, or another user's
real data in a report.

---

## Title

`<SHORT TITLE, e.g. BOLA: User B can read User A's invoices via /api/v1/invoices/{id}>`

## Severity

`<Critical / High / Medium / Low>` — justify in one line below.

**Justification:** `<e.g. Any authenticated user can read any other user's invoices, including billing details.>`

## Summary

`<2–4 sentences: what the vulnerability is, where it is, and what an attacker can do with it. Mention the affected endpoint and the authorization check that is missing.>`

## Affected Endpoint

```text
<METHOD> https://api.example.com/<PATH>
```

**Parameter / field involved:** `<e.g. id in the URL path / invoice_id in the JSON body>`

## Steps to Reproduce

All steps use two test accounts you control. Replace the placeholders with
your test account identifiers.

1. Log in as **User A** (`<USER_A_EMAIL>`) and create/access the resource:
   `<e.g. create an invoice; note its ID <USER_A_INVOICE_ID>.>`
2. Confirm User A owns the resource:
   ```http
   GET https://api.example.com/api/v1/invoices/<USER_A_INVOICE_ID>
   Authorization: Bearer <USER_A_TOKEN>
   ```
   Response: `200 OK` — returns User A's invoice.
3. Log in as **User B** (`<USER_B_EMAIL>`, a different account with no
   relationship to User A).
4. Replay the same request with User B's token:
   ```http
   GET https://api.example.com/api/v1/invoices/<USER_A_INVOICE_ID>
   Authorization: Bearer <USER_B_TOKEN>
   ```
5. **Observed result:** `<e.g. 200 OK — User B receives User A's full invoice, including billing address and payment details.>`

**Expected result:** `403 Forbidden` (or `404 Not Found` if the application
hides resource existence).

## Evidence

`<Attach or paste:>`

- Request/response pair for User A (owner) — shows ownership.
- Request/response pair for User B (attacker) — shows unauthorized access.
- Redact tokens and any sensitive values before sharing. Keep the full
  unredacted evidence in your own secure notes only if the program asks
  for it.

## Impact

`<What an attacker can do, in concrete terms:>`

- `<e.g. Read any user's invoices by enumerating sequential IDs.>`
- `<e.g. Modify or delete another user's data via PUT/DELETE on the same endpoint.>`
- `<e.g. Escalate privilege if the exposed record contains an admin token or reset link.>`

State the impact for the **affected data** only — do not speculate about
data you did not verify.

## Remediation

`<Concrete fix, e.g.:>`

- Enforce object-level authorization on **every** endpoint that takes an
  object identifier: verify the authenticated user owns the object (or has
  an explicit role granting access) before returning or modifying it.
- Apply the check on all HTTP methods and all ID locations (path, query,
  body, headers), not just the `GET` path.
- For function-level issues: enforce role checks server-side on every
  admin/internal endpoint; never rely on the UI hiding the route.
- For mass assignment: use an allowlist of writable fields per endpoint;
  never bind request bodies directly to the data model.
- Return `403` (or `404` to avoid leaking existence) for unauthorized
  access, and log the attempt.
- Add tests: an automated test where User B requests User A's resource
  must fail with `403`/`404`.

## References

- OWASP API Security Top 10 — API1 Broken Object Level Authorization
- OWASP API Security Top 10 — API5 Broken Function Level Authorization
- OWASP API Security Top 10 — API6 Unrestricted Access to Sensitive Business Flows
- OWASP Testing Guide — Testing for Insecure Direct Object References

---

**Tester:** `<YOUR NAME / HANDLE>`
**Date:** `<YYYY-MM-DD>`
**Target:** `<PROGRAM OR TARGET NAME, as defined by the scope>`
