# Vibe Coding POC Corpus — README and Agent Assignment Index

**Purpose of this document:** Orient anyone loading this corpus into a multi-agent system (e.g., CrewAI) for evaluation. Defines what each document is, which agent consumes it, and how the documents relate to each other.

**Read this first.** Every other document assumes the reader already understands the gate structure and Type 1 / Type 1B classification. This document supplies that framing.

---

## 1. What This Corpus Is

The corpus models a six-gate governance pipeline for an internal application development program ("Vibe Coding"), where non-technical Product Owners build internal applications with AI-assisted development. The pipeline routes submissions through:

```
Pre-Stage  →  Gate A  →  Gate B & C (parallel)  →  Gate D  →  Gate E
Validator     Arch.        Testing  Security        Code Rev   Go-Live
```

Each gate is enforced by a specialized agent. The corpus provides each agent with the rubrics it needs to evaluate submissions deterministically, plus test fixtures so agent behavior can be observed and compared against expected outcomes.

The corpus is **scope-limited** to two application classifications:

- **Type 1** — Internal · Non-Sensitive (lighter governance, no InfoSec involvement)
- **Type 1B** — Internal · Restricted (heavier governance, InfoSec involvement at Gates A, C, D)

External-facing applications (Type 2) and excluded categories are explicitly out of scope.

---

## 2. Document Inventory

| # | Filename                                                | Role                              | Type           |
| - | ------------------------------------------------------- | --------------------------------- | -------------- |
| 1 | `PRD_Template_with_Gate_Triggers.md`                    | Submission structure rubric       | Rubric         |
| 2 | `Data_Classification_Standard_Vibe_Coding.md`           | Data classification rubric        | Rubric         |
| 3 | `Approved_Tooling_Catalog_Vibe_Coding.md`               | Approved stack rubric             | Rubric         |
| 4 | `Approved_Architecture_Patterns_Library.md`             | Approved architecture rubric      | Rubric         |
| 5 | `Test_Coverage_and_Quality_Gates_Standard.md`           | Test pass/fail rubric             | Rubric         |
| 6 | `Security_Controls_Baseline_and_Severity_Matrix.md`     | Security scan and severity rubric | Rubric         |
| 7 | `Code_Review_Checklist_and_Reviewer_Matrix.md`          | Code review rubric                | Rubric         |
| 8 | `Project_A_DeployBoard_Brief.md`                        | Type 1 submission                 | Test Fixture   |
| 9 | `Project_B_VendorTrack_Brief.md`                        | Type 1B submission                | Test Fixture   |
| 10 | `Example_PRD_DeployBoard_Type1.md`                     | Canonical passing PRD             | Reference      |

**Rubric documents** define decision logic — the agents validate submissions *against* them.

**Test fixtures** are sample submissions the pipeline processes. Every gate agent receives these as input.

**Reference example** is a gold-standard "passing" PRD that conforms to the template. The Pre-Stage Validator can use it for calibration; other agents can consult it as a positive example.

---

## 3. Agent Assignment Matrix

For each of the six agents in the orchestration, the table below lists primary inputs (must be loaded into agent context every invocation) and secondary inputs (loaded once for orientation, or consulted when a specific question arises).

### 3.1 Agent 1 — Pre-Stage Requirements Validator

**Role:** Validates that submissions meet the four YES conditions and are eligible for Vibe Coding scope. Routes valid submissions to Gate A; rejects invalid ones with structured reasons.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Primary inputs:**                                   |                                                                |
| `PRD_Template_with_Gate_Triggers.md`                  | Defines the structure to validate, the 4 YES conditions, and the rejection-reason taxonomy |
| `Data_Classification_Standard_Vibe_Coding.md`         | Validates classification declarations in PRD Section 12 against data flows in PRD Section 4 |
| `Approved_Tooling_Catalog_Vibe_Coding.md`             | Validates PRD Section 6 stack against the approved/conditional/prohibited tiers |
| **Secondary inputs:**                                 |                                                                |
| `Example_PRD_DeployBoard_Type1.md`                    | Calibration reference — what a clean passing submission looks like |

