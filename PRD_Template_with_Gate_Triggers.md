# Vibe Coding Project Brief Template (PRD)

**Document Type:** Template / Rubric
**Owner:** Security Architecture (in coordination with Vibe Coding governance)
**Audience:** Submitting Product Owners (POs), Pre-Stage Validator agent, downstream gate agents
**Scope:** Vibe Coding Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications only
**Out of Scope:** Type 2 (External · Non-Sensitive), Excluded (External · Restricted) — these route through full R&D process

---

## How to Use This Template

This template is the canonical input format for all Vibe Coding submissions. Every section is mandatory unless explicitly marked optional. The Pre-Stage Validator agent parses each section against the validation rules at the end of this document. Submissions failing any required check are returned to the PO with a structured rejection reason.

This is not a free-form proposal document. POs who want their project approved should treat the section structure as required and the validation rules as binding.

---

## Section 1 — Executive Summary

**Purpose:** Two-paragraph plain-English description of what the application does, who uses it, and the data it touches.

**Required content:**
- One sentence on what the application is
- One sentence on the primary problem it solves
- One paragraph on the data it interacts with, including an explicit statement on PCI scope, customer PII, and regulated data flows ("none" is the expected answer for Type 1 / 1B)

**Validation rule:** Section must explicitly state PCI scope status and PII status. Absence of these statements is a Pre-Stage rejection.

---

## Section 2 — Business Case

**Purpose:** Justification for why this project deserves engineering effort.

**Required content:**
- Quantified problem statement (time lost, cost incurred, risk borne, audit exposure, etc.)
- Baseline measurement (current state, with numbers)
- Why a Vibe Coding intervention is the right shape — versus a full R&D project, a SaaS purchase, or no action

**Validation rule:** Business case must contain at least one quantified baseline metric. "It would be nice to have" is not a business case. Vague benefits ("improve productivity") without numbers fail validation.

**Maps to:** Pre-Stage YES Condition #1 (Business Case)

---

## Section 3 — Target Audience

**Purpose:** Defines the population of users and is the primary input to the Internal-vs-External axis of classification.

**Required content:** A table of audience segments with population estimates and access levels. Mandatory rows for "External users," "Customer-facing," and "Cross-tenant exposure" — even if the answer is "None."

**Validation rules:**
- If "External users" ≠ None → submission is **not eligible for Vibe Coding Type 1 / 1B** and must be rerouted to full R&D process. Pre-Stage rejection with reroute instruction.
- If "Customer-facing" = Yes → same rejection.
- Population estimates must be numeric (not "lots" or "all of engineering").

**Maps to:** Classification axis 1 (Audience: Internal / External)

---

## Section 4 — Data Interaction

**Purpose:** Defines what data flows through the application and is the primary input to the Non-Sensitive-vs-Restricted axis of classification.

**Required content:** A table of data flows with the following columns:
- Direction (READ / WRITE)
- Source or target system
- Data fields touched
- Sensitivity classification per the Data Classification Standard

**Validation rules:**
- Every data flow must have a sensitivity classification — referencing the Data Classification Standard.
- If any flow is classified higher than "Internal · Restricted" (e.g., PCI, customer PII, employee PII at scale) → submission is **not eligible for Vibe Coding** and must be rerouted to full R&D. Pre-Stage rejection.
- If any flow is "Internal · Restricted" → minimum classification is Type 1B. Submission classified as Type 1 with restricted data flows is rejected for inconsistency.
- An explicit statement of "no PCI scope, no customer PII" is required.

**Maps to:** Classification axis 2 (Data: Non-Sensitive / Restricted)

---

## Section 5 — Functional Requirements

**Purpose:** Numbered list of what the application does.

**Required content:**
- Numbered functional requirements
- Explicit "Out of scope for v1" subsection

**Validation rule:** Functional requirements must be specific enough that a developer could begin work. "User-friendly interface" is not a functional requirement; "List view of vendor records sortable by renewal date" is.

---

## Section 6 — Proposed Tech Stack

**Purpose:** Technology choices, validated against the Approved Tooling Catalog.

**Required content:** Table covering frontend, backend, auth, datastore, hosting, CI/CD, observability.

**Validation rules:**
- Every entry must match the Approved Tooling Catalog. Stack components outside the catalog require an exception note in Section 13.
- "TBD" in any required row → Pre-Stage rejection.

---

## Section 7 — Integrations

**Purpose:** External dependencies and authentication patterns.

**Required content:** Table of integrations with direction, auth method, and a "Pre-Cleared in Catalog?" column.

**Validation rule:** Any integration not in the Integration Catalog must be flagged as a risk in Section 13 and triggers an automatic Gate A architecture review item.

---

## Section 8 — KPIs and Success Metrics

**Purpose:** How success will be measured post-launch.

**Required content:** Table with metric name, baseline, target, and measurement window for each KPI.

**Validation rules:**
- Minimum 3 KPIs.
- Every KPI must have a numeric baseline (or "N/A — new capability") and a numeric target.
- Targets without measurement windows fail validation.

**Maps to:** Pre-Stage YES Condition #2 (KPIs)

---

## Section 9 — Budget and Resources

**Purpose:** Effort estimate and funding source.

**Required content:** Table of cost line items + named funding source.

**Validation rules:**
- Funding source must be explicitly named (budget line, cost center, or sponsor).
- "TBD" funding source → Pre-Stage rejection.
- For Type 1B projects, the budget table must include a line item for InfoSec review effort and Architect review effort. Absence flags the project for Pre-Stage clarification.

