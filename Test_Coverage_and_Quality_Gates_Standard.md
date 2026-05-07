# Test Coverage and Quality Gates Standard

**Document Owner:** Engineering Platform (in coordination with Security Architecture)
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Gate B of the Vibe Coding pipeline for Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications

---

## 1. Purpose

This document is consumed by the Gate B Automated Testing Controller agent. It defines what tests must run, what counts as passing, what auto-remediation the agent may apply when tests fail, and what counts as suppressing failures rather than fixing them.

Gate B is where automation has the most autonomy in the pipeline. The agent runs tests, evaluates results, and may modify code to fix failures — within strict boundaries. The boundaries exist because the easiest way to make a failing test pass is to break the test rather than fix the code, and the agent must not be allowed to take that path. Equally, the easiest way to satisfy a coverage threshold is to write tests that execute code without asserting anything meaningful, and the agent must not be allowed to count those as coverage.

The standard accordingly has three parts: what to measure (Sections 3-5), what counts as a meaningful measurement (Sections 6 and 8), and what to do when the measurement is failing (Section 7).

---

## 2. Required Test Categories

The required test categories depend on the application's classification. The Gate B agent determines applicability from the project's classification (Type 1 or Type 1B, per the Data Classification Standard) and the architectural patterns identified at Gate A.

| Test Category                  | Type 1   | Type 1B   | Notes                                          |
| ------------------------------ | -------- | --------- | ---------------------------------------------- |
| Unit tests                     | Required | Required  | Cover business logic, data transformations, validation |
| Integration tests              | Required when external dependencies exist | Required | Cover database access, upstream API calls, message handling |
| End-to-end tests               | Optional | Required  | Cover at least the primary happy-path user flows |
| Authorization tests            | Optional | **Required** | Explicit tests verifying scope boundaries between users/groups |
| Audit trail tests              | N/A      | **Required** | Explicit tests verifying audit entries are written for write operations |
| Build and packaging            | Required | Required  | Build must produce deployable artifact          |
| Linting                        | Required | Required  | Per language/framework standard                 |
| Type checking                  | Required | Required  | Where the language supports static typing       |
| Dependency installation        | Required | Required  | Clean install must succeed in CI environment    |

**Authorization tests** for Type 1B must specifically verify that a user with scope X cannot read or modify scope Y's data via any path: direct ID lookup, search, list, export, or aggregation. Authorization tests that only verify "user without role gets 401" are insufficient for Type 1B; they must also verify scope boundary enforcement when both users have valid sessions.

**Audit trail tests** for Type 1B must verify that every write to Restricted data produces an audit entry with the actor, timestamp, before-value, and after-value populated correctly. Tests that only verify "an audit entry exists" without checking the content are insufficient.

---

## 3. Coverage Thresholds

Coverage is measured at the line level on the changed code in the current pull request, not against the codebase as a whole. The agent's threshold check applies to changed lines because applying a global threshold to a low-coverage legacy codebase punishes new code for old coverage gaps.

| Metric                             | Type 1 Threshold | Type 1B Threshold |
| ---------------------------------- | ---------------- | ----------------- |
| Line coverage on changed code      | ≥ 70%            | ≥ 80%             |
| Line coverage on authorization-related code | N/A     | ≥ 95%             |
| Line coverage on audit-trail-related code   | N/A     | ≥ 95%             |
| Branch coverage on changed code    | Tracked, not gating | ≥ 70%          |

"Authorization-related code" and "audit-trail-related code" are identified by file path conventions or explicit annotations declared in the project's CI configuration. The agent does not infer authorization-relevance from naming alone; the project must declare which paths fall in this category. Misclassification (e.g., declaring all code as non-authorization to dodge the higher threshold) is a Gate D code review concern, not a Gate B blocker.

**Coverage exclusions.** The following are excluded from coverage calculation: auto-generated code, vendored dependencies, framework boilerplate (e.g., scaffolded Program.cs entries), and code paths gated behind explicit feature flags that are off in the current environment. Exclusions must be declared in CI configuration; ad-hoc exclusion via comment annotations is not permitted.

---

## 4. Quality Gates Beyond Coverage

Coverage is necessary but not sufficient. The following gates also apply.

