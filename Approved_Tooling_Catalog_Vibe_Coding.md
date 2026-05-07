# Approved Tooling Catalog — Vibe Coding Internal Applications

**Document Owner:** Security Architecture (in coordination with DevOps and Engineering Platform)
**Version:** 0.1
**Status:** Draft
**Effective Date:** TBD upon approval
**Scope:** Vibe Coding Type 1 (Internal · Non-Sensitive) and Type 1B (Internal · Restricted) applications only
**Out of Scope:** External-facing applications, PCI-scoped workloads, customer-facing services, regulated data flows. These are governed by applicable AWS infrastructure standards and applicable EKS infrastructure standards.

---

## 1. Purpose

This catalog defines the technologies, frameworks, and tools approved for use in Vibe Coding internal applications. It exists to serve three goals:

First, to reduce decision overhead for non-technical Product Owners who submit Vibe Coding projects. The catalog presents a curated short list rather than an enterprise menu, so a PO does not need to evaluate technology choices that have already been pre-cleared by Security Architecture and DevOps.

Second, to provide a deterministic rubric for the Pre-Stage Validator and Gate A Architecture Reviewer agents. Stack components in the catalog pass automatically; components outside the catalog trigger an exception path with a documented decision.

Third, to constrain the operational support burden. Vibe Coding apps are built quickly, often with AI assistance, by people who are not the long-term operators. Restricting the stack to technologies that DevOps and Engineering Platform already support means a Vibe Coding app does not become a one-off operational liability after launch.

---

## 2. How to Use This Catalog

A PRD's Section 6 (Proposed Tech Stack) must list a technology for each required layer, drawn from the Approved entries in Sections 4 through 14 below. Components in the Conditional tier (Section 15) are permitted only with an exception note in PRD Section 13 and Architecture review at Gate A. Components in the Prohibited tier (Section 16) are not permitted under any circumstances and trigger an immediate Pre-Stage rejection.

When in doubt, default to the Approved tier. The Conditional tier exists to support legitimate edge cases, not to be the path of least resistance.

---

## 3. Catalog Structure and Versioning Policy

For each technology, the catalog specifies an Approved status and a versioning policy rather than a frozen version number. The policies are:

- **LTS only** — only Long-Term Support releases are permitted. Active LTS or Maintenance-LTS versions are acceptable; STS / current versions are not.
- **Current stable** — the latest stable release plus the immediately prior stable release are permitted. Pre-release, beta, and end-of-life versions are not.
- **Vendor-current** — for SaaS or managed services, the vendor's current production version is the implicit approved version.

Specific version floors are reviewed and refreshed quarterly by Security Architecture in coordination with DevOps.

---

## 4. Frontend

Internal applications surface a web UI to authenticated internal users. Native mobile, desktop, or terminal-only applications are out of scope for this catalog.

| Technology      | Status    | Versioning Policy | Notes                                          |
| --------------- | --------- | ----------------- | ---------------------------------------------- |
| React           | Approved  | Current stable    | Default frontend library                        |
| Next.js         | Approved  | Current stable    | Preferred meta-framework for full-stack patterns |
| TypeScript      | Approved  | Current stable    | Required for all new frontend code              |
| Tailwind CSS    | Approved  | Current stable    | Approved utility-first CSS framework            |
| shadcn/ui       | Approved  | Current stable    | Approved component library for internal UIs     |
| Vue, Angular    | Conditional | —              | Permitted only with exception note; existing apps may use, new apps default to React/Next |
| jQuery          | Prohibited | —                | No new development                              |

---

## 5. Backend

Backend services for Vibe Coding internal applications must use the standard backend stack to align with Engineering Platform support capability.

| Technology         | Status      | Versioning Policy   | Notes                                               |
| ------------------ | ----------- | ------------------- | --------------------------------------------------- |
| C# / .NET          | Approved    | LTS only            | Primary backend stack                         |
| ASP.NET Core       | Approved    | LTS only            | For HTTP APIs and BFF patterns                      |
| Entity Framework Core | Approved | LTS only            | Approved ORM for MSSQL access                       |
| Node.js            | Conditional | LTS only            | Permitted for BFF or lightweight integration layers; not for primary business logic |
| Python             | Conditional | Current stable      | Permitted for data-processing or scripting components, not for HTTP-serving applications |
| Go, Rust, Java     | Prohibited  | —                   | Not part of the supported language set for Vibe Coding |
| PHP, Ruby, Perl    | Prohibited  | —                   | No new development                                  |

---

## 6. Authentication and Identity

Authentication for all Vibe Coding internal applications is delegated to Microsoft Entra ID via the standard the standard SSO pattern. Custom authentication is prohibited.

