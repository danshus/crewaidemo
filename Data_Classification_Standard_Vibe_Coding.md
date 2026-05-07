# Data Classification Standard — Vibe Coding Internal Applications

**Document Owner:** Security Architecture
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Vibe Coding Type 1 and Type 1B applications only

---

## 1. Purpose

This standard defines how data is classified in applications built under the the Vibe Coding initiative. It exists for two reasons:

First, classification drives the Vibe Coding gate path. Type 1 (Architects review only) and Type 1B (Architects + InfoSec) are determined by the most sensitive data flow in the application. Misclassification at intake produces the wrong reviewer set and either blocks low-risk projects unnecessarily or admits higher-risk projects without adequate review.

Second, classification drives application-level controls. The handling requirements in Section 4 are not aspirational — they are the minimum controls a developer must implement, and the minimum controls a reviewer must verify. An application classified as touching Internal · Restricted data and shipped without server-side authorization, audit logging, or least-privilege access is non-compliant regardless of whether the project passed the gates.

---

## 2. Scope

**In scope:** Internal applications built under the Vibe Coding initiative classified as Type 1 (Internal · Non-Sensitive) or Type 1B (Internal · Restricted).

**Out of scope:**
- External-facing applications (Type 2 and Excluded cells of the Vibe Coding matrix)
- Applications processing PCI cardholder data, customer PII, or other regulated data
- Customer-facing portals or APIs
- Cross-tenant or partner-shared platforms

Applications outside this scope are governed by the the enterprise Data Classification Policy and the full R&D process. If a Vibe Coding submission is found to contain data flows that fall outside this scope, it is rerouted at Pre-Stage.

---

## 3. Classification Tiers

Three tiers apply to Vibe Coding internal applications:

### 3.1 Public

Data that has been deliberately approved for unrestricted disclosure outside the organization. Examples: published product documentation, marketing materials on the public website, press releases, public job postings.

For practical purposes, **Public data rarely originates inside a Vibe Coding internal application** — these applications consume internal data sources, not public ones. Public data is included in this standard for completeness, not because most Vibe Coding apps will encounter it.

### 3.2 Internal · Non-Sensitive

Data that is not intended for public disclosure but whose accidental disclosure within the company would have negligible impact. This is the "default tier" for most operational metadata in internal systems.

Examples:
- Service deployment metadata (service name, version, environment, deployment timestamp)
- Internal API documentation
- Public organizational chart, internal contact directory entries (name, title, team, business email)
- General internal announcements, all-hands materials
- Aggregate operational metrics (DAU, error rates, deployment frequency)
- Source code references in the form of paths, repo names, and commit SHAs
- Project status updates, sprint reports
- Entra ID identity in the context of "this user is logged in" — display name, business email, group membership

### 3.3 Internal · Restricted

Data that is sensitive within the organization and whose disclosure — even internally to the wrong audience — would cause harm. The harm may be commercial (negotiating leverage lost), reputational (perception of mishandling), legal (audit findings, regulatory exposure short of formal breach), or organizational (loss of trust, manager-employee relationship damage).

Examples relevant to Vibe Coding internal applications:
- Vendor commercial terms (contract values, negotiated discounts, renewal terms, MSA exception clauses)
- Internal financial data not published in earnings filings (margin by product, cost-center spend detail, internal forecasts)
- Procurement records of in-flight negotiations
- Internal audit findings, control test results, exception logs
- Security findings prior to remediation (open vulnerabilities, configuration gaps, pen test results)
- Personnel data beyond directory level (compensation, performance ratings, disciplinary records, hiring decisions in flight)
- BU-specific commercial data where another BU should not have visibility (cross-BU competitive insights)
- M&A or strategic-partnership materials prior to public disclosure
- Audit trail data — records of who changed what and when, in any system. Audit trails are inherently restricted because they reveal patterns of internal decision-making.

**A note on aggregation.** Data that is Non-Sensitive in isolation may become Restricted in aggregation. A list of every vendor the organization uses is approximately Non-Sensitive. A list of every vendor with their contract values is Restricted. Classification must consider the whole data set the application surfaces, not just individual fields.

---

## 4. Handling Requirements per Tier

| Requirement                          | Public            | Internal · Non-Sensitive | Internal · Restricted |
| ------------------------------------ | ----------------- | ------------------------ | --------------------- |
| Authentication required              | No                | Yes (Entra ID SSO)       | Yes (Entra ID SSO)    |
| Authorization required               | No                | Per application baseline | Yes — explicit RBAC, server-side enforced |
| Encryption in transit                | Recommended       | Required (TLS 1.2+)      | Required (TLS 1.2+)   |
| Encryption at rest                   | Not required      | Per platform default     | Required (per platform — TDE, KMS-CMK, or equivalent) |
| Secrets management                   | N/A               | Akeyless                 | Akeyless              |
| Audit logging of access              | Not required      | Recommended              | Required — write events at minimum, read events for high-sensitivity records |
| Audit logging of changes             | N/A               | Recommended              | Required — append-only audit trail with user, timestamp, before/after values |
| Change-event integrity               | N/A               | Application-level OK     | Required — application service account must not have DELETE/UPDATE on audit records |
| Forwarding to QRadar                 | No                | Application logs only    | Audit trail forwarded |
| Retention                            | Per business need | Per business need        | Minimum 1 year for audit trail; data retention per business need |
| Export controls                      | None              | None                     | RBAC must apply to exports — a user's export must contain only what they could see in the application |
| Cross-BU visibility                  | No constraint     | No constraint            | Restricted — BU-scoped access enforced where commercially relevant |
| Display in UI                        | No constraint     | No constraint            | No bulk display of restricted fields without justified use case |

