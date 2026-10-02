---
description: Runs the next goal of plan/plan.md (one per run), verifies it, updates the documentation, commits and pushes the loop branch
argument-hint: "[optional G<nn>: only to check that it is the selected goal]"
---

# /goal — one goal of the spec plan per run

You are the executor of the agentic loop of the awesome-lang-auth specification repository. Work
from the repository root (`git rev-parse --show-toplevel`). Run **exactly one goal**, then stop.
The rules below repeat `plan/plan.md` § "Loop rules": when in doubt, that file wins.

Argument received: `$ARGUMENTS`. If it is a goal ID it is only a check: if the goal selected by the
rules is a different one, stop and report the difference without changing anything. Never use it
to pick a goal out of order.

## 1. Bootstrap check

The owner commits the bootstrap files on `main` before the first run. If neither
`git cat-file -e main:plan/plan.md` nor `git cat-file -e origin/main:plan/plan.md` succeeds, stop
without changing anything and report "bootstrap commit missing".

## 2. Branch and synchronisation

- If the current branch is not `agent/plan`:
  - with a dirty working tree, **stop without changing anything** and report;
  - otherwise `git switch agent/plan` if it exists locally; else
    `git switch -c agent/plan --track origin/agent/plan` if that remote branch exists; else
    `git switch -c agent/plan main` (or `origin/main`).
- If `origin` exists and is reachable: `git fetch origin`. Then, only when the working tree is
  clean:
  1. if `origin/agent/plan` exists: `git merge --ff-only origin/agent/plan`. If it fails (the
     branches diverged), **stop without changing anything** and report.
  2. if `git merge-base --is-ancestor origin/main HEAD` fails: `git merge --no-edit origin/main`.
     On a conflict: `git merge --abort`, add the waiting entry
     `- sync — merging origin/main into agent/plan conflicts; the owner resolves it`, commit it
     (`plan: waiting items updated`), push (step 7.6) and stop.
- The owner writes inputs, ADR reviews and block clearances on `main` only; you see them only after
  this synchronisation. Evaluate preconditions and blocks after it.

## 3. Read the context

1. Read `plan/plan.md` in full.
2. Read `plan/baseline.md`, `plan/memory/project.md`, `plan/memory/owner-inputs.md` (read only:
   **never edit it**) and `plan/memory/decisions.md`.
3. Read `AGENTS.md` and `CONTRIBUTING.md` if they exist, and `spec/00-overview.md` once G04 is done.

## 4. Select the goal

- **Orphan work.** If `git status --porcelain` is not empty and no goal is `DOING`, **stop without
  changing anything** and report the dirty files. The owner decides.
- **Open block.** A block is open when `## Blocks` of `plan/memory/project.md` has
  `- [ ] G<nn> — <date> — …` and `## Block clearances` of `plan/memory/owner-inputs.md` has no line
  `- unblock: G<nn> <date>` with the same goal and date.
- If a goal has `Status: DOING`:
  - with an open block, **stop without changing anything** and report the block;
  - otherwise it is a **resume** (interrupted run, or a cleared block): tick a cleared block entry
    (`- [x]`) in `plan/memory/project.md`, skip step 5 and restart from step 6 with 3 fresh
    attempts.
- Otherwise walk the goals in document order and take the **first eligible** one: `Status: TODO`,
  every goal in `Depends on` is `DONE`, for `Type: HUMAN` the `Precondition` command exits 0 (run
  it from the root with `bash -euo pipefail`), and its tools are available.
- **Tools.** `node` (major 24 or later: `node -p "process.versions.node.split('.')[0]"`), `npm`
  and `git` always; `npm ping` when `node_modules/` is absent or the goal runs `npm ci` or
  `npm install`; `gh` with `gh auth status` succeeding when the Verification calls `gh`;
  `git ls-remote https://github.com/<repo>.git HEAD` for each repository the goal clones;
  `curl -fsI` of one licence URL for G02.
- A HUMAN goal whose precondition fails, or a goal with a missing tool, **never becomes `DOING`**:
  it stays `TODO`; create or update its entry under `## Waiting for the owner` in
  `plan/memory/project.md` (what is needed and where to write it, or "missing tool: <name>"), then
  keep looking for the next eligible goal. No block, no attempt used.
- Keep `## Waiting for the owner` current: remove entries of goals that are `DONE`; after the first
  push add once `- PR — open one draft pull request from agent/plan into main, then add
  "- loop-pr: open" under ## Owner inputs`, and remove it once that owner input exists.
- If no goal is eligible, report "no runnable goal" with the waiting items and the blocks, and
  stop. If waiting entries changed and `git diff --quiet plan/memory/project.md` fails, commit
  `plan/memory/project.md` alone with the message `plan: waiting items updated` and push
  (step 7.6); otherwise commit nothing.

## 5. Start the goal (start only, not on resume)

1. Set `- **Status:** DOING` on the goal in `plan/plan.md` (that line only).
2. Run the `Verification` **before** writing anything else and observe that it fails. If it
   already passes, that is an anomaly: open a block (step 8) and stop.

## 6. Deliver exactly

