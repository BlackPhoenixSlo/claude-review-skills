# claude-review-skills

Two [Claude Code](https://claude.com/claude-code) skills for reviewing code seriously,
plus the workflow that ties them together.

Code review has two independent lenses, and a clean pass on one tells you nothing
about the other. A file can be beautifully structured and still lose data; it can be
airtight and still be a tangle nobody can maintain. These skills give you one lens each,
and a loop that runs both.

## The skills

### `thermo-nuclear-code-quality-review` — the structure lens
An unusually strict maintainability audit: abstraction quality, file-size sprawl,
spaghetti-condition growth, wrong-layer logic, and "code-judo" simplifications that
delete whole categories of complexity rather than rearrange them. Generic and reusable
across any codebase.

Invoke with the bundled `/th` command:
- `/th` — review the current branch's changes vs. the main branch
- `/th <path | commit | description>` — scope the review to a target

### `review-loop` — the workflow
Codifies a build-and-review loop that runs both lenses and drives fixes to completion:

1. **Build** with a strong model.
2. **Review** with two passes **in parallel** — `/th` for structure, a *project-specific*
   correctness prompt for behavior. Both read-only.
3. **Merge** the findings into one ranked, deduplicated list.
4. **Fix** with a single writer, in ranked order (blockers first).
5. **Re-review** both passes until each finding is CLOSED and nothing new broke.
6. **Commit** — the only step that pauses for human approval.

Invoke with `/review-loop` or `/rwl`, or ask Claude to "run the review loop."
Use `/rwl-codexedits` for a Codex Sol writer with Claude reviewers.
The loop also uses bounded Codex review, devil's-advocate, and final-check calls;
missing Codex is reported without blocking the Claude loop.

## The one rule that makes it work

**One writer at a time.** Never run the fixing agent while a review agent is reading the
same files — the reviewer analyzes a moving target and its report goes stale. Reviews may
run in parallel with each other (they only read); fixing runs alone, against a quiet tree.

## Why the correctness review isn't in this repo

The structure review (`/th`) is generic and ships here. The **correctness** review is
deliberately *project-specific* — it traces behavior against a project's own spec and
findings, and that specificity is its whole strength. Genericizing it would turn a sharp
tool into a vague one. `review-loop` generates a fresh correctness prompt per project (from
the project's spec/design docs) rather than shipping one.

## Repository review rules

Both skills explicitly read the target repository's root `REVIEW.md` when present,
alongside applicable `CLAUDE.md`/`AGENTS.md` instructions. RWL passes those rules to
all reviewers and the fixer. Root wins over a `.claude/REVIEW.md` backup copy.
Keep durable review rules there and current-change acceptance criteria in the
per-run correctness prompt. App-specific rules do not belong in this skill repo.

Choose validation by relevance: slop is not a universal gate. Explain a skip;
retain applicable runtime, regression, and UI proof. Codex review calls explicitly
load `REVIEW.md` through `codex exec` rather than assuming auto-discovery.

## Install

Copy into your Claude Code config:

```sh
mkdir -p ~/.claude/skills ~/.claude/commands
# skills (user-wide)
cp -r skills/* ~/.claude/skills/
# /th, /rwl, and /rwl-codexedits commands
cp commands/*.md ~/.claude/commands/
```

Or clone anywhere and symlink the directories into `~/.claude/`.

## License

MIT — see [LICENSE](LICENSE).
