# Governance

Draft. This document describes how the awesome-lang-auth specification is decided, versioned and
used. It will be revised before spec 1.0.0.

## Roles

- **Owner.** The maintainer of the awesome-lang-auth organisation. Merges into `main`, accepts or
  rejects ADRs, decides what may be published about the security of a port, files every issue
  sent to a port repository, approves releases and tags them.
- **Port maintainers.** Maintain one implementation. They propose changes, publish their port's
  declaration and decide how and when their port conforms.
- **Contributors.** Anyone opening issues or pull requests here.
- **Agentic loop.** An AI agent that executes one goal of the plan per run on the branch
  `agent/plan`. It drafts text, schemas, vectors, tools, reports and issue drafts; it never
  merges, never tags, never releases, and never writes to another repository.

## Decision process

1. A change starts as an issue or as a goal of the plan.
2. Any change to a requirement (new, stricter, looser, retired) needs an **ADR** in `adr/`,
   opened with status `Proposed`.
3. The owner accepts, amends, rejects or supersedes the ADR. Port maintainers are invited to
   comment on the pull request; a change that makes a port non-conforming says so in the ADR's
   consequences.
4. The spec text, the vectors and the changelog are updated in the same pull request as the
   accepted ADR, or in the next goal of the plan.

Default rule: for wire shapes, names, status codes and configuration defaults, the spec follows
the recorded behaviour of the pinned reference implementation (awesome-node-auth). It departs from
it only through an accepted ADR, typically for consistency across ports. Security requirements
form a baseline that every port meets, the reference included; the baseline is written from the
threat model and public security practice, not from any port. The reference has no special
authority beyond being the default: where the spec and the reference disagree after an accepted
ADR, the reference is the one that changes.

## Spec versioning

The specification has its own Semantic Version, in the `VERSION` file, independent of every port.

- **MAJOR**: a change that can make a conforming port non-conforming: a new MUST requirement in an
  existing profile, a stricter requirement, a changed wire shape, a removed requirement or profile.
- **MINOR**: a new optional profile, new SHOULD or MAY requirements, a requirement relaxed, a
  deprecation (deprecated requirements are removed only in the next MAJOR).
- **PATCH**: clarifications, editorial fixes, new or corrected vectors for existing requirements,
  tooling.

Requirement identifiers are never reused; retired identifiers are listed in
`spec/retired-ids.json`. Pre-release versions (`-draft`, `-rc.N`) carry no conformance guarantee.
Releases are tagged `vX.Y.Z` by the owner on the merge commit in `main`.

## Conformance claims

A port may state "conforms to awesome-lang-auth spec vX.Y, profiles: P1, P2, …" only when:

- its declaration in `parity/ports/<id>.json` targets the same `MAJOR.MINOR`;
- every MUST requirement of the claimed profiles is `pass`, backed by a runner report produced
  with the vectors of that version;
- every SHOULD requirement that does not pass is a declared `deviation` with a public tracking
  link;
- the report is linked from the port's README.

A port that does not meet these conditions may still publish its declaration; the parity matrix
shows its status without a claim. The owner may mark a claim as disputed in the matrix when a
report cannot be reproduced.

## Agentic loop

The plan in `plan/plan.md` is the only source of goal state. The loop runs one goal per run,
verifies it with the commands written in the goal, commits as `G<nn>: <title>` on `agent/plan`
and pushes that branch. The owner opens one draft pull request from `agent/plan` into `main` once,
reviews it and merges it with a merge commit.

The owner writes only on `main`, and only in these places: `plan/memory/owner-inputs.md` (inputs,
approvals, block clearances, filed issue URLs) and the `Status` and `## Decision` of ADRs. The loop
never edits `plan/memory/owner-inputs.md`; it merges `main` into `agent/plan` at the start of every
run and reads the owner's changes from there. Goals of type HUMAN wait for an owner input described
in their precondition. A goal that fails three times opens a block that only the owner can clear.

## Security disclosure

Weaknesses in the specification itself are reported privately through GitHub private
vulnerability reporting on this repository. Weaknesses in a port are reported through that port's
own security policy. Undisclosed weaknesses are never described in public issues, pull requests,
reports, declarations, inventories or divergence tables here; the spec states the safe behaviour
without naming the ports that do not yet implement it. The owner decides when material about a
port's security may become public, after the port's advisory is published.
