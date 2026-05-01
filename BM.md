# Budget Manager → GCP Migration

> One-line pitch: Migrating Budget Manager from <legacy stack> to GCP to <primary outcome>.

**Status:** 🟢 Frontend Live | 🟡 Backend In Progress
**Phase:** Backend migration — Backfill & Dual-Write
**TL:** @tl  |  **PM:** @pm  |  **Eng Manager:** @em
**Slack:** #budget-manager-migration  |  **Bug component:** <id>
**Last updated:** <date> by @owner

---

## At a Glance

| | |
|---|---|
| **Frontend on GCP** | ✅ Live as of <date> |
| **Backend on GCP** | 🟡 In progress |
| **Source** | <current platform / stack> |
| **Target** | GCP (<services: GCE / GKE / Cloud Run / Spanner / etc.>) |
| **Scope** | <services, DAOs, pipelines in scope> |
| **Out of scope** | <explicitly excluded> |
| **Target backend cutover** | <date / quarter> |
| **Dependencies** | <teams, services, infra> |

---

## Current Status

### ✅ Frontend — Live on GCP
- Migration complete and serving production traffic as of **<date>**.
- <# of users / QPS / regions> running on GCP.
- Key wins: <e.g. latency improvement, cost reduction, simplified deploys>.
- Link to launch announcement / postmortem of cutover: <link>

