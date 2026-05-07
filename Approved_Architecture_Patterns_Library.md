# Approved Architecture Patterns Library

**Document Owner:** Security Architecture
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Gate A of the Vibe Coding pipeline for Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications

---

## 1. Purpose

This library catalogs the architectural patterns approved for use in Vibe Coding internal applications. Where the Approved Tooling Catalog answers "which tools may I use?", this library answers "how do I combine those tools correctly?".

The Gate A Architecture Gate Reviewer agent uses this library as the positive-form rubric for evaluating proposed designs. A submission whose architecture matches an approved pattern receives a fast review focused on parameter choices and edge cases. A submission that does not match any pattern is not automatically rejected, but is escalated for design discussion and must produce an Architecture Decision Record explaining the deviation. New patterns can be added to the library through the process in Section 13.

The patterns in this library are deliberately narrow and opinionated. The Vibe Coding philosophy favors a small set of well-supported templates over a flexible toolkit. A new pattern that diverges meaningfully from any in this library is more likely a sign that the project does not fit Vibe Coding scope than a reason to expand the library.

---

## 2. How to Use This Library

A PRD's Section 6 (Tech Stack) and Section 7 (Integrations) implicitly describe an architecture. The Gate A agent's first action is to match that description to one of the patterns in Sections 4 through 12. The match may combine patterns — most non-trivial applications combine several (an authentication pattern, an authorization pattern, a data-handling pattern). The agent identifies which patterns apply, applies the per-pattern checklist, and surfaces deviations.

A pattern match is structural, not literal. An application using PostgreSQL instead of MSSQL still matches the CRUD pattern provided the structural elements are present (server-side authorization, audit trail, RBAC enforcement). Tooling choices are governed by the Tooling Catalog; pattern conformance is governed by this library.

---

## 3. Pattern Catalog Index

| Pattern                                          | Section | Type 1 | Type 1B | Notes                                       |
| ------------------------------------------------ | ------- | ------ | ------- | ------------------------------------------- |
| SSO Authentication                               | 4       | ✓      | ✓       | Mandatory for all Vibe Coding apps          |
| Group-Claim RBAC Authorization                   | 5       | ✓      | ✓       | Mandatory; required to be server-side       |
| Secrets Injection from Vault                     | 6       | ✓      | ✓       | Mandatory whenever any secret is used       |
| Stateless Read-Only Aggregator                   | 7       | ✓      | (rare)  | Default pattern for read-only dashboards    |
| CRUD with RBAC and Audit Trail                   | 8       | (rare) | ✓       | Default pattern for Type 1B applications    |
| Append-Only Audit Trail                          | 9       | (optional) | ✓   | Mandatory for Type 1B; embedded in Pattern 8 |
| Internal API Integration                         | 10      | ✓      | ✓       | Service account auth, retry, circuit break  |
| M365 Graph API Integration                       | 11      | ✓      | ✓       | For document storage and email notifications |
| Background Job / Scheduled Task                  | 12      | ✓      | ✓       | For renewal alerts, periodic processing     |

Patterns marked "(rare)" are technically permitted but unusual for that classification — flagged for additional Architecture review.

---

## 4. Pattern: SSO Authentication

**Intent.** Authenticate users to internal applications via the corporate identity provider with no application-side credential handling.

**When to use.** Every Vibe Coding application. There are no exceptions.

**When NOT to use.** Never — there is no approved alternative.

**Components.**

| Component               | Role                                                          |
| ----------------------- | ------------------------------------------------------------- |
| Identity provider (Entra ID) | Authentication authority; issues OIDC tokens             |
| Frontend SPA            | Initiates auth flow, stores token in memory only              |
| Backend (BFF or API)    | Validates token on every request via SDK; extracts claims     |
| Token validation library | Framework-provided (Microsoft.Identity.Web for .NET, MSAL.js for frontend) |

**Data flow.**
1. User opens application URL.
2. Frontend redirects to identity provider for authentication.
3. User authenticates (with MFA per organizational policy).
4. Identity provider returns an ID token and access token to the frontend.
5. Frontend includes access token in `Authorization: Bearer` header on every backend request.
6. Backend validates the token signature, expiration, audience, and issuer using the SDK.
7. Backend extracts user identity and group claims from the validated token for use in authorization decisions.

**Embedded security controls.**
- No application-side password handling (the only safe credential is the credential the application never sees).
- Token expiration enforced server-side on every request; client-side expiration is advisory only.
- MFA enforcement is the identity provider's responsibility; the application does not implement step-up authentication independently.

