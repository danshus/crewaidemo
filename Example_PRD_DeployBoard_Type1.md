# PRD: DeployBoard

**Vibe Coding Classification:** Type 1 (Internal · Non-Sensitive)
**Submitting PO:** Sarah Mitchell, Engineering PMO
**Executive Sponsor:** N/A — not required for Type 1
**Submission Date:** 2026-05-04
**Document Status:** Submitted to Vibe Coding intake — awaiting Pre-Stage Validator review
**Conforms to:** PRD Template v0.1

---

## 1. Executive Summary

DeployBoard is a stateless internal dashboard that consolidates deployment status across all internal services and environments into a single read-only view, replacing the current ad-hoc mix of Slack queries, manual lookups, and tribal knowledge engineering uses today.

The application reads from existing internal APIs and writes to no upstream system. **The application has no PCI scope. It processes no customer PII, no employee personal data beyond Entra ID display name in the browser session, and no regulated data of any kind.**

---

## 2. Business Case

The current state of deployment-status discovery is producing measurable productivity loss. An engineering survey conducted in March 2026 found that engineers spend approximately **40 minutes per week per person** searching across Slack, GitLab, and direct queries to DevOps to answer the question "what version is deployed where?" At the current engineering headcount of approximately 120 staff, this represents about **80 hours per week** of low-value work, and is a recurring source of friction during incident response.

A consolidated read-only dashboard is the smallest viable intervention. It does not change any deployment process, does not introduce any new permissions, and does not write to any upstream system. It only surfaces information that already exists in tools the team already uses. A SaaS purchase is not justified for this scope; a full R&D project would be disproportionate.

---

## 3. Target Audience

| Audience          | Population | Access Level             |
| ----------------- | ---------- | ------------------------ |
| Engineers         | 120        | Read-only, all services  |
| Engineering Mgrs  | 15         | Read-only, all services  |
| VP Engineering    | 1          | Read-only, all services  |
| DevOps            | 10         | Read-only, all services  |
| External users    | 0          | None                     |
| Customer-facing   | No         | —                        |
| Cross-tenant exposure | No     | —                        |

---

## 4. Data Interaction

| Direction | Source / Target       | Data                                                     | Sensitivity (per Data Classification Standard) |
| --------- | --------------------- | -------------------------------------------------------- | ---------------------------------------------- |
| READ      | Internal Deploy API   | Service name, version, environment, deployment timestamp, status | Internal · Non-Sensitive                       |
| READ      | GitLab API            | Commit SHA, MR title, MR author, branch                  | Internal · Non-Sensitive                       |
| READ      | Entra ID (browser)    | Logged-in user display name (session context only)       | Internal · Non-Sensitive                       |
| WRITE     | None                  | —                                                        | —                                              |

DeployBoard persists no data in its own datastore. The application is fully stateless: every page load queries upstream APIs in real time and caches results in browser memory only.

**Explicit scope statements:**
- No PCI cardholder data is processed.
- No customer PII is processed.
- No employee personal data is processed beyond Entra ID display name visible in the user's own browser session.
- No GDPR special categories, no SOX-material financial data, no DORA-scoped operational data flows through this application.

---

## 5. Functional Requirements

1. List view of all services with current deployed version per environment (Dev, QA, Staging, Production)
2. Filter by team, environment, and deployment status
3. Click-through to GitLab commit and merge request for each deployed version
4. Last-deployed-by attribution sourced from the upstream Deploy API
5. Auto-refresh every 5 minutes
6. Dark mode toggle

**Out of scope for v1:** Triggering deployments, rolling back deployments, modifying any upstream state, exporting data, alerting or notifications, mobile-optimized layout.

---

## 6. Proposed Tech Stack

| Layer         | Technology                              | Notes                                              |
| ------------- | --------------------------------------- | -------------------------------------------------- |
| Frontend      | Next.js / React 18                      | Matches Approved Tooling Catalog                   |
| Backend       | C# / .NET 8 (BFF pattern)               | Matches Approved Tooling Catalog                   |
| Auth          | Entra ID SSO                            | Standard standard pattern; no custom auth code        |
| Datastore     | None                                    | Application is fully stateless                     |
| Hosting       | Internal dev-tooling Kubernetes cluster | Existing infra; no new infra requested             |
| CI/CD         | GitLab + ArgoCD                         | Existing pipeline                                  |
| Observability | Existing internal logging + metrics stack | Standard structured logging; no custom telemetry |

All stack components are present in the Approved Tooling Catalog. No exception note required.

---

## 7. Integrations

| Integration         | Direction | Auth Method                       | Pre-Cleared in Catalog? |
| ------------------- | --------- | --------------------------------- | ----------------------- |
| Entra ID SSO        | Inbound   | OIDC, standard standard pattern      | Yes                     |
| Internal Deploy API | Outbound  | Service-to-service mTLS           | Yes                     |
| GitLab API          | Outbound  | Service-account PAT via Akeyless  | Yes                     |

