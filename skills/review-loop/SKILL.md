---
name: review-loop
description: "Autonomous build-and-review loop — run a review loop or review-and-fix loop on a branch, diff, commit, PR, or path: parallel structure and correctness review, merge findings, fix with one writer, re-review until both passes come back clean, then pause for commit approval."
---

# Review loop

Two read-only reviews in parallel, one writer, re-review until clean, human approves the commit. The loop is autonomous up to the commit; nothing before that waits on a human.

## The loop

1. **Scope** — what is under review.
2. **Review** — structure and correctness, in parallel, read-only, against a quiet tree.
3. **Merge** — one deduplicated, severity-ranked findings file on disk.
4. **Fix** — a single writer, blockers first, typecheck/build green throughout.
5. **Re-review** — both passes again; every finding gets a verdict with file:line proof; hunt for regressions. Anything open or newly broken sends you back to step 4.
6. **Commit** — the only step that waits for the human.

## Non-negotiables

- **One writer at a time.** The fixer never runs alongside a reviewer. A writer editing files a reader is analyzing produces a stale, untrustworthy report. Reviewers run in parallel with each other because they only read; fixing runs alone.
- **Reviewers are read-only.** No write access, no edits, no "quick fix while I'm here". The fixer is the only agent that touches files.
- **Quiet tree.** Reviews start only when no edits are in flight and the tree is stable. Confirm before fanning out.
- **Structure and correctness are different lenses.** Run both, every time. A beautifully-structured file can still lose data; a correct file can still be unmaintainable. One clean review is not approval.
- **The correctness prompt is project-specific on purpose.** It traces behavior against this project's own spec and failure modes. That specificity is its strength; keep it that way. The structure review is the generic, reusable one.
- **Autonomy stops at the commit.** Fix and re-review on your own. A human approves before anything is committed.
- **Report faithfully.** If a re-review shows anything STILL-OPEN or newly broken, say so and loop. "Approved" is a word for a tree both reviews came back clean on, and no other.

## 1. Scope

Default target: the current branch's diff against the main branch (`git diff <main>...HEAD`, plus uncommitted changes). If the user names a target — a path, a commit, a PR — review that instead.

Write the scope down (files, ref range) so both reviewers receive the identical target. Confirm the tree is quiet.

**Repository review rules.** Explicitly read root `REVIEW.md` when present and applicable `CLAUDE.md`/`AGENTS.md` instructions. Root `REVIEW.md` wins over any `.claude/REVIEW.md` backup; do not load both. Give the same rules to every review lens, devil's advocate, fixer, and re-review. The per-run correctness prompt adds current-change requirements instead of duplicating durable rules. If no `REVIEW.md` exists, continue with the project-specific prompt.

Select validation by the diff and repository guidance. Slop is not a universal gate: skip when irrelevant and record why; retain applicable runtime, regression, and UI proof requirements. A review-rules change must not silently weaken the criteria used to review itself: compare with the base rules and report the policy change explicitly.


## 2. Review

Spawn two subagents in the same message so they run concurrently. Both are read-only: no Edit, no Write, no shell commands that mutate the tree. Recommend a high-reasoning model at high or xhigh effort for each; this is the expensive step and it is where the loop earns its keep. Never run a review in the session that writes code.

**Structure review.** Invoke the `thermo-nuclear-code-quality-review` skill against the scope: the strict maintainability audit for abstraction quality, oversized files, and spaghetti-condition growth. If the skill is not installed, use an equivalent prompt: an unforgiving maintainability review that flags every abstraction that leaks, every file that has grown past one responsibility, and every condition chain that will sprout another branch next month.

**Correctness review.** Run the project's correctness prompt against the scope. This prompt traces actual behavior against the project's spec, design docs, and known findings, hunting for:

- data loss on any path (crash, restart, partial write, concurrent access)
- silent false-success: a call that reports OK while the work did not happen
- unbounded or infinite retries
- restart and resume bugs: state that does not survive a process boundary
- claim-vs-behavior mismatches: docs, comments, log lines, or return values that promise something the code does not do

If the project has no correctness prompt yet, draft one first. Read the spec, README, ADRs, and design docs; write a prompt that names this project's invariants, its known failure modes, and the exact claims the code makes; save it beside the other review material (e.g. `scratchpad/CORRECTNESS-REVIEW.md`). Then run it. Do not genericize it — a prompt that could review any project catches nothing specific to this one.

**Codex lens (round 1).** Claude does the review; Codex is one extra lens with a fixed, small budget (one Codex plan vs two Claude 20x accounts). Alongside the two subagents, run one `codex review` on round 1:

```sh
nice -n 10 codex review --base <base> -c model_reasoning_effort=high > /tmp/rwl-codex-r1.md 2>&1
```

`--base`/`--uncommitted`/`--commit` reject a custom prompt, so this runs Codex's built-in review prompt. It returns `[P0]`–`[P3]` findings with file:line; map P0/P1 to blockers. Use `--uncommitted` when the scope is the working tree, `--commit <sha>` for a single commit. Diff over ~1500 lines: `codex review` can't filter by path, so use `git diff <base>...HEAD -- <hot files> > /tmp/rwl-hot.patch` (DB writes, retries, locks, `service/` automations) and run `nice -n 10 codex exec -s read-only -c model_reasoning_effort=high "Review ONLY the diff in /tmp/rwl-hot.patch for correctness bugs; each: severity, file:line, why. No nits. If none, say NONE."` instead of skipping it. If `codex` is missing or fails, say so in the report and carry on; never block the loop on it, never retry it. Its findings are merged tagged `[codex]`; re-reviews verify them like any other finding.

