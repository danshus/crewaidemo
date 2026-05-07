# Security Controls Baseline and Vulnerability Severity Matrix

**Document Owner:** Security Architecture
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Gate C of the Vibe Coding pipeline for Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications
**Out of Scope:** External-facing applications, PCI-scoped workloads, regulated data flows. These are governed by the enterprise application security baseline and full-R&D security review process.

---

## 1. Purpose

This document defines what security controls are evaluated at Gate C, what scanning tools produce those evaluations, and what pass/fail policy is applied to the findings. It exists in two parts because the Gate C Security Gate Enforcer agent needs both: a list of required scans (the Baseline) and a decision rule for evaluating their output (the Matrix). Either part alone is incomplete — the Baseline without the Matrix is observation without judgment; the Matrix without the Baseline has nothing to judge.

The Gate C agent applies this document mechanically. Pass/fail decisions are deterministic given the inputs; human judgment enters only at the exception path (Section 9) and at the Type 1B InfoSec sign-off (Section 8).

---

## 2. Required Security Scans

The following scan types are mandatory at Gate C for all Vibe Coding applications. Absence of any required scan is a Gate C blocker independent of findings — a clean SAST report from a tool that does not actually scan the language is not a passing scan.

| # | Scan Type                          | Required For       | Purpose                                                                 |
| - | ---------------------------------- | ------------------ | ----------------------------------------------------------------------- |
| 1 | Static Application Security Testing (SAST) | All apps   | Detects vulnerable patterns in first-party code (injection, deserialization, weak crypto, etc.) |
| 2 | Software Composition Analysis (SCA) | All apps          | Detects known vulnerabilities in third-party dependencies               |
| 3 | Secrets Detection                  | All apps           | Detects hardcoded credentials, API keys, tokens in source and config    |
| 4 | License Compliance                 | All apps           | Detects incompatible or prohibited open-source licenses                 |
| 5 | Container Image Scanning           | All containerized apps | Detects vulnerabilities in base images and installed packages       |
| 6 | Infrastructure-as-Code (IaC) Scanning | Apps with IaC artifacts | Detects misconfigurations in Terraform, Helm, Kubernetes manifests |
| 7 | SBOM Generation                    | All apps           | Produces a Software Bill of Materials for the build artifact            |

**Out of scope at Gate C for Vibe Coding apps:**
- Dynamic Application Security Testing (DAST) — not applicable to internal-only apps with no external attack surface
- Penetration testing — reserved for higher-risk application categories outside Vibe Coding
- Manual code review of scan results — that is Gate D's role, not Gate C's

---

## 3. Approved Scanning Tools

The following tools are approved for each scan type. Tooling decisions are operationally owned by DevOps in coordination with Security Architecture. The Gate C agent expects scan output in one of these tool's standard formats; output from non-approved tools is not evaluated.

| Scan Type            | Approved Tools (one or more)                                            |
| -------------------- | ----------------------------------------------------------------------- |
| SAST                 | Semgrep, SonarQube, Checkmarx, Veracode, GitHub CodeQL                  |
| SCA                  | Snyk, Mend, OWASP Dependency-Check, GitHub Dependabot                   |
| Secrets Detection    | GitGuardian, TruffleHog, Gitleaks, GitHub Secret Scanning               |
| License Compliance   | Snyk License, FOSSA, Mend License (typically bundled with SCA tooling)  |
| Container Scanning   | Trivy, Snyk Container, Wiz Container Scanning, Aqua                     |
| IaC Scanning         | Checkov, tfsec, Snyk IaC, KICS                                          |
| SBOM Generation      | Syft, CycloneDX-CLI, SPDX tooling                                       |

A project must use at least one tool per applicable scan type. Multiple tools per type are permitted; results are unioned for pass/fail evaluation.

---

## 4. Vulnerability Severity Matrix

Severity is determined first by CVSS v3.1 or v4.0 base score, then adjusted for exploitability and context per Section 5.

| Severity  | CVSS Range  | Default Pass/Fail Behavior                       |
| --------- | ----------- | ------------------------------------------------ |
| Critical  | 9.0 – 10.0  | **Always blocking.** Zero tolerance.             |
| High      | 7.0 – 8.9   | **Always blocking.** Zero tolerance.             |
| Medium    | 4.0 – 6.9   | Tracked, not blocking. Must be remediated within SLA. |
| Low       | 0.1 – 3.9   | Informational. Logged, not tracked.              |
| Info      | 0.0         | Informational. Not logged.                       |

Findings without a CVSS score (typical for SAST findings, secrets, license issues) follow the per-scan-type severity rules in Section 6 below.

---

## 5. Exploitability and Context Adjustments

The default CVSS-based severity is adjusted up (never down via this section) when one of the following applies:

| Adjustment Trigger                                                  | Effect                                |
| ------------------------------------------------------------------- | ------------------------------------- |
| Finding is in the CISA Known Exploited Vulnerabilities (KEV) catalog | Severity bumped up one tier (min: High) |
| Public exploit available (Metasploit module, public PoC, exploit-db entry) | Severity bumped up one tier      |
| Vulnerable code path is reachable from an authenticated user (per SAST reachability analysis where supported) | Severity bumped up one tier |
| Finding is in a Type 1B-classified application                      | High and Critical findings receive InfoSec review even if auto-remediated |