- Create or modify **only**:
  - the files listed in `Deliverables`;
  - the files listed in `Documentation`;
  - the **integration files**, only to wire in what the deliverables create: `package.json` and
    `package-lock.json` (scripts and dev dependencies), `.gitignore`, `CHANGELOG.md` (one entry
    under `## [Unreleased]`), `README.md` (only `## Repository layout`), `tools/headings.json`,
    and from G08 on the rows of `### Error code catalogue` in `spec/11-errors.md` and
    `tools/check-codes-allow.json` for codes the goal introduces;
  - unavoidable generated files (lockfiles, `parity/matrix.md` written by its generator);
  - the goal's `Status` line and, in `plan/memory/project.md`, only `## Waiting for the owner` and
    `## Blocks`.
- Never edit `plan/memory/owner-inputs.md`, never change an existing ADR's `Status` line, never
  create an ADR the goal does not list (a needed one becomes an `## Open questions` item).
- **Update the documentation of the `Documentation` field before verifying.**
- Respect `Out of scope`. Never widen the scope, never touch later goals, never reorder or add
  goals. A deliverable that turns out wrong, ambiguous or too large for one run is a block, not an
  improvisation.
- **Writing the spec** (plan rule 13): take wire shapes, names, codes and defaults from the pinned
  inventory `inventory/node-1.10.8/`; depart from it only where the goal lists a departure with its
  ADR; write the goal's "Security baseline" as requirements that say nothing about any port and
  never contrast them with the reference or another port except through a public source; ADR
  contexts cite only public sources; BCP 14 keywords only inside requirement items; route aliases
  listed and never required; `## Known divergences` rows name a short repository name of
  `parity/ports.json` and a public source.
- **Public repository** (plan rule 12): English only; no secrets, no personal data, no local
  absolute paths, nothing copied from `$SPEC_PRIVATE_DIR`. Never write an undisclosed, exploitable
  weakness of any port, directly or by implication (a sentence, a table row, an inventory field,
  a declaration status), in a file, a commit message or anything pushed: put it in
  `$SPEC_PRIVATE_DIR` and say in your final report, without detail, that the owner has private
  material to review. From G28 on, a requirement that mitigates a threat is `fail` or `deviation`
  in a public file only with an existing public tracking URL; otherwise `untested` with evidence
  `review pending`.
- **External actions** (plan rule 14): the only write outside the working tree is
  `git push origin agent/plan`. No issue, comment, pull request, release, advisory, tag or package
  publication anywhere. Read-only access (clones, `git ls-remote`, `gh issue view`,
  `gh issue list`, `npm view`) is allowed where the goal needs it.

## 7. Verify and close

0. Before any Verification run (here and in step 5), run `npm ci` when `package.json` exists and
   `node_modules/` is missing or `package-lock.json` differs from `node_modules/.package-lock.json`.
1. Run the `Verification` block **as one script**: copy it to a temporary file and run it from the
   root with `bash -euo pipefail <file>`; steps in subdirectories use subshells `( cd … && … )`;
   negations are written `if <cmd>; then exit 1; fi`; output is captured before matching
   (`out=$(<cmd>); grep -Eq '…' <<<"$out"`), never piped into `grep -q`.
2. Check the output against `Expected:`.
3. Run the regression `npm run verify` (from G01 on). It must exit 0. It is offline: checks that
   need the reference clone or the registry run only where a goal's Verification calls them.
4. Commands that can take more than a few minutes (clones, installs, harness runs) run in the
   background and you wait for their result; a foreground timeout counts as a failed attempt.
5. You have **3 attempts** in total to make verification and regression pass, fixing only the
   files allowed in step 6.

**If it passes:**
1. Set `- **Status:** DONE` on the goal.
2. Check that the `Documentation` files are already updated.
3. For every non-obvious decision add an entry to `plan/memory/decisions.md`
   (`## YYYY-MM-DD — Title` with **Decision.** **Why.** **Consequences.**; existing entries are
   never rewritten).
4. Remove the goal's entry under `## Waiting for the owner`, if any.
5. `git add` only the files you touched and commit with the message exactly
   `G<nn>: <goal title>` (plus the attribution trailer the session requires). No `--no-verify`.
   If the commit fails, set `Status: DOING` back, open a block with the hook's message and stop.
6. Push: `git push origin agent/plan` (never `--force`, never another branch, never tags). If the
   push is not possible, add a `## Waiting for the owner` entry, commit it with the message
   `plan: waiting items updated` and stop.
7. Report briefly: goal closed, main files, verification result, next eligible goal, waiting items,
   and whether private material awaits the owner.

## 8. Blocks

When verification still fails after 3 attempts, or a rule above says "open a block":
1. Leave (or set) `- **Status:** DOING`.
2. Add under `## Blocks` in `plan/memory/project.md` the entry
   `- [ ] G<nn> — <YYYY-MM-DD> — <one-line symptom> — <what unblocks it>`.
3. Commit with the message `G<nn>: blocked` (`plan/plan.md`, `plan/memory/project.md` and any
   useful partial work that passes `npm run verify`), then push as in step 7.6.
4. Stop and report the block. The owner clears it on `main` by adding
   `- unblock: G<nn> <YYYY-MM-DD>` under `## Block clearances` in `plan/memory/owner-inputs.md`;
   on the next run the goal resumes with 3 fresh attempts.

## Use with /loop

The command is meant to be repeated:

```
/loop 15m /goal
```

Each round runs at most one goal. On an open block it stops without doing anything until the owner
clears it; when a HUMAN goal waits for an input it moves on to independent goals and, if there are
none, only reports the waiting items (committing them only when they changed).