**Common mistakes.**
- Storing tokens in `localStorage` instead of memory — exposes the token to any XSS that may exist in the app.
- Trusting the ID token on the backend (the ID token is for the frontend; the backend validates the access token).
- Hand-rolling JWT validation instead of using the SDK — typically gets at least one of `aud`, `iss`, `exp`, or signature validation wrong.
- Caching the user's group membership in application state across requests rather than re-extracting from the validated token each request.

**Applicability.** Type 1 and Type 1B both required to use this pattern. No simplification permitted for Type 1.

---

## 5. Pattern: Group-Claim RBAC Authorization

**Intent.** Authorize requests using the user's group membership as conveyed in the SSO token's claims, with enforcement on the server side of every protected endpoint.

**When to use.** Every Vibe Coding application that has more than one user role, or that has any access boundary at all (which is approximately every application).

**When NOT to use.** Applications that genuinely have a single uniform user population with identical access. These are rare; most applications discover after launch that they do, in fact, need authorization differentiation.

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Identity provider groups   | Source of truth for role membership                        |
| SSO token group claims     | Conveyance mechanism; backend reads from validated token   |
| Authorization middleware   | Per-endpoint enforcement; declarative attribute-based      |
| Resource-level checks      | Additional checks where group alone is insufficient (e.g., BU scoping) |

**Data flow.**
1. User's group memberships are managed in the identity provider (out-of-band, by the appropriate identity governance process).
2. Authentication (Pattern 4) produces a token containing group claims.
3. The backend's authorization middleware reads the group claims from the validated token.
4. Each endpoint declares the groups (or roles derived from groups) authorized to access it.
5. For endpoints returning multi-tenant data (e.g., BU-scoped views), an additional resource-level check applies the user's permitted scope to the query.
6. Failed authorization returns a 403 (not 401, which would indicate authentication failure) and produces an audit log entry (Pattern 9).

**Embedded security controls.**
- Authorization is server-side; the frontend may hide UI elements based on claims for usability, but it is not the security boundary.
- Group claims come only from the validated SSO token; never from request body, URL parameters, or client-supplied headers.
- Resource-level scoping (e.g., "this user may access BU-A's data only") is enforced at the database query level, not by post-filtering the result set.

**Common mistakes.**
- Loading all records and filtering in the application after the database returns — slow, memory-heavy, and fundamentally insecure if memory or pagination boundaries leak.
- Using a separate user-to-permission table maintained in the application database rather than the identity provider's groups — drifts out of sync, becomes a separate identity store.
- Allowing the user to specify which BU's data they want via a query parameter without verifying their access to that BU.
- Implementing authorization in the controller layer for some endpoints and in the service layer for others — inconsistent placement breeds gaps.

**Applicability.** Mandatory for both Type 1 and Type 1B. Type 1B applications additionally require explicit authorization tests (per the Code Review Checklist).

---

## 6. Pattern: Secrets Injection from Vault

**Intent.** Make secrets available to running application code without those secrets ever appearing in source code, configuration files in source control, or environment variables defined statically.

**When to use.** Any time the application needs a database connection string, an API key for an external service, or any other credential.

**When NOT to use.** When the application has no secrets — vanishingly rare.

**Components.**

| Component                       | Role                                                  |
| ------------------------------- | ----------------------------------------------------- |
| Secrets vault                   | Source of truth for all secrets                       |
| Secrets injection mechanism     | Retrieves secrets at runtime; injects into application environment |
| Application secrets configuration | References secret name only, not value             |

**Data flow.**
1. Secret is created in the vault with appropriate access policies (which workload identities may read it).
2. Application's deployment manifest references the secret by name and target injection point (environment variable, mounted file).
3. Injection mechanism retrieves the secret value from the vault using the workload's identity.
4. Application reads the secret from its environment at startup or, for rotatable secrets, on each use.
5. Secret rotation in the vault is automatically reflected in the application's environment without code change or redeployment.

**Embedded security controls.**
- Source code never contains secret values.
- Source-controlled config files contain secret names, not secret values.
- Application logs do not log environment variables on startup or any other path.
- Secret access is auditable per workload identity in the vault's audit log.

**Common mistakes.**
- Embedding secrets in `.env` files committed to source control.
- Using Kubernetes Secrets directly without vault sourcing — Kubernetes Secrets are base64-encoded, not encrypted by default, and are visible to anyone with namespace read access.
- Logging the entire request or response for debugging when those payloads contain secrets.
- Caching secrets in long-lived application state such that rotation requires a restart.
- Constructing connection strings from individual secret components in source code — leaks the structure even if individual secrets are protected.