**Adjustments cannot reduce severity.** A finding with no fix available, or one in a transitively imported dependency, remains at its CVSS-derived severity. Mitigating context (e.g., "the vulnerable function is not called") is handled through the exception process in Section 9, not by adjusting severity.

---

## 6. Per-Scan-Type Severity Rules

Findings without CVSS scores are classified by scan-type-specific rules.

### 6.1 SAST Findings

| Pattern                                                              | Severity |
| -------------------------------------------------------------------- | -------- |
| SQL injection, command injection, path traversal, deserialization of untrusted input | Critical |
| Cross-site scripting (XSS), server-side request forgery (SSRF), XXE  | High     |
| Hardcoded credentials, weak cryptography (MD5, SHA1, ECB mode), insecure random | High |
| Open redirect, missing authorization check on sensitive endpoint     | High     |
| Information disclosure (stack trace exposure, verbose errors)        | Medium   |
| Code quality issues with security implications (TODO/FIXME on security-relevant code) | Low |

### 6.2 Secrets Detection Findings

All confirmed secret detections are **Critical** regardless of secret type. There are no medium or low secrets findings — a leaked credential is binary. False-positive suppression is the only path; no severity-based deferral applies.

### 6.3 License Compliance Findings

| License Tier                                                         | Severity / Action |
| -------------------------------------------------------------------- | ----------------- |
| Strong copyleft (GPL-3.0, AGPL-3.0) in non-GPL distribution          | Critical (block)  |
| Weak copyleft (LGPL, MPL) without compliance review                  | High (block until reviewed) |
| Permissive (MIT, Apache-2.0, BSD)                                    | Pass              |
| Unknown / unidentified license                                       | High (block until identified) |
| License explicitly on the prohibited list                            | Critical (block)  |

### 6.4 Container Image Findings

CVE-based findings follow the standard CVSS matrix. Additionally:

| Pattern                                                              | Severity |
| -------------------------------------------------------------------- | -------- |
| Image runs as root (UID 0)                                           | High     |
| Base image not from approved registry                                | High     |
| Image uses floating tag (`latest`, `stable`) in production manifest  | High     |
| Image has no defined `USER` directive                                | Medium   |
| Image lacks healthcheck                                              | Low      |

### 6.5 IaC Findings

CVSS does not apply. Findings follow tool-specific severity (Checkov, tfsec, etc.) with the following adjustments for Vibe Coding context:

| Misconfiguration                                                     | Severity |
| -------------------------------------------------------------------- | -------- |
| Public-facing resource (S3 bucket, security group, load balancer)    | Critical (Vibe Coding apps are internal-only — public exposure is out of scope) |
| Unencrypted data at rest                                             | High     |
| Missing logging or audit configuration                               | High     |
| Overly permissive IAM policy (`*` action or resource)                | High     |
| Missing tagging required by enterprise policy                        | Low      |

---

## 7. Auto-Remediation Boundaries

The Gate C agent supports limited auto-remediation. Auto-remediation runs before pass/fail evaluation; if a finding is auto-fixed cleanly, it does not contribute to the pass/fail count. The agent attempts a maximum of **two auto-remediation loops** per scan run; further failures escalate to human review.

| Finding Type                       | Auto-Remediation Allowed? | Method                                          |
| ---------------------------------- | ------------------------- | ----------------------------------------------- |
| SCA — vulnerable dependency, patch available in same major version | Yes | Bump dependency version, re-run scan          |
| SCA — vulnerable dependency, patch only in next major version      | No  | Major version bumps require human review (breaking change risk) |
| Secrets detection                  | **Never**                 | Secrets must be rotated and removed by a human; auto-removal would mask the leak |
| SAST — auto-fixable patterns where the tool produces a confident fix (e.g., adding parameterization to SQL) | Yes (with caveats) | Auto-fix applied, **PR flagged for mandatory human review at Gate D** even if scan passes |
| SAST — pattern requiring business logic understanding | No | Always escalate                       |
| License compliance                 | No                        | License changes require review                 |
| Container — base image update with patches | Yes                | Bump base image to current patched version     |
| Container — runs-as-root remediation | No                      | Application impact requires human review       |
| IaC — additive misconfiguration fixes (add encryption, add logging) | Yes | Apply fix, re-run scan                          |
| IaC — restrictive changes (remove public access, tighten IAM) | No | Human must validate intentionality              |

**Hard prohibitions on auto-remediation:**
- Auto-remediation may not modify business logic.
- Auto-remediation may not suppress, mute, or downgrade findings.
- Auto-remediation may not disable security controls or scans.
- Auto-remediation may not commit changes that fail tests (Gate B must still pass after remediation).

---

## 8. Type 1 vs Type 1B Differential at Gate C