### 3.2 Agent 2 — Architecture Gate Reviewer (Gate A)

**Role:** Reviews proposed architecture against approved patterns. Coordinates InfoSec involvement for Type 1B. Produces architecture review outcome and any required ADRs.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Primary inputs:**                                   |                                                                |
| `Approved_Architecture_Patterns_Library.md`           | The 9 approved patterns with components, embedded controls, and common mistakes |
| `Approved_Tooling_Catalog_Vibe_Coding.md`             | Stack and integration validation                               |
| `Data_Classification_Standard_Vibe_Coding.md`         | Classification context drives Type 1 vs 1B reviewer assignment |
| **Secondary inputs:**                                 |                                                                |
| `PRD_Template_with_Gate_Triggers.md`                  | Understands which PRD sections to read (Sections 4, 6, 7, 12, 13 per the gate consumption map) |

### 3.3 Agent 3 — Automated Testing Gate Controller (Gate B)

**Role:** Runs the test suite, evaluates coverage, attempts auto-remediation within strict bounds, and produces a structured pass/fail outcome.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Primary inputs:**                                   |                                                                |
| `Test_Coverage_and_Quality_Gates_Standard.md`         | All Gate B logic: required test categories, thresholds, auto-remediation boundaries, escalation triggers |
| `Data_Classification_Standard_Vibe_Coding.md`         | Determines which test categories and coverage thresholds apply (Type 1 vs 1B) |
| **Secondary inputs:**                                 |                                                                |
| `PRD_Template_with_Gate_Triggers.md`                  | Understands project context for routing decisions              |

### 3.4 Agent 4 — Security Gate Enforcer (Gate C)

**Role:** Runs required security scans, evaluates findings against the severity matrix, applies bounded auto-remediation, and decides pass/fail. Coordinates InfoSec sign-off for Type 1B.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Primary inputs:**                                   |                                                                |
| `Security_Controls_Baseline_and_Severity_Matrix.md`   | All Gate C logic: required scans, approved tools, severity classification, pass/fail thresholds, auto-remediation boundaries, license policy, SBOM requirements |
| `Approved_Tooling_Catalog_Vibe_Coding.md`             | Cross-reference for license tier definitions and approved scanning tools |
| `Data_Classification_Standard_Vibe_Coding.md`         | Determines Type 1 vs 1B differential at Gate C (InfoSec sign-off requirement) |

### 3.5 Agent 5 — Code Review Orchestrator (Gate D)

**Role:** Routes the code review to the right reviewer set based on classification, manages SLAs and escalations, validates auto-remediations from Gates B and C, produces final review outcome.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Primary inputs:**                                   |                                                                |
| `Code_Review_Checklist_and_Reviewer_Matrix.md`        | Reviewer assignment by classification, universal checklist, security review checklist, Type 1B InfoSec checklist, sign-off rules, SLA enforcement |
| `Approved_Architecture_Patterns_Library.md`           | Reasoned-backward verification — does the implementation actually realize the pattern declared at Gate A? |
| `Data_Classification_Standard_Vibe_Coding.md`         | Determines reviewer set (Type 1 = peer; Type 1B = peer + Architect + InfoSec) |
| **Secondary inputs:**                                 |                                                                |
| `Security_Controls_Baseline_and_Severity_Matrix.md`   | Context for evaluating Gate C auto-remediations under InfoSec review |
| `Test_Coverage_and_Quality_Gates_Standard.md`         | Context for evaluating Gate B auto-remediations and warnings |

### 3.6 Agent 6 — Go-Live Deployment Coordinator (Gate E)

**Role:** Coordinates production deployment with stakeholders, validates the signed-off commit matches what is being deployed, executes phased rollout for Type 1B.