**Applicability.** Mandatory for both Type 1 and Type 1B. The pattern is identical for both classifications.

---

## 7. Pattern: Stateless Read-Only Aggregator

**Intent.** Surface a unified view of data from multiple upstream APIs without persisting any state in the application's own datastore.

**When to use.** Read-only dashboards, status pages, reporting views that compose existing API surfaces. Project A (DeployBoard) is the canonical instance.

**When NOT to use.** Applications that need to write back to upstream systems, applications with multi-step workflows that must survive page reloads, applications with significant compute cost where caching is required for performance.

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Frontend SPA               | Renders the aggregated view; holds no persistent state     |
| Backend BFF                | Composes upstream API responses; stateless between requests |
| Upstream APIs              | Authoritative data sources; the aggregator never owns the data |
| Browser-side cache         | Short-lived in-memory cache for performance                |

**Data flow.**
1. User loads the dashboard; SSO authentication completes (Pattern 4).
2. Frontend requests aggregated view from BFF endpoint.
3. BFF, on each request, calls each upstream API in parallel using its service-account credentials (Pattern 6 + Pattern 10).
4. BFF composes the responses into a single payload; applies any user-specific filtering based on group claims (Pattern 5).
5. Frontend renders the payload; periodic refresh re-runs the flow.

**Embedded security controls.**
- No application database means no application database to compromise.
- Upstream APIs remain the single source of truth, including for authorization (the BFF must enforce that the user can see what the upstream allows them to see).
- Failure of any upstream API does not corrupt application state because there is no application state.

**Common mistakes.**
- Adding a "small cache" that gradually grows into a partial replica of the upstream data — at which point the data classification of the upstream data flows into the aggregator and changes its tier.
- Polling upstream APIs faster than the upstream can sustain, especially when scaled to many concurrent users without client-side jitter.
- Forwarding upstream API errors to the user verbatim, exposing internal API details.
- Aggregating data such that the aggregate is more sensitive than any individual upstream record — see the Data Classification Standard's note on aggregation.

**Applicability.** Type 1 default. Type 1B is technically possible but uncommon — most Type 1B requirements drive toward Pattern 8 (CRUD with audit) rather than read-only aggregation.

---

## 8. Pattern: CRUD with RBAC and Audit Trail

**Intent.** Standard pattern for internal applications that create, read, update, and delete records of restricted internal data with full accountability.

**When to use.** Type 1B applications managing internal commercial, operational, or governance data. Project B (VendorTrack) is the canonical instance.

**When NOT to use.** Pure read-only applications (use Pattern 7), applications without record-level state (use Pattern 7), applications where the data is non-sensitive (use a simpler pattern even though this one would also work).

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Frontend SPA               | UI for record CRUD operations                              |
| Backend API                | Business logic, authorization enforcement, validation      |
| Application database       | Persistent storage for records                             |
| Audit table (separate schema) | Append-only audit trail (Pattern 9)                     |
| Database role separation   | Application service account vs. audit-read-only account    |

**Data flow.**
1. Authentication (Pattern 4) and authorization (Pattern 5) on every request.
2. Read operations: filtered by user's authorized scope; no records leak across scope boundaries.
3. Write operations: validate input, write to records table, write to audit table in the same transaction, return success or fail atomically.
4. Audit trail records actor, timestamp, entity, before/after, operation type — for every write.
5. Soft-delete only: hard deletes are prohibited because they would erase the audit trail's referential integrity.

**Embedded security controls.**
- Server-side authorization on every endpoint, including resource-level scope.
- Audit trail integrity via database role separation: the application service account cannot UPDATE or DELETE rows in the audit table.
- Transactional consistency between record write and audit write (the audit entry is part of the same database transaction as the change it records).
- Soft delete with a `deleted_at` timestamp preserves audit trail history while removing the record from active queries.

**Common mistakes.**
- Implementing audit logging in application code as a separate write after the main write — opens a window where the main write succeeds but the audit fails (or vice versa). Use a single transaction.
- Using the same database account for application writes and audit writes — defeats the integrity guarantee.
- Allowing hard deletes "for cleanup" — undermines the audit trail's value.
- Storing before/after values as serialized blobs without parseable structure — makes audit queries useless.
- Failing to apply the same authorization filter to data exports as to UI reads — common path for accidental scope leaks.

**Applicability.** Type 1B default. Type 1 may use this pattern but it is overweight for Non-Sensitive data; Type 1 typically uses simpler patterns.

---

## 9. Pattern: Append-Only Audit Trail