The scans run and severity matrix are identical for both classifications. The differential is in the human-review layer above the automated pass/fail.

| Step                                  | Type 1                              | Type 1B                                           |
| ------------------------------------- | ----------------------------------- | ------------------------------------------------- |
| All required scans run                | Required                            | Required                                          |
| Auto-remediation attempts             | Up to 2 loops                       | Up to 2 loops                                     |
| Pass/fail determination               | Automated, by this matrix           | Automated, by this matrix                         |
| InfoSec review of scan summary        | Not required                        | **Required** — even if all scans pass clean       |
| InfoSec sign-off on findings          | Not required                        | **Required before Gate C closes**                 |
| InfoSec review of any auto-remediation applied | Not required               | **Required** — InfoSec validates auto-fixes were appropriate |
| Exception requests (Section 9)        | Architect-approved                  | Architect + InfoSec-approved                      |

A Type 1B project that passes all scans cleanly still cannot exit Gate C without InfoSec sign-off. The sign-off may be quick, but it is mandatory and must be recorded.

---

## 9. Exception Process

Findings that cannot be remediated within Gate C may be granted an exception and the project allowed to proceed. Exceptions are time-bound, documented, and tracked. They are not a routine path — the default is "fix it."

Exception requests must contain:
1. **Finding identifier** — CVE, rule ID, or tool-specific finding ID
2. **Severity** — as determined by Section 4 + Section 5
3. **Technical justification** — why the finding cannot be remediated within Gate C scope (e.g., upstream fix not available, fix breaks API compatibility, false positive verified)
4. **Compensating controls** — what mitigates the residual risk (network restriction, monitoring rule, logical impossibility of exploitation)
5. **Exception duration** — maximum 90 days for Critical / High; 180 days for Medium; 365 days for Low. Renewals require fresh review.
6. **Approval** — Architect for Type 1; Architect and InfoSec for Type 1B
7. **Tracking ID** — registered in the security exception tracker

Exception requests received by the Gate C agent are routed to the appropriate approver based on classification. The agent does not approve exceptions itself; it routes and waits.

**Exceptions never apply to:**
- Hardcoded secrets — always remediate, no exception path
- Strong copyleft license violations in non-GPL distributions — must be remediated or replaced
- Public-facing IaC misconfigurations in Vibe Coding apps — Vibe Coding is internal-only; an app that genuinely needs public exposure is no longer Vibe Coding

---

## 10. Findings Routing

The Gate C agent produces a structured output for downstream consumption regardless of pass/fail outcome.

| Output Section          | Audience                     | Format                                                      |
| ----------------------- | ---------------------------- | ----------------------------------------------------------- |
| Pass/fail summary       | Pre-Stage Validator (for return-to-PO if blocked), Gate D, Gate E | Boolean per scan type + overall                |
| Findings detail         | Gate D Code Review Orchestrator | Per-finding: ID, severity, location, suggested fix     |
| Auto-remediation report | Gate D                       | List of fixes applied with file/line references            |
| Exception register entries | Security exception tracker | Per-exception: full justification, expiry date, approver  |
| SBOM                    | Gate E (deployment), audit trail | Standard SBOM format (CycloneDX or SPDX)              |
| InfoSec review request (Type 1B) | InfoSec on-call queue | Scan summary + findings detail                       |

---

## 11. License Policy

The following license tiers apply universally to Vibe Coding applications.

**Permitted (no review required):**
- MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, Zlib, Unlicense

**Permitted with review (License Compliance review at Gate C):**
- LGPL-2.1, LGPL-3.0, MPL-2.0, EPL-2.0, CDDL-1.0, Apache-1.1

**Prohibited:**
- GPL-2.0, GPL-3.0, AGPL-3.0 (all variants)
- SSPL, BUSL (Business Source License) for production use
- "Custom" or unidentified licenses
- Licenses on the organization's explicit prohibited list (maintained separately by Legal in coordination with Security Architecture)

**Special handling:**
- Cryptographic libraries — even under permitted licenses, require Security Architecture review at Gate A before adoption
- Dual-licensed dependencies — the more restrictive license applies for Gate C purposes

---

## 12. SBOM Requirements

Every Vibe Coding application produces a Software Bill of Materials at build time. The SBOM is generated by the build pipeline and forwarded to the Gate C agent and the deployment artifact store.

| Requirement                        | Detail                                                  |
| ---------------------------------- | ------------------------------------------------------- |
| Format                             | CycloneDX 1.5+ or SPDX 2.3+                             |
| Coverage                           | All direct and transitive dependencies                  |
| Generation timing                  | Build phase, before container image construction        |
| Storage                            | Attached to deployment artifact; retained ≥ 1 year      |
| Type 1B additional requirement     | SBOM forwarded to security log aggregation for audit trail |

The SBOM is consumed by the Vulnerability Management process post-deployment to enable rapid impact assessment when new CVEs are disclosed against components in active use.

---

## 13. Document History

| Version | Date         | Author     | Changes                                          |
| ------- | ------------ | ---------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author]   | Initial draft for CrewAI POC corpus              |