| Technology / Pattern                         | Status      | Notes                                                |
| -------------------------------------------- | ----------- | ---------------------------------------------------- |
| Entra ID SSO via OIDC (delegated permissions) | Approved   | Default authentication for all Vibe Coding apps      |
| Entra ID group-claim-based RBAC              | Approved    | Default authorization model                          |
| Microsoft.Identity.Web (.NET library)        | Approved    | Approved Entra ID integration library                |
| MSAL.js (frontend)                           | Approved    | Approved for SPA token handling                      |
| Local username/password authentication       | Prohibited  | No custom auth permitted                             |
| Application Permissions (App Registration)   | Conditional | Permitted only for service-to-service flows with explicit scope review at Gate A; never for user-facing flows |
| Custom JWT issuance                          | Prohibited  | Use Entra-issued tokens only                         |
| Hardcoded API keys for user authentication   | Prohibited  | —                                                    |

---

## 7. Datastores

| Technology          | Status      | Versioning Policy | Notes                                                      |
| ------------------- | ----------- | ----------------- | ---------------------------------------------------------- |
| MSSQL (internal cluster) | Approved | Vendor-current   | Default relational store; existing capacity, no new infra  |
| Redis (internal)    | Approved    | Current stable    | For caching, session storage, rate limiting                |
| Amazon S3 (where app runs in AWS) | Approved | Vendor-current | For object storage in AWS-hosted apps                  |
| SharePoint document library (Graph API) | Approved | Vendor-current | For document storage in M365-integrated apps     |
| PostgreSQL          | Conditional | LTS only          | Permitted for application-specific use cases not well-served by MSSQL; requires DevOps capacity confirmation |
| MongoDB, DynamoDB, Cassandra | Conditional | —       | Permitted only with explicit Architecture and DevOps approval at Gate A |
| MySQL, MariaDB      | Prohibited  | —                 | Not supported by the DBA team                            |
| SQLite (production) | Prohibited  | —                 | Permitted in development/test only, never in production    |
| Local file-based storage for application data | Prohibited | — | Not durable, not backed up, not auditable             |

---

## 8. Messaging and Eventing

Most Vibe Coding internal applications do not require asynchronous messaging. Where they do, the following apply.

| Technology      | Status      | Versioning Policy | Notes                                          |
| --------------- | ----------- | ----------------- | ---------------------------------------------- |
| Amazon SQS      | Approved    | Vendor-current    | For AWS-hosted apps                            |
| MQTT (internal broker) | Approved | Vendor-current  | For event-driven internal integrations         |
| Apache Kafka / Confluent | Conditional | —          | POC ongoing; permitted only for use cases approved by Architecture |
| RabbitMQ        | Conditional | —                 | Permitted only with DevOps capacity confirmation |
| Self-hosted message brokers not on this list | Prohibited | —    | —                                              |

---

## 9. Secrets Management

All application secrets must be stored in and retrieved from Akeyless. Hardcoded secrets, environment-variable-only secrets without retrieval from a vault, and secrets in source control are prohibited.

| Technology / Pattern                       | Status      | Notes                                                     |
| ------------------------------------------ | ----------- | --------------------------------------------------------- |
| Akeyless                                   | Approved    | Default secrets vault for all Vibe Coding apps            |
| AWS Secrets Manager (for AWS-only apps)    | Conditional | Permitted only in AWS-hosted apps where Akeyless integration is impractical; requires Security review at Gate A |
| Hardcoded secrets in source code           | Prohibited  | Detected at Gate C secrets scanning                       |
| Secrets in environment variables alone     | Prohibited  | Permitted only as the runtime injection mechanism after retrieval from Akeyless |
| Secrets in configuration files committed to Git | Prohibited | —                                                    |
| Secrets in Kubernetes Secrets (alone)      | Prohibited  | Acceptable as runtime injection target only when sourced from Akeyless via External Secrets Operator or equivalent |

---

## 10. Hosting and Compute

| Technology / Platform                | Status      | Notes                                                       |
| ------------------------------------ | ----------- | ----------------------------------------------------------- |
| Internal Kubernetes (dev-tooling cluster) | Approved | Default for internal Vibe Coding apps                       |
| Amazon EKS (where app runs in AWS)   | Approved    | Per applicable EKS infrastructure standards                                 |
| AWS Lambda                           | Conditional | Permitted for stateless event-driven workloads; requires DevOps capacity confirmation |
| AWS ECS / Fargate                    | Conditional | Permitted for AWS-only apps where EKS is over-engineered    |
| On-premises VMware VMs               | Conditional | Permitted only when containerization is impractical; requires Architecture approval |
| Self-managed VPS, container hosts not in catalog | Prohibited | —                                              |
| Public-facing cloud platforms (Vercel, Netlify, etc.) | Prohibited | Vibe Coding is internal-only; no external hosting |

---

## 11. CI/CD

Vibe Coding applications must use the standard the standard CI/CD pipeline. This is non-negotiable: the security scans at Gate C are pipeline-integrated, and out-of-pipeline deployment paths cannot be evaluated by the Security Gate Enforcer agent.

| Technology   | Status      | Notes                                          |
| ------------ | ----------- | ---------------------------------------------- |
| GitLab       | Approved    | Default source control and CI                  |
| ArgoCD       | Approved    | Default GitOps deployment controller           |
| GitHub Actions, CircleCI, Jenkins, Travis CI | Prohibited | —                              |
| Manual deployments via SSH, kubectl, AWS CLI | Prohibited | —                              |

---

## 12. Observability

