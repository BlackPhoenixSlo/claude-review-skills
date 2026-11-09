# Review rules

<!-- Example for a multi-tenant web app with background jobs. Copy to your
repository root as REVIEW.md, replace the domain rules with your actual
invariants, and remove sections that do not apply. Keep development commands
and general coding instructions in CLAUDE.md / AGENTS.md. -->

## Evidence and severity

- Each finding includes severity, file:line, the concrete failure path or
  structural problem, its impact, and what fixed looks like.
- Distinguish reproduced behavior from inference. Investigate uncertain claims
  before presenting them as blockers.
- Block on data loss, unauthorized access, duplicate external effects, false
  success, or a demonstrated regression in a supported user flow.
- Explain structural findings through concrete coupling, branching, or ownership
  problems. Prefer a simpler existing abstraction over a speculative framework.
- Separate defects introduced by this change from pre-existing issues. Keep
  unrelated cleanup outside the fix loop.

## Domain invariants — adapt these examples

- Tenant isolation: routes, queries, background jobs, and cache keys must preserve
  the caller's tenant boundary. Test with two tenants, not just one happy path.
- Retry safety: retrying a payment or notification must not duplicate its external
  effect. Inspect the failure window between remote success and local persistence.
- Durable jobs: interrupted work must resume safely or report failure; an in-memory
  success flag must not stand in for persisted completion.
- Partial updates: a failed multi-step operation must not leave related records
  inconsistent. A partial remote sync must not delete unseen local records.
- Bounded work: pagination, retries, and polling must terminate; refreshes and
  concurrent callers must not create request storms.
- Honest success: a successful response means the promised action happened.
  If it only queues work, the API and UI must say so.

## Verification

- Choose checks for the affected behavior. Docs-only changes need relevant
  content/link checks, not a full application test suite.
- For runtime changes, exercise the real local path. For bug fixes, demonstrate
  that the regression check fails without the fix and passes with it.
- For UI changes, inspect a settled screenshot and verify the relevant API,
  persisted state, or other observable effect.
- Use slop/load scenarios when they add coverage for changed flows, concurrency,
  or volume. Skip irrelevant scenarios with a reason; missing slop alone is not
  a finding. Follow repository instructions for any selected scenario.
- Use local/dev data and stub external effects. Do not validate by changing
  production data or invoking paid services.

## Noise and scope

- Do not repeat formatter/linter findings already reported by automated checks.
- Do not request tests that merely repeat the implementation or cosmetic changes
  with no meaningful benefit.
- Review generated behavior at its source; do not request manual edits to generated
  output. Still flag consequential dependency or generated-output regressions.
- Check changed docs, UI text, logs, and return values against actual behavior.

## Keeping rules useful

Keep durable, verified lessons here. Put branch-specific acceptance criteria and
findings in per-run review material. Revisit rules when the architecture changes.
If `.claude/REVIEW.md` is kept as a backup, keep it identical; this root file wins.
