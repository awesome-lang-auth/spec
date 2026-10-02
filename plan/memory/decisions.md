# Decisions

Non-obvious decisions taken while bootstrapping and running the plan. Format:
`## YYYY-MM-DD — Title` with **Decision.** **Why.** **Consequences.** Entries are never rewritten:
a changed decision gets a new entry that supersedes the old one. Normative decisions about the
protocol are ADRs in `adr/`, not entries here.

## 2026-10-02 — Repository structure

**Decision.** One Markdown file per spec area under `spec/` (17 files, numbered), long-form vision
in `docs/vision.md`, reference evidence in `inventory/`, vectors, schemas, runner contract and
porting guide in `conformance/`, per-port declarations and the generated matrix in `parity/`,
review reports and issue drafts in `reports/`, ADRs in `adr/`, security documents in `security/`,
the plan and its memory in `plan/`, tools in `tools/` (Node scripts), the reference runner and the
node reference harness in `runner/`.
**Why.** One file per area keeps requirement prefixes, divergence tables and goal deliverables
aligned; evidence (inventory), norms (spec) and tests (conformance) stay separate so each can be
checked on its own.
**Consequences.** `spec/areas.json` is the registry every linter reads; adding an area is a minor
spec change with its own goal. Abuse protection (ABU) and API keys (APK) have their own areas;
tenants, roles and permissions are outside 1.x (ADR 0027 in G15) apart from reserved names.

## 2026-10-02 — English, BCP 14 and requirement identifiers

**Decision.** The repository is in English. Requirements use the BCP 14 keywords (RFC 2119,
RFC 8174) only inside identified items `- **<PFX>-<nnn>** …`; identifiers are never reused.
**Why.** The repository is public and international, and every requirement must map to vectors
and to port declarations.
**Consequences.** `tools/req-lint.mjs` enforces the format; coverage by vectors becomes mandatory
from G25.

## 2026-10-02 — Reference default and security baseline

**Decision.** awesome-node-auth 1.10.8 (commit `7e640b4681c1fc899ba4d9f4de0c9efb4ad8ae43`, the
same commit as the tag `v1.10.8`, verified on 2026-10-02) is pinned as reference. For wire shapes,
names, codes and defaults the spec follows its recorded behaviour and departs from it only through
an ADR. Security requirements form a baseline written from the threat model, the cited RFCs and
public practice; the plan, the spec and the ADRs never contrast that baseline with the behaviour of
the reference or of another port except through a public source.
**Why.** The reference is the most complete implementation and the one every port tracks, so it is
the right default for the wire contract. Framing security requirements as "departures from the
reference" would tell readers which safe behaviours the reference lacks before the owner has
handled them privately.
**Consequences.** Spec goals carry a "Reference default" bullet and a "Security baseline" bullet;
ADR contexts cite only public sources. Some baseline requirements will require changes in the
reference; they surface in the node harness run (G27) only after `disclosure-cleared: node`.

## 2026-10-02 — Security-sensitive findings stay private, including indirect disclosure

**Decision.** Undisclosed, exploitable weaknesses of any port found by the survey or by later
reviews are kept in `$SPEC_PRIVATE_DIR` and handed to the owner for the port's private advisory
process. Public files cite only wire facts, documented behaviour and public issues. The reference
inventory (per-route authentication and CSRF, token keys and verification rules) is gated behind
the owner input `disclosure-cleared: node` (G05). From G28 on, a requirement that mitigates a
threat may be `fail` or `deviation` in a public file only with an existing public tracking URL;
the tools enforce it.
**Why.** All ports are public repositories with users, and `agent/plan` is a public branch pushed
before review; publishing unfixed weaknesses, even as a machine-readable inventory or a
declaration status, would be a disclosure without a fix.
**Consequences.** The baseline omits a few survey facts that sit close to private findings. Public
declarations keep such requirements `untested` with evidence `review pending` until the advisory is
published. G03 and G04 can run while G05 waits.

## 2026-10-02 — One writer per file between owner and loop

**Decision.** The loop writes `plan/memory/project.md` (waiting items, blocks) on `agent/plan`; the
owner writes `plan/memory/owner-inputs.md` (inputs, block clearances, approved and filed issues)
and ADR statuses on `main`. The loop merges `origin/agent/plan` (fast-forward only) and then
`origin/main` at the start of every run with a clean tree, and evaluates preconditions and blocks
after that.
**Why.** With both writing one file on two branches, every owner input would conflict with the
loop's next waiting-items update; with block ticks only on `main`, a blocked goal would never see
its clearance.
**Consequences.** A block is cleared by `- unblock: G<nn> <date>` in owner-inputs; the loop then
ticks its own entry.

## 2026-10-02 — The loop never writes to GitHub except its own branch

**Decision.** The loop pushes only `agent/plan` (never forced, never tags). The owner opens the
loop's draft pull request once, files issues by running a script the loop generates
(`tools/gen-file-issues.mjs`, G29) and tags releases on the merge commit in `main`.
**Why.** `gh auth status` does not prove write access, and the owner keeps control of `main`, of
releases and of every message sent to other repositories.
**Consequences.** G33 only records the URLs the owner pasted and checks them with read-only
`gh issue view`. The owner merges with a merge commit; a squash merge would make `agent/plan`
diverge.

## 2026-10-02 — 35 goals instead of about 20

**Decision.** The plan has 35 goals, more than the 15 to 25 first asked for.
**Why.** A review of the first draft showed that several goals could not be finished in one run
(the node inventory, the errors and transport areas, admin with events, configuration with the
identity provider, the vectors of eleven prefixes in one goal, three port reviews per goal), and
that steps were missing: a disclosure gate, ADR alignment as its own goal, a run of the vectors
against the real reference, the release split between evidence and approval. Lowering the
minimum requirement counts instead would have produced a thinner security baseline.
**Consequences.** The owner's inputs are needed at G02, G05, G20, G33 and G35; G03 and G04 run
while the owner prepares G02 and G05.

## 2026-10-02 — Thresholds checked against the pinned reference

**Decision.** The fixed thresholds in G05 and G06 (at least 80 routes, 30 error codes, 26 event
names) and every route, code and configuration path named in their Verification were checked in a
scratch clone of the pinned commit on 2026-10-02: about 104 literal route registrations, 49
upper-case code-like literals, 26 `identity.*` names; `2FA_REQUIRED`, `NOT_IMPLEMENTED` and
`OAUTH_ORIGIN_ALLOWLIST_EMPTY` are present; `RATE_LIMITED` is not (G08 introduces it).
**Why.** A wrong threshold would block a goal permanently.
**Consequences.** The pin uses the full SHA directly; G05 has no SHA-resolution step.

## 2026-10-02 — Client vectors are review-checked in 1.x

**Decision.** `kind: client` vectors count for coverage because they specify the expected client
behaviour, but the reference runner of spec 1.x does not execute them; reports check them by
review unless the client provides a client harness.
**Why.** A client runner needs a harness in each client language; that is post-1.0 work and each
client report drafts an issue for it.
**Consequences.** Client declarations rest on review evidence in 1.0; a client runner is a 1.x
minor addition.
