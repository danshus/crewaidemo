# Project Brief: DeployBoard

**Vibe Coding Classification:** Type 1 (Internal · Non-Sensitive)
**Submitting PO:** Sarah Mitchell, Engineering PMO
**Document Status:** Submitted to Vibe Coding intake — Pre-Stage validation pending
**Submission Date:** [DATE]

---

## 1. Executive Summary

DeployBoard is an internal-only engineering dashboard that consolidates deployment status across all internal services and environments into a single read-only view. It replaces the current ad-hoc process of Slack messages, manual queries, and tribal knowledge that engineers rely on today to answer "what version is deployed where, and who deployed it?"

The application reads from existing internal APIs only and writes to no upstream system. There is no customer data, no employee PII beyond Entra ID display names already visible in the browser context, no financial data, and no PCI scope.

---

## 2. Business Case

Engineering currently loses meaningful productivity time to deployment-status discovery. Internal survey (Aug 2026) showed engineers spend ~40 minutes per week per person chasing this information across Slack, GitLab, and direct queries to DevOps. At ~120 engineers, that is 80 hours/week of low-value work and a recurring source of cross-team friction during incident response.

A consolidated read-only dashboard is the simplest possible intervention: it does not change any deployment process, it does not introduce any new permissions, and it does not write to any system. It only surfaces information that already exists in tools the team already uses.

---

## 3. Target Audience

| Audience          | Population | Access Level             |
| ----------------- | ---------- | ------------------------ |
| Engineers         | ~120       | Read-only, all services  |
| Engineering Mgrs  | ~15        | Read-only, all services  |
| VP Engineering    | 1          | Read-only, all services  |
| DevOps            | ~10        | Read-only, all services  |

**External users:** None.
**Customer-facing:** No.
**Cross-tenant exposure:** No.

---

## 4. Data Interaction

| Direction | Source             | Data                                                                  | Sensitivity     |
| --------- | ------------------ | --------------------------------------------------------------------- | --------------- |
| READ      | Internal Deploy API | Service name, version, env, deployment timestamp, status              | Internal · Non-Sensitive |
| READ      | GitLab API          | Commit SHA, MR title, MR author, branch                               | Internal · Non-Sensitive |
| READ      | Entra ID            | Logged-in user display name (browser session only)                    | Internal · Non-Sensitive |
| WRITE     | None                | —                                                                     | —               |

No data is persisted in DeployBoard's own datastore. The application is fully stateless — every page load queries upstream APIs in real time and caches results in browser memory only.

**No PCI scope. No PII beyond Entra display name. No financial data. No customer data.**

---

## 5. Functional Requirements

1. List view of all services with current deployed version per environment (Dev, QA, Staging, Prod)
2. Filter by team, environment, deployment status
3. Click-through to GitLab commit / MR for each deployed version
4. Last-deployed-by attribution (Entra ID display name from upstream API)
5. Auto-refresh every 5 minutes
6. Dark mode toggle (engineering team request)

**Out of scope for v1:** Triggering deployments, rolling back deployments, modifying any upstream state, exporting data, alerting/notifications.

---

## 6. Proposed Tech Stack

| Layer       | Technology                            | Notes                                          |
| ----------- | ------------------------------------- | ---------------------------------------------- |
| Frontend    | Next.js / React 18                    | Per the approved frontend stack              |
| Backend     | C# / .NET 8 (BFF pattern)             | Per the approved backend stack               |
| Auth        | Entra ID SSO (existing pattern)       | No custom auth logic                           |
| Datastore   | None                                  | Fully stateless                                |
| Hosting     | Internal dev-tooling Kubernetes       | Existing cluster, no new infra                 |
| CI/CD       | GitLab + ArgoCD                       | Existing pipeline                              |
| Observability | Existing internal stack             | Standard logging + metrics, no custom telemetry |

---

## 7. Integrations