**Intent.** Maintain a tamper-evident record of every modification to Restricted data, queryable for accountability and forensic purposes.

**When to use.** Mandatory for every Type 1B application. Optional but recommended for Type 1 applications that touch operational metadata where accountability matters.

**When NOT to use.** As a substitute for application logs (audit trails are about state changes, not application events). As a reporting source for non-audit purposes (the audit table should not be a denormalized reporting cache).

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Audit schema/table         | Separate from the application schema                       |
| Database role for app      | Has INSERT on audit; lacks UPDATE/DELETE                   |
| Database role for audit reader | Has SELECT on audit; used by reporting and security tooling |
| Audit forwarding           | Sends audit entries to security log aggregation in near-real-time |

**Data flow.**
1. Application performs a write to a Restricted-data record.
2. In the same database transaction, the application inserts a corresponding row into the audit table containing actor, timestamp, entity reference, operation, before-value, after-value.
3. Audit forwarding (database trigger, CDC stream, or application-level forwarder) sends the entry to the SIEM.
4. SIEM ingestion confirms receipt; failures alert the security operations team.

**Embedded security controls.**
- Append-only enforcement at the database role level, not the application level.
- Transactional atomicity: if the audit insert fails, the record write fails.
- Forwarding to an external system means deletion of the on-application audit data cannot suppress evidence; the SIEM has the canonical record.
- Audit entries are themselves Restricted (per the Data Classification Standard) and protected accordingly.

**Common mistakes.**
- Implementing the append-only constraint in application code only — circumvented by any direct database access.
- Using a single role for both application writes and audit reads — every application bug becomes a potential audit-tampering vector.
- Relying on database triggers for audit writes without verifying they fire for ORM-generated queries (some ORMs bypass triggers under specific conditions).
- Forwarding audit entries to the SIEM asynchronously without retry, such that a forwarding failure silently loses entries.

**Applicability.** Mandatory for Type 1B. Optional but recommended for Type 1 where operational accountability matters.

---

## 10. Pattern: Internal API Integration

**Intent.** Consume an internal API from a Vibe Coding application using a service-account credential with appropriate retry, circuit-breaking, and observability.

**When to use.** Whenever the application calls another internal API.

**When NOT to use.** External (third-party) API integration — those follow the third-party integration pattern outside Vibe Coding scope.

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Service-account credential | Identifies the calling application to the upstream API     |
| HTTP client with retry     | Handles transient upstream failures with exponential backoff |
| Circuit breaker            | Stops cascading failures when upstream is degraded         |
| Observability (tracing)    | Correlates upstream calls to user requests for debugging   |

