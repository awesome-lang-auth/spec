# Project memory

Written only by the `/goal` loop, on the branch `agent/plan`. The loop changes only the sections
"Waiting for the owner" and "Blocks". The owner never edits this file: owner inputs, approvals and
block clearances go in `plan/memory/owner-inputs.md`, on `main`.

## Environment

- Runtime: Node.js 24 or later with npm; git. `gh`, authenticated, only for read-only checks in
  the goals that call it. Network access to GitHub and to the npm registry where a goal needs it.
- Private directory: `$SPEC_PRIVATE_DIR`, default `$HOME/awesome-lang-auth-private`. It holds the
  full survey notes (`surveys/2026-10-01.json`), `security-notes.md`, per-port review notes
  (`reports/<id>/`) and the optional `denylist.txt` read by `tools/check-private.sh`. Never
  committed, never quoted in public files.
- Caches (ignored by git): `.cache/reference/` (pinned reference clone), `.cache/ports/<id>/`
  (port clones for reviews), `.cache/reports/` (runner reports).
- Loop branch: `agent/plan`. The owner opens one draft pull request from it into `main` and merges
  it with a merge commit (not a squash), so the branch keeps fast-forwarding.

## Waiting for the owner

- G02 — choose the licence of the specification text and the licence of code, schemas and vectors;
  enable private vulnerability reporting, protect `main` and require the CI check; then write
  `- spec-text-licence: <SPDX>`, `- code-licence: <SPDX>` and `- repo-settings: done` under
  `## Owner inputs` in `plan/memory/owner-inputs.md` (see G02 Precondition).
- G05 — the reference inventory is published only after the owner writes
  `- disclosure-cleared: node` under `## Owner inputs` in `plan/memory/owner-inputs.md`, once the
  private findings about awesome-node-auth 1.10.8 are fixed and published or judged safe to
  publish (see G05 Precondition). G06 onwards, except G03 and G04, wait for it.

## Blocks

<!-- - [ ] G<nn> — YYYY-MM-DD — symptom — what unblocks it -->
