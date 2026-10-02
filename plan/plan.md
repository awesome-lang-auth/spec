# awesome-lang-auth/spec — goal-driven plan

This file is read and updated by the `/goal` command (`.claude/commands/goal.md`). It is the **only
source of goal state**. Starting facts about the ten repositories of the family:
`plan/baseline.md`. Loop memory, written only by the loop: `plan/memory/project.md` (waiting items
and blocks) and `plan/memory/decisions.md` (non-obvious decisions). Owner inputs, written only by
the owner on `main`: `plan/memory/owner-inputs.md`.

## Loop rules

1. **One goal per run.** A `/goal` run works on exactly one goal, then stops.
2. **Order.** The document order is a topological order: every goal depends only on goals with a
   lower number (`tools/plan-lint.mjs` checks this from G01 on).
3. **Bootstrap.** The owner commits the bootstrap files (`README.md`, `GOVERNANCE.md`, `plan/`,
   `.claude/commands/goal.md`) on `main` before the first run. The loop runs only when
   `git cat-file -e main:plan/plan.md` or `git cat-file -e origin/main:plan/plan.md` succeeds;
   otherwise it stops without changing anything and reports "bootstrap commit missing".
4. **Branch and synchronisation.** The loop works only on the branch `agent/plan`.
   - If the current branch is not `agent/plan` and the working tree is clean: `git switch agent/plan`
     when it exists locally; otherwise `git switch -c agent/plan --track origin/agent/plan` when
     that remote branch exists; otherwise `git switch -c agent/plan main` (or `origin/main`). With a
     dirty working tree on another branch, stop without changing anything and report.
   - If a remote `origin` exists and is reachable: `git fetch origin`. Then, only when the working
     tree is clean:
     1. if `origin/agent/plan` exists, `git merge --ff-only origin/agent/plan`; when that fails
        (the branches diverged), stop without changing anything and report;
     2. if `origin/main` is not an ancestor of `HEAD`, `git merge --no-edit origin/main`; on a
        conflict run `git merge --abort`, add the waiting entry
        `- sync — merging origin/main into agent/plan conflicts; the owner resolves it`, commit
        and push it (rule 6), and stop.
   - The owner writes inputs, ADR reviews and block clearances on `main` only. They reach the loop
     through this synchronisation; preconditions and blocks are evaluated after it.
5. **Goal selection.**
   - **Dirty tree.** If `git status --porcelain` is not empty and no goal is `DOING`, stop without
     changing anything and report the dirty files (orphan work of an interrupted run).
   - **Blocks.** A block is open when `plan/memory/project.md`, section `## Blocks`, has an entry
     `- [ ] G<nn> — <date> — …` and `plan/memory/owner-inputs.md`, section `## Block clearances`,
     has no line `- unblock: G<nn> <date>` with the same goal and date.
   - If a goal is `DOING`: when it has an open block, **stop without changing anything** and report
     the block; otherwise **resume** that goal. On resume a cleared block entry is ticked
     (`- [x]`) in `plan/memory/project.md`, the "Verification must fail first" check is skipped and
     the run restarts from the deliverables with 3 fresh attempts.
   - Otherwise: the **first eligible** goal in document order: `Status: TODO`, every goal in
     `Depends on` is `DONE`, for `Type: HUMAN` the `Precondition` command exits 0, and the tools
     used by the goal are available (rule 7).
   - If no goal is eligible: report "no runnable goal" with the waiting items and the blocks, and
     stop.
6. **Waiting items.** The loop keeps `## Waiting for the owner` in `plan/memory/project.md` up to
   date: one entry per HUMAN goal whose precondition fails (what is needed, where to write it),
   per missing tool, per synchronisation conflict, and the one-time entry
   `- PR — open one draft pull request from agent/plan into main, then add "- loop-pr: open" under
   ## Owner inputs` after the first push, removed once that owner input exists. Entries of goals
   that became `DONE` are removed. A run that changes only waiting entries commits them with the
   message `plan: waiting items updated` and pushes (rule 11), **only when**
   `git diff --quiet plan/memory/project.md` fails; otherwise it commits nothing.
7. **HUMAN goals and tools.**
   - A HUMAN goal needs an owner input described in its `Precondition` (run from the repository
     root with `bash -euo pipefail`). While the precondition fails the goal **never becomes
     `DOING`**: it stays `TODO`, its waiting entry is created or updated and selection moves on to
     the next eligible goal. Goals depending on it keep waiting.
   - Before declaring a goal eligible, check the tools it needs: `node` (major version 24 or later),
     `npm` and `git` always; registry access (`npm ping`) when `node_modules/` is absent or the goal
     runs `npm ci` or `npm install`; `gh` with `gh auth status` succeeding when the Verification
     calls `gh`; network access to GitHub (`git ls-remote https://github.com/<repo>.git HEAD`) for
     each repository the goal clones; `curl -fsI` of one licence URL in G02. A missing tool is
     treated like an unmet precondition: the goal stays `TODO`, its waiting entry says
     "missing tool: <name>", no block is opened and no attempt is used.
8. **Verification must fail first.** On start (not on resume) set `Status: DOING`, run the
   `Verification` and observe that it **fails**. If it already passes, that is an anomaly: open a
   block and stop.
9. **Exact deliverables.** Create or modify only:
   - the files listed in `Deliverables`;
   - the files listed in `Documentation`;
   - **integration files**, and only to wire in what the deliverables create: `package.json` and
     `package-lock.json` (scripts and dev dependencies), `.gitignore`, `CHANGELOG.md` (one entry
     under `## [Unreleased]` per goal that changes normative text, schemas, vectors or tools),
     `README.md` (only its `## Repository layout` section, to link files the goal creates),
     `tools/headings.json` (required headings of files the goal creates), and, from G08 on, the
     rows of the `### Error code catalogue` table in `spec/11-errors.md` and
     `tools/check-codes-allow.json` for codes the goal introduces;
   - unavoidable generated files (lockfiles, `parity/matrix.md` regenerated by its tool);
   - the `Status` line of the goal in this file and, in `plan/memory/project.md`, only the sections
     `## Waiting for the owner` and `## Blocks`.

   The loop **never** edits `plan/memory/owner-inputs.md`, never edits an existing ADR's `Status`
   line, and never creates an ADR that its goal does not list: a decision that needs one more ADR
   becomes an item in the area's `## Open questions`. `Out of scope` is binding. Never widen the
   scope, never touch other goals, never reorder or add goals. A goal that turns out wrong or too
   large for one run is a block.
10. **Documentation before verification.** The files in `Documentation` are part of the deliverable
    and are updated **before** running the Verification.
11. **Verification and outcome.**
    - The `Verification` block runs **as one script** from the repository root with
      `bash -euo pipefail` (saved to a temporary file and run with `bash -euo pipefail <file>`);
      steps in subdirectories use subshells `( cd … && … )`. Negations are written
      `if <cmd>; then exit 1; fi`, never `! <cmd>`. Output is captured before it is matched
      (`out=$(<cmd>); grep -Eq '<pattern>' <<<"$out"`), never piped into `grep -q`, which can make
      the writer fail with SIGPIPE under `pipefail`. After the goal's Verification run the
      regression `npm run verify` (from G01 on). Commands that may take more than a few minutes
      (cloning, installing, running a harness) run in the background and their result is awaited.
      Everything must exit 0 and produce the `Expected:` output.
    - Before running any Verification (including the "must fail first" run), run `npm ci` when
      `package.json` exists and `node_modules/` is missing or `package-lock.json` differs from
      `node_modules/.package-lock.json`, so that a missing install never makes a check fail.
    - `npm run verify` is offline and self-contained: it never fetches, and never reads `.cache/`.
      Checks that need the reference clone or a registry (`inventory-check`, the node reference
      harness) run in their own CI workflows and in the Verification of the goals that create them.
    - Pass → `Status: DONE`, an entry in `plan/memory/decisions.md` for every non-obvious decision,
      waiting entry of the goal removed, commit with message `G<nn>: <title>` (no `--no-verify`).
      Then `git push origin agent/plan` (never `--force`, never another branch, never tags). If the
      push fails, add a waiting entry and stop.
    - If the commit fails (hook, check), set `Status: DOING` back, open a block and stop.
    - Fails after **3 attempts** → the status stays `DOING`; add under `## Blocks` in
      `plan/memory/project.md` an entry `- [ ] G<nn> — <YYYY-MM-DD> — <symptom> — <what unblocks it>`;
      commit with message `G<nn>: blocked` (plan, memory and any partial work that passes
      `npm run verify`), push as above and stop. The owner clears it by adding
      `- unblock: G<nn> <YYYY-MM-DD>` under `## Block clearances` in
      `plan/memory/owner-inputs.md` on `main`.
12. **Public repository and responsible disclosure.** This repository and every branch pushed to
    it are public.
    - Everything is written in English. Normative text uses the BCP 14 keywords (RFC 2119 and
      RFC 8174) only inside requirement items, as defined in `spec/00-overview.md` once G04 is done.
    - Never commit secrets, personal data (names, emails, handles of individuals), local absolute
      paths, or content of the private directory `$SPEC_PRIVATE_DIR` (default
      `$HOME/awesome-lang-auth-private`), which holds the full survey notes and the security notes.
    - Undisclosed, exploitable weaknesses of any port are never written in this repository, in
      commits, in pull requests or anywhere public. This includes indirect disclosure: a sentence,
      table, inventory field or declaration status that lets a reader infer that a named port
      lacks a security property. Security-sensitive findings go to `$SPEC_PRIVATE_DIR` and are
      mentioned to the owner in the run report, without detail.
    - The machine-readable inventory of the reference (G05 to G07) is published only after the
      owner records `disclosure-cleared: node` (G05 Precondition).
    - From G28 on, a requirement that mitigates a threat of `security/threat-model.md` may carry
      the status `fail` or `deviation`, in a declaration, a report or an issue draft, only with a
      public tracking URL that already exists (an issue, a pull request or a published advisory);
      otherwise it stays `untested` with evidence `review pending` and the finding goes to
      `$SPEC_PRIVATE_DIR`. `tools/gen-parity.mjs` and `tools/check-reports.mjs` enforce this.
