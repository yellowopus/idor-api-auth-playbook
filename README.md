# IDOR & API Authorization Playbook

A focused, practical playbook for testing **IDOR / BOLA** (Broken Object Level
Authorization), **BFLA** (Broken Function Level Authorization), and related API
authorization weaknesses — object-level access, function-level access, mass
assignment, excessive data exposure, rate limiting, and JWT header tricks.

## Scope & Authorization

> **Only test targets you own or have explicit written authorization for.**
>
> - Test **in-scope targets only**, exactly as defined by the program or
>   engagement rules you are working under.
> - Use **your own accounts only**. Never read, modify, or delete another
>   user's data. When a finding needs proof, prove it with **two test
>   accounts you control** — one as the "owner" and one as the "attacker".
> - All examples in this repository use generic placeholders such as
>   `api.example.com` and do not target any real third party. Do not copy
>   these examples against a real site or company without authorization.
> - If you find a vulnerability, stop, document it, and report it through
>   the proper channel. Do not escalate, exfiltrate, or probe further than
>   needed to confirm impact.
> - Follow the program's rules of engagement and the law in your
>   jurisdiction. This playbook is for authorized security testing and
>   defensive learning only.

## Who It's For

- Bug bounty hunters testing APIs and web apps.
- Security engineers and developers learning how authorization fails.
- Anyone reviewing an API for object- and function-level access control.

## Contents

| File | What It Covers |
| --- | --- |
| [IDOR.md](IDOR.md) | IDOR/BOLA methodology: identifier enumeration, object- vs function-level checks, proving impact with two test accounts |
| [API-AUTH.md](API-AUTH.md) | API authorization issues: mass assignment, excessive data exposure, rate limiting, JWT header tricks (`kid`, `alg`, `typ`) |
| [CHECKLIST.md](CHECKLIST.md) | Copy-paste checklist for API authorization testing |
| [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md) | Report template for an IDOR / API-auth finding |
| [LICENSE](LICENSE) | MIT License |

## How to Use This Playbook

1. Read [IDOR.md](IDOR.md) for the core testing loop.
2. Read [API-AUTH.md](API-AUTH.md) for the wider authorization surface.
3. Work through [CHECKLIST.md](CHECKLIST.md) during a test.
4. When you confirm a finding, write it up with
   [REPORT-TEMPLATE.md](REPORT-TEMPLATE.md).

## Key Concepts at a Glance

| Term | Meaning |
| --- | --- |
| **IDOR** | Insecure Direct Object Reference — accessing an object you don't own by changing its identifier |
| **BOLA** | Broken Object Level Authorization — the OWASP API Top 10 name for the same class (API context) |
| **BFLA** | Broken Function Level Authorization — a low-privilege user reaching a high-privilege function (e.g. admin endpoints) |
| **Mass assignment** | The API lets a client set model fields that were never meant to be user-controlled (e.g. `role`, `isAdmin`) |
| **Excessive data exposure** | An API returns more fields than the client needs, leaking sensitive data |

## License

MIT — see [LICENSE](LICENSE).