**Maps to:** Pre-Stage YES Condition #4 (Budget)

---

## Section 10 — Timeline

**Purpose:** Phased plan from Pre-Stage through Go-Live.

**Required content:** Phase table covering all six gates plus development.

**Validation rules:**
- Type 1 timeline must reflect lighter Gate A and Gate D durations.
- Type 1B timeline must include time for InfoSec involvement at Gates A, C, and D.
- Total timeline shorter than 4 weeks → flagged for clarification (likely understates effort).

---

## Section 11 — Pre-Stage YES Conditions Self-Attestation

**Purpose:** Explicit, structured attestation that the four mandatory conditions are met.

**Required content:** A 4-row table with these exact rows:

| Condition       | Status | Evidence |
| --------------- | ------ | -------- |
| Business case   | YES / NO | Reference to Section 2 |
| KPIs defined    | YES / NO | Reference to Section 8 |
| PO readiness    | YES / NO | Named PO, named executive sponsor for Type 1B |
| Budget approved | YES / NO | Reference to Section 9 |

**Validation rules:**
- All four conditions must read YES.
- Any NO → Pre-Stage rejection with reason.
- Type 1B projects without a named executive sponsor → Pre-Stage rejection (Type 1 may proceed with PO only).

**Maps to:** Pre-Stage YES Conditions #1–4 (formal attestation)

---

## Section 12 — Classification Justification

**Purpose:** Explicit reasoning for why the project is Type 1 or Type 1B.

**Required content:**
- Stated classification (Type 1 or Type 1B)
- Audience axis justification (must reference Section 3)
- Data axis justification (must reference Section 4)
- Statement of regulated-data scope ("no PCI, no GDPR-restricted PII, no SOX-material data" — or, if any apply, reroute to full R&D)
- For Type 1B: explicit acknowledgment that InfoSec involvement at Gates A, C, and D is required

**Validation rules:**
- Classification must be consistent with Sections 3 and 4. A Type 1 declaration with restricted data in Section 4 is a logical inconsistency → Pre-Stage rejection.
- A Type 1B declaration with no restricted data and internal-only audience is over-classification — flagged but not rejected (PO may downgrade to Type 1 to save cycle time).

---

## Section 13 — Open Questions / Risks

**Purpose:** Known unknowns and risk surface that downstream gates should attend to.

**Required content:** Bulleted list of risks with proposed mitigations + bulleted list of open questions with target gate for resolution.

**Validation rule:** This section may be empty for very simple Type 1 projects, but absence of any risks for a Type 1B project is itself a flag — most Type 1B projects have at least authorization-model and audit-trail risks worth surfacing.

---

# Appendix A — Pre-Stage Validator Routing Rules

The Pre-Stage Validator agent applies the following decision logic in order:

1. **Eligibility check.** Is the submission a Vibe Coding-eligible project (Section 3 audience = Internal only, Section 4 data ≤ Internal · Restricted)? If no → reroute to full R&D process.
2. **Completeness check.** Are all 13 sections present and non-empty? If no → reject with list of missing sections.
3. **YES conditions check.** Are all 4 attestations YES? If no → reject with reason.
4. **Consistency check.** Is Section 12 classification consistent with Sections 3 and 4? If no → reject with reason.
5. **Type 1B sponsor check.** If classified Type 1B, is an executive sponsor named in Section 11? If no → reject.
6. **Routing.** If all checks pass, route to Gate A with classification flag (Type 1 or Type 1B).

---

# Appendix B — Gate Consumption Map

This table tells each downstream agent which sections of the PRD it consumes as primary input.

| Agent                              | Primary Sections             | Secondary Sections   |
| ---------------------------------- | ---------------------------- | -------------------- |
| Pre-Stage Validator                | 11, 12 (+ all for completeness) | 1–13                 |
| Gate A Architecture Reviewer       | 4, 5, 6, 7, 12, 13           | 3, 9, 10             |
| Gate B Automated Testing Controller | 5, 6                         | 4, 13                |
| Gate C Security Gate Enforcer      | 4, 6, 7, 12, 13              | 3, 5                 |
| Gate D Code Review Orchestrator    | 5, 6, 12, 13                 | 4, 7                 |
| Gate E Go-Live Coordinator         | 8, 10, 13                    | 3, 9, 12             |

---

# Appendix C — Common Pre-Stage Rejection Reasons

The following are the most frequent reasons Pre-Stage submissions are returned to the PO. The Validator should produce one of these reasons (or a structured combination) on rejection rather than free-text feedback.

1. **Section 2 lacks a quantified baseline.** Business case is qualitative only.
2. **Section 3 indicates external users.** Project is not Vibe Coding eligible.
3. **Section 4 contains a data flow that exceeds Internal · Restricted.** Project is not Vibe Coding eligible.
4. **Section 4 contains restricted data but Section 12 declares Type 1.** Classification inconsistency.
5. **Section 6 references tooling outside the Approved Tooling Catalog without exception note.** Stack out of bounds.
6. **Section 8 has fewer than 3 KPIs, or KPIs lack baselines/targets.** YES Condition #2 fails.
7. **Section 9 funding source is "TBD."** YES Condition #4 fails.
8. **Section 11 contains a NO attestation.** YES Condition fails by self-declaration.
9. **Type 1B project missing executive sponsor in Section 11.** PO readiness incomplete.

---

# Appendix D — Document History

| Version | Date         | Author        | Changes                                          |
| ------- | ------------ | ------------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author] | Initial draft for CrewAI POC corpus              |