All integrations are pre-cleared in the Internal Integration Catalog. No new integrations required. No Gate A architecture review items triggered by this section.

---

## 8. KPIs and Success Metrics

| Metric                                                  | Baseline | Target   | Measurement Window     |
| ------------------------------------------------------- | -------- | -------- | ---------------------- |
| Engineering team weekly active users                    | 0        | ≥ 80%    | 90 days post-launch    |
| "What's deployed where?" queries in #engineering Slack  | ~50/wk   | ≤ 15/wk  | 90 days post-launch    |
| Median self-reported time to find deployment status     | ~5 min   | ≤ 30 sec | 90 days post-launch    |
| Page load p95 latency                                   | N/A      | ≤ 2 sec  | Continuous             |

Four KPIs defined, all with numeric baselines or "N/A — new capability" notation, all with numeric targets and explicit measurement windows.

---

## 9. Budget and Resources

| Item                          | Estimate                       |
| ----------------------------- | ------------------------------ |
| Development effort            | 6 weeks × 1 FTE engineer       |
| PM/PO effort                  | 1 day/week × 8 weeks           |
| Infrastructure (incremental)  | $0 — uses existing dev cluster |
| Third-party licenses          | $0                             |
| **Total budgeted cost**       | **~$24K** (loaded engineer time)   |

**Funding source:** Engineering Productivity Initiative budget (FY27), line item EPI-2026-014. Approved by VP Engineering on 2026-04-12.

---

## 10. Timeline

| Phase                          | Duration   | Notes                                            |
| ------------------------------ | ---------- | ------------------------------------------------ |
| Pre-Stage validation           | 2 days     | This document                                    |
| Gate A — Architecture review   | 5 days     | Architect-only review path (Type 1)              |
| Development                    | 6 weeks    | Single engineer, weekly demos to PMO             |
| Gate B — Automated testing     | 1 week     | CI/CD with auto-remediation (max 2 loops)        |
| Gate C — Security scanning     | 1 week     | Standard scans, no InfoSec sign-off (Type 1)     |
| Gate D — Code review           | 3 days     | Peer review path (Type 1)                        |
| Gate E — Go-Live               | 2 days     | Single-phase internal rollout                    |
| **Total elapsed**              | **~10 weeks** |                                               |

---

## 11. Pre-Stage YES Conditions Self-Attestation

| Condition       | Status | Evidence                                                                            |
| --------------- | ------ | ----------------------------------------------------------------------------------- |
| Business case   | YES    | Section 2 — quantified at 80 hours/week of engineering productivity loss            |
| KPIs defined    | YES    | Section 8 — four metrics with baselines, targets, and measurement windows           |
| PO readiness    | YES    | Sarah Mitchell committed for full project duration; Type 1 path does not require executive sponsor |
| Budget approved | YES    | Section 9 — line item EPI-2026-014, approved 2026-04-12 by VP Engineering           |

---

## 12. Classification Justification

**Stated classification:** Type 1 (Internal · Non-Sensitive)

**Audience axis (Internal):** Confirmed by Section 3. The audience consists exclusively of internal users with Entra ID access. There are no external users, no customer-facing surface, and no cross-tenant exposure.

**Data axis (Non-Sensitive):** Confirmed by Section 4. Every data flow is read-only of operational metadata that, per the Data Classification Standard, qualifies as Internal · Non-Sensitive: service deployment metadata, source-control commit metadata, and Entra ID directory-level identity. No flows touch Internal · Restricted data, and no flows touch regulated data of any kind.

**Regulated-data scope:** None. No PCI scope. No GDPR-restricted PII. No SOX-material data. No DORA-scoped operational data.

**Type 1 path acknowledgment:** The PO acknowledges that a Type 1 classification routes the project through Architecture review only at Gate A and standard automated security scanning at Gate C, with no mandatory InfoSec involvement at Gates A, C, or D. The PO further acknowledges that if any data flow is subsequently identified as Internal · Restricted or higher, the project will be re-evaluated and rerouted through the Type 1B path.

---

## 13. Open Questions / Risks

**Risks:**
- Internal Deploy API is not formally rate-limited. DeployBoard's 5-minute polling, multiplied by approximately 120 concurrent users, could create load on the upstream service. *Mitigation:* Coordinate with DevOps prior to development to confirm acceptable polling frequency and implement client-side jitter to avoid synchronized request spikes.

**Open questions:**
- Should v1 include a manual refresh button in addition to auto-refresh? *Resolution target:* Beta feedback during development phase.

---

**Prepared by:** Sarah Mitchell, Engineering PMO
**Submitted to:** Vibe Coding intake queue — Pre-Stage Validator agent
**Submission timestamp:** 2026-05-04 09:14 UTC