| Integration       | Direction | Auth Method                  | Pre-Cleared in Catalog? |
| ----------------- | --------- | ---------------------------- | ----------------------- |
| Entra ID SSO      | Inbound   | OIDC, existing standard pattern | Yes                     |
| Internal Deploy API | Outbound | Service-to-service mTLS      | Yes                     |
| GitLab API        | Outbound  | Service account PAT (Akeyless) | Yes                   |

No new integrations. All upstream systems are already in the the Integration Catalog with Internal · Non-Sensitive clearance.

---

## 8. KPIs and Success Metrics

| Metric                                                  | Baseline | Target    | Measurement Window |
| ------------------------------------------------------- | -------- | --------- | ------------------ |
| Engineering team weekly active users                    | N/A      | ≥ 80%     | 90 days post-launch |
| "What's deployed where?" Slack queries (#engineering)   | ~50/wk   | ≤ 15/wk   | 90 days post-launch |
| Time to find deployment status (median, self-reported)  | ~5 min   | ≤ 30 sec  | 90 days post-launch |
| Page load p95 latency                                   | N/A      | ≤ 2 sec   | Continuous         |

---

## 9. Budget and Resources

| Item                       | Estimate                       |
| -------------------------- | ------------------------------ |
| Development effort         | 6 weeks × 1 FTE engineer       |
| PM/PO effort               | 1 day/week × 8 weeks           |
| Infrastructure (incremental) | $0 — uses existing dev cluster |
| Third-party licenses       | $0                             |
| Total budgeted cost        | ~$24K (loaded engineer time)   |

**Funding source:** Engineering productivity initiative budget (already approved, line-item available).

---

## 10. Timeline

| Phase                     | Duration | Notes                                   |
| ------------------------- | -------- | --------------------------------------- |
| Pre-Stage validation      | 2 days   | This brief                              |
| Gate A — Architecture     | 5 days   | Architect review only (Type 1 path)     |
| Development               | 6 weeks  | Single engineer, weekly demos to PMO    |
| Gate B — Automated Testing | 1 week   | CI/CD with auto-remediation             |
| Gate C — Security         | 1 week   | Standard scans, no InfoSec involvement  |
| Gate D — Code Review      | 3 days   | Peer review, no InfoSec sign-off needed |
| Gate E — Go-Live          | 2 days   | Internal rollout, no staged release     |
| **Total**                 | **~10 weeks** |                                    |

---

## 11. Pre-Stage YES Conditions Self-Attestation

| Condition          | Status | Evidence                                                                 |
| ------------------ | ------ | ------------------------------------------------------------------------ |
| Business case      | YES    | Section 2 — quantified time loss, simple intervention                    |
| KPIs defined       | YES    | Section 8 — four metrics with baselines, targets, and measurement windows |
| PO readiness       | YES    | Sarah Mitchell committed for full project duration; engineering sponsor confirmed |
| Budget approved    | YES    | Line item exists in engineering productivity budget                       |

---

## 12. Classification Justification

This project is classified **Type 1 (Internal · Non-Sensitive)** because:

- **Audience is Internal:** No external users, no customers, no partners. Only internal users with Entra ID access.
- **Data is Non-Sensitive:** Read-only access to operational metadata (service names, versions, deployment timestamps). No PCI scope, no PII beyond Entra display names, no commercial terms, no employee performance data, no customer information.
- **No write paths:** Application cannot mutate any upstream system.
- **No regulated data flows:** No GDPR, PCI-DSS, SOX, or DORA in-scope data is processed.

Per the Vibe Coding Classification Policy, Type 1 projects require Architecture Review only at Gate A and standard automated security scans at Gate C, with no mandatory InfoSec sign-off.

---

## 13. Open Questions / Risks

- **Risk:** Internal Deploy API is not formally rate-limited; DeployBoard's 5-min polling at scale could create load. **Mitigation:** Coordinate with DevOps to confirm acceptable polling frequency before development.
- **Open question:** Should v1 include a manual refresh button in addition to auto-refresh? — Defer to engineering team feedback in beta.

---

**Prepared by:** Sarah Mitchell, Engineering PMO
**Reviewed by:** [Engineering Sponsor — TBD]
**Submitted to:** Vibe Coding intake queue