All Vibe Coding applications must emit structured logs, application metrics, and (for Type 1B) audit events.

| Layer / Concern   | Approved Technology                       | Notes                                          |
| ----------------- | ----------------------------------------- | ---------------------------------------------- |
| Application logging | Standard structured JSON logs             | Forwarded to existing log aggregation          |
| Application metrics | Existing the standard metrics stack              | Standard counters, gauges, histograms          |
| Distributed tracing | OpenTelemetry SDK                         | Standard across .NET and Node.js ecosystems    |
| Error tracking      | Existing the standard error reporting platform   | —                                              |
| Audit log forwarding (Type 1B) | QRadar via existing forwarding pattern | Required for Type 1B applications |
| Custom logging stacks (ELK, Loki, etc.) | Prohibited            | Use existing the standard stack                       |
| Third-party APM (Datadog, New Relic, etc.) | Prohibited         | —                                              |

---

## 13. AI Coding Tools (Vibe Coding-Specific)

Vibe Coding by definition involves AI-assisted development. The catalog explicitly governs which AI coding tools are approved, and how they must be used.

| Tool / Pattern                                      | Status     | Notes                                                              |
| --------------------------------------------------- | ---------- | ------------------------------------------------------------------ |
| Anthropic Claude (via Portkey AI Gateway)           | Approved   | Default AI coding assistant; routed through the organization-managed AI gateway  |
| GitHub Copilot                                      | Conditional | Permitted only with InfoSec review of data-handling configuration |
| OpenAI, Anthropic direct API access (bypassing gateway) | Prohibited | All AI traffic must traverse the Portkey gateway                |
| Free-tier or personal-account AI tools for company code | Prohibited | Code is company IP; cannot be exposed to non-tenant AI systems  |
| AI tools without enterprise data-handling agreements | Prohibited | —                                                                  |

**Required configuration:** AI coding tools must be configured to operate within the organization's data boundary. Pasting company source code, internal data, or PRD content into consumer-tier AI tools (e.g., free chat.openai.com sessions, personal Claude.ai accounts) is a prohibited pattern regardless of which tool is in use.

---

## 14. Testing Frameworks

Test framework approval is consumed by the Gate B Automated Testing Controller agent, which must know what test runners and reporters are valid.

| Layer              | Approved Framework                | Versioning Policy | Notes                                          |
| ------------------ | --------------------------------- | ----------------- | ---------------------------------------------- |
| .NET unit testing  | xUnit                             | Current stable    | Default unit framework                         |
| .NET unit testing  | NUnit                             | Current stable    | Acceptable alternative                         |
| .NET mocking       | Moq, NSubstitute                  | Current stable    | Either acceptable                              |
| .NET integration testing | TestContainers              | Current stable    | Approved for DB and dependency-bound tests     |
| Frontend unit testing | Jest, Vitest                   | Current stable    | Either acceptable                              |
| Frontend component testing | React Testing Library     | Current stable    | —                                              |
| End-to-end testing | Playwright                        | Current stable    | Default e2e framework                          |
| Cypress, Selenium  | Conditional                       | —                 | Permitted only for legacy app migration scenarios |

---

## 15. Conditional Tier — Exception Path

Tools and technologies in the Conditional tier are not prohibited but require explicit handling:

1. The PRD must include an exception note in Section 13 (Open Questions / Risks) explaining why the Approved tier alternative is unsuitable.
2. The exception is reviewed at Gate A by the Architect (and InfoSec for Type 1B).
3. Approved exceptions are documented in the project's ADR.
4. Approved exceptions are scoped to the project; they do not constitute approval for general use across other Vibe Coding apps.

A Conditional approval is project-specific. Other Vibe Coding apps wanting to use the same conditional technology must each go through their own Gate A exception review.

---

## 16. Prohibited Tier — Hard Stop

Items in the Prohibited tier are not permitted under any circumstance for Vibe Coding applications. Submission of a PRD listing a Prohibited technology in Section 6 results in immediate Pre-Stage rejection. The PO must revise the stack to use Approved or Conditional alternatives before resubmission.

If a legitimate use case is genuinely not served by anything in the Approved or Conditional tiers, the appropriate path is not a Vibe Coding exception — it is a request to amend this catalog. Catalog amendments are reviewed by Security Architecture quarterly, or out-of-cycle for material business needs.

---

## 17. Catalog Amendment Process

Requests to add, modify, or remove catalog entries follow this process:

1. Submitter (any engineer, architect, or PO from any team) opens a request in the Standards project in Jira.
2. Request must include: technology, proposed tier, rationale, alternatives considered, operational support implications, and security implications.
3. Security Architecture and DevOps jointly review.
4. Approved amendments take effect at the next quarterly catalog refresh, or immediately for security-driven changes.

This process exists to keep the catalog current without making it a free-for-all. The default outcome of a casual request is "no" — the burden is on the requester to demonstrate why the existing options are insufficient.

---

## 18. Document History

| Version | Date         | Author          | Changes                                          |
| ------- | ------------ | --------------- | ------------------------------------------------ |
| 0.1     | [DATE]       | [Author] | Initial draft for CrewAI POC corpus              |
