# How we used Gemini CLI to automate a large-scale Spanner migration

**Category:** Databases (cross-post: AI & Machine Learning, Developers & Practitioners)
**Estimated read time:** 7 minutes
**Author:** [Your Name], [Your Title], Google

---

Migrating a service's data layer from one storage system to another is one
of the most repetitive — and most consequential — tasks in software
engineering. The same shape of code change has to be applied across many
data access objects (DAOs). Each change is small. Each change matters. And
a single inconsistency can cause data drift that takes weeks to find.

We recently went through this exact exercise: migrating Budget Manager from
Cloud Spanner to a different Spanner deployment, end-to-end, with no
downtime. Rather than write the migration code by hand, DAO by DAO, we
built an automation pipeline powered by **Gemini CLI in headless mode** —
and the results changed how we think about AI-assisted engineering.

This post covers what we built, why the headless mode was the unlock,
and what we'd recommend to any team facing a similar migration.

---

## The migration we needed to do

Budget Manager has many DAOs that perform writes. To migrate without
downtime we needed three phases of repetitive work across each of them:

1. **Backfill** — generate code to copy historical data from the source
   Spanner instance to the destination.
2. **Dual-write** — modify each DAO to write to both stores in parallel
   during the migration window, so neither store goes stale.
3. **API verification** — exercise every write RPC end-to-end and
   confirm that the dual write actually landed identically in both stores.

Each phase involved making essentially the same change across a long list
of DAOs and APIs. The pattern was clear; the volume was the problem.

## Why we reached for Gemini CLI — and specifically headless mode

Interactive AI coding assistants are great for exploration. They are not
great for "do the same thing carefully across N targets and don't miss
any of them." That kind of work needs an *automatable* AI invocation.

Gemini CLI in **headless mode** gave us exactly that. It runs as a normal
shell command — no terminal UI, no human prompt loop — which means we
could compose it with the rest of our build tooling. Concretely, headless
mode lets us:

- Drive Gemini from a shell script, the same way we drive any other CLI.
- Pin a curated, version-controlled prompt as the input — so the same DAO
  always produces the same shape of output.
- Run the entire pipeline unattended overnight and come back to a list
  of reviewable changelists in the morning.
- Layer presubmit checks, AI self-review, and CL creation as further
  shell steps on top.

Without headless mode, we would have been back to a human sitting in
front of a chat window, doing one DAO at a time. With it, AI becomes a
deterministic step in a pipeline.

## The pipeline shape

For each phase, the pipeline takes a CSV list of DAOs (or, for
verification, the service definition) and loops:

1. Create an isolated workspace for the target.
2. Invoke Gemini CLI (headless) to generate the code change and tests.
3. Run presubmit — build, lint, unit tests.
4. Invoke Gemini CLI again to perform an AI self-review pass; auto-fix
   findings; re-run presubmit.
5. Sanity-check the diff so that nothing outside the expected scope is
   touched.
6. Auto-author a changelist with a structured description and test plan.
7. Mail the CL to the right reviewers.
8. Log per-target status.

The orchestrator is a thin shell script. The intelligence is in the
prompts. The safety is in the gates. Humans review the final CL.

> **Why this layering matters.** Every CL produced by the pipeline
> passes through five layers — build, lint, tests, AI self-review, diff
> sanity — *before* a human ever sees it. We never put a half-finished
> CL in front of a reviewer.

## Verification: where AI earns its keep

The most interesting part wasn't the code generation — it was the
verification phase. Once dual-write CLs were submitted, we needed to
prove that every write RPC actually wrote identical data to both stores.

We pointed a single Gemini-driven workflow at the service definition.
It enumerated every write RPC, synthesized valid request payloads from
the proto types, called each one through stubby, and read the result
back from both stores to confirm the payloads matched.

The crucial detail: the **first call almost always failed.** A missing
field. A wrong enum. A stale fixture. In a manual flow, this is where
engineer time disappears. In our pipeline, Gemini reads the failure
context — status code, error message, server logs — and decides what
to do:

- If the request is malformed, it adjusts the payload and retries.
- If the failure is environmental, it surfaces a single, actionable
  flag for the engineer.
- If the failure points to a real bug in the dual-write code, it
  records a verification finding rather than papering over it.

When the verification run completes, the same pipeline emits a
comprehensive verification document — every RPC, every request,
every response, every dual-write diff, every troubleshooting step —
with no separate "now write it up" task. The document is an output of
the run, not a sibling of it.

## What changed for the team

Three things, concretely:

**Velocity.** Per-DAO authoring cost dropped to near-zero. The
engineer's cost is one CL review, not one CL authoring plus one
review. Across many DAOs, the difference is measured in weeks.

**Uniformity.** Every CL follows the same structure, error handling,
logging, retry policy, and metric naming. Reviewers built muscle
memory. On-call engineers see one signature in production, not many.
This is the kind of consistency that's nearly impossible when many
engineers write similar code by hand.

**Reviewer leverage.** Because boilerplate is gone, reviewer attention
shifts to where it actually pays off — schema correctness, blast
radius, rollout sequencing. The same review headcount can credibly
ship many more DAOs.

## Lessons for teams considering this pattern

A few things we'd do the same way, and a few we'd flag:

1. **Use headless mode for anything that loops.** Interactive AI is
   for exploration. Headless AI is for production.
2. **Keep the orchestrator dumb.** Shell scripts for sequencing,
   prompts for intelligence. Don't try to put logic in both places.
3. **Layer your safety gates.** Build → lint → tests → AI review →
   diff sanity → human review. Each layer catches a different class
   of issue.
4. **Make the AI self-heal where it's cheap, surface where it's not.**
   Most failures during verification are payload details, not bugs.
   Letting AI fix those quietly removes a lot of toil; making sure
   real bugs still bubble up is what keeps the pipeline trustworthy.
5. **Make documentation an output, not a task.** Generating the
   verification doc programmatically meant it was always current,
   never skipped, and uniform across runs.

## Where we're going next

The pipeline shape — CSV input, per-target workspace, headless Gemini
for the AI work, deterministic shell for orchestration, AI self-review
and self-healing for quality — is reusable for any future migration
that has the same "uniform change across many targets" structure.
We're already lining up the next places to apply it.

If you're staring down a similar migration, the punchline is simple:
**the cost of doing it carefully has dropped sharply.** AI tooling
that you can drive from a shell script changes the economics of
repetitive engineering work in a way that, six months ago, would
have been hard to argue for. Now it's the obvious choice.

---

**Want to learn more?**
- [Gemini CLI documentation](https://cloud.google.com/gemini/docs)
- [Cloud Spanner](https://cloud.google.com/spanner)
- [Building with the Gemini API](https://ai.google.dev/)
