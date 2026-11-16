# IDOR / BOLA Methodology

How to systematically test for Insecure Direct Object References and Broken
Object Level Authorization. All examples use `api.example.com` as a generic
placeholder and assume you are testing a target you own or are authorized to
test, using only your own test accounts.

## 1. IDOR vs BOLA vs BFLA

| Term | Meaning | Typical Impact |
| --- | --- | --- |
| **IDOR** | Insecure Direct Object Reference | Read or modify another user's data |
| **BOLA** | Broken Object Level Authorization (OWASP API Top 10) | Same as IDOR, in an API context |
| **BFLA** | Broken Function Level Authorization | A low-privilege user reaches high-privilege functions (e.g. admin endpoints) |

The key distinction:

- **BOLA** = accessing an **object** you shouldn't own (data belonging to
  another user).
- **BFLA** = accessing a **function** you shouldn't be authorized for
  (admin CRUD, bulk actions, user management).

## 2. Where Object IDs Hide

Don't stop at URL path parameters — identifiers appear in many places:

```text
URL path:       GET /api/v1/users/1234/profile
URL query:      GET /orders?order_id=982
Request body:   {"userId": 1234, "action": "view"}
JSON fields:    {"resource": {"id": 5678, "type": "invoice"}}
Headers:        X-User-ID: 1234
                X-Account-ID: 9999
Cookies:        user_id=1234; account=org_5678
GraphQL args:   query { user(id: "1234") { ... } }
Form fields:    <input name="documentId" value="5678">
WebSocket msgs: {"event": "subscribe", "channel_id": 9999}
```

Map every identifier the app sends or receives before you start swapping
values. An ID you never notice is a test you never run.

## 3. The A–B Test Loop

The most systematic way to test for BOLA:

1. **Create two test accounts you control**: User A and User B.
2. **Act as User A.** Perform every action (view profile, open an order,
   edit a document, download a file) and capture the requests.
3. **Note every object ID** User A creates or accesses.
4. **Authenticate as User B.**
5. **Replay User A's requests using User B's session/token**, changing
   nothing except the session.
6. **If User B can read or modify User A's data → BOLA confirmed.**

Report evidence must show both sides: that User A owns the resource, and
that User B accessed it. A successful request alone is not proof — capture
the response showing the other account's data.

## 4. Identifier Types and What They Imply

| ID Pattern | Example | Notes |
| --- | --- | --- |
| Sequential integer | `id=1001` → `id=1002` | Easy to predict; try ±1, ±10 from your own ID |
| UUID v4 | `550e8400-...` | Random; harvest UUIDs from your own account's responses first |
| UUID v1 | Time-based UUID | Timestamp is embedded — adjacent values may be predictable |
| Hashed IDs | `md5(user_id)` | Try hashing sequential integers to see if the scheme matches |
| Encoded IDs | base64 of `{"id": 1001}` | Decode, modify, re-encode |
| Compound IDs | `/api/users/1/orders/5` | Both IDs may be checked independently — test each one |

## 5. Horizontal vs Vertical Escalation

**Horizontal** — same privilege level, another user's data:

```http
GET /api/account/1234/statement    ← you are user 5678
```

**Vertical** — a low-privilege user reaches admin-only functions:

```http
POST /api/admin/users/delete       ← normal user calling an admin endpoint
GET  /api/admin/all-users
PUT  /api/users/1234/role {"role": "admin"}
```

**Combined** — an IDOR that leads to privilege escalation, for example
reading another user's data that contains an admin token or a password-reset
link.

## 6. HTTP Method Escalation

When `GET /resource/1234` is properly restricted, test the other verbs —
authorization is often implemented per method, and one path gets forgotten:

```http
GET    /api/v1/users/<USER_A_ID>   ← might be blocked
POST   /api/v1/users/<USER_A_ID>   ← different code path, might not check
PUT    /api/v1/users/<USER_A_ID>   ← update another user's data
DELETE /api/v1/users/<USER_A_ID>   ← delete another user's account
PATCH  /api/v1/users/<USER_A_ID>   ← partial update, often missed
```

## 7. Parameter Pollution & Type Confusion

When `id=1234` is validated, try variations the validator may handle
differently:

```text
id[]=1234&id[]=5678          ← array — the app may use the first or last value
id=5678&id=1234              ← duplicate — the app may prefer first or last
{"id": "1234"}               ← string vs integer: may hit a different code path
{"id": [1234]}               ← array in JSON
{"userId": 1234, "id": 5678} ← two ID fields — which one is used for authz?
```

## 8. BFLA: Function-Level Testing

### Common admin-only functions to probe

```http
GET    /api/v1/admin/users
DELETE /api/v1/users/{any_user_id}
PUT    /api/v1/users/{user_id}/role
POST   /api/v1/users/bulk-delete
GET    /api/v1/export/all-data
POST   /api/v1/admin/subscription/modify
GET    /api/v1/reports/all-users-activity
```

### How to find hidden admin endpoints

1. Read JavaScript bundles — admin routes are often exposed in frontend code.
2. Check API documentation (Swagger/OpenAPI) for `admin`, `internal`, or
   `privileged` tags.
3. Enumerate common prefixes: `/api/v1/admin/`, `/api/v1/manage/`,
   `/api/v1/internal/`.
4. Compare the regular-user docs against any admin section in the docs.

## 9. Indirect IDOR (Reference Chains)

The app checks permission on **object A** but not on a **referenced object B**:

```text
User A can read their own messages.
GET /api/messages/1234 → checks: "does the user own message 1234?" ✓

But messages have attachments:
GET /api/attachments/5678 → does NOT check whether the attachment
                             belongs to a message the user owns.
```

Test sub-resources directly via their own IDs instead of going through the
parent endpoint. GraphQL has the same pattern: a query on your own profile
can traverse relationships into other users' private fields.

## 10. State Machine Abuse

When a resource has a status (e.g. an order: `pending → confirmed → shipped
→ delivered`), test whether you can skip states or change another user's
resource state:

```http
PUT /api/orders/1234 {"status": "delivered"}   ← from "pending"
PUT /api/orders/1234 {"status": "refunded"}    ← from "pending"
PUT /api/orders/<USER_A_ORDER_ID> {"status": "cancelled"}  ← as User B
```

## 11. Systematic Test Categories

| # | Category | Test Method |
| --- | --- | --- |
| 1 | Direct ID reference | Change a numeric/UUID ID in the URL |
| 2 | Predictable UUID | If UUIDs are v1 (time-based), adjacent IDs may be calculable |
| 3 | Batch / bulk operations | `/api/users/bulk?ids=123,456` — add other users' IDs |
| 4 | Export / download | Export endpoints that accept a `user_id` parameter |
| 5 | Linked object IDOR | Change `order.address_id` to another user's address |
| 6 | Resource replacement | Update your own profile with another user's resource ID |
| 7 | Write IDOR | PUT / PATCH / DELETE with another user's ID |
| 8 | Nested object | `/api/orgs/1/users/2` — change the org ID to reach another org's users |