These requirements are minimums. Applications may apply stricter controls. Applications may not apply weaker controls without an approved exception.

---

## 5. Decision Tree

The following decision tree classifies a single data flow. Apply it independently to every flow in Section 4 of the PRD. The application's overall classification is the highest classification of any individual flow.

```
START: I have a data flow to classify.

Q1: Is the data published, or already approved for public disclosure?
    → YES → Public
    → NO  → Continue to Q2

Q2: Is this regulated data?
    (PCI cardholder data, customer PII, GDPR special categories,
     SOX-material financial data not yet disclosed, employee personal data
     beyond directory level)
    → YES → NOT VIBE CODING ELIGIBLE. Reroute to full R&D process.
    → NO  → Continue to Q3

Q3: Would unauthorized internal disclosure of this data cause
    commercial, reputational, legal, or organizational harm?
    → YES → Internal · Restricted
    → NO  → Continue to Q4

Q4: Does this data, in aggregate, reveal commercially sensitive patterns
    even if individual records are not sensitive?
    → YES → Internal · Restricted
    → NO  → Internal · Non-Sensitive
```

If the answer to any question is genuinely uncertain, default to the higher classification. Conservative classification is reversible at low cost; under-classification produces controls that are missing rather than excessive.

---

## 6. Mapping to Vibe Coding Application Types

| Highest Data Flow Classification | Allowed Vibe Coding Type |
| -------------------------------- | ------------------------ |
| Public only                      | Type 1                   |
| Internal · Non-Sensitive         | Type 1                   |
| Internal · Restricted            | Type 1B                  |
| Anything higher                  | Not Vibe Coding eligible |

A Type 1 application that is later found to handle Internal · Restricted data must be re-evaluated and re-routed through the Type 1B path, including retroactive InfoSec review at Gates A, C, and D for the work already done.

---

## 7. Common Misclassification Pitfalls

The following are the most frequent misclassification patterns seen in submissions. The Pre-Stage Validator and Gate A reviewer should screen for these.

**"It's just internal."** Internal does not mean Non-Sensitive. The classification axis is sensitivity, not audience. The audience axis is what determines Vibe Coding eligibility in the first place; sensitivity determines Type 1 vs 1B.

**"It's just metadata."** Metadata about commercial activity (who deployed what, who approved what, who accessed what) is often more revealing than the underlying data. Audit trails and access logs default to Restricted.

**"It's just names and emails."** Directory-level identity (name, title, team, business email) is Non-Sensitive. Identity in the context of an action — "Eitan changed vendor X's contract value from $A to $B at time T" — is Restricted because the action is restricted.

**"It's read-only, so it's safe."** Read scope can still expose Restricted data. A read-only dashboard that surfaces every BU's commercial metrics to every employee has the same exposure as an editable one. Classification is about the data, not the operation.

**"It's not regulated, so it's not Restricted."** Many forms of internally sensitive data are not regulated by external frameworks. Vendor commercial terms, internal forecasts, and audit findings are Restricted by internal policy regardless of regulatory status.

**Aggregation effects.** A single deployment record is Non-Sensitive. A complete deployment history with deployer attribution across all production services is closer to Restricted because it surfaces internal operational patterns useful to an attacker. Classify the dataset, not just the field.

---

## 8. Application-Level Implications by Type

Type 1 (Internal · Non-Sensitive) applications must implement:
- Entra ID SSO authentication for all access
- Standard application logging (no PII redaction overhead; no Restricted data flows to redact)
- Standard CI/CD security scanning at Gate C
- Standard architecture review at Gate A by an Architect

Type 1B (Internal · Restricted) applications must additionally implement:
- Server-side authorization enforcement, including for export paths and aggregation views
- BU-scoped access controls where commercial sensitivity warrants
- Append-only audit trail of write operations, retained ≥ 1 year, forwarded to QRadar
- Database role separation such that the application service account cannot modify audit records
- InfoSec review at Gate A (authorization model), Gate C (scan results sign-off), and Gate D (authorization enforcement in code review)
- Phased rollout at Gate E with a smaller initial audience

---

## 9. Exceptions

Exceptions to this standard follow the standard the standard exception process. Note specifically:

- **Downgrade exception** (treating Restricted data as Non-Sensitive in handling): Requires CISO approval and documented compensating controls. Rare and time-bound.
- **Upgrade voluntary** (treating Non-Sensitive data with Restricted controls): Permitted without exception process; no approval required.
- **Out-of-scope data discovered post-launch**: The application is suspended pending re-evaluation. This is not a policy exception — it is a remediation event.

---

## 10. Document History

| Version | Date         | Author          | Changes                                  |
| ------- | ------------ | --------------- | ---------------------------------------- |
| 0.1     | [DATE]       | [Author] | Initial draft for CrewAI POC corpus       |