13. **Writing the spec (goals G08 to G18).**
    - **Reference default.** For wire shapes, route paths, field names, codes, status codes and
      configuration defaults, the spec follows the behaviour of the pinned reference implementation
      as recorded in `inventory/node-1.10.8/` (awesome-node-auth 1.10.8), not survey prose. Where
      the inventory and `plan/baseline.md` disagree, the inventory wins and the disagreement is
      noted in the area's `## Open questions`. A goal's "Reference default" bullet lists what to
      take from the inventory; departures from it are listed with their ADR.
    - **Security baseline.** Each spec goal also lists a "Security baseline": requirements every
      port meets, written from the threat model, the cited RFCs and public security practice. The
      baseline says nothing about any port. Its text, its ADRs and the area file never contrast it
      with the behaviour of the reference or of another port, except through a public source
      (an issue, a pull request, a published advisory, or a port's own documentation).
    - **ADRs.** Each ADR listed by the goal is written in `adr/` with status `Proposed`; the spec
      text uses the proposed position. An ADR's `## Context` cites only public sources and general
      security literature. The owner accepts, amends or rejects ADRs before vectors are written
      (G20).
    - **Aliases.** An area lists in `## Reference behaviour` every route of its surface that the
      inventory marks with `aliasOf`, and says whether a port MAY provide it; an alias is never
      required.
    - **Divergences.** Each area file has a `## Known divergences` table
      `| Port | Behaviour | Requirement | Source |`. The Port cell is the short repository name
      (for example `awesome-go-auth`) of a port in `parity/ports.json`; the Source cell is a public
      reference (issue, pull request, the port's own documentation, or "survey 2026-10-01" for wire
      facts in `plan/baseline.md`). Rows describe wire shapes, names, codes and defaults; a row
      about a security baseline requirement needs a public issue, pull request, advisory or
      documentation page as its source. No BCP 14 keyword appears in these tables.
14. **External actions.** The loop never creates or edits issues, pull requests, comments,
    releases, advisories, tags or packages on any repository, this one included. Its only write
    outside the working tree is `git push origin agent/plan`. Read-only access (clones, `git
    ls-remote`, `gh issue view`, `gh issue list`, `npm view`) is allowed where a goal needs it. The
    owner opens the loop's pull request, files issues and tags releases.

## Goal format

```
## G<nn> — <title>
- **Status:** TODO | DOING | DONE
- **Type:** AGENT | HUMAN
- **Depends on:** G.., G.. | —
- **Precondition:** (HUMAN only) what the owner must provide + a command that exits 0 when it is there
- **Goal:** 1-2 sentences
- **Deliverables:** list of paths from the repository root with what to create or change
- **Out of scope:** what not to touch
- **Verification:** exact commands + `Expected:`
- **Documentation:** what to update (README, CHANGELOG, plan/memory/decisions.md)
```

## Index

| ID | Title | Type | Depends on |
|---|---|---|---|
| G01 | Scaffold, linters and CI | AGENT | — |
| G02 | Licences and repository settings | HUMAN | G01 |
| G03 | Area registry, spec skeletons and requirement linter | AGENT | G01 |
| G04 | Overview, conformance model and vision | AGENT | G03 |
| G05 | Reference pin and route inventory | HUMAN | G03 |
| G06 | Node inventory: errors and events | AGENT | G05 |
| G07 | Node inventory: cookies, headers, tokens and configuration | AGENT | G06 |
| G08 | Spec: errors | AGENT | G04, G07 |
| G09 | Spec: transport and abuse protection | AGENT | G08 |
| G10 | Spec: tokens and sessions | AGENT | G09 |
| G11 | Spec: password, email and account lifecycle | AGENT | G10 |
| G12 | Spec: passwordless and two-factor | AGENT | G11 |
| G13 | Spec: OAuth and account linking | AGENT | G12 |
| G14 | Spec: admin console and API keys | AGENT | G11 |
| G15 | Spec: events, webhooks and tools | AGENT | G14 |
| G16 | Spec: configuration and secure defaults | AGENT | G13, G15 |
| G17 | Spec: identity provider and resource server | AGENT | G16 |
| G18 | Spec: client contract | AGENT | G13 |
| G19 | Threat model and release checklist | AGENT | G17, G18 |
| G20 | ADR review and spec alignment | HUMAN | G19 |
| G21 | Vector format, runner contract and porting guide | AGENT | G20 |
| G22 | Vectors: errors, transport, abuse protection, tokens and sessions | AGENT | G21 |
| G23 | Vectors: credentials, account lifecycle, passwordless and two-factor | AGENT | G22 |
| G24 | Vectors: OAuth, admin, API keys, events, webhooks and tools | AGENT | G23 |
| G25 | Vectors: configuration, identity provider, clients and full coverage | AGENT | G24 |
| G26 | Reference runner | AGENT | G25 |
| G27 | Node reference harness | AGENT | G26 |
| G28 | Port declarations and parity matrix | AGENT | G27 |
| G29 | Conformance report: node | AGENT | G28 |
| G30 | Conformance reports: go and lambda | AGENT | G29 |
| G31 | Conformance reports: python, rust and dart | AGENT | G30 |
| G32 | Conformance reports: flutter, angular and react | AGENT | G31 |
| G33 | Record filed issues | HUMAN | G32 |
| G34 | Release evidence | AGENT | G02, G33 |
| G35 | Release 1.0.0 | HUMAN | G34 |

## G01 — Scaffold, linters and CI
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** —
- **Goal:** Make the repository operable for the loop: agent guide, contributing guide, an npm-based
  `verify`, the plan linter, offline Markdown link and heading checks, a private-content check and
  CI on GitHub Actions.
- **Deliverables:**
  - `AGENTS.md` — operating guide with the sections `## Commands` (`npm ci`, `npm run verify`,
    each `lint:*` script), `## Layout` (planned top-level directories, in code spans for paths that
    do not exist yet), `## Writing rules` (English; BCP 14 keywords only in requirement items;
    reference default, security baseline, ADRs, aliases and divergence tables of plan rule 13),
    `## Prohibitions` (plan rules 12 and 14, the private directory `$SPEC_PRIVATE_DIR`, the
    owner-only file `plan/memory/owner-inputs.md`).
  - `CLAUDE.md` — one line: "Read AGENTS.md: it is the operating guide of this repository."
  - `CONTRIBUTING.md` — sections `## Ways to contribute`, `## Proposing a normative change`
    (issue first, ADR for anything that changes a requirement, requirement IDs are never reused),
    `## Running the checks`, `## Security issues` (never public issues for undisclosed
    weaknesses; pointer to the affected port's advisory process).
  - `CHANGELOG.md` — Keep a Changelog layout with `## [Unreleased]`.
  - `VERSION` — `0.1.0-draft`.
  - `.gitignore` — `node_modules/`, `.cache/`, `coverage/`, `*.log`, `.DS_Store`.
  - `.editorconfig`; `.nvmrc` with content `24`.
  - `package.json` — `"private": true`, `"name": "awesome-lang-auth-spec"`, `"type": "module"`,
    `engines.node` `>=24`, no dependency yet, scripts `lint:plan`, `lint:links`, `lint:headings`,
    `lint:private`, `test` (`node --test tools/`) and `verify` (all of them in that order).
    `package-lock.json` generated by `npm install`.
  - `tools/plan-lint.mjs` — validates `plan/plan.md`, ignoring fenced code blocks (fences may be
    indented, as inside the Verification bullets): headings `## G<nn> — <title>` contiguous from
    G01; field bullets in the order Status, Type, Depends on, Precondition, Goal, Deliverables,
    Out of scope, Verification, Documentation; Precondition present if and only if Type is HUMAN;
    Status in {TODO, DOING, DONE}; Type in {AGENT, HUMAN}; every dependency exists and has a lower
    number; at most one DOING; every Verification holds a fenced `bash` block and a line whose
    first non-blank characters are `Expected:` (indentation allowed); the rows of `## Index`
    identical (ID, title, type, dependencies) to the sections. Prints `plan-lint: OK (<n> goals)`
    or the errors and exits 1.
  - `tools/plan-lint.test.mjs` — `node:test` cases with inline valid and invalid plans (forward
    dependency, missing field, Precondition on an AGENT goal, two DOING, index mismatch, indented
    `Expected:` accepted).
  - `tools/check-links.mjs` — for every tracked `*.md` file (`git ls-files`, plus untracked
    non-ignored files from `git ls-files --others --exclude-standard`): inline links and reference
    definitions outside fenced and inline code; a relative target must exist; a `#fragment` must
    match a heading slug of the target (GitHub algorithm: lower-case, drop characters other than
    letters, digits, spaces, `-` and `_`, spaces to `-`, duplicates suffixed `-1`, `-2`);
    `http(s):` and `mailto:` targets are not fetched. Prints
    `check-links: OK (<files> files, <links> links)`. `tools/check-links.test.mjs`.
  - `tools/headings.json` — map from path pattern (`*` matches within one path segment) to the
    list of required `## ` headings. Initial entries: `README.md` (`## Vision`, `## Scope`,
    `## Repository layout`, `## How ports conform`, `## Versioning`, `## Status`, `## Licence`),
    `GOVERNANCE.md` (`## Roles`, `## Decision process`, `## Spec versioning`,
    `## Conformance claims`, `## Agentic loop`, `## Security disclosure`), `CONTRIBUTING.md`,
    `AGENTS.md` (the sections above), `plan/memory/project.md` (`## Environment`,
    `## Waiting for the owner`, `## Blocks`) and `plan/memory/owner-inputs.md` (`## Owner inputs`,
    `## Block clearances`, `## Approved issue drafts`, `## Filed issues`).
  - `tools/check-headings.mjs` and `tools/check-headings.test.mjs` — every file matching a pattern
    holds each required heading as an exact line; a pattern matching no file is an error. Prints
    `check-headings: OK (<n> files)`.
  - `tools/check-private.sh` — scans tracked and untracked non-ignored files (excluding itself and
    its test) and fails on: absolute paths under the macOS or Linux home roots or the macOS
    per-user temporary root (case-sensitive match, so API paths such as `/users/{id}` pass),
    PEM private-key headers, and any non-blank, non-`#` line of
    `$SPEC_PRIVATE_DIR/denylist.txt` (fixed string, case-insensitive) when that file exists. When
    the deny-list is absent it first prints `check-private: denylist absent, phrase check skipped`;
    when nothing is found its **last** line is `check-private: OK`.
  - `tools/check-private.test.mjs` — builds the forbidden patterns at run time (never as literals).
  - `.github/workflows/ci.yml` — one job on `push` and `pull_request`: checkout,
    `actions/setup-node` with `node-version-file: .nvmrc` and npm cache, `npm ci`,
    `npm run verify`; `permissions: contents: read`.
- **Out of scope:** `spec/`, `adr/`, `inventory/`, `conformance/`, `parity/`, `security/`; JSON
  Schema validation (G03); the content of this plan except the Status of G01; README and
  GOVERNANCE text beyond what the link and heading checks require.
- **Verification:**
  ```bash
  npm ci
  npm run verify
  out=$(node tools/plan-lint.mjs); grep -Eq '^plan-lint: OK \([0-9]+ goals\)$' <<<"$out"
  out=$(node tools/check-links.mjs); grep -Eq '^check-links: OK \([0-9]+ files, [0-9]+ links\)$' <<<"$out"
  out=$(node tools/check-headings.mjs); grep -Eq '^check-headings: OK \([0-9]+ files\)$' <<<"$out"
  out=$(bash tools/check-private.sh); test "$(tail -n 1 <<<"$out")" = 'check-private: OK'
  test -f AGENTS.md && test -f CLAUDE.md && test -f CONTRIBUTING.md && test -f CHANGELOG.md
  test "$(cat VERSION)" = 0.1.0-draft && test "$(cat .nvmrc)" = 24
  grep -q 'npm run verify' .github/workflows/ci.yml
  git check-ignore -q node_modules/x && git check-ignore -q .cache/reference/x
  ```
  Expected: every command exits 0. Before the goal `npm ci` fails (no `package.json`).
- **Documentation:** `README.md` `## Repository layout` lists `tools/`, `AGENTS.md`,
  `CONTRIBUTING.md`; `plan/memory/decisions.md` entries for the link-check slug algorithm and for
  the choice of npm scripts over a Makefile.

## G02 — Licences and repository settings
- **Status:** TODO
- **Type:** HUMAN
- **Depends on:** G01
- **Precondition:** The owner chooses the licence of the specification text and the licence of the
  code, schemas and vectors, and configures the repository on GitHub: private vulnerability
  reporting enabled (GOVERNANCE.md and the future SECURITY.md rely on it), branch protection on
  `main`, and the CI check of G01 required before merge. The owner records under `## Owner inputs`
  in `plan/memory/owner-inputs.md` the lines `- spec-text-licence: <SPDX>`,
  `- code-licence: <SPDX>` and `- repo-settings: done`. Command:
  `grep -Eq '^- spec-text-licence: [A-Za-z0-9.+-]+$' plan/memory/owner-inputs.md && grep -Eq '^- code-licence: [A-Za-z0-9.+-]+$' plan/memory/owner-inputs.md && grep -Eq '^- repo-settings: done$' plan/memory/owner-inputs.md`
- **Goal:** License the repository so that ports and contributors can reuse the text, the
  schemas and the vectors.
- **Deliverables:**
  - `LICENSES/<spec SPDX>.txt` and `LICENSES/<code SPDX>.txt` — official texts from
    `https://raw.githubusercontent.com/spdx/license-list-data/main/text/<SPDX>.txt`.
  - `LICENSE` — byte copy of the code licence text.
  - `README.md` `## Licence` — which licence covers which paths (`spec/`, `docs/`, `adr/`,
    `security/` under the text licence; `tools/`, `conformance/`, `runner/`, `parity/`,
    `inventory/` under the code licence).
  - `CONTRIBUTING.md` — inbound licensing equals outbound licensing.
  - `package.json` — `"license"` set to the code SPDX identifier.
- **Out of scope:** any normative text; copyright headers in existing files; repository settings
  (the owner's).
- **Verification:**
  ```bash
  SPEC_ID=$(sed -nE 's/^- spec-text-licence: ([A-Za-z0-9.+-]+)$/\1/p' plan/memory/owner-inputs.md)
  CODE_ID=$(sed -nE 's/^- code-licence: ([A-Za-z0-9.+-]+)$/\1/p' plan/memory/owner-inputs.md)
  test -n "$SPEC_ID" && test -n "$CODE_ID"
  test -s "LICENSES/$SPEC_ID.txt" && test -s "LICENSES/$CODE_ID.txt"
  cmp -s LICENSE "LICENSES/$CODE_ID.txt"
  test "$(node -p "require('./package.json').license")" = "$CODE_ID"
  grep -qF "$SPEC_ID" README.md && grep -qF "$CODE_ID" README.md
  npm run verify
  ```
  Expected: every command exits 0. Before the goal `LICENSES/` does not exist.
- **Documentation:** `plan/memory/decisions.md` entry with the chosen licences and their scope.

## G03 — Area registry, spec skeletons and requirement linter
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G01
- **Goal:** Fix the machine-checkable shape of the specification: one file per area, the
  requirement item format, the linter that enforces it, JSON Schema validation, the port registry
  and the ADR format.
- **Deliverables:**
  - `tools/validate-json.mjs` — `node tools/validate-json.mjs <schema> <file>...`, Ajv in JSON
    Schema 2020-12 mode with `ajv-formats`; prints `validate-json: OK (<n> files)` or the errors
    and exits 1. `tools/validate-json.test.mjs`. Dev dependencies `ajv` (^8) and `ajv-formats` (^3).
  - `spec/areas.json` and `spec/areas.schema.json` — array of
    `{file, title, prefixes: [{prefix, title, appliesTo, profile}]}` (profiles themselves live in
    `spec/profiles.json`). Exactly these 17 entries:
    `00-overview.md` CNF (appliesTo `any`, profile `core`);
    `01-transport.md` TRN (`server`, `core`);
    `02-tokens.md` TOK (`server`, `core`);
    `03-sessions.md` SES (`server`, `core`);
    `04-password-and-email.md` PWD (`server`, `core`) and EML (`server`, `email`);
    `05-passwordless.md` MLK and OTP (`server`, `passwordless`);
    `06-two-factor.md` MFA (`server`, `two-factor`);
    `07-oauth-and-linking.md` OAU and LNK (`server`, `oauth`);
    `08-account-lifecycle.md` ACC (`server`, `core`);
    `09-admin.md` ADM (`server`, `admin`);
    `10-events-and-webhooks.md` EVT (`server`, `events`), WHK (`server`, `webhooks`),
    TOL (`server`, `tools`);
    `11-errors.md` ERR (`server`, `core`);
    `12-configuration.md` CFG (`server`, `core`);
    `13-identity-provider.md` IDP (`server`, `idp`) and RSV (`server`, `resource-server`);
    `14-clients.md` CLI (`client`, `client-core`);
    `15-abuse-protection.md` ABU (`server`, `core`);
    `16-api-keys.md` APK (`server`, `api-keys`).
  - `spec/profiles.json` and `spec/profiles.schema.json` — `[{name, kind: server|client|any,
    description, requires: [profile]}]` with the profiles `core`, `email`, `passwordless`,
    `two-factor`, `oauth`, `admin`, `api-keys`, `events`, `webhooks` (requires `events`), `tools`,
    `idp`, `oidc-provider` (requires `idp`), `resource-server`, `client-core`, `client-cookie` and
    `client-bearer` (both require `client-core`).
  - The 17 files `spec/<file>` as skeletons: `# <title>`, a line `Status: skeleton.`, and the
    headings `## Scope`, `## Requirements`, `## Reference behaviour`, `## Known divergences`,
    `## Security considerations`, `## Open questions`, each followed by `_To be written._`.
    `spec/00-overview.md` instead has `## Conventions`, `## Terminology`, `## Conformance model`,
    `## Profiles`, `## Requirement identifiers`, `## Requirements`, `## Versioning`.
  - `spec/retired-ids.json` — `[]`.
  - `tools/req-lint.mjs` — on `spec/*.md`, ignoring fenced code, inline code spans (so prose may
    name a level as `MUST`), HTML comments and regions between `<!-- bcp14:off -->` and
    `<!-- bcp14:on -->`:
    a requirement item is a line `- **<ID>** ` (ID `[A-Z]{3}-[0-9]{3}`), optionally followed by
    `{profile=<name>} `, plus its continuation lines (indented by at least two spaces);
    (1) an uppercase BCP 14 keyword (MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD,
    SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL) as a whole word appears only inside a
    requirement item; (2) requirement items appear only under `## Requirements` (including its
    `###` subsections); (3) each item holds at least one keyword, its level is the strongest one;
    (4) the prefix belongs to that file in `spec/areas.json`; (5) IDs are unique across files and
    not in `spec/retired-ids.json`; (6) a profile override exists in `spec/profiles.json`.
    Options: `--json` prints all requirements `{id, file, line, level, profile, appliesTo, text}`;
    `--min PREFIX=N` (repeatable) fails when a prefix has fewer than N requirements. Prints
    `req-lint: OK (<n> requirements)`. (Coverage options come in G21.)
  - `tools/req-lint.test.mjs` — fixtures in temporary directories for each rule and option.
  - `parity/ports.json` and `parity/ports.schema.json` — the 9 implementations of
    `plan/baseline.md` § Ports: `{id, repo, role: server|serverless|client, language,
    reference: boolean, surveyed: {commit (7 to 40 hex characters), version, date}}`; ids `node`,
    `go`, `python`, `rust`, `dart`, `lambda`, `flutter`, `angular`, `react`; only `node` has
    `reference: true`.
  - `adr/template.md` and `adr/0001-record-decisions.md`,
    `adr/0002-normative-language-and-requirement-ids.md`,
    `adr/0003-reference-default-and-security-baseline.md` (the spec is normative; the pinned
    reference is evidence and the default for wire shapes and defaults; the security baseline is
    independent of every port; plan rule 13), all three `Proposed` (the owner accepts them in
    G20). Every numbered ADR has a title `# NNNN. <title>`, the lines
    `- **Status:** <Proposed|Accepted|Rejected|Superseded by NNNN>` and `- **Date:** YYYY-MM-DD`,
    and the headings `## Context`, `## Decision`, `## Consequences`, `## Alternatives considered`.
  - `tools/headings.json` — entries for `spec/*.md` with `exclude: ["spec/00-overview.md"]`
    (area headings), `spec/00-overview.md` (overview headings) and `adr/*.md` with
    `exclude: ["adr/template.md"]`; `tools/check-headings.mjs` extended with the per-pattern
    `exclude` list and an optional per-pattern line regex (used for the ADR Status line).
  - `package.json` — scripts `lint:req` (`node tools/req-lint.mjs`) and `validate:registries`
    (validate-json of `spec/areas.json`, `spec/profiles.json`, `parity/ports.json`), both added to
    `verify`.
- **Out of scope:** normative requirements (no requirement item yet); vector format; coverage;
  inventory.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/req-lint.mjs); grep -Eq '^req-lint: OK \(0 requirements\)$' <<<"$out"
  node --test tools/req-lint.test.mjs tools/validate-json.test.mjs
  node tools/validate-json.mjs spec/areas.schema.json spec/areas.json
  node tools/validate-json.mjs spec/profiles.schema.json spec/profiles.json
  node tools/validate-json.mjs parity/ports.schema.json parity/ports.json
  test "$(node -p "require('./spec/areas.json').length")" = 17
  test "$(node -p "require('./spec/profiles.json').length")" = 16
  test "$(node -p "require('./parity/ports.json').length")" = 9
  for f in $(node -p "require('./spec/areas.json').map(a => 'spec/' + a.file).join(' ')"); do test -f "$f"; done
  test -f adr/template.md
  for a in 0001 0002 0003; do grep -qxF -- '- **Status:** Proposed' adr/$a-*.md; done
  ```
  Expected: every command exits 0. Before the goal `spec/areas.json` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `spec/`, `adr/`, `parity/ports.json`;
  `AGENTS.md` `## Writing rules` gives the requirement item format; `CHANGELOG.md`.

## G04 — Overview, conformance model and vision
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G03
- **Goal:** Write the overview every area builds on (conventions, terminology, conformance model,
  profiles, identifiers, versioning) and the long-form vision.
- **Deliverables:**
  - `spec/00-overview.md`:
    - `## Conventions` — the RFC 2119 and RFC 8174 boilerplate inside a
      `<!-- bcp14:off -->` … `<!-- bcp14:on -->` region; the requirement item format; "server",
      "client" and "port" as requirement subjects; "security baseline" as the set of requirements
      that every port meets regardless of the reference.
    - `## Terminology` — at least: port, reference implementation, server, serverless deployment,
      client, harness, runner, vector, profile, API prefix, cookie transport, bearer transport,
      access token, refresh token, temp token, admin token, one-time token, API key, session,
      first factor, second factor, trusted proxy, deviation.
    - `## Conformance model` — a port conforms to spec `MAJOR.MINOR` for a set of profiles when
      every requirement of level `MUST` in those profiles passes and no failure is undeclared;
      failures of `SHOULD` requirements are allowed as declared deviations; how declarations and
      reports relate (pointer to GOVERNANCE.md). Outside requirement items, levels are named only
      in code spans.
    - `## Profiles` — table written by hand from `spec/profiles.json` and `spec/areas.json`
      (name, kind, prefixes, requires, description).
    - `## Requirement identifiers` — format, never reused, retirement, profile override.
    - `## Requirements` — at least 8 CNF requirements: conformance claim wording
      ("conforms to awesome-lang-auth spec vX.Y, profiles: …"), declaration file, harness only in
      test builds, deviations declared with a public tracking link, and so on.
    - `## Versioning` — summary of GOVERNANCE.md § Spec versioning.
  - `docs/vision.md` — `## Problem` (facts from `plan/baseline.md`: the reference moves fast,
    ports track different baselines, no shared vectors, the existing prior art), `## Goals`,
    `## Non-goals` (including tenants, roles and permissions in 1.x), `## Principles`,
    `## How the family uses the spec`.
  - `tools/headings.json` — entry for `docs/vision.md`.
  - `spec/profiles.json` and `spec/areas.json` — adjustments only if the overview requires them
    (recorded in decisions.md).
- **Out of scope:** area files other than the overview; ADRs.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min CNF=8
  grep -q 'RFC 2119' spec/00-overview.md && grep -q 'RFC 8174' spec/00-overview.md
  grep -qxF '<!-- bcp14:off -->' spec/00-overview.md
  for p in $(node -p "require('./spec/profiles.json').map(p => p.name).join(' ')"); do grep -qF "\`$p\`" spec/00-overview.md; done
  for t in 'temp token' 'bearer transport' 'one-time token' 'trusted proxy' 'security baseline' 'harness' 'deviation'; do grep -qi "$t" spec/00-overview.md; done
  test -f docs/vision.md && grep -qF 'docs/vision.md' README.md
  ```
  Expected: every command exits 0; `req-lint` reports at least 8 requirements. Before the goal the
  CNF minimum fails.
- **Documentation:** `README.md` `## Repository layout` links `docs/vision.md` and
  `spec/00-overview.md`; `CHANGELOG.md`.

## G05 — Reference pin and route inventory
- **Status:** TODO
- **Type:** HUMAN
- **Depends on:** G03
- **Precondition:** A machine-readable inventory of the reference (per-route authentication and
  CSRF, token kinds and keys, verification rules) can reveal undisclosed weaknesses. The owner
  confirms that every private finding about awesome-node-auth 1.10.8 is either fixed in a
  published release with a published advisory, or judged safe to publish, by adding
  `- disclosure-cleared: node` (a comma-separated list of port ids) under `## Owner inputs` in
  `plan/memory/owner-inputs.md`. Command:
  `grep -Eq '^- disclosure-cleared: ([a-z]+, )*node(, [a-z]+)*$' plan/memory/owner-inputs.md`
- **Goal:** Pin the reference implementation (awesome-node-auth 1.10.8) and record every HTTP route
  it mounts as machine-readable, source-cited evidence.
- **Deliverables:**
  - `inventory/reference.json` — `{repo: "awesome-lang-auth/awesome-node-auth", package:
    "@awesome-lang-auth/node", version: "1.10.8", commit:
    "7e640b4681c1fc899ba4d9f4de0c9efb4ad8ae43", tag: "v1.10.8", date: "2026-09-29"}` and
    `inventory/schema/reference.schema.json`.
  - `tools/fetch-reference.sh` — full (not shallow) clone into
    `.cache/reference/awesome-node-auth`, reused when present; checks out `commit` (detached),
    never the tag; fails unless `HEAD` equals `commit`, `v1.10.8^{commit}` equals `commit` and
    `package.json` has version `1.10.8`. Prints `fetch-reference: OK <full SHA>`.
  - `inventory/schema/routes.schema.json` and `inventory/node-1.10.8/routes.json` — one entry per
    route registration: `router` (`auth`, `admin`, `tools`, `ui`, `docs`), `method` (upper case),
    `path` (Express syntax, relative to the router mount), `mount` (`<apiPrefix>`,
    `<apiPrefix>/admin`, `<toolsMount>`, `<apiPrefix>/ui`), `auth` (`none`, `access`,
    `access-optional`, `admin`, `api-key`, `temp-token`, `configurable`), `csrf` (`enforced`,
    `manual`, `not-enforced`), `mountedWhen` (conditions such as `sessionStore`,
    `!resourceServer.enabled`), `requestBody` (field names), `source` (`src/<file>:<line>`),
    `aliasOf` (optional, `"<METHOD> <path>"` of the canonical route, for duplicate registrations
    such as the admin `/api/users/…` and `/users/…` pairs), `stub` (optional boolean, for
    registrations that only answer "not configured"). Status codes and error codes per route are
    not recorded here (G06).
  - `inventory/node-1.10.8/extraction-exceptions.json` — `{routes: [{source, reason, route}]}`:
    registrations whose path is not a plain literal (template literals containing `${`, such as
    the `/oauth/${…}` routes, and identifier paths such as the JWKS path), each with the route it
    produces; and matches that are not routes (for example examples in comments), with
    `route: null`. Its schema is `inventory/schema/extraction-exceptions.schema.json`.
  - `tools/inventory-check.mjs` with `--routes` — extracts every
    `<identifier>.(get|post|put|patch|delete|all)(` whose first argument is a single- or
    double-quoted string literal starting with `/`, from
    `.cache/reference/awesome-node-auth/src/**/*.ts`; compares the **multiset** of
    (method, path, file) with `routes.json` entries that are not exceptions (the real handler and
    the not-configured stub of the same route are two entries), and checks every exception's
    `source` line; fails on a difference in either direction; checks that every `source` file
    exists with at least that many lines. Prints `inventory-check: OK (<n> routes)`.
  - `inventory/README.md` — what the inventory is (non-normative evidence), how to regenerate it,
    the pinned version and commit.
- **Out of scope:** normative text; other ports; any change to the reference repository; status
  and error codes per route (G06).
- **Verification:**
  ```bash
  out=$(bash tools/fetch-reference.sh); grep -Eq '^fetch-reference: OK 7e640b4681c1fc899ba4d9f4de0c9efb4ad8ae43$' <<<"$out"
  node -e "const r=require('./inventory/reference.json'); if(r.commit!=='7e640b4681c1fc899ba4d9f4de0c9efb4ad8ae43'||r.version!=='1.10.8') process.exit(1)"
  node tools/validate-json.mjs inventory/schema/reference.schema.json inventory/reference.json
  node tools/validate-json.mjs inventory/schema/routes.schema.json inventory/node-1.10.8/routes.json
  out=$(node tools/inventory-check.mjs --routes); grep -Eq '^inventory-check: OK \([0-9]+ routes\)$' <<<"$out"
  node -e "
  const r=require('./inventory/node-1.10.8/routes.json');
  if (r.length < 80) { console.error('too few routes', r.length); process.exit(1); }
  const need=[['auth','POST','/login'],['auth','POST','/refresh'],['auth','POST','/logout'],['auth','GET','/me'],['auth','POST','/2fa/verify'],['auth','POST','/magic-link/verify'],['auth','POST','/forgot-password'],['auth','DELETE','/account'],['auth','POST','/sessions/cleanup'],['admin','POST','/login'],['admin','POST','/logout']];
  for (const [ro,m,p] of need) if (!r.some(x => x.router===ro && x.method===m && x.path===p)) { console.error('missing', ro, m, p); process.exit(1); }
  if (!r.some(x => x.aliasOf)) process.exit(1);"
  npm run verify
  ```
  Expected: every command exits 0; at least 80 routes (the pinned commit has about 104 literal
  registrations). Before the goal `tools/fetch-reference.sh` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `inventory/README.md`;
  `plan/memory/decisions.md` entry on the multiset comparison and the exception format.

## G06 — Node inventory: errors and events
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G05
- **Goal:** Record every error answer of the reference with the routes that produce it, and every
  event name with where it is published.
- **Deliverables:**
  - Schemas `inventory/schema/errors.schema.json` and `inventory/schema/events.schema.json`.
  - `inventory/node-1.10.8/errors.json` — `[{code (string or null), message, status, routes:
    ["<router> <METHOD> <path>"], source: ["src/<file>:<line>"]}]`, including code-less errors
    such as the missing-access-token answer of the auth middleware.
  - `inventory/node-1.10.8/events.json` — `[{name, constant, emitted (boolean), emittedFrom:
    ["src/<file>:<line>"], data: [field]}]`, including names that are defined but never
    published.
  - `tools/inventory-check.mjs` — options `--errors` and `--events`: every non-null code appears as
    a string literal in the reference `src/`, and every string literal of `src/` matching
    `^[A-Z0-9]+(_[A-Z0-9]+)+$` is a code of `errors.json` or is listed, with a reason, in a new
    `literals` array of `inventory/node-1.10.8/extraction-exceptions.json` (for example
    `NODE_ENV`; schema updated); every
    `'identity.*'` string literal of `src/` is an event name and vice versa; every `routes` entry
    exists in `routes.json`; every `source` exists. Prints
    `inventory-check: OK (<n> routes, <e> errors, <v> events)` when called with
    `--routes --errors --events`.
  - `inventory/README.md` updated.
- **Out of scope:** normative text; other ports; cookies, headers, tokens, configuration (G07).
- **Verification:**
  ```bash
  bash tools/fetch-reference.sh > /dev/null
  for k in errors events; do node tools/validate-json.mjs inventory/schema/$k.schema.json inventory/node-1.10.8/$k.json; done
  out=$(node tools/inventory-check.mjs --routes --errors --events); grep -Eq '^inventory-check: OK \([0-9]+ routes, [0-9]+ errors, [0-9]+ events\)$' <<<"$out"
  node -e "
  const e=require('./inventory/node-1.10.8/errors.json'); const c=new Set(e.map(x=>x.code).filter(Boolean));
  for (const k of ['INVALID_CREDENTIALS','SESSION_REVOKED','CSRF_INVALID','INVALID_OAUTH_STATE','2FA_REQUIRED','2FA_SETUP_REQUIRED','USER_EXISTS','EMAIL_NOT_VERIFIED','EMAIL_VERIFICATION_REQUIRED','NOT_IMPLEMENTED','OAUTH_ORIGIN_ALLOWLIST_EMPTY']) if (!c.has(k)) { console.error('missing', k); process.exit(1); }
  if (c.size < 30) process.exit(1);
  if (!e.some(x => x.code === null && x.status === 403)) process.exit(1);"
  node -e "const v=require('./inventory/node-1.10.8/events.json'); if (v.length < 26 || !v.some(x => x.emitted === false) || !v.some(x => x.name === 'identity.user.deleted' && x.emitted)) process.exit(1)"
  npm run verify
  ```
  Expected: every command exits 0; at least 30 distinct codes (the pinned commit has about 47) and
  26 event names. Before the goal `inventory/node-1.10.8/errors.json` does not exist.
- **Documentation:** `inventory/README.md`; `plan/memory/decisions.md` for every place where the
  inventory contradicts `plan/baseline.md`.

## G07 — Node inventory: cookies, headers, tokens and configuration
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G06
- **Goal:** Complete the reference evidence with cookies, headers, token kinds and configuration
  defaults, and check the inventory against the pinned clone in CI.
- **Deliverables:**
  - Schemas `inventory/schema/{cookies,headers,tokens,config}.schema.json`.
  - `inventory/node-1.10.8/cookies.json` — `[{name, prefixedVariants, httpOnly, sameSite, secure,
    path, maxAge, setBy, readBy, source}]` (accessToken, refreshToken, csrf-token,
    oauth_nonce_<provider>, the admin cookie).
  - `inventory/node-1.10.8/headers.json` — request and response headers with meaning and source
    (Authorization, X-Auth-Strategy, X-CSRF-Token, X-Api-Key, X-Correlation-Id, the
    X-Webhook-* headers, CORS headers, Cache-Control).
  - `inventory/node-1.10.8/tokens.json` — per kind (`access`, `refresh`, `temp`, `admin`,
    `idp-access`, `idp-refresh`, `oauth-state`): algorithm, key or secret used, default lifetime,
    claims, reserved claims, verification rules, source.
  - `inventory/node-1.10.8/config.json` — `[{path, type, default, source, notes}]` for AuthConfig
    and the router options.
  - `tools/inventory-check.mjs` `--all` — routes, errors, events, plus: the last segment of every
    config path appears as an identifier in the reference `src/`; every cookie name appears as a
    string literal; every `source` exists. Prints
    `inventory-check: OK (<n> routes, <e> errors, <v> events, <k> cookies, <c> config)`.
  - `package.json` — script `verify:inventory` (`bash tools/fetch-reference.sh && node
    tools/inventory-check.mjs --all`), **not** part of `verify` (plan rule 11).
  - `.github/workflows/inventory.yml` — on `push` and `pull_request` touching `inventory/**` or
    `tools/inventory-check.mjs` or `tools/fetch-reference.sh`, and weekly: checkout, setup-node,
    `actions/cache` on `.cache/reference` keyed by the pinned commit, `npm ci`,
    `npm run verify:inventory`; `permissions: contents: read`.
  - `inventory/README.md` updated.
- **Out of scope:** normative text; other ports.
- **Verification:**
  ```bash
  bash tools/fetch-reference.sh > /dev/null
  for k in cookies headers tokens config; do node tools/validate-json.mjs inventory/schema/$k.schema.json inventory/node-1.10.8/$k.json; done
  out=$(node tools/inventory-check.mjs --all); grep -Eq '^inventory-check: OK \([0-9]+ routes, [0-9]+ errors, [0-9]+ events, [0-9]+ cookies, [0-9]+ config\)$' <<<"$out"
  node -e "const t=require('./inventory/node-1.10.8/tokens.json'); for (const k of ['access','refresh','temp','admin']) if (!t.some(x => x.kind === k)) process.exit(1)"
  node -e "const c=require('./inventory/node-1.10.8/config.json'); for (const p of ['accessTokenExpiresIn','refreshTokenExpiresIn','apiPrefix','cookieOptions.secure','cookieOptions.sameSite','csrf.enabled','emailVerificationMode','bcryptSaltRounds']) if (!c.some(x => x.path === p)) { console.error('missing', p); process.exit(1); }"
  node -e "const k=require('./inventory/node-1.10.8/cookies.json'); for (const n of ['accessToken','refreshToken','csrf-token']) if (!k.some(x => x.name === n)) process.exit(1)"
  grep -qF 'inventory-check.mjs --all' package.json && grep -qF 'verify:inventory' .github/workflows/inventory.yml
  if node -e "process.exit(require('./package.json').scripts.verify.includes('inventory') ? 0 : 1)"; then exit 1; fi
  npm run verify
  ```
  Expected: every command exits 0. Before the goal `inventory/node-1.10.8/tokens.json` does not
  exist.
- **Documentation:** `inventory/README.md`; `plan/memory/decisions.md` for every place where the
  inventory contradicts `plan/baseline.md`.

## G08 — Spec: errors
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G04, G07
- **Goal:** Specify the error envelope, the status mapping and the code catalogue that every later
  area uses.
- **Deliverables:**
  - `spec/11-errors.md` — at least 15 ERR requirements; `## Reference behaviour` holds a
    `### Error code catalogue` subsection with the table `| Code | Status | Meaning | Aliases |`
    covering every non-null code of `inventory/node-1.10.8/errors.json` plus the codes this goal
    introduces (`NO_ACCESS_TOKEN`, `INVALID_BODY`, `RATE_LIMITED`, `WEAK_PASSWORD` at least).
    Reference default: the `{error, code}` envelope and the statuses of the inventory.
    Content required: `code` is the only machine-readable part, the `error` message text is
    non-normative; validation failures answer 400 with `INVALID_BODY` (including an empty or
    non-JSON body); 501 `NOT_IMPLEMENTED` when an optional store or delivery channel is absent;
    429 `RATE_LIMITED`; OIDC endpoints use RFC 6749 §5.2 bodies; unknown fields in an error body
    are ignored by clients.
  - ADRs, status `Proposed`:
    - `adr/0004-error-envelope-and-codes.md` — every error answer of the auth, admin, API-key and
      tools surfaces is `{error, code}` with `code` required; message text non-normative; 400 with
      `INVALID_BODY` for validation errors (context: FastAPI's 422 in awesome-python-auth,
      awesome-go-auth #34); 501 for absent optional features; RFC 6749 bodies for OIDC.
    - `adr/0005-missing-or-invalid-access-token-status.md` — 401 with `NO_ACCESS_TOKEN` or
      `INVALID_ACCESS_TOKEN` instead of the code-less 403 recorded in the inventory; clients keep
      treating a code-less 403 as refreshable until spec 2.0 (awesome-flutter-auth #33 asks for a
      code).
  - `tools/check-codes.mjs` — the catalogue is the code column of the table above; every
    backticked token matching `^[A-Z0-9]+(_[A-Z0-9]+)+$` in `spec/*.md` is in the catalogue (code
    or alias) or in `tools/check-codes-allow.json`; every non-null inventory code is in the
    catalogue. Prints `check-codes: OK (<n> codes)`. `tools/check-codes.test.mjs`;
    `tools/check-codes-allow.json`; `package.json` script `lint:codes` added to `verify`.
  - Known divergences from `plan/baseline.md` § Errors.
- **Out of scope:** transport (G09) and every other area file except the catalogue rows of plan
  rule 9.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min ERR=15
  out=$(node tools/check-codes.mjs); grep -Eq '^check-codes: OK \([0-9]+ codes\)$' <<<"$out"
  grep -qxF '### Error code catalogue' spec/11-errors.md
  for a in 0004 0005; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for c in NO_ACCESS_TOKEN INVALID_ACCESS_TOKEN INVALID_BODY RATE_LIMITED SESSION_REVOKED CSRF_INVALID NOT_IMPLEMENTED; do grep -qF "\`$c\`" spec/11-errors.md; done
  grep -qi 'non-normative' spec/11-errors.md && grep -qF 'RFC 6749' spec/11-errors.md
  for r in awesome-python-auth awesome-dart-auth; do grep -qF "$r" spec/11-errors.md; done
  ```
  Expected: every command exits 0. Before the goal the ERR minimum fails.
- **Documentation:** `CHANGELOG.md`; `plan/memory/decisions.md` for each alias choice
  (for example `INVALID_OAUTH_STATE` versus awesome-go-auth's `OAUTH_STATE_INVALID`).

## G09 — Spec: transport and abuse protection
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G08
- **Goal:** Specify cookie and bearer delivery, cookie names and attributes, CSRF, CORS and
  response headers, and the abuse protections every server applies: rate limits, lockout and
  client IP derivation.
- **Deliverables:**
  - `spec/01-transport.md` — at least 25 TRN requirements.
    Reference default: cookie names `accessToken`, `refreshToken`, `csrf-token`; prefix rules
    (`__Host-` when Secure with no Domain and Path `/`, `__Secure-` when Secure otherwise, bare when
    not Secure) and read priority `__Host-` > `__Secure-` > bare; `X-Auth-Strategy: bearer`
    returning tokens in the body and setting no cookie; `Authorization: Bearer` taking precedence
    over the cookie; refresh cookie Path `<apiPrefix>/refresh` except under `__Host-`; Max-Age
    equal to the token lifetime; SameSite default `Lax`; OAuth callbacks deliver cookies; CORS
    allow-list echoing allowed origins with credentials and allowing `X-Auth-Strategy`.
    Departures (with their ADRs): CSRF on by default with cookie transport; Secure cookies by
    default; whether a `GET /csrf` endpoint is required or optional.
    Security baseline: CSRF is enforced on every state-changing request that a cookie credential
    authenticates, whatever other headers it carries (the bearer opt-in never disables it); the
    CSRF token is compared in constant time; `Cache-Control: no-store` on every response that
    carries a token or a secret; state-changing JSON endpoints require
    `Content-Type: application/json`; SameSite `None` only with Secure; a deprecated route alias,
    when provided, behaves exactly as its canonical route.
  - `spec/15-abuse-protection.md` — at least 12 ABU requirements: 429 `RATE_LIMITED` with
    `Retry-After`; per-account and per-client-IP throttles on password login, admin login, 2FA and
    OTP verification, magic-link and SMS send, forgot-password, register and API-key
    authentication; a lockout or back-off policy that does not let an attacker lock a victim out
    indefinitely; uniform timing on credential checks; client IP derivation: `X-Forwarded-For` and
    `Forwarded` honoured only from configured trusted proxies, otherwise the socket peer address;
    the same derived IP feeds rate limits and the `ip` field of events.
  - ADRs, status `Proposed`:
    - `adr/0006-csrf-default-and-exemptions.md` — CSRF on by default with cookie transport; the
      exemption table pinned by awesome-go-auth's `TestCSRFExemptionsMatchTheReference` as the
      starting point, checked against `inventory/node-1.10.8/routes.json`; the exemption keys on
      how the request is authenticated; `GET /csrf` optional (awesome-go-auth has none).
    - `adr/0007-secure-cookie-default.md` — Secure by default; insecure cookies only through an
      explicit development opt-in.
    - `adr/0008-rate-limiting-and-client-ip.md` — required throttled operations, `Retry-After`,
      trusted-proxy configuration (context: awesome-go-auth and awesome-lambda-auth already answer
      429 `RATE_LIMITED`).
  - Known divergences from `plan/baseline.md` § Transport.
- **Out of scope:** token contents and sessions (G10); client behaviour (G18); configuration names
  (G16) beyond those the requirements need; other area files.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min TRN=25 --min ABU=12
  for a in 0006 0007 0008; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for r in awesome-node-auth awesome-go-auth awesome-python-auth awesome-lambda-auth awesome-dart-auth; do grep -qF "$r" spec/01-transport.md; done
  for t in 'X-Auth-Strategy' '__Host-' 'no-store' 'application/json' 'SameSite' 'alias'; do grep -qF "$t" spec/01-transport.md; done
  grep -qF 'Retry-After' spec/15-abuse-protection.md && grep -qF '`RATE_LIMITED`' spec/15-abuse-protection.md
  grep -qi 'trusted prox' spec/15-abuse-protection.md && grep -qF 'X-Forwarded-For' spec/15-abuse-protection.md
  ```
  Expected: every command exits 0. Before the goal the TRN and ABU minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G10 — Spec: tokens and sessions
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G09
- **Goal:** Specify token kinds, claims, keys, validation and lifetimes, and the session model:
  refresh, rotation, revocation, device list and logout.
- **Deliverables:**
  - `spec/02-tokens.md` — at least 22 TOK requirements. Reference default: HS256 session tokens;
    claims `sub`, `email`, `role`, `loginProvider`, `isEmailVerified`, `isTotpEnabled`, `sid`, plus
    application claims that cannot override reserved ones; access 15 minutes and refresh 7 days by
    default; temp token 5 minutes; admin token 24 hours.
    Security baseline: a `typ` claim on every token and verifiers that reject any other `typ` (no
    refresh, temp, admin, state or IdP token is ever accepted as an access token, and the
    reverse); the algorithm pinned per token kind before verification, `alg: none` rejected;
    `exp` and `iat` required, `nbf` honoured, a bounded clock skew; distinct keys per token
    purpose; no secret shorter than 32 bytes; the decision on `iss` and `aud`; the decision on
    whether the refresh token is opaque or a JWT.
  - `spec/03-sessions.md` — at least 22 SES requirements. Reference default: refresh from the
    body in bearer transport and from the cookie in cookie transport; `sid` per session;
    `checkOn` modes `allcalls`, `refresh` (default) and `none`, with `allcalls` answering 401
    `SESSION_REVOKED`; device list `GET /sessions` and `DELETE /sessions/:handle`;
    `singleSessionPerUser`.
    Security baseline: rotation on every refresh; reuse of a rotated-out refresh token revokes the
    session; logout revokes the server-side session; reset-password revokes all sessions,
    change-password and change-email confirmation revoke every other session; an absolute
    session lifetime besides the sliding refresh; the session list returns only an allow-listed
    set of fields; `DELETE /sessions/:handle` acts only on the caller's own sessions and answers
    404 for any other handle; session cleanup is not callable without admin authorization;
    refresh tokens are stored only as hashes.
  - ADRs, status `Proposed`: `adr/0009-token-types-claims-and-validation.md` (`typ`, keys,
    algorithm pinning, `iss`/`aud`, time claims and skew, opaque or JWT refresh; context:
    awesome-go-auth #25 and PR #97, the documented awesome-lambda-auth refuse-to-start rule on equal
    secrets, RFC 8725), `adr/0010-refresh-rotation-and-reuse-detection.md` (context: the
    documented awesome-lambda-auth refresh families, RFC 9700),
    `adr/0011-revocation-on-credential-change-and-logout.md`,
    `adr/0012-session-check-default-and-absolute-lifetime.md` (keep `refresh` as default; state the
    revocation latency).
  - Known divergences from `plan/baseline.md` § Tokens and § Sessions.
- **Out of scope:** IdP tokens and JWKS (G17); 2FA temp-token flows beyond the token definition
  (G12); client storage (G18).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min TOK=22 --min SES=22
  for a in 0009 0010 0011 0012; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for t in '`typ`' '`sid`' '`iss`' '`aud`' '`exp`' 'alg' 'skew' 'opaque'; do grep -qF "$t" spec/02-tokens.md; done
  for t in '`SESSION_REVOKED`' 'allcalls' '404' 'absolute' 'reuse' 'logout'; do grep -qiF "$t" spec/03-sessions.md; done
  grep -qF 'awesome-go-auth' spec/02-tokens.md && grep -qF 'awesome-lambda-auth' spec/03-sessions.md
  ```
  Expected: every command exits 0. Before the goal the TOK and SES minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G11 — Spec: password, email and account lifecycle
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G10
- **Goal:** Specify password login, registration, reset and change, password storage, email
  verification and email change, the profile and `/me`, re-authentication for sensitive
  operations, and account deletion with its hooks.
- **Deliverables:**
  - `spec/04-password-and-email.md` — at least 15 PWD and 12 EML requirements. Reference default:
    login answers and codes; forgot-password always 200 `{success:true}`; reset token 1 hour,
    verification token 24 hours, email-change token 1 hour; `emailVerificationMode` `none`
    (default), `lazy` and `strict` with `EMAIL_NOT_VERIFIED` and `EMAIL_VERIFICATION_REQUIRED`;
    change-email request and confirmation with a notice to the old address; change-password
    without `currentPassword` only for accounts without a password (awesome-go-auth #30).
    Security baseline: one-time tokens from a CSPRNG with at least 128 bits, single use, bound to
    one purpose and stored as hashes; uniform answers on every unauthenticated "send" endpoint;
    a minimum password policy applied on register, reset and change (`WEAK_PASSWORD`); password
    hashing with an adaptive algorithm and a minimum cost, a maximum accepted length and the
    handling of algorithms that truncate (bcrypt's 72-byte limit), Unicode normalisation of
    passwords; links in emails built only from the configured public base URL, never from `Host`
    or `X-Forwarded-Host`.
  - `spec/08-account-lifecycle.md` — at least 15 ACC requirements: register (mounted only when
    enabled; 201 `{success, userId}`; no session unless `issueSessionOnRegister`), `/me`,
    `PATCH /profile`, `POST /add-phone`, `DELETE /account` with `onBeforeDeleteUser`.
    Security baseline: re-authentication (current password, or a second factor for accounts
    without a password, within a bounded time) before changing email, changing password,
    deleting the account and disabling 2FA; the decision on `USER_EXISTS` against
    anti-enumeration.
  - ADRs, status `Proposed`: `adr/0013-register-response-session-and-enumeration.md` (context:
    awesome-go-auth #21 and PR #98, awesome-python-auth PR #14, awesome-rust-auth PR #15,
    awesome-dart-auth PR #15, the documented awesome-lambda-auth deviation; `USER_EXISTS` versus a
    uniform answer completed by email), `adr/0014-account-deletion-pipeline.md` (the hook is
    awaited and may veto with an explicit status; deletion revokes sessions and removes linked
    accounts, API keys and every other datum the server holds about the user;
    `identity.user.deleted` is published; admin deletion uses the same pipeline with
    `source: "admin"`; context: awesome-node-auth #20), `adr/0015-password-storage-and-policy.md`,
    `adr/0016-re-authentication-for-sensitive-operations.md`.
  - Known divergences from `plan/baseline.md` § Password and email and § Account lifecycle.
- **Out of scope:** magic link, SMS and 2FA (G12); admin routes (G14).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min PWD=15 --min EML=12 --min ACC=15
  for a in 0013 0014 0015 0016; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  grep -qF 'onBeforeDeleteUser' spec/08-account-lifecycle.md && grep -qF 'issueSessionOnRegister' spec/08-account-lifecycle.md
  grep -qF '`USER_EXISTS`' spec/08-account-lifecycle.md && grep -qi 're-authenticat' spec/08-account-lifecycle.md
  for t in '`lazy`' '`strict`' '`WEAK_PASSWORD`' '72' 'base URL' 'X-Forwarded-Host'; do grep -qF "$t" spec/04-password-and-email.md; done
  grep -qi 'normali' spec/04-password-and-email.md
  grep -qF 'awesome-go-auth' spec/08-account-lifecycle.md
  ```
  Expected: every command exits 0. Before the goal the PWD, EML and ACC minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G12 — Spec: passwordless and two-factor
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G11
- **Goal:** Specify magic links, one-time codes by SMS, TOTP enrolment and verification, the 2FA
  challenge and step-up modes.
- **Deliverables:**
  - `spec/05-passwordless.md` — at least 10 MLK and 10 OTP requirements. Reference default:
    `POST /magic-link/send` and `/verify` with `mode` `login` or `2fa`; 15-minute links; SMS codes
    of 6 digits valid 10 minutes; `POST /sms/send` and `/verify`; first magic-link login marks the
    email verified.
  - `spec/06-two-factor.md` — at least 18 MFA requirements. Reference default: the login
    challenge `200 {requiresTwoFactor: true, tempToken, available2faMethods}` with methods `totp`,
    `sms`, `magic-link`; forced enrolment `403 {requires2FASetup: true, tempToken, code:
    "2FA_SETUP_REQUIRED"}`; `POST /2fa/setup` returning `secret` and `otpauthUrl` (`qrCode`
    optional: awesome-go-auth and awesome-lambda-auth omit it, awesome-flutter-auth #22 made it
    optional); `POST /2fa/verify-setup`, `/2fa/verify`, `/2fa/disable` with `2FA_REQUIRED`; TOTP per
    RFC 6238 with one step of tolerance.
    Security baseline: the second factor applies to every route that opens a session from a first
    factor (password, magic link, SMS, OAuth callback, link verification with login); codes come
    from a CSPRNG; numeric codes are stored as a keyed hash (HMAC with a server key), never as a
    plain or unkeyed hash; a code is invalidated after at most 5 failed attempts; a TOTP time step
    is never accepted twice for the same account; the temp token is single use and bound to the
    user and its purpose, and the 2FA verification accepts only a valid temp token, never a bare
    user identifier; the TOTP secret is generated and kept by the server and verify-setup never
    accepts a secret from the client; disabling 2FA requires a current second-factor code
    (ADR 0016).
  - ADRs, status `Proposed`: `adr/0017-second-factor-on-every-first-factor-login.md` (context: the
    reference documentation states that magic-link and SMS logins skip TOTP; awesome-lambda-auth
    documents the OAuth callback skipping 2FA as a deviation),
    `adr/0018-one-time-code-storage-attempts-and-replay.md`.
  - Known divergences from `plan/baseline.md` § Passwordless and two-factor.
- **Out of scope:** OAuth (G13); client mapping of the challenge (G18).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min MLK=10 --min OTP=10 --min MFA=18
  for a in 0017 0018; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for t in 'available2faMethods' '`2FA_SETUP_REQUIRED`' 'RFC 6238' 'HMAC' 'verify-setup' 'temp token' 'awesome-flutter-auth'; do grep -qF "$t" spec/06-two-factor.md; done
  grep -qF 'HMAC' spec/05-passwordless.md
  ```
  Expected: every command exits 0. Before the goal the MLK, OTP and MFA minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G13 — Spec: OAuth and account linking
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G12
- **Goal:** Specify the OAuth client flow towards upstream providers (state, PKCE, return path,
  redirect allowlist, provisioning and email-verification policy) and account linking.
- **Deliverables:**
  - `spec/07-oauth-and-linking.md` — at least 18 OAU and 10 LNK requirements. Reference default:
    `GET /oauth/:provider?return_path=` and `/callback`; a signed state with a 10-minute expiry
    bound to a nonce cookie `oauth_nonce_<provider>` scoped to the callback path, to the origin
    and to the return path; return-path syntax rules (single leading `/`, no backslash or control
    characters, at most 512 characters, optional allowlist); redirect origins from the site URLs
    and CORS origins; the account-conflict redirect; codes `INVALID_OAUTH_STATE`,
    `OAUTH_RETURN_PATH_INVALID`, `OAUTH_ORIGIN_ALLOWLIST_EMPTY`, `OAUTH_ACCOUNT_CONFLICT`,
    `OAUTH_PROFILE_FAILED`; linking routes `GET /linked-accounts`,
    `DELETE /linked-accounts/:provider/:providerAccountId`, `POST /link-request`,
    `POST /link-verify`.
    Security baseline: the state nonce compared in constant time and single use; the state
    signing key distinct from every token key; PKCE S256 towards providers that support it;
    OAuth start and every post-login redirect refused when the redirect allowlist is empty, in
    every environment; no automatic linking by email unless the provider asserts a verified email
    (`email_verified` or the provider's equivalent) and the policy allows it, with the conflict
    flow as the default; link verification that opens a session goes through the second factor
    (G12).
  - ADRs, status `Proposed`: `adr/0019-pkce-towards-providers.md` (RFC 7636, RFC 9700),
    `adr/0020-email-verification-policy-for-linking.md` (context: awesome-go-auth #36 and PR #99),
    `adr/0021-redirect-allowlist-required.md` (context: awesome-lambda-auth documents a refusal to
    start without one).
  - `## Open questions` includes whether the 2FA redirect after an OAuth callback may carry the temp
    token in the URL.
  - Known divergences from `plan/baseline.md` § OAuth.
- **Out of scope:** the OIDC provider role (G17); client OAuth helpers (G18).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min OAU=18 --min LNK=10
  for a in 0019 0020 0021; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for t in 'PKCE' 'return_path' '`INVALID_OAUTH_STATE`' 'email_verified' 'awesome-go-auth'; do grep -qF "$t" spec/07-oauth-and-linking.md; done
  ```
  Expected: every command exits 0. Before the goal the OAU and LNK minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G14 — Spec: admin console and API keys
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G11
- **Goal:** Specify the admin API surface and its access control, and API-key authentication with
  scopes, IP allow-lists and expiry.
- **Deliverables:**
  - `spec/09-admin.md` — at least 15 ADM requirements: admin login and token, access policies
    (`is-admin-flag`, `first-user`, `open`, predicate), a table of the admin API routes a port
    claiming `admin` provides (from `inventory/node-1.10.8/routes.json`, with the duplicate
    `/api/…` registrations listed as aliases), pagination.
    Security baseline: the admin credential is distinct from the application session and its
    cookie name never collides with an application cookie; CSRF on cookie-authenticated admin
    mutations; `open` only outside production; admin login is rate-limited (ABU) and SHOULD
    support a second factor; admin authorization is evaluated against the current stored state of
    the user on every request, never against a role or flag cached in a token; admin secrets are
    compared in constant time.
  - `spec/16-api-keys.md` — at least 10 APK requirements. Reference default: the `X-Api-Key`
    header, the API-key routes and codes of the inventory (`API_KEY_MISSING`, `API_KEY_INVALID`,
    `API_KEY_EXPIRED`, `API_KEY_REVOKED`, `API_KEY_INSUFFICIENT_SCOPE`, `API_KEY_IP_BLOCKED`).
    Security baseline: keys come from a CSPRNG, are shown once and stored only as hashes; scopes,
    IP allow-lists, expiry and revocation are enforced on every request; keys are removed by the
    account deletion pipeline (ADR 0014).
  - ADRs, status `Proposed`: `adr/0022-admin-authentication-and-authorization.md` (context:
    awesome-lambda-auth documents its refusal of `first-user`; awesome-go-auth documents its
    refusal to mount without a policy), `adr/0023-api-keys-profile.md`.
  - Known divergences from `plan/baseline.md` § Admin.
- **Out of scope:** UI pages and admin SPA assets; account deletion pipeline details (G11);
  tenants, roles and permissions (G15, ADR 0027).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min ADM=15 --min APK=10
  for a in 0022 0023; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for t in 'is-admin-flag' 'first-user' 'CSRF' 'awesome-lambda-auth'; do grep -qF "$t" spec/09-admin.md; done
  for t in 'X-Api-Key' '`API_KEY_INVALID`' '`API_KEY_IP_BLOCKED`' 'hash'; do grep -qF "$t" spec/16-api-keys.md; done
  ```
  Expected: every command exits 0. Before the goal the ADM and APK minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G15 — Spec: events, webhooks and tools
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G14
- **Goal:** Specify the `identity.*` event catalogue and payload, outgoing and inbound webhooks, the
  tools router and the event stream, and record that tenants, roles and permissions stay outside
  spec 1.x.
- **Deliverables:**
  - `spec/10-events-and-webhooks.md` — at least 15 EVT, 10 WHK and 8 TOL requirements: canonical
    `identity.*` names with the operation that publishes each; payload fields `event`,
    `timestamp`, `userId`, `tenantId`, `sessionId`, `data`, `ip`, `userAgent`, `correlationId`;
    correlation-id pattern; outgoing webhook headers `X-Webhook-Event`, `X-Webhook-Delivery`,
    `X-Webhook-Timestamp`, `X-Webhook-Signature: sha256=<hex>` and what is signed (from the
    inventory); retry schedule; tools routes `track`, `notify`, `stream`, `telemetry`, `webhook`.
    The names defined but never published by the reference become required in the `events`
    profile, except the tenant, role and permission names, which are reserved (ADR 0027).
    Security baseline: the webhook signature covers the timestamp and receivers reject deliveries
    outside a stated tolerance; outgoing webhook targets and every other server-side fetch are
    protected against SSRF (no loopback, private or link-local destination unless explicitly
    allowed, redirects re-checked); the tools router is never mounted without explicit access
    control; `track` and `notify` take the identity from the authenticated principal, never from
    the body; inbound webhooks verify a signature; inbound scripts never run in the server
    process; the stream credential is not a long-lived token in the URL.
  - ADRs, status `Proposed`: `adr/0024-event-catalogue-and-emission.md` (context:
    awesome-rust-auth #12, awesome-dart-auth #12), `adr/0025-tools-router-access.md` (context:
    awesome-node-auth #10), `adr/0026-event-stream-credential.md` (a short-lived, single-use
    stream ticket obtained with the normal credential),
    `adr/0027-tenants-roles-and-permissions-outside-1x.md` (event names and the `tid` claim
    reserved; tenant, role and permission APIs not specified in 1.x; deletion still removes the
    related data per ADR 0014).
  - Known divergences from `plan/baseline.md` § Events and webhooks.
- **Out of scope:** admin (G14); configuration names (G16).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min EVT=15 --min WHK=10 --min TOL=8
  for a in 0024 0025 0026 0027; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  for e in $(node -p "require('./inventory/node-1.10.8/events.json').map(e => e.name).join(' ')"); do grep -qF "\`$e\`" spec/10-events-and-webhooks.md; done
  for t in 'X-Webhook-Signature' 'X-Webhook-Timestamp' 'SSRF' 'tolerance' 'stream' 'awesome-rust-auth'; do grep -qF "$t" spec/10-events-and-webhooks.md; done
  ```
  Expected: every command exits 0; every reference event name appears in the events file. Before
  the goal the EVT, WHK and TOL minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G16 — Spec: configuration and secure defaults
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G13, G15
- **Goal:** Specify the configuration surface with its secure defaults and refuse-to-start rules,
  and the cross-cutting rules on secrets.
- **Deliverables:**
  - `spec/12-configuration.md` — at least 25 CFG requirements: canonical option names (the
    reference's AuthConfig names), a defaults table built from `inventory/node-1.10.8/config.json`,
    the default API prefix `/auth`, development opt-ins that are explicit and logged at start.
    Security baseline: refuse to start on a secret shorter than 32 bytes, on equal secrets for
    different token purposes, on a well-known default secret (a normative list of placeholder
    values, such as `secret`, `changeme`, `password`, and the empty string), on SameSite
    `None` without Secure, on cookie transport without CSRF unless explicitly opted out, on OAuth
    without a redirect allowlist, on an admin console without an access policy, on a tools router
    without access control, and on missing trusted-proxy configuration when forwarded headers are
    honoured; IdP mode off by default; every comparison of a secret, token, code, API key or admin
    secret in constant time; tokens, codes, cookies, API keys and delivery-gateway credentials never
    logged and never placed in URLs (except protocol parameters that a cited RFC requires).
  - ADR, status `Proposed`: `adr/0028-secure-defaults-and-refuse-to-start.md` (context: the
    documented awesome-lambda-auth rules RS-1 to RS-18 and awesome-go-auth mount refusals).
  - Known divergences from `plan/baseline.md` § Configuration.
- **Out of scope:** deployment tooling of any port; client configuration (G18); IdP options (G17).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min CFG=25
  grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/0028-*.md
  for t in '32 bytes' '`changeme`' '`/auth`' 'awesome-lambda-auth'; do grep -qF "$t" spec/12-configuration.md; done
  grep -Eqi 'constant[- ]time' spec/12-configuration.md && grep -qi 'logged' spec/12-configuration.md
  ```
  Expected: every command exits 0. Before the goal the CFG minimum fails.
- **Documentation:** `CHANGELOG.md`.

## G17 — Spec: identity provider and resource server
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G16
- **Goal:** Specify the identity-provider mode (signing keys, JWKS, the optional OIDC provider) and
  resource-server verification.
- **Deliverables:**
  - `spec/13-identity-provider.md` — at least 18 IDP and 10 RSV requirements, of which at least 12
    carry `{profile=oidc-provider}`. Reference default: JWKS at
    `<apiPrefix>/.well-known/jwks.json` (path configurable) with CORS and
    `Cache-Control: max-age=3600`; RS256.
    Security baseline: the JWKS publishes only public asymmetric keys, never symmetric keys or
    material derived from a secret; tokens issued in IdP mode are verifiable with the published
    JWKS; several keys for rotation with stable `kid` values; access and refresh tokens
    distinguishable by `typ`; every token-minting endpoint requires client authentication; IdP
    mode is off by default (G16).
    `oidc-provider` profile: discovery with the required metadata fields; exact `redirect_uri`
    match; `/authorize` with PKCE S256 required; a single-use authorization code with a short
    lifetime, and revocation of the tokens issued from a code that is presented twice; `/token`
    with the `authorization_code` and `refresh_token` grants; single-use refresh tokens with family
    revocation; `client_secret_basic` and `client_secret_post`; an `id_token` with `iss`, `sub`,
    `aud`, `exp`, `iat` and the request `nonce`; `/userinfo`; RFC 6749 §5.2 errors.
    RSV: JWKS fetching and caching, `kid` refetch limits, algorithm pinned before key lookup,
    issuer and audience checks, SSRF protection on JWKS and discovery URLs.
  - ADRs, status `Proposed`: `adr/0029-idp-tokens-and-key-rotation.md` (context: the documented
    awesome-lambda-auth `kid` derivation and `kmsPreviousKeyIds`),
    `adr/0030-oidc-provider-profile.md` (context: awesome-node-auth #36, awesome-go-auth #14 and
    PR #97, awesome-lambda-auth #20; OpenID Connect Core 1.0, RFC 6749, RFC 7636, RFC 9700).
  - Known divergences from `plan/baseline.md` § Identity provider and resource server.
- **Out of scope:** deployment tooling; certification against OpenID Connect.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min IDP=18 --min RSV=10
  for a in 0029 0030; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  node tools/req-lint.mjs --json | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const r=JSON.parse(s); if(r.filter(x=>x.profile==='oidc-provider').length<12) process.exit(1)})"
  for t in 'jwks.json' '`kid`' 'id_token' 'nonce' 'redirect_uri' 'SSRF' 'public' 'awesome-go-auth'; do grep -qF "$t" spec/13-identity-provider.md; done
  ```
  Expected: every command exits 0; at least 12 requirements carry the `oidc-provider` profile.
  Before the goal the IDP and RSV minimums fail.
- **Documentation:** `CHANGELOG.md`.

## G18 — Spec: client contract
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G13
- **Goal:** Specify what a client library does on the wire: transports, CSRF, refresh and retry,
  revocation, token storage, the result and event model, the 2FA challenge and OAuth helpers.
- **Deliverables:**
  - `spec/14-clients.md` — at least 25 CLI requirements, with `{profile=client-cookie}` and
    `{profile=client-bearer}` overrides where they apply. Reference behaviour: the three clients
    surveyed (awesome-flutter-auth 1.10.5, awesome-angular-auth 1.10.0 on `develop`,
    awesome-react-auth 0.1.0) and the bundled `auth.js` of the reference. Required content: the
    endpoint and body mapping of every client action; CSRF cookie lookup priority and the
    `X-CSRF-Token` header; bearer transport with `X-Auth-Strategy: bearer`; single-flight refresh
    and one retry; `SESSION_REVOKED` ends the session without a refresh; the result shape
    (`success`, `error`, `code`, `status`); events (signed in, signed out, session revoked, session
    expired); mapping of the 2FA challenge and of forced enrolment; OAuth URL building; SSR
    behaviour.
    Security baseline: credentials (cookies through the credentials mode, the CSRF header, the
    bearer token) are sent only to the configured backend origin; no refresh once logout or local
    clearing has started (awesome-flutter-auth #33); the no-refresh list covers every route that
    verifies a credential or a code; a persistent token storage, when configured, also persists
    the refresh token; tokens never appear in URLs or logs.
  - ADRs, status `Proposed`: `adr/0031-client-refresh-and-retry-policy.md` (aligned with ADR 0005:
    refresh on 401 and on a code-less 403 until spec 2.0, never on another coded error),
    `adr/0032-credential-scoping-in-clients.md` (context: general guidance only, RFC 6750 and the
    browser credentials model).
  - Known divergences from `plan/baseline.md` § Clients.
- **Out of scope:** UI components, theming, admin clients, offline token verification.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --min CLI=25
  for a in 0031 0032; do grep -Eqx -- '- \*\*Status:\*\* (Proposed|Accepted)' adr/$a-*.md; done
  node tools/req-lint.mjs --json | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const r=JSON.parse(s); for (const p of ['client-cookie','client-bearer']) if(!r.some(x=>x.profile===p)) process.exit(1)})"
  for r in awesome-flutter-auth awesome-angular-auth awesome-react-auth; do grep -qF "$r" spec/14-clients.md; done
  for t in '`SESSION_REVOKED`' 'X-CSRF-Token' 'X-Auth-Strategy' 'origin' 'single-flight'; do grep -qF "$t" spec/14-clients.md; done
  ```
  Expected: every command exits 0. Before the goal the CLI minimum fails.
- **Documentation:** `CHANGELOG.md`.

## G19 — Threat model and release checklist
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G17, G18
- **Goal:** Tie the requirements to the threats they mitigate, and give spec releases and port
  conformance claims a security checklist.
- **Deliverables:**
  - `security/threat-model.md` — `## Assets`, `## Actors`, `## Trust boundaries`, `## Threats`
    (table `| ID | Threat | Flow | Mitigating requirements | Residual risk |`, IDs `THR-001` …,
    at least 40 threats, STRIDE categories named, every mitigating requirement an existing ID),
    `## Out of scope`. Generic only (plan rule 12): the writer may read
    `$SPEC_PRIVATE_DIR/security-notes.md` to check coverage but copies nothing port-specific.
  - `security/release-checklist.md` — `## Spec release`, `## Port conformance claim`; items
    `- [ ] RC-<nn> …`, at least 20, each naming the requirement IDs or tools it checks; the spec
    release section includes "no ADR is `Proposed`" and "no public file reveals an undisclosed
    weakness".
  - `SECURITY.md` — reporting weaknesses in the spec (GitHub private vulnerability reporting on
    this repository) and in ports (the port's own process); never public issues for undisclosed
    weaknesses.
  - `tools/check-threats.mjs` — parses the threats table; every ID unique; every mitigating ID
    exists (`req-lint --json`); `--json` prints `{threats, mitigating: [<sorted unique IDs>]}`;
    otherwise prints `check-threats: OK (<n> threats)`. `tools/check-threats.test.mjs`;
    `package.json` script `lint:threats` added to `verify`.
  - `tools/headings.json` entries for the three files.
  - Requirement gaps found here are recorded in the affected area's `## Open questions` (no new
    requirement text).
- **Out of scope:** changes to requirement items.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/check-threats.mjs); grep -Eq '^check-threats: OK \(([4-9][0-9]|[1-9][0-9]{2,}) threats\)$' <<<"$out"
  node tools/check-threats.mjs --json | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const j=JSON.parse(s); if(j.mitigating.length<60) process.exit(1)})"
  test "$(grep -Ec '^- \[ \] RC-[0-9]{2} ' security/release-checklist.md)" -ge 20
  test -f SECURITY.md
  ```
  Expected: every command exits 0; at least 40 threats, 60 mitigating requirements and 20
  checklist items. Before the goal `tools/check-threats.mjs` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `security/` and `SECURITY.md`;
  `CHANGELOG.md`.

## G20 — ADR review and spec alignment
- **Status:** TODO
- **Type:** HUMAN
- **Depends on:** G19
- **Precondition:** The owner has reviewed every ADR on `main`: none is `Proposed` any more (each
  is `Accepted`, `Rejected` or `Superseded by NNNN`, with any amendment written in its
  `## Decision`), and the last ADR planned (0032) exists. Command:
  `test -z "$(grep -lxF -- '- **Status:** Proposed' adr/[0-9]*.md || true)" && ls adr/0032-*.md > /dev/null`