**Data flow.**
1. Service account credential is retrieved at startup from the secrets vault (Pattern 6).
2. Application makes upstream API call with the credential in the appropriate header (Bearer token, mTLS certificate, or API key per upstream's contract).
3. Retries on transient failures (5xx, network errors) with exponential backoff capped at 3 retries.
4. Circuit breaker opens after consecutive failures; half-open after a cooldown to test recovery.
5. Distributed trace context propagated to the upstream so request paths are correlatable.

**Embedded security controls.**
- Service-account credentials are vault-managed and rotated independently of application deployment.
- Upstream calls use TLS; certificate validation is not disabled.
- Authentication is service-to-service; the application does not pass through user credentials except where the upstream API is designed for delegated user identity (rare).

**Common mistakes.**
- Disabling certificate validation "to make it work in dev" and forgetting to re-enable in production.
- Logging request/response payloads at INFO level — captures secrets and Restricted data.
- Implementing infinite retry loops that hammer a degraded upstream further into degradation.
- Using the user's identity to call upstream APIs ("on-behalf-of" flows) without the upstream actually expecting delegated identity — leaks user context unnecessarily.

**Applicability.** Type 1 and Type 1B identical.

---

## 11. Pattern: M365 Graph API Integration

**Intent.** Integrate with Microsoft 365 services (SharePoint, mail, calendar) for document storage, notifications, and shared resources.

**When to use.** Document storage when the existing organizational pattern is SharePoint. Email notifications via the corporate mail platform. Calendar integration for scheduled actions.

**When NOT to use.** As a general database substitute (SharePoint is not a database). For high-volume programmatic email (use a transactional mail service, not user-facing mail).

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Graph API app registration | Application identity in the identity provider              |
| App-only Graph credential  | Vault-managed credential for backend Graph calls           |
| Graph SDK or HTTP client   | API consumption layer                                      |
| Permission scopes          | Specific Graph permissions granted to the application      |

**Data flow.**
1. App registration created in the identity provider with explicit Graph permissions (e.g., `Sites.ReadWrite.Selected` for SharePoint, `Mail.Send` for sending notifications).
2. Permissions are scoped as narrowly as possible — `Sites.Selected` rather than `Sites.ReadWrite.All`, `Mail.Send` rather than full mailbox access.
3. App-only credential stored in vault (Pattern 6).
4. Application uses Graph SDK or HTTP client to call Graph APIs with app-only authentication.
5. Operations are logged with the actor identity (the app registration, not the application service account on its host).

**Embedded security controls.**
- Permissions are scoped explicitly at the app registration; the application cannot exceed the granted permissions.
- App-only credentials are rotated on the same cadence as other vault secrets.
- Graph operations are audited at the Graph layer, providing an independent audit trail beyond the application's own audit.

**Common mistakes.**
- Requesting overly broad permissions (`Files.ReadWrite.All`) when narrower permissions would suffice — opens unnecessary blast radius.
- Using delegated permissions for backend service operations — fails when no user is present (e.g., scheduled jobs).
- Embedding the app secret in source code or non-vault storage.
- Treating Graph permissions as a "set once and forget" — periodic review is required to remove permissions no longer needed.

**Applicability.** Type 1 and Type 1B both. Type 1B requires explicit InfoSec review of the requested Graph permissions at Gate A.

---

## 12. Pattern: Background Job / Scheduled Task

**Intent.** Run periodic or scheduled work outside the request/response path — typical examples are renewal alert emails, daily summary generation, periodic cleanup.

**When to use.** Any time work must run on a schedule, in response to a time-based trigger, or as a long-running batch outside an HTTP request.

**When NOT to use.** When the work is genuinely interactive and should be tied to a user request. When the work needs guaranteed exactly-once semantics under all failure modes (background jobs are typically at-least-once; design for idempotency).

**Components.**

| Component                  | Role                                                       |
| -------------------------- | ---------------------------------------------------------- |
| Scheduler                  | Triggers job execution (cron, Kubernetes CronJob, scheduled function) |
| Job worker                 | Performs the work; idempotent                              |
| Job state tracking         | Records which jobs have completed (for idempotency and alerting) |
| Failure notification       | Alerts on job failures or excessive duration               |

**Data flow.**
1. Scheduler triggers job at the configured cadence.
2. Job worker authenticates as a service identity (no human user context).
3. Job worker checks job state to determine work scope (e.g., which renewals need alerts that haven't been sent).
4. Job worker performs work, writing to application state and audit trail (Pattern 9 if Type 1B).
5. Job worker records completion in job state.
6. Failures are surfaced via observability and notification.

**Embedded security controls.**
- Background jobs use service identities, not user identities — preventing privilege drift over time.
- Job state tracking prevents duplicate work in the case of scheduler retries.
- Audit trail entries from background jobs are clearly attributed to the job identity, not impersonating a user.

**Common mistakes.**
- Storing user credentials in scheduled job configuration to "act as that user" — antipattern that compounds privilege over time.
- Implementing non-idempotent jobs that misbehave on retry (e.g., sending multiple emails because the scheduler retried after a partial failure).
- Logging full record contents at job completion — creates large logs containing potentially Restricted data.
- Long-running jobs without progress tracking, making failures invisible until they exceed the schedule interval.

**Applicability.** Type 1 and Type 1B identical, except that Type 1B background jobs writing to records must follow Pattern 9 (audit trail) for those writes.

---

## 13. Adding New Patterns

When a Vibe Coding project's architecture genuinely does not match any pattern in this library, the appropriate path depends on how often the deviation will recur:

**Single-project deviation.** Document in an Architecture Decision Record at Gate A. The deviation is approved (or rejected) for the specific project. Future projects do not inherit the approval.

**Recurring pattern.** If the same deviation appears across multiple Vibe Coding submissions, propose a new pattern via the Standards intake process. Proposals must include intent, when-to-use, components, embedded security controls, and common mistakes — i.e., the pattern must arrive complete, not as a request for someone else to flesh out. New patterns are reviewed quarterly by Security Architecture.

**Out-of-scope architecture.** If the deviation reflects a fundamentally different application class — external-facing, regulated data, complex integration topology — the appropriate path is not a new Vibe Coding pattern but a reroute to the full R&D process.

The bias of this library is toward stability. Adding patterns is permitted; modifying existing patterns is rare. A pattern's stability is part of its value.

---

## 14. Document History

| Version | Date         | Author     | Changes                                          |
| ------- | ------------ | ---------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author]   | Initial draft for CrewAI POC corpus              |