### 🟡 Backend — In Progress
- Currently in **<phase: Backfill / Dual-Write / Verification>**.
- <X of Y> DAOs migrated; <X of Y> APIs verified.
- Next milestone: <name + date>.
- See [Phase Plan](#phase-plan) for the full roadmap.

---

## 1. Background & Context

- What is Budget Manager (one paragraph for newcomers)
- Where it runs today and why
- What changed / what triggered this migration
- Links to historical docs, prior proposals

## 2. Why Migrate (Motivation)

- **Business drivers** — cost, compliance, customer asks
- **Technical drivers** — scaling limits, deprecation of current stack, reliability, latency
- **Strategic alignment** — org-level GCP mandate, platform consolidation
- **Cost of not migrating** — what breaks or stalls if we skip this

## 3. Goals & Non-Goals

### Goals
- <measurable goal 1>
- <measurable goal 2>

### Non-Goals
- <what we explicitly will not do in this migration>

### Success Metrics
| Metric | Baseline | Target | Frontend (actual) | Backend (target) |
|---|---|---|---|---|
| Latency p99 | | | | |
| Monthly cost | | | | |
| Availability | | | | |
| Migration completeness | 0% | 100% | ✅ 100% | <X%> |

---

## 4. Scope

### In Scope
- Services / components
- DAOs / data stores
- Pipelines / jobs
- Clients / callers

### Out of Scope
- <list>

### Stakeholders
| Team | Role | Contact |
|---|---|---|
| | Owner | |
| | Reviewer | |
| | Dependency | |
| | Customer | |

---

## 5. Current Architecture

- Diagram (link or embed)
- Key components and how they interact
- Known pain points

## 6. Target Architecture (on GCP)

- Diagram (link or embed)
- GCP services chosen and why
- Data flow
- Key design decisions (link to design doc)

## 7. Migration Strategy

- High-level approach (lift-and-shift, re-platform, re-architect)
- Phasing (frontend → backfill → dual-write → verify → cutover → cleanup)
- Rollback strategy
- Cutover plan (link to detailed runbook)

### Phase Plan
| Phase | Description | Owner | Status | ETA |
|---|---|---|---|---|
| 0 | Design & approvals | | ✅ Done | |
| 1 | **Frontend migration** | | ✅ **Live** | <date> |
| 2 | Backend — Backfill | | 🟡 In progress | |
| 3 | Backend — Dual write | | 🟡 In progress | |
| 4 | Backend — API verification | | ⬜ | |
| 5 | Backend — Read migration | | ⬜ | |
| 6 | Backend cutover | | ⬜ | |
| 7 | Decommission legacy | | ⬜ | |

---

## 8. AI-Driven Automation

A meaningful portion of the backend migration work is being automated using **Gemini CLI** in headless mode, orchestrated by shell scripts. This section highlights what we built and why it matters.

### Why Automate

The backend migration involves repeating the same shape of code change across many DAOs and APIs. Doing this by hand would be slow, inconsistent, and error-prone. Automation gives us **velocity, uniformity, and built-in safety gates** — and lets engineers spend their time on the parts that actually need human judgment.

### What We Automated

| Workstream | What it does | AI role | Status |
|---|---|---|---|
| **Backfill code generation** | For each DAO, generates the code that copies historical data from the legacy store to the GCP store. | Gemini CLI generates code, runs presubmit, performs AI self-review, and creates a CL. | 🟢 In production use |
| **Dual-write code generation** | Modifies each DAO to write to both legacy and GCP stores in parallel during the migration window. | Same pipeline shape as backfill — Gemini generates the change, AI reviews it, CL is mailed for human review. | 🟢 In production use |
| **API verification & documentation** | Once dual-write CLs are submitted, exercises every write RPC end-to-end via stubby, troubleshoots failures, and produces a verification document. | Gemini discovers APIs from the service definition, builds requests, self-heals on failure, and emits a comprehensive doc using the workspace skill. | 🟢 In production use |

### How It Works (High Level)

For backfill and dual-write, the pipeline is:

1. **Input** — CSV list of DAOs in scope.
2. **Per-DAO loop** — for each DAO:
   - Create an isolated workspace.
   - Call Gemini CLI (headless) to generate the code change + tests.
   - Run presubmit checks (build, lint, unit tests).
   - Call Gemini CLI again for an AI self-review pass; auto-fix and re-run presubmit.
   - Auto-create a CL with description and test plan.
   - Mail the CL to the appropriate reviewers.
   - Log per-DAO status.
3. **Output** — N reviewable CLs, one per DAO, uniform in structure.

For API verification, the pipeline reads the service definition, enumerates every write RPC, builds valid requests from the proto types, calls them via stubby, **self-heals on failure** (Gemini reads the error and adjusts the request or flags real bugs), verifies the dual-write actually landed in both stores, and finally produces a polished verification document.

### Benefits

- **Velocity** — Per-DAO authoring cost drops to near zero; engineer cost is one CL review, not one CL authoring + one review.
- **Uniformity** — Every CL follows the same structure, error handling, logging, and retry semantics. No drift between the first DAO and the last.
- **Safety gates** — Multiple layers (build, lint, tests, AI self-review, diff sanity) before any CL reaches a human.
- **Reviewer leverage** — Humans focus on logic and blast radius, not boilerplate.
- **Reproducibility** — Update the prompt once, re-run the pipeline, change propagates uniformly.
- **Self-documenting** — API verification produces evidence as a default, not an afterthought.
- **Reusable pattern** — The same orchestration shape applies to future migrations and release-readiness work.

### Detailed Documents

| Document | Description |
|---|---|
| [Backfill Automation](<link>) | Full design and operating model for backfill code generation. |
| [Dual-Write Automation](<link>) | Per-DAO dual-write generation pipeline. |
| [API Verification Automation](<link>) | End-to-end stubby verification with self-healing and auto-generated docs. |

---

## 9. Documents

| Doc | Type | Owner | Status |
|---|---|---|---|
| Design Doc | Design | | Approved |
| Frontend Cutover Postmortem | Ops | | ✅ Published |
| Capacity Plan | Plan | | Draft |
| Security Review | Review | | In review |
| Privacy Review | Review | | |
| Cost Model | Plan | | |
| Backend Cutover Runbook | Ops | | |
| Rollback Runbook | Ops | | |
| Test Plan | QA | | |
| Backfill Automation | Eng | | ✅ |
| Dual-Write Automation | Eng | | ✅ |
| API Verification Automation | Eng | | ✅ |
| Postmortems (if any) | Ops | | |

## 10. Approvals

| Approval | Approver | Date | Link |
|---|---|---|---|
| Design | | | |
| Security | | | |
| Privacy | | | |
| Capacity | | | |
| SRE / Prod readiness — Frontend | | ✅ | |
| SRE / Prod readiness — Backend | | | |
| Launch — Frontend | | ✅ | |
| Launch — Backend | | | |

---

## 11. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation | Owner |
|---|---|---|---|---|
| Data drift during dual-write | High | Med | Verification pipeline + diff job | |
| Latency regression on GCP | Med | Low | Load test pre-cutover | |
| Cost overrun | Med | Med | Cost model + budget alerts | |
| Backend cutover incident | High | Low | Staged rollout + rollback runbook | |
| AI-generated code introduces subtle bugs | Med | Low | Presubmit + AI self-review + human CL review (3-layer gate) | |

## 12. Open Questions

- [ ] <question 1> — owner: @x
- [ ] <question 2> — owner: @y

## 13. Decisions Log

| Date | Decision | Rationale | Decided by |
|---|---|---|---|
| | Frontend cutover approach | | |
| | Use Gemini CLI for backend migration automation | Velocity + uniformity across many DAOs | |

---

## 14. Tracking

- **Bug / Tracker:** <link>
- **Dashboards:** <migration progress, error rates, cost>
- **Burndown:** <link>
- **OKRs:** <link>

## 15. Timeline / Milestones

| Milestone | Date | Status |
|---|---|---|
| Design approved | | ✅ |
| **Frontend live on GCP** | <date> | ✅ |
| Backfill complete | | 🟡 |
| Dual write enabled (all DAOs) | | 🟡 |
| API verification green | | ⬜ |
| Read traffic shifted | | ⬜ |
| Backend cutover | | ⬜ |
| Legacy decommissioned | | ⬜ |

## 16. Communication Plan

- Weekly status: <where, when>
- Stakeholder updates: <cadence>
- Incident comms: <channel>

---

## 17. FAQ

**Q: Is the frontend already on GCP?**
A: Yes — frontend has been serving production traffic on GCP since <date>. Backend migration is in progress.

**Q: Why GCP and not <alternative>?**
A:

**Q: How is AI being used in this migration?**
A: We use Gemini CLI in headless mode to automate backfill code generation, dual-write code generation, and end-to-end API verification. See [AI-Driven Automation](#8-ai-driven-automation).

**Q: How does this affect <dependent team>?**
A:

**Q: What changes for end users?**
A:

---

## 18. Appendix

- Glossary
- Reference architecture links
- Related projects
- Historical context / prior attempts

---

## Changelog

| Date | Author | Change |
|---|---|---|
| <date> | | Frontend live on GCP |
| <date> | | Added AI-driven automation section |
| | | Initial page |