- **Goal:** Align the specification with the outcome of every ADR before vectors are written.
- **Deliverables:**
  - `spec/*.md` — requirement items whose ADR was rejected, amended or superseded are reworded to
    the decided position or retired; text outside requirement items updated to match. Nothing
    else changes.
  - `spec/retired-ids.json` — IDs retired here (never reused).
  - `CHANGELOG.md` — under `## [Unreleased]`, one line per ADR `0001` to `0032` of the form
    `- ADR NNNN <Accepted|Rejected|Superseded by NNNN>: <one-line effect on the spec>`.
- **Out of scope:** new requirements; ADR files (the owner's); vectors.
- **Verification:**
  ```bash
  npm run verify
  if grep -lxF -- '- **Status:** Proposed' adr/[0-9]*.md; then exit 1; fi
  for f in adr/[0-9]*.md; do n=$(basename "$f" | cut -c1-4); grep -Eq "^- ADR $n (Accepted|Rejected|Superseded by [0-9]{4}):" CHANGELOG.md; done
  out=$(node tools/req-lint.mjs); grep -Eq '^req-lint: OK \([0-9]+ requirements\)$' <<<"$out"
  ```
  Expected: every command exits 0. Before the goal the CHANGELOG has no `- ADR NNNN` lines.
- **Documentation:** `plan/memory/decisions.md` entry listing retired IDs and reworded
  requirements per ADR.

## G21 — Vector format, runner contract and porting guide
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G20
- **Goal:** Define the vector format, the contract between the runner and the port harnesses, the
  coverage rules, and the guide port maintainers follow to adopt the suite.
- **Deliverables:**
  - `conformance/schema/vector.schema.json` — a vector file is
    `{area, vectors: [{id, title, requirements, profile, kind, given, steps, notes}]}`: `id`
    `<PREFIX>-V<nnn>`; `kind` `http`, `client` or `static`; `given` (users, configuration,
    clock, upstream provider behaviour); `steps` with `request` (method, path, headers, query,
    body, cookie jar use), `harness` calls, and `expect` (status, headers, set-cookie names and
    attributes, body `equals`, `contains`, `schema` or `absentKeys`, captured mail, SMS, events
    and webhook deliveries), and `capture` (JSON Pointer into a variable).
  - `conformance/schema/harness.schema.json` — payloads of the server harness control API.
  - `conformance/schema/report.schema.json` — a runner report: port, spec version, runner
    version, date, profiles, capabilities, results per vector and per requirement (`pass`,
    `fail`, `skipped`, with a reason).
  - `conformance/runner-contract.md` — `## Roles`, `## Server harness` (control endpoints under
    `/__conformance` in test builds only: reset, create users, read the captured mail and SMS
    outbox, set or advance the clock, report capabilities), `## Upstream provider mock` (the
    runner plays the OAuth and OIDC upstream provider), `## Webhook sink` (the runner receives
    outgoing webhooks), `## Configuration control` (start the server with a given configuration
    and observe a refusal to start), `## Event capture` (read published events), `## Client
    harness` (a program reading a JSON script of client actions on stdin and driving the client
    library against the runner acting as the server), `## Runner` (command line, environment,
    exit codes), `## Capabilities and profiles` (prior art: the awesome-lambda-auth `test/contract`
    capability probe and `AWESOME_AUTH_CONTRACT_REQUIRE`), `## Reports`,
    `## Security of the harness`.
  - `conformance/porting-guide.md` — `## Adding the harness` (test builds only, never in a
    production artefact), `## Running the suite`, `## Publishing a declaration`,
    `## Claiming conformance`.
  - `conformance/vectors/README.md` — format, coverage rules, and the statement that in spec 1.x
    `client` vectors are executed by a client harness when a port provides one and are otherwise
    checked by review in the port's report; they count for coverage because they specify the
    expected behaviour.
  - `conformance/vectors/transport.json` with at least 3 vectors as worked examples.
  - `conformance/untested.json` — `[]`; `conformance/schema/untested.schema.json` — array of
    `{id, reason}` with a non-empty reason.
  - `tools/req-lint.mjs` — new options: `--coverage` checks that every ID is referenced by at least
    one vector (any string inside any `requirements` array of any `conformance/vectors/*.json`) or
    listed in `conformance/untested.json`, never both, and that every referenced ID exists and is
    not retired; `--only PREFIX,PREFIX` restricts `--coverage`; `--max-untested P` fails when more
    than P percent of the checked IDs are untested. With coverage it prints
    `req-lint: OK (<n> requirements, covered <c>, untested <u>)`. Tests in
    `tools/req-lint.test.mjs`.
  - `tools/validate-vectors.mjs` — every `conformance/vectors/*.json` validates; vector IDs
    unique; the vector prefix is the prefix of each requirement it lists; every requirement exists
    (from `req-lint --json`) and its profile equals the vector profile or one the vector profile
    requires. Prints `validate-vectors: OK (<n> vectors)`. `tools/validate-vectors.test.mjs`;
    `package.json` script `validate:vectors` added to `verify`; `validate:registries` also
    validates `conformance/untested.json`.
  - `tools/headings.json` entries for `conformance/runner-contract.md` and
    `conformance/porting-guide.md`.
- **Out of scope:** vectors beyond the worked examples; the runner implementation (G26).
- **Verification:**
  ```bash
  npm run verify
  for s in vector harness report untested; do test -f conformance/schema/$s.schema.json; done
  out=$(node tools/validate-vectors.mjs); grep -Eq '^validate-vectors: OK \([0-9]+ vectors\)$' <<<"$out"
  for h in '## Roles' '## Server harness' '## Upstream provider mock' '## Webhook sink' '## Configuration control' '## Event capture' '## Client harness' '## Runner' '## Capabilities and profiles' '## Reports' '## Security of the harness'; do grep -qxF "$h" conformance/runner-contract.md; done
  for h in '## Adding the harness' '## Running the suite' '## Publishing a declaration' '## Claiming conformance'; do grep -qxF "$h" conformance/porting-guide.md; done
  node --test tools/req-lint.test.mjs tools/validate-vectors.test.mjs
  node tools/validate-json.mjs conformance/schema/untested.schema.json conformance/untested.json
  ```
  Expected: every command exits 0. Before the goal `conformance/schema/vector.schema.json` does not
  exist.
- **Documentation:** `README.md` `## Repository layout` links `conformance/runner-contract.md` and
  `conformance/porting-guide.md`; `CHANGELOG.md`.

## G22 — Vectors: errors, transport, abuse protection, tokens and sessions
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G21
- **Goal:** Cover the ERR, TRN, ABU, TOK and SES requirements with vectors, or list each uncovered
  one in `conformance/untested.json` with the reason.
- **Deliverables:**
  - `conformance/vectors/errors.json`, `conformance/vectors/transport.json`,
    `conformance/vectors/abuse-protection.json`, `conformance/vectors/tokens.json`,
    `conformance/vectors/sessions.json`.
  - `conformance/untested.json` — entries for these prefixes that no vector can check black-box
    (for example storage-at-rest rules), each with a reason.
- **Out of scope:** other prefixes; requirement text (a requirement that cannot be tested as
  written is recorded in the area's `## Open questions`, and listed as untested).
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --coverage --only ERR,TRN,ABU,TOK,SES --max-untested 20
  out=$(node tools/validate-vectors.mjs); grep -Eq '^validate-vectors: OK \([0-9]+ vectors\)$' <<<"$out"
  ```
  Expected: every command exits 0; at most 20 percent of these requirements untested. Before the
  goal the coverage check fails.
- **Documentation:** `CHANGELOG.md`.

## G23 — Vectors: credentials, account lifecycle, passwordless and two-factor
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G22
- **Goal:** Cover the PWD, EML, ACC, MLK, OTP and MFA requirements with vectors or untested
  entries.
- **Deliverables:**
  - `conformance/vectors/password-and-email.json`, `conformance/vectors/account-lifecycle.json`,
    `conformance/vectors/passwordless.json`, `conformance/vectors/two-factor.json`.
  - `conformance/untested.json` — entries for these prefixes.
- **Out of scope:** other prefixes; requirement text.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --coverage --only PWD,EML,ACC,MLK,OTP,MFA --max-untested 20
  out=$(node tools/validate-vectors.mjs); grep -Eq '^validate-vectors: OK \([0-9]+ vectors\)$' <<<"$out"
  ```
  Expected: every command exits 0. Before the goal the coverage check fails.
- **Documentation:** `CHANGELOG.md`.

## G24 — Vectors: OAuth, admin, API keys, events, webhooks and tools
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G23
- **Goal:** Cover the OAU, LNK, ADM, APK, EVT, WHK and TOL requirements with vectors or untested
  entries, using the upstream provider mock, the webhook sink and event capture of the runner
  contract.
- **Deliverables:**
  - `conformance/vectors/oauth-and-linking.json`, `conformance/vectors/admin.json`,
    `conformance/vectors/api-keys.json`, `conformance/vectors/events-and-webhooks.json`.
  - `conformance/untested.json` — entries for these prefixes.
- **Out of scope:** other prefixes; requirement text; the runner.
- **Verification:**
  ```bash
  npm run verify
  node tools/req-lint.mjs --coverage --only OAU,LNK,ADM,APK,EVT,WHK,TOL --max-untested 25
  out=$(node tools/validate-vectors.mjs); grep -Eq '^validate-vectors: OK \([0-9]+ vectors\)$' <<<"$out"
  ```
  Expected: every command exits 0. Before the goal the coverage check fails.
- **Documentation:** `CHANGELOG.md`.

## G25 — Vectors: configuration, identity provider, clients and full coverage
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G24
- **Goal:** Cover the remaining prefixes (CFG with the configuration-control harness, IDP, RSV,
  CLI, CNF) and turn on full coverage in `verify`.
- **Deliverables:**
  - `conformance/vectors/configuration.json`, `conformance/vectors/identity-provider.json`,
    `conformance/vectors/clients.json` (`kind: client`), and CNF entries in
    `conformance/untested.json` or vectors of `kind: static` in `conformance/vectors/overview.json`.
  - `conformance/untested.json` — entries for these prefixes.
  - `package.json` — `lint:req` becomes `node tools/req-lint.mjs --coverage --max-untested 25`.
  - `conformance/vectors/README.md` — coverage figures per prefix.
- **Out of scope:** requirement text; the runner.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/req-lint.mjs --coverage --max-untested 25); grep -Eq '^req-lint: OK \([0-9]+ requirements, covered [0-9]+, untested [0-9]+\)$' <<<"$out"
  grep -qF -- '--coverage' package.json
  ```
  Expected: every command exits 0; every requirement is covered or untested, at most 25 percent
  untested overall. Before the goal the full coverage check fails.
- **Documentation:** `CHANGELOG.md`.

## G26 — Reference runner
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G25
- **Goal:** Ship a dependency-light Node runner that executes `http` vectors against a server
  harness, with the upstream provider mock, webhook sink, configuration control and event capture
  of the runner contract, proven against an in-repository fixture server.
- **Deliverables:**
  - `runner/bin/conformance.mjs` — `--base-url`, `--vectors <dir>` (default
    `conformance/vectors`), `--profiles a,b`, `--require a,b`, `--report <file>`, `--self-test`;
    cookie jar per vector, variable capture, harness calls per `conformance/runner-contract.md`;
    vectors needing a capability the harness does not report are `skipped` with the reason;
    `client` and `static` vectors reported as `skipped` with a reason; exit 0 when no required
    profile has a failure.
  - `runner/lib/*.mjs` — HTTP execution, matchers, upstream provider mock, webhook sink, event and
    outbox readers, report writer.
  - `runner/test/fixture-server.mjs` — a minimal in-process server implementing the harness and
    enough of the TRN, TOK and SES surface for at least 5 vectors, plus one OAuth vector through
    the provider mock and one webhook vector through the sink; a `broken` mode that violates one
    of them.
  - `runner/test/*.test.mjs` — the fixture passes those vectors, the broken mode fails exactly the
    expected one, reports validate against `conformance/schema/report.schema.json`.
  - `runner/README.md`; `package.json` script `test:runner` (`node --test runner/test/`) added to
    `verify`.
- **Out of scope:** client-harness execution; changes to vectors or schemas (a schema problem is a
  block).
- **Verification:**
  ```bash
  npm run verify
  node --test runner/test/
  out=$(node runner/bin/conformance.mjs --self-test); grep -q '^runner: self-test OK$' <<<"$out"
  ```
  Expected: every command exits 0. Before the goal `runner/bin/conformance.mjs` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `runner/README.md`; `CHANGELOG.md`.

## G27 — Node reference harness
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G26
- **Goal:** Run the vectors against the real pinned reference before any declaration is written:
  every vector of reference-default behaviour passes, and every vector of a decided departure
  fails as listed.
- **Deliverables:**
  - `runner/harness/node/package.json` (private, exact dependency `@awesome-lang-auth/node`
    `1.10.8`, `express` ^5) and its own `package-lock.json`.
  - `runner/harness/node/server.mjs` — an Express app on that package with in-memory
    implementations of the store interfaces it needs, mail and SMS captured through the
    configuration hooks, events captured from the event bus, and the `/__conformance` endpoints
    of the runner contract; capabilities it cannot provide (for example the clock) are reported
    and the dependent vectors are skipped.
  - `runner/harness/node/run.mjs` — starts the harness on a free local port, runs
    `runner/bin/conformance.mjs` against it with `--report <file>`, stops it.
  - `conformance/expected-failures/node.json` — `[{vector, adr, reason}]`: the vectors the
    reference is expected to fail because of an accepted ADR departure or a security baseline
    requirement (each entry names the ADR or the requirement).
  - `conformance/reports/node-1.10.8.json` — the runner report of this run, tracked (the copy in
    `.cache/reports/` is lost on a fresh checkout); G28 and G29 read it.
  - `tools/compare-expected.mjs` — `<report> <expected>`: every listed vector fails, every other
    `http` vector passes or is skipped with a capability reason, at most `--max-skipped P` percent
    of `http` vectors skipped. Prints
    `compare-expected: OK (<p> passed, <f> expected failures, <s> skipped)`.
    `tools/compare-expected.test.mjs`.
  - `package.json` script `conformance:node` (install and run the harness), **not** part of
    `verify`; `.github/workflows/conformance-node.yml` running it on changes to `conformance/**`,
    `runner/**`; `permissions: contents: read`.
  - Mismatches are resolved here: a vector that contradicts reference-default behaviour the spec
    meant to follow is a vector bug and is fixed in `conformance/vectors/*.json`; a mismatch that
    needs a change of requirement text is a block.
- **Out of scope:** requirement text; other ports; publishing anything.
- **Verification:**
  ```bash
  npm run verify
  ( cd runner/harness/node && npm ci )
  test "$(node -p "require('./runner/harness/node/node_modules/@awesome-lang-auth/node/package.json').version")" = 1.10.8
  mkdir -p .cache/reports
  node runner/harness/node/run.mjs --report .cache/reports/node.json
  node tools/validate-json.mjs conformance/schema/report.schema.json .cache/reports/node.json
  out=$(node tools/compare-expected.mjs --max-skipped 25 .cache/reports/node.json conformance/expected-failures/node.json); grep -Eq '^compare-expected: OK \([0-9]+ passed, [0-9]+ expected failures, [0-9]+ skipped\)$' <<<"$out"
  node tools/validate-json.mjs conformance/schema/report.schema.json conformance/reports/node-1.10.8.json
  out=$(node tools/compare-expected.mjs --max-skipped 25 conformance/reports/node-1.10.8.json conformance/expected-failures/node.json); grep -Eq '^compare-expected: OK ' <<<"$out"
  ```
  Expected: every command exits 0; at most 25 percent of `http` vectors skipped. Before the goal
  `runner/harness/node/package.json` does not exist.
- **Documentation:** `runner/README.md` (the harness); `CHANGELOG.md`; `plan/memory/decisions.md`
  for every vector fixed here and why.

## G28 — Port declarations and parity matrix
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G27
- **Goal:** Give every port a machine-readable declaration against the spec, generate the parity
  matrix from the declarations, and mark the spec as release candidate.
- **Deliverables:**
  - `conformance/schema/declaration.schema.json` — `{port, specVersion, reviewedCommit,
    claimedProfiles, requirements: {<ID>: {status: pass|fail|deviation|not-applicable|untested,
    evidence, tracking}}}`.
  - `parity/ports/<id>.json` for the 9 ports of `parity/ports.json`: every requirement present;
    `not-applicable` for client requirements on servers and server requirements on clients; for
    `node`, `pass` and `fail` from `conformance/reports/node-1.10.8.json` with `reviewedCommit` the
    pinned commit (plan rule 12 applied); `deviation` with a public `tracking` link for the
    documented, public divergences of `plan/baseline.md`; `untested` otherwise.
  - `tools/gen-parity.mjs` — writes `parity/matrix.md` (per port and profile: counts by status;
    per requirement: status per port); `--check` fails when the file is stale, when a declaration
    misses a requirement, when a declaration's `specVersion` has a different MAJOR.MINOR than
    `VERSION`, or when a requirement listed by `check-threats --json` as mitigating has status
    `fail` or `deviation` without a `tracking` URL of an issue, pull request or published advisory
    of that port's repository (plan rule 12). `tools/gen-parity.test.mjs`; `package.json` script
    `parity:check` added to `verify`.
  - `parity/matrix.md` (generated).
  - `VERSION` — `1.0.0-rc.1`; `spec/00-overview.md` and `README.md` `## Status` updated.
- **Out of scope:** reviewing port code (G29 to G32); any external action.
- **Verification:**
  ```bash
  npm run verify
  for p in $(node -p "require('./parity/ports.json').map(p => p.id).join(' ')"); do node tools/validate-json.mjs conformance/schema/declaration.schema.json parity/ports/$p.json; done
  node tools/gen-parity.mjs --check
  node --test tools/gen-parity.test.mjs
  test "$(cat VERSION)" = 1.0.0-rc.1
  ```
  Expected: every command exits 0. Before the goal `parity/ports/node.json` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `parity/matrix.md`; `CHANGELOG.md`
  `## [1.0.0-rc.1]`.

## G29 — Conformance report: node
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G28
- **Goal:** Review awesome-node-auth at the pinned commit (the one the harness ran) against the
  `MUST` requirements of the profiles it implements, record the results, draft (not file) the
  issues, and build the report and issue-filing tools that the later reports reuse.
- **Deliverables:**
  - The reviewed commit is the pinned commit of `inventory/reference.json` (clone in
    `.cache/reference/awesome-node-auth`). The current head of the default branch is resolved
    read-only with `git ls-remote` and recorded in the report; each draft says whether the head
    already changes the observed behaviour.
  - `reports/node/1.0.0-rc.1.md` with `## Scope and method` (commit reviewed, head at review time,
    `conformance/reports/node-1.10.8.json` as evidence, static review of what the harness cannot
    reach; `SHOULD` requirements not covered by the harness stay `untested`), `## Summary`,
    `## Failures`, `## Deviations`, `## Untested`, `## Proposed issues`.
  - Drafts `reports/node/issues/<nn>-<slug>.md` with front matter `repo`, `title`, `requirements`,
    `issue` (empty) and the body sections `## Requirement`, `## Observed behaviour`,
    `## Expected behaviour`, `## Acceptance`. One draft groups related requirements; existing
    public issues are referenced instead of duplicated; drafts of deliberate spec departures cite
    the ADR; one draft proposes adopting the conformance harness upstream.
  - `reports/site/issues/01-link-the-specification.md` — a draft for the documentation site
    repository `awesome-lang-auth/awesome-lang-auth` (link to the spec and the parity matrix).
  - `parity/ports/node.json` — statuses from the review with `reviewedCommit`; `parity/matrix.md`
    regenerated.
  - Security-sensitive findings (plan rule 12) are written only to
    `$SPEC_PRIVATE_DIR/reports/node/`.
  - `tools/check-reports.mjs` — report headings; draft front matter; `repo` equal to the port's
    repository in `parity/ports.json` (or `awesome-lang-auth/awesome-lang-auth` under
    `reports/site/`); requirement IDs exist; `issue` empty or a URL of that repository; no draft
    and no report failure lists a requirement that `check-threats --json` reports as mitigating,
    unless the draft front matter has `public: <URL of an existing issue, pull request or
    published advisory>`. Prints `check-reports: OK (<r> reports, <d> drafts)`.
    `tools/check-reports.test.mjs`; `package.json` script `lint:reports` added to `verify`.
  - `tools/gen-file-issues.mjs` — reads the ticked drafts under `## Approved issue drafts` in
    `plan/memory/owner-inputs.md`; default mode prints, for the owner to review and run, a bash
    script that for each ticked draft with an empty `issue` searches the target repository
    (`gh issue list --repo <repo> --state all --search "<title> in:title"`), creates the issue
    with `gh issue create --repo <repo> --title <title> --body-file <body without front matter>`
    when none exists, and prints `- filed: <draft path> <issue URL>`; `--check-filed` exits 0 when
    `- approved-issues: none` is present, or when at least one draft is ticked and every ticked
    draft has a `- filed:` line under `## Filed issues`. `tools/gen-file-issues.test.mjs`.
- **Out of scope:** creating issues, comments or pull requests anywhere; changing the spec.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/check-reports.mjs); grep -Eq '^check-reports: OK \(1 reports, [0-9]+ drafts\)$' <<<"$out"
  node -e "const d=require('./parity/ports/node.json'); if(d.reviewedCommit!==require('./inventory/reference.json').commit) process.exit(1); if(!Object.values(d.requirements).some(r=>r.status==='pass')) process.exit(1)"
  node --test tools/check-reports.test.mjs tools/gen-file-issues.test.mjs
  test -f reports/site/issues/01-link-the-specification.md
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `reports/` does not exist.
- **Documentation:** `README.md` `## Repository layout` links `reports/`; `CHANGELOG.md`.

## G30 — Conformance reports: go and lambda
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G29
- **Goal:** Same as G29 for awesome-go-auth and awesome-lambda-auth, against the `MUST`
  requirements of the profiles each implements.
- **Deliverables:** as in G29 for `go` and `lambda` (reports, drafts including one "implement the
  conformance harness" draft per port, declarations, regenerated matrix, private findings outside
  the repository), except that the reviewed commit is the head of the default branch, resolved
  with `git ls-remote` and checked out in `.cache/ports/<id>` before the review starts. The
  awesome-lambda-auth `test/contract` suite may be run against the reference harness or used as
  evidence.
- **Out of scope:** as in G29; changes to `tools/check-reports.mjs` and
  `tools/gen-file-issues.mjs` other than bug fixes.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/check-reports.mjs); grep -Eq '^check-reports: OK \(3 reports, [0-9]+ drafts\)$' <<<"$out"
  for p in go lambda; do node -e "const d=require('./parity/ports/$p.json'); if(!/^[0-9a-f]{40}$/.test(d.reviewedCommit)) process.exit(1); if(!Object.values(d.requirements).some(r=>r.status==='pass')) process.exit(1)"; done
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `reports/go/` does not exist.
- **Documentation:** `CHANGELOG.md`.

## G31 — Conformance reports: python, rust and dart
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G30
- **Goal:** Same as G29 for awesome-python-auth, awesome-rust-auth and awesome-dart-auth.
- **Deliverables:** as in G30 for `python`, `rust` and `dart`. A port without an HTTP surface for
  a profile declares that profile unclaimed; its requirements are `fail` only when the port's own
  documentation claims the feature.
- **Out of scope:** as in G30.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/check-reports.mjs); grep -Eq '^check-reports: OK \(6 reports, [0-9]+ drafts\)$' <<<"$out"
  for p in python rust dart; do node -e "const d=require('./parity/ports/$p.json'); if(!/^[0-9a-f]{40}$/.test(d.reviewedCommit)) process.exit(1)"; done
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `reports/python/` does not exist.
- **Documentation:** `CHANGELOG.md`.

## G32 — Conformance reports: flutter, angular and react
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G31
- **Goal:** Same as G29 for the clients awesome-flutter-auth, awesome-angular-auth (branch
  `develop`) and awesome-react-auth, against the CLI requirements; `client` vectors are checked by
  review.
- **Deliverables:** as in G30 for `flutter`, `angular` and `react`, with one "implement the client
  harness" draft per client.
- **Out of scope:** as in G30.
- **Verification:**
  ```bash
  npm run verify
  out=$(node tools/check-reports.mjs); grep -Eq '^check-reports: OK \(9 reports, [0-9]+ drafts\)$' <<<"$out"
  for p in flutter angular react; do test -f reports/$p/1.0.0-rc.1.md; node -e "const d=require('./parity/ports/$p.json'); if(!/^[0-9a-f]{40}$/.test(d.reviewedCommit)) process.exit(1)"; done
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `reports/flutter/` does not exist.
- **Documentation:** `CHANGELOG.md`.

## G33 — Record filed issues
- **Status:** TODO
- **Type:** HUMAN
- **Depends on:** G32
- **Precondition:** The owner reads the drafts, ticks the approved ones under
  `## Approved issue drafts` in `plan/memory/owner-inputs.md` (`- [x] reports/<id>/issues/<file>.md`),
  runs `node tools/gen-file-issues.mjs > file-issues.sh`, reviews and runs the script outside the
  repository, and pastes its `- filed: <draft> <URL>` lines under `## Filed issues`; or approves
  none with `- approved-issues: none` under `## Approved issue drafts`. Command:
  `node tools/gen-file-issues.mjs --check-filed`
- **Goal:** Record in the repository the issues the owner filed, so that declarations and the
  matrix link them.
- **Deliverables:**
  - The `issue` field of every filed draft set to its URL. Unapproved drafts are left untouched.
  - `parity/ports/<id>.json` — `tracking` set to the issue URL for the requirements of each filed
    draft; `parity/matrix.md` regenerated.
  - `reports/filed-issues.md` — `# Filed issues`, the spec version, and one line per filed draft
    `- <draft path> → <issue URL>`, or the line `No drafts were approved for filing.`.
- **Out of scope:** any external action; editing draft bodies.
- **Verification:**
  ```bash
  npm run verify
  grep -qx '# Filed issues' reports/filed-issues.md
  while read -r f url; do
    repo=$(sed -nE 's/^repo: *//p' "$f" | head -1)
    grep -Eq "^https://github.com/$repo/issues/[0-9]+$" <<<"$url"
    test "$(sed -nE 's/^issue: *//p' "$f" | head -1)" = "$url"
    gh issue view "$url" --json url > /dev/null
    grep -qF "$f → $url" reports/filed-issues.md
  done < <(sed -nE 's/^- filed: ([^ ]+) ([^ ]+)$/\1 \2/p' plan/memory/owner-inputs.md)
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0; every filed draft carries the URL of an existing issue of its
  repository and is listed in `reports/filed-issues.md`. Before the goal `reports/filed-issues.md`
  does not exist.
- **Documentation:** `CHANGELOG.md` (one line: issues filed, by port).

## G34 — Release evidence
- **Status:** TODO
- **Type:** AGENT
- **Depends on:** G02, G33
- **Goal:** Prepare the evidence the owner needs to approve release 1.0.0, without approving
  anything.
- **Deliverables:**
  - `security/releases/1.0.0.md` — every item of `## Spec release` of
    `security/release-checklist.md`, left unticked (`- [ ] RC-<nn> …`), each followed by an
    indented `Evidence:` line (command run, result, date) or `Evidence: not met — <why>`.
  - `CHANGELOG.md` — `## [Unreleased]` summarised for the release.
- **Out of scope:** `VERSION`; ticking checklist items (G35, after the owner's approval);
  normative changes.
- **Verification:**
  ```bash
  npm run verify
  n_rc=$(sed -n '/^## Spec release$/,/^## /p' security/release-checklist.md | grep -Ec '^- \[ \] RC-[0-9]{2} ')
  test "$(grep -Ec '^- \[ \] RC-[0-9]{2} ' security/releases/1.0.0.md)" = "$n_rc"
  test "$(grep -Ec '^ +Evidence: ' security/releases/1.0.0.md)" = "$n_rc"
  if grep -lxF -- '- **Status:** Proposed' adr/[0-9]*.md; then exit 1; fi
  node tools/req-lint.mjs --coverage --max-untested 25 > /dev/null
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `security/releases/1.0.0.md` does not exist.
- **Documentation:** `plan/memory/decisions.md` entry for any item marked "not met".

## G35 — Release 1.0.0
- **Status:** TODO
- **Type:** HUMAN
- **Depends on:** G34
- **Precondition:** The owner has read `security/releases/1.0.0.md`, accepted each item by adding
  one line `- rc-ok: RC-<nn>` per item under `## Owner inputs` in `plan/memory/owner-inputs.md`,
  and approved the release with `- release-approved: 1.0.0` in the same section. Command:
  `grep -Eq '^- release-approved: 1\.0\.0$' plan/memory/owner-inputs.md && test -f security/releases/1.0.0.md && for id in $(grep -oE '^- \[ \] RC-[0-9]{2}' security/releases/1.0.0.md | grep -oE 'RC-[0-9]{2}'); do grep -qxF -- "- rc-ok: $id" plan/memory/owner-inputs.md || exit 1; done`
- **Goal:** Mark version 1.0.0 of the specification; the owner tags it.
- **Deliverables:**
  - `security/releases/1.0.0.md` — every item ticked (`- [x]`), each followed by the line
    `  Accepted: rc-ok` (the owner's acceptance recorded in `plan/memory/owner-inputs.md`).
  - `VERSION` — `1.0.0`; `CHANGELOG.md` `## [1.0.0] — <date>`; `spec/00-overview.md`,
    `README.md` `## Status` and `GOVERNANCE.md` (if it names a version) updated.
  - The run report asks the owner to merge the pull request and tag the merge commit on `main` as
    `v1.0.0` (the loop creates no tag).
- **Out of scope:** normative changes; declarations (they keep `1.0.0-rc.1`, same MAJOR.MINOR);
  tags and GitHub releases (the owner's).
- **Verification:**
  ```bash
  npm run verify
  test "$(cat VERSION)" = 1.0.0
  grep -Eq '^## \[1\.0\.0\] — [0-9]{4}-[0-9]{2}-[0-9]{2}$' CHANGELOG.md
  if grep -q '^- \[ \]' security/releases/1.0.0.md; then exit 1; fi
  for id in $(grep -oE '^- \[x\] RC-[0-9]{2}' security/releases/1.0.0.md | grep -oE 'RC-[0-9]{2}'); do grep -qxF -- "- rc-ok: $id" plan/memory/owner-inputs.md; done
  if grep -lxF -- '- **Status:** Proposed' adr/[0-9]*.md; then exit 1; fi
  node tools/req-lint.mjs --coverage --max-untested 25 > /dev/null
  node tools/gen-parity.mjs --check
  ```
  Expected: every command exits 0. Before the goal `VERSION` is `1.0.0-rc.1`.
- **Documentation:** `README.md` `## Status`; `plan/memory/decisions.md` entry for the release.