| Gate                              | Type 1 | Type 1B | Failure Mode                                       |
| --------------------------------- | ------ | ------- | -------------------------------------------------- |
| All required tests pass           | Block  | Block   | Any test failure that auto-remediation cannot resolve |
| Build produces a deployable artifact | Block | Block  | Build error                                        |
| Linting passes                    | Warn   | Block   | New lint violations introduced                     |
| Type checking passes              | Block  | Block   | New type errors introduced                         |
| No new compiler warnings          | Warn   | Warn    | New warnings introduced (above existing baseline)  |
| No skipped tests without annotation | Warn  | Block   | `[Skip]`, `it.skip`, `@Disabled` etc. without justification comment and tracking ticket |
| Test execution time per category  | Warn   | Warn    | Unit test suite > 5 minutes; e2e suite > 20 minutes |
| Flaky test rate                   | Warn   | Warn    | More than 5% of tests fail intermittently across the last 10 runs |

"Block" means the agent fails Gate B and routes back to development. "Warn" means the agent records the issue but allows progression. Warns accumulate in the Gate B output for downstream visibility (Gate D code review will see them).

---

## 5. Test Quality Heuristics

A test that runs without asserting anything counts toward coverage but is meaningless. The agent applies the following heuristics to flag low-quality tests. These do not block Gate B (the agent cannot reliably distinguish "low quality test" from "intentional minimal test"), but they accumulate as warnings for Gate D review.

| Heuristic                                                      | Signal                                  |
| -------------------------------------------------------------- | --------------------------------------- |
| Test contains no assertion statements                          | Low-quality test                        |
| Test only asserts on values it set itself in the same function | Tautological test                       |
| Test name does not describe the behavior under test            | Likely coverage-targeting               |
| Test is a copy-paste of another test with only literal values changed | Non-orthogonal test                |
| Test mocks all dependencies including the unit under test      | Tests the mock, not the code            |
| Test asserts only that a function "did not throw"              | Insufficient unless throwing is the entire contract |

These heuristics are advisory. The agent surfaces them; the human reviewer at Gate D decides whether to act on them. Patterns of low-quality tests across a project are themselves a Gate D escalation trigger.

---

## 6. Auto-Remediation Boundaries

The agent attempts auto-remediation when tests fail. The maximum is **two auto-remediation loops** per Gate B run; exceeding two without resolution escalates to human review without a Gate B pass.

### 6.1 Permitted Auto-Remediation

| Failure Pattern                                                | Permitted Action                              |
| -------------------------------------------------------------- | --------------------------------------------- |
| Missing import / unresolved symbol in new test                 | Add the missing import                        |
| Async test without `await` on the asserted promise             | Add `await` to the assertion                  |
| Timeout on a single test where the underlying operation is genuinely slow | Increase timeout to documented framework cap, with comment explaining why |
| Snapshot mismatch where the diff matches the intentional code change in the same PR | Update snapshot, **but flag the snapshot update for Gate D human review** |
| Type error introduced by an upstream type definition change in a dependency bump | Update the local type usage to match new upstream type |
| Linting auto-fixable issues (formatting, import ordering, simple style) | Apply linter's `--fix` and re-run         |
| Flaky test confirmed by 3 consecutive passes after retry      | Mark test as flaky in tracking system; allow proceed; **file ticket for investigation** |

### 6.2 Prohibited Auto-Remediation

The agent must never attempt the following, even when doing so would make Gate B pass:

- **Modifying business logic to match a failing test.** The test asserts the expected behavior; if the code disagrees, the code is wrong, not the test. (Exception: when the assertion in the test is itself obviously incorrect — e.g., asserting `expect(2 + 2).toBe(5)` — the agent escalates rather than auto-fixing in either direction.)
- **Modifying assertions to match incorrect behavior.** A failing assertion is the test doing its job. Modifying the assertion to pass is suppressing the failure.
- **Disabling, skipping, or removing tests.** Any test that is `[Skip]`'d, removed, or marked as expected-to-fail by the agent is grounds for immediate human escalation.
- **Lowering coverage thresholds.** Threshold values come from this document and the project's CI configuration; the agent does not modify them.
- **Excluding code paths from coverage calculation.** Exclusions are declared in CI configuration upfront; the agent does not add new exclusions.
- **Modifying authorization tests.** Authorization tests are security-critical; auto-modification of them is prohibited regardless of failure mode.
- **Modifying audit trail tests.** Same rationale as authorization tests.
- **Adjusting test fixtures to make assertions pass.** Fixture changes are only permitted when the schema or interface they reference has changed; the agent must establish the schema-change basis before adjusting fixtures.
- **Suppressing compiler warnings via pragma directives.** Warnings are surfaced to the human reviewer; suppression is a code review decision.

### 6.3 Loop Behavior

