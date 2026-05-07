# Code Review Checklist and Reviewer Assignment Matrix

**Document Owner:** Security Architecture (in coordination with Engineering Leadership)
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Gate D of the Vibe Coding pipeline for Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications

---

## 1. Purpose

This document is consumed by the Gate D Code Review Orchestrator agent. It encodes three things: who reviews code at Gate D depending on the project's classification, what each reviewer is expected to evaluate, and what constitutes a passing review.

Gate D is the most visible point in the Vibe Coding pipeline where the Type 1 / Type 1B differential takes effect. The same code, the same scan results, and the same automated tests produce a different reviewer set and a different checklist depending on classification. The agent must apply this differential mechanically — Type 1B projects do not get to skip InfoSec because the project happens to be small, and Type 1 projects do not gain InfoSec involvement because the team feels uncertain.

---

## 2. Reviewer Assignment Matrix

### 2.1 Type 1 (Internal · Non-Sensitive)

| Reviewer Role          | Required? | SLA       | Sign-Off Authority                          |
| ---------------------- | --------- | --------- | ------------------------------------------- |
| Peer Engineer (senior) | Required  | 48 hours  | Pass / Request Changes / Block              |
| Architect              | Optional* | —         | May be requested by peer reviewer if architectural concern surfaces |
| InfoSec                | Not required | —      | —                                           |

*Architect involvement at Gate D for Type 1 is optional and triggered by the peer reviewer's escalation when an architectural issue is identified that was not flagged at Gate A. It is not a routine reviewer slot.

**Type 1 pass criteria:** Peer reviewer signs off as Pass with no Block-level comments.

### 2.2 Type 1B (Internal · Restricted)

| Reviewer Role          | Required? | SLA       | Sign-Off Authority                          |
| ---------------------- | --------- | --------- | ------------------------------------------- |
| Peer Engineer (senior) | Required  | 72 hours  | Pass / Request Changes / Block              |
| Architect              | Required  | 72 hours  | Pass / Request Changes / Block              |
| InfoSec                | Required  | 72 hours  | Pass / Request Changes / Block              |

All three reviewers must sign off independently. Sign-offs are not delegated, not implicit, and not overridable by any single reviewer.

**Type 1B pass criteria:** All three reviewers sign off as Pass with no Block-level comments. Request Changes from any reviewer halts Gate D until addressed and re-reviewed. A single Block from any reviewer prevents the project from advancing regardless of other sign-offs.

### 2.3 SLA Definitions

| SLA Term         | Meaning                                                                 |
| ---------------- | ----------------------------------------------------------------------- |
| 48 / 72 hours    | Wall-clock hours from review request to first reviewer response (Pass, Request Changes, or Block) |
| Re-review SLA    | Same as initial review — reviewers commit to the SLA each round, not amortized over the project |
| Reviewer absence | If a required reviewer is unavailable for the SLA window, an alternate from the same role is assigned by the Code Review Orchestrator |

---

## 3. Universal Code Review Checklist

Every Gate D review — Type 1 or 1B, peer or architect or InfoSec — applies this checklist as the baseline. It is not exhaustive, but absence of any item is a Request Changes outcome at minimum.

### 3.1 Functional Correctness

- The code implements the functional requirements stated in PRD Section 5.
- Edge cases identified during development are handled (null inputs, empty collections, boundary values).
- Error handling is present and produces actionable error messages without leaking implementation detail.
- The code does not introduce regressions to functionality outside its declared scope.

### 3.2 Code Quality

- The code follows the language/framework conventions documented in the Secure Coding Patterns guide.
- Functions and classes have a single, clear purpose.
- Naming is consistent and self-documenting; abbreviations are used only where the abbreviation is more readable than the full term.
- No dead code, commented-out blocks, or TODO markers without a tracked ticket reference.
- No obvious anti-patterns (god objects, deeply nested conditionals, primitive obsession on security-relevant types).

### 3.3 Test Coverage

- Unit tests exist for new code paths.
- Test coverage on changed lines meets the threshold defined in the Test Coverage and Quality Gates Standard.
- Tests are deterministic; no time-dependent or order-dependent flakiness.
- Test names describe the behavior under test, not just the function name.
- Negative test cases exist where the function has authorization, validation, or error-handling branches.

### 3.4 Documentation

- Public APIs (HTTP endpoints, library exports) have docstring or comment documentation.
- README or operational notes are updated for new configuration, new environment variables, or new operational dependencies.
- ADRs are present for any non-obvious design decision, per the ADR Template.
- Database schema changes are documented in a migration with a reversal path.

