# Owner inputs

Written only by the owner, on `main`. The `/goal` loop reads this file after merging `main` into
`agent/plan` and never edits it. Every input is a line starting at column 0 with `- `; the
examples in the comments below are indented on purpose so that no precondition matches them.

## Owner inputs

<!-- Examples:
    - spec-text-licence: CC-BY-4.0
    - code-licence: Apache-2.0
    - repo-settings: done                 (G02: private vulnerability reporting, branch protection, required CI)
    - disclosure-cleared: node            (G05; a comma-separated list of port ids, e.g. "node, go")
    - loop-pr: open                       (after opening the draft pull request from agent/plan)
    - rc-ok: RC-01                        (G35, one line per item of security/releases/1.0.0.md)
    - release-approved: 1.0.0             (G35)
-->

## Block clearances

<!-- Clear a block of plan/memory/project.md by repeating its goal and date. Example:
    - unblock: G07 2026-10-12
-->

## Approved issue drafts

<!-- Approve an issue draft with a ticked line holding its path. Unticked or absent drafts are
never filed. To approve none, write "- approved-issues: none". Examples:
    - [x] reports/go/issues/01-typed-tokens.md
    - approved-issues: none
-->

## Filed issues

<!-- Paste here the lines printed by the script of `node tools/gen-file-issues.mjs` (G33). Example:
    - filed: reports/go/issues/01-typed-tokens.md https://github.com/awesome-lang-auth/awesome-go-auth/issues/40
-->