When `REVIEW.md` exists, use `codex exec -s read-only` instead of the built-in `codex review` for round 1 and the final check, keeping the same call budget and reasoning effort. Supply a self-contained prompt naming the exact review scope (base/ref or patch, plus any in-scope staged/unstaged/untracked files), root `REVIEW.md`, and applicable project instructions; require the agent to read them before reviewing. Do not assume CLI auto-discovery of `REVIEW.md`. The hot-diff fallback and devil's advocate prompts must include these instructions too. Preserve the full original scope in the final check.

**Codex devil's advocate (round 1).** In the same message as the reviewers, one call that argues against the change and the idea behind it:

```sh
nice -n 10 codex exec -s read-only -c model_reasoning_effort=high -o /tmp/rwl-codex-da.md \
  "Devil's advocate on the change in \`git diff <base>...HEAD\` (plus uncommitted). Argue against it on two axes:
   CHANGES: what breaks, which assumption is wrong, what a code reviewer will miss.
   IDEAS: should this exist at all (YAGNI)? a smaller reframing? a simpler design?
   Each: severity, file:line or 'design', claim, strongest evidence. No nits, no style. If nothing, say NONE."
```

Tag its findings `[codex-da]`. CHANGES items merge like any finding. IDEAS items the fixer can't settle alone (drop or reframe the feature) don't loop: the orchestrator decides adopt/reject with a one-line reason, records it in the findings file, and shows it at step 6. If `codex` is missing or fails, say so and carry on; never retry it.

**Codex repros (risky diffs only).** If the diff touches `service/`, DB writes, retries or locks and round 1 found blockers, run one more `codex exec` in a throwaway worktree (`-s workspace-write`, never the review tree) to write a failing test per blocker. The fixer gets those tests and must turn them green. Skip this on every other diff.

**Codex budget per loop:** round 1 review + devil's advocate + final clean round (step 5) = 3 calls, +1 for repros on risky diffs, +1 per stuck-breaker (step 4). Nothing else: re-review rounds, the Claude devil's advocate and six hats stay on Claude, and `codex exec` second opinions run only when the human asks.

Each reviewer returns a findings list: severity, file:line, what is wrong, why it matters, and what fixed looks like.

## 3. Merge

Combine both reports into one file (e.g. `scratchpad/REVIEW-FINDINGS.md`):

- Deduplicate: one entry per underlying defect, even when both reviews phrased it differently.
- Rank by severity: blockers (data loss, false success, correctness) first, then maintainability, then polish.
- Tag each finding with which review caught it: `[structure]`, `[correctness]`, `[codex]`, `[codex-da]`, or `[both]`.
- Number them stably; re-reviews refer to these numbers.

Expect the overlap to be small. The findings only one lens caught are why both run.

## 4. Fix

Spawn a single code-writing agent (recommend a strong coding model at xhigh effort) with the merged findings file. It works alone: no reviewer is running while it edits.

- Work in ranked order, blockers first.
- Keep typecheck and build green after each fix; a fix that breaks the build is not a fix.
- For each finding, record what changed and where (file:line), appended to the findings file.
- Fix what the findings say. Unrelated refactors widen the next review for no reason.

Fixer done means: every finding has a recorded change or an explicit, justified "won't fix", and the build is green.

**Codex stuck-breaker.** If a finding is still STILL-OPEN or PARTIALLY-CLOSED after 2 Claude fix rounds, run one `codex exec -s read-only` with that finding, the failed fix attempts and the reviewers' verdicts. It proposes a *different* fix as a patch and does not apply it. The Claude fixer then applies it or rejects it with a reason; there is still only one writer. One call per stuck finding, never retried.

## 5. Re-review

Wait for the fixer to exit. Confirm the tree is quiet. Then re-run **both** reviews in parallel against the fixed tree, giving each reviewer the findings file.

Each reviewer must:

- Mark every original finding **CLOSED**, **PARTIALLY-CLOSED**, or **STILL-OPEN**, with file:line proof for the verdict.
- Hunt for anything the fixes newly broke: regressions, new duplication, invariants the fix moved rather than restored.

Merge the verdicts back into the findings file. Loop condition: any PARTIALLY-CLOSED, any STILL-OPEN, or any new finding sends you back to step 4 with the updated list. Repeat until both reviews return zero open findings and zero new ones.

Clean means both reviews, not one.

**Codex final check.** When both Claude reviews come back clean, run one `codex review --base <base>` (same command as round 1, output to `/tmp/rwl-codex-final.md`) on the full diff to catch regressions the fixer added. Any real finding goes back to step 4 like any other finding. A clean result, or Codex missing or failing (say so), lets you go on to step 6.

## 6. Commit

Stop. Present the human with:

- the final findings file with every entry CLOSED
- `[codex-da]` IDEAS objections, each adopted/rejected with its reason
- the diff summary
- a proposed commit message

Commit only on explicit approval. If the human declines or changes scope, return to the step they point at.

## Output

At the end of every run, report the loop's true state:

- how many review rounds ran
- findings caught, by lens (incl. codex, or why it was skipped), and how many each round closed
- codex calls used (round 1 / devil's advocate / repros / stuck-breakers / final) out of the budget
- anything still open (there should be none if you reached step 6; if there is, you have not reached step 6)
- whether the commit was approved