---

## 4. Security Review Checklist (Architect Lens)

Applied at Gate D for Type 1B projects (and for Type 1 only when escalated). The Architect verifies the implementation of security-relevant decisions, not the decisions themselves — the decisions were ratified at Gate A.

### 4.1 Authentication

- Every HTTP endpoint and every backend service interface is behind authentication; no anonymous access exists except for explicitly public endpoints (which should not exist for Vibe Coding internal apps).
- Authentication is delegated to the SSO identity provider; no custom authentication code is introduced.
- Token validation uses the SDK or framework-provided library; no hand-rolled JWT parsing.
- Token expiration is enforced server-side, not relied upon at the client.

### 4.2 Authorization

- Every endpoint that returns or modifies data has an explicit authorization check.
- Authorization checks are server-side; client-side filtering is not the only line of defense.
- Authorization is based on identity-provider claims (group membership), not on values supplied in the request body or URL.
- Resource-level authorization (this user may access this specific record) is implemented where multiple users may share an endpoint but should see different data.

### 4.3 Input Validation and Output Encoding

- All inputs from untrusted sources (HTTP requests, file uploads, message queues) are validated against an explicit schema before use.
- String inputs are validated for length and character class, not just non-empty.
- SQL queries are parameterized; no string concatenation produces SQL.
- HTML output is encoded by the templating engine; no raw insertion of user input into HTML.
- File uploads validate content type by inspecting bytes, not just by trusting the Content-Type header or file extension.

### 4.4 Cryptography Usage

- Cryptographic operations use platform-provided libraries (no hand-rolled crypto).
- TLS 1.2 or higher is enforced for all outbound HTTP calls.
- Symmetric encryption uses authenticated modes (AES-GCM, ChaCha20-Poly1305); ECB and unauthenticated CBC are not used.
- Hashing of secrets uses a slow KDF (Argon2id, scrypt, bcrypt); SHA-family is used only for non-secret integrity, not password hashing.
- Random values used in security contexts come from the cryptographic RNG, not the general-purpose RNG.

### 4.5 Secrets Handling

- No secret values appear in source code, config files committed to source control, log output, or HTTP responses.
- All secrets are retrieved from the secrets vault at runtime via the approved injection pattern.
- Secret rotation is not blocked by code (rotated secrets do not require code redeployment to take effect).

### 4.6 Error Handling and Information Disclosure

- Stack traces are never returned in HTTP responses for production builds.
- Error responses to authenticated users are descriptive enough to act on; error responses to unauthenticated users are generic ("Unauthorized," "Bad Request") and do not reveal whether a resource exists.
- Logged errors include sufficient context for debugging without including secrets or full request/response bodies for sensitive endpoints.

---

## 5. Type 1B Additional InfoSec Review

Applied at Gate D **only** for Type 1B projects. This checklist addresses the controls specifically required by the Internal · Restricted classification.

### 5.1 Authorization Enforcement Verification

- BU-scoped access (or equivalent multi-tenancy boundary) is enforced at the database query level, not in application post-filtering.
- Authorization tests exist that explicitly verify a user from BU-A cannot read or modify BU-B's data, including via direct ID guessing, search, and export paths.
- Privilege escalation paths are absent: no endpoint allows a user to grant themselves additional permissions or to assume another user's identity.

### 5.2 Audit Trail Integrity

- An append-only audit trail captures every write operation on Restricted data with: actor identity (from authenticated session, not request body), timestamp (server-side, UTC), entity affected, before/after values, and operation type.
- The application service account does not have UPDATE or DELETE permissions on the audit table — verified at the database role level, not at the application code level.
- Audit trail entries are forwarded to the security log aggregation system (per the Security Controls Baseline) before the user-facing transaction completes.
- Failed authorization attempts are also logged, not just successful operations.

### 5.3 Export and Aggregation Path RBAC

- Data export endpoints (CSV, Excel, PDF) apply the same authorization filter as the underlying read endpoints; a user's export contains exactly what they could see in the UI.
- Aggregation endpoints (summary statistics, dashboards) do not allow inference of restricted data points by users without access to the underlying records.
- Pagination and search do not bypass authorization (a search across "all records" is filtered by what the user can see, not surfaced and then filtered client-side).

### 5.4 Auto-Remediation Validation