**Note:** A dedicated rubric for Gate E (Production Readiness Checklist) was deferred from the POC corpus. The agent operates on context already produced by upstream gates.

| Document                                              | Role                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| **Secondary inputs:**                                 |                                                                |
| `Code_Review_Checklist_and_Reviewer_Matrix.md`        | Defines Gate E's commit-hash verification expectation (Section 9 of that doc) |
| `Data_Classification_Standard_Vibe_Coding.md`         | Determines deployment cadence (Type 1 = single rollout; Type 1B = phased) |

---

## 4. Test Fixtures (Pipeline Inputs)

The two project briefs are sample submissions that flow through the entire pipeline. Every gate agent receives them as input, evaluates them under its rubrics, and produces output for the next gate.

### 4.1 Project A — DeployBoard

- **Classification:** Type 1 (Internal · Non-Sensitive)
- **Architecture pattern:** Stateless Read-Only Aggregator (Library Pattern 7)
- **Expected pipeline behavior:**
  - Pre-Stage: Pass cleanly (all 4 YES conditions met, classification consistent)
  - Gate A: Architect-only review, fast pass
  - Gate B: Standard coverage threshold (≥70%), no audit/auth tests required
  - Gate C: Standard scans, no InfoSec sign-off needed
  - Gate D: Peer review only
  - Gate E: Single-phase rollout

### 4.2 Project B — VendorTrack

- **Classification:** Type 1B (Internal · Restricted)
- **Architecture patterns:** CRUD with RBAC and Audit Trail (Library Pattern 8) + Append-Only Audit Trail (Pattern 9) + M365 Graph Integration (Pattern 11) + Background Jobs (Pattern 12)
- **Expected pipeline behavior:**
  - Pre-Stage: Pass cleanly, with executive sponsor named
  - Gate A: Architect + InfoSec review; particular focus on RBAC enforcement and audit trail design
  - Gate B: Elevated coverage threshold (≥80% changed code, ≥95% on auth/audit code paths); authorization tests and audit trail tests required
  - Gate C: Standard scans + mandatory InfoSec sign-off even on clean results
  - Gate D: Peer + Architect + InfoSec review; Section 5 of Code Review Checklist applies
  - Gate E: Phased rollout

### 4.3 Reference Example — Example PRD (DeployBoard)

A template-conformant PRD using DeployBoard's content. Slightly tighter than the Project A Brief in form: every required field is explicitly populated, all 9 Pre-Stage validation rules satisfied. Used as a calibration reference, not as a test fixture for the pipeline.

---

## 5. Type 1 vs Type 1B Differential

Where the differential manifests across the corpus:

| Gate | Differential                                                                                       | Encoded In                              |
| ---- | -------------------------------------------------------------------------------------------------- | --------------------------------------- |
| Pre-Stage | Type 1B requires named executive sponsor                                                       | PRD Template Section 11                 |
| Gate A | Type 1B adds InfoSec to architecture review                                                        | Architecture Patterns Library + Code Review Matrix |
| Gate B | Type 1B requires auth tests, audit tests, e2e tests; coverage thresholds elevated                  | Test Coverage and Quality Gates Standard Sections 2-3 |
| Gate C | Type 1B requires InfoSec sign-off even on clean scans; InfoSec validates auto-remediations         | Security Controls Baseline Section 8    |
| Gate D | Type 1B requires peer + Architect + InfoSec sign-off; Section 5 InfoSec checklist applies          | Code Review Checklist Sections 2 and 5  |
| Gate E | Type 1B uses phased rollout                                                                        | (Gate E rubric deferred)                |

This differential is the primary thing the POC should test. If both Project A and Project B receive identical pipeline behavior, the orchestration has failed regardless of whether it produced "passing" outcomes.

---

## 6. Suggested POC Test Scenarios

The corpus supports several test scenarios beyond the two happy-path fixture runs:

| Scenario                                                            | Purpose                                                       |
| ------------------------------------------------------------------- | ------------------------------------------------------------- |
| Project A through full pipeline                                     | Baseline — does the orchestration mechanically work for Type 1? |
| Project B through full pipeline                                     | Differential — does InfoSec actually get pulled in at Gates A, C, D? |
| Project A with classification artificially upgraded to Type 1B      | Does the orchestration adapt when classification changes?     |
| Project B with classification artificially downgraded to Type 1     | Does Pre-Stage Validator catch the inconsistency between data flows and declared classification? |
| Project A with TBD funding source                                   | Does Pre-Stage Validator reject with a structured reason citing Rule 7? |
| Project A with deliberately introduced critical SAST finding        | Does Gate C block correctly? Does it produce a useful exception path output? |
| Project B with audit trail tests intentionally weak                 | Does Gate D Section 5.2 catch the gap, or does it pass through? |
| Project B where Architect signs Pass but InfoSec signs Block        | Does Gate D correctly halt despite Architect approval?        |

The first two scenarios validate that the pipeline runs at all. The remainder validate specific decision branches the orchestration must handle. A POC that only runs the first two has not really tested the agents.

---

## 7. Known Gaps and Out of Scope

The following were intentionally not included in this corpus:

- **Production Readiness Checklist (Gate E rubric)** — deferred. Gate E operates on outputs from upstream gates without its own dedicated rubric.
- **Adversarial test fixtures** — no PRD intentionally crafted to fail Pre-Stage. Suggested test scenarios above describe how to construct these by modifying existing fixtures.
- **External-facing application support (Type 2)** — out of scope. The Vibe Coding program in this scoping handles internal applications only.
- **PCI / regulated-data flows** — out of scope. The corpus assumes no regulated data is processed.
- **ADR Template** — referenced by Gates A and B but not included as a separate document. ADRs in the POC may be free-form.
- **Specific scanning tool configurations** — the Security Controls Baseline names approved tools but does not include per-tool configuration files.

---

## 8. Document Relationships

```
                  PRD Template
                       |
                       v
          Data Classification Standard
                       |
       +---------------+---------------+
       v               v               v
  Tooling Catalog  Patterns Library  Test Coverage Std
       |               |                    |
       v               v                    v
  Pre-Stage --------> Gate A <---+      Gate B
                                 |
                       Security Baseline ---> Gate C
                                 |
                                 v
                            Code Review Std ---> Gate D
                                                  |
                                                  v
                                              Gate E (no rubric)
```

The PRD Template and Data Classification Standard are upstream of every other rubric — they define the inputs and the classification axis that all subsequent gates branch on. The Tooling Catalog and Architecture Patterns Library are paired (catalog says what's allowed; patterns say how to combine). The Code Review rubric depends on the Architecture Patterns Library because Gate D verifies that implementation actually realizes the pattern declared at Gate A.

---

## 9. Loading Order (Suggested)

When initially populating the multi-agent system, load documents in this order:

1. This README
2. `PRD_Template_with_Gate_Triggers.md`
3. `Data_Classification_Standard_Vibe_Coding.md`
4. `Approved_Tooling_Catalog_Vibe_Coding.md`
5. `Approved_Architecture_Patterns_Library.md`
6. `Test_Coverage_and_Quality_Gates_Standard.md`
7. `Security_Controls_Baseline_and_Severity_Matrix.md`
8. `Code_Review_Checklist_and_Reviewer_Matrix.md`
9. `Example_PRD_DeployBoard_Type1.md` (reference)
10. `Project_A_DeployBoard_Brief.md` (test fixture)
11. `Project_B_VendorTrack_Brief.md` (test fixture)

Loading the rubrics before the fixtures matters: each agent should have its decision rules in context before it processes its first submission.

---

## 10. Document History

| Version | Date         | Author     | Changes                                          |
| ------- | ------------ | ---------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author]   | Initial corpus README for CrewAI POC             |