| Loop | Permitted Scope                                           |
| ---- | --------------------------------------------------------- |
| Loop 1 | Attempt the most direct fix matching the failure pattern |
| Loop 2 | If Loop 1 did not resolve, attempt a broader fix or escalate decision back to scoping (e.g., is this a flaky test rather than a real failure?) |
| After Loop 2 | No further auto-remediation. Escalate to human with full diagnostic output |

Each loop must produce a discrete, reviewable change set. The agent does not chain multiple speculative fixes within a single loop.

---

## 7. Pass/Fail Policy

| Outcome                          | Conditions                                                |
| -------------------------------- | --------------------------------------------------------- |
| **Pass**                         | All required tests pass + all quality gates clear (no Block) + coverage thresholds met |
| **Pass with warnings**           | All required tests pass + coverage met + at least one quality gate at Warn (linting, warnings, test quality) |
| **Auto-remediated pass**         | Tests passed after Loop 1 or Loop 2 auto-remediation. Output flags which fixes were applied; Gate D reviews the fixes |
| **Fail — coverage**              | Tests pass but coverage threshold not met                 |
| **Fail — test failure**          | One or more required tests fail and auto-remediation cannot resolve |
| **Fail — quality gate**          | Build error, type error, or other Block-level quality gate failure |
| **Fail — escalation required**   | After Loop 2, failure remains; human review required      |

A "Pass with warnings" or "Auto-remediated pass" outcome is still a pass for the purpose of advancing to Gate C, but the warnings and remediations are forwarded to Gate D for review.

---

## 8. Type 1 vs Type 1B Differential at Gate B

| Dimension                        | Type 1                              | Type 1B                              |
| -------------------------------- | ----------------------------------- | ------------------------------------ |
| End-to-end tests                 | Optional                            | Required (primary happy paths)       |
| Authorization tests              | Optional                            | Required                             |
| Audit trail tests                | N/A                                 | Required                             |
| Coverage threshold (changed code) | ≥ 70%                              | ≥ 80%                                |
| Coverage threshold (auth/audit code) | N/A                            | ≥ 95%                                |
| Linting                          | Warn on violation                   | Block on violation                   |
| Skipped tests without annotation | Warn                                | Block                                |
| Auto-remediation rules           | Same as Type 1B (no relaxation)     | Same as Type 1                       |
| InfoSec review of auto-remediations | Not at Gate B (Gate D)            | Not at Gate B (Gate D)               |

Auto-remediation rules deliberately do not differ between Type 1 and Type 1B. The agent should not be more aggressive about modifying tests for less-sensitive applications; the prohibitions exist because of the integrity of the test process, not the sensitivity of the application.

The differential at Gate B manifests in what tests are required and what coverage threshold applies, not in how the agent behaves when running them.

---

## 9. Findings Routing

The Gate B agent produces structured output for downstream consumption.

| Output                          | Audience                | Format                                          |
| ------------------------------- | ----------------------- | ----------------------------------------------- |
| Pass/fail summary               | Gate C, Gate D          | Pass / Pass-with-warnings / Auto-remediated-pass / Fail |
| Coverage report                 | Gate D                  | Per-file coverage with changed-line attribution |
| Test execution log              | Gate D                  | Test names, durations, pass/fail status         |
| Auto-remediation report         | Gate D                  | Per-fix: failure pattern, action taken, files modified, link to commit |
| Quality gate warnings           | Gate D                  | Per-warning: gate, severity, location           |
| Test quality heuristic flags    | Gate D                  | Per-flagged-test: heuristic triggered, suggestion |
| Flaky test tickets filed        | Engineering tracking    | Per-flaky-test: failure history, frequency      |

Auto-remediations applied at Gate B are flagged for explicit Gate D review even when Gate B passes. The Gate D Code Review Orchestrator agent treats auto-remediated changes as commits requiring human verification, not as commits already reviewed.

---

## 10. Escalation Triggers

The Gate B agent escalates rather than continuing auto-remediation when any of the following occurs:

- Two auto-remediation loops have not resolved the failure.
- An auto-remediation attempt would require entering prohibited territory (modifying business logic, disabling tests, etc.).
- A snapshot update is required but the diff does not match an intentional code change in the same PR.
- Coverage drops in code paths declared as authorization-related or audit-trail-related.
- A flaky test was previously resolved and has reappeared.
- The failure mode is unrecognized — the agent does not have a remediation pattern matching the observed failure.
- Tests pass locally but consistently fail in CI, suggesting environment differences the agent cannot inspect.

Escalations route to the project's engineering owner via the standard CI notification channel and are recorded in the project's tracking tool.

---

## 11. Document History

| Version | Date         | Author     | Changes                                          |
| ------- | ------------ | ---------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author]   | Initial draft for CrewAI POC corpus              |