- Any auto-remediation applied at Gate C (per the Security Controls Baseline) is reviewed in this code review for correctness.
- Auto-remediated SAST findings (e.g., parameterization fixes) are inspected to confirm the fix is semantically correct, not just syntactically passing the rule.
- Auto-remediated dependency bumps do not break behavior; the dependency upgrade was tested at Gate B but the InfoSec reviewer confirms no regression in security-relevant behavior.

### 5.5 License and SBOM Verification

- The SBOM produced at Gate C is attached to the deployment artifact reference for this review.
- No prohibited licenses are present (cross-checked against the Security Controls Baseline license policy).
- License exceptions granted at Gate C are validated as still applicable.

### 5.6 Data Handling

- Restricted data fields are not logged in application logs (verified by reviewing log statements adjacent to Restricted data access).
- Restricted data is not cached in browser-accessible storage (localStorage, sessionStorage) beyond the working session.
- Restricted data shown in the UI is paginated or otherwise prevented from being scraped en masse without an audit trail entry.

---

## 6. Sign-Off Definitions

| Sign-Off Outcome     | Meaning                                                                 |
| -------------------- | ----------------------------------------------------------------------- |
| Pass                 | The reviewer has no concerns that prevent advancing to Gate E. Comments may exist for future improvement but do not block. |
| Request Changes      | The reviewer has identified one or more issues that should be addressed before advancing. The reviewer commits to re-review within the SLA. |
| Block                | The reviewer has identified an issue that must be resolved before advancing. Block is reserved for security, correctness, or policy issues; not stylistic preferences. Block from any required reviewer halts Gate D regardless of other sign-offs. |

A reviewer's sign-off is recorded against the specific commit reviewed. New commits invalidate prior sign-offs and require re-review (Section 8).

---

## 7. Escalation and SLA Enforcement

The Gate D agent enforces SLA mechanically. If a required reviewer has not produced a sign-off within the SLA, the agent applies the following escalation:

| Trigger                                              | Action                                                          |
| ---------------------------------------------------- | --------------------------------------------------------------- |
| Reviewer at 24 hours into SLA with no response       | Reminder notification to reviewer + their manager               |
| Reviewer at SLA expiry with no response              | Reviewer marked unavailable; alternate from the same role assigned by the agent; SLA clock resets for alternate reviewer |
| Same reviewer alternate fails to respond at SLA      | Escalation to InfoSec leadership (for InfoSec) or Architecture leadership (for Architect); peer absence escalates to the project's PO and engineering manager |
| Type 1B project stalled at Gate D for > 7 calendar days | Project flagged for executive review; no automatic timeout-to-pass under any circumstance |

There is no implicit pass through SLA expiry. A required reviewer's silence does not constitute approval — silence triggers reassignment, not advancement.

---

## 8. Re-Review Triggers

The Gate D agent invalidates prior sign-offs and requires re-review when any of the following occurs:

| Change                                                              | Re-Review Required                              |
| ------------------------------------------------------------------- | ----------------------------------------------- |
| New commits added to the branch under review                        | All sign-offs invalidated; full re-review of new commits |
| Auto-remediation is re-run after additional Gate C findings         | Architect (Type 1B: + InfoSec) re-review of remediated areas |
| Dependency bumped to a new major version                            | Architect re-review of dependency-related code |
| Authorization-related code or configuration modified                | Architect + InfoSec (Type 1B) full re-review of authorization paths |
| Data classification of any flow changes during development          | Full Gate D re-review under the new classification |
| Branch is rebased onto a new base with material changes             | Full re-review                                  |

Trivial changes (typo fixes, comment-only edits) may be acknowledged by reviewers via a "no further review needed" comment but are otherwise treated as new commits that invalidate prior sign-offs.

---

## 9. Pass Criteria Summary

| Classification | Required Sign-Offs                          | Block Threshold                            |
| -------------- | ------------------------------------------- | ------------------------------------------ |
| Type 1         | Peer (Pass)                                 | Any Block from peer or escalated Architect |
| Type 1B        | Peer (Pass) + Architect (Pass) + InfoSec (Pass) | Any Block from any required reviewer    |

Gate D produces a structured output for Gate E consumption: classification, list of reviewers, sign-off outcomes, link to review thread, hash of the reviewed commit, and a summary of significant comments. Gate E will refuse to deploy a commit hash that does not match the one signed off at Gate D.

---

## 10. Document History

| Version | Date         | Author     | Changes                                          |
| ------- | ------------ | ---------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author]   | Initial draft for CrewAI POC corpus              |
