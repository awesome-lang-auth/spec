# awesome-lang-auth specification

The shared vision, normative specification and conformance suite of the
[awesome-lang-auth](https://github.com/awesome-lang-auth) family: authentication libraries and
deployments for several languages that speak one wire contract.

> **Status: pre-1.0 draft.** Nothing in this repository is normative yet, and no port claims
> conformance. The content is being written goal by goal by an agentic loop under owner review
> (see [the plan](plan/plan.md)).

## Vision

Every port of the family (servers in TypeScript, Go, Python, Rust and Dart, a serverless AWS
deployment, and clients for Flutter, Angular and React) should behave the same on the wire, so
that any client works with any server and an application can change server language without
changing its front end.

Today that contract lives in the code of the reference implementation, awesome-node-auth, and in
prose spread across READMEs. The survey of 2026-10-01 ([baseline](plan/baseline.md)) found that
ports track different versions of the reference, that none shares test vectors with another, and
that error shapes, status codes and defaults differ in ways no test catches.

This repository turns that implicit contract into:

- a **normative specification**, one file per area, with numbered, testable requirements written
  with the RFC 2119 and RFC 8174 keywords;
- a **security baseline** that every port meets, stated as requirements and tied to a threat
  model;
- a **conformance suite**: JSON vectors validated by JSON Schema, and a runner contract that every
  port implements through a small test harness;
- a **parity matrix** generated from per-port declarations;
- **ADRs** that record every decision, in particular every place where the spec deliberately
  departs from the reference.

## Scope

In scope: the HTTP wire contract of an auth server (transport, cookies, CSRF, CORS, abuse
protection, tokens and claims, sessions, refresh and revocation, password, email verification and
reset, magic links and one-time codes, TOTP, OAuth with account linking, account lifecycle and
deletion hooks, the admin API, API keys, events and webhooks, errors, configuration and secure
defaults, identity-provider mode and JWKS, resource-server verification), and the behaviour of
client libraries on that wire.

Out of scope: language-level APIs of each port (each stays idiomatic), storage schemas, UI pages
and styling, deployment tooling, and certification against external standards such as OpenID
Connect. Tenants, roles and permissions are outside spec 1.x except for reserved names (see the
plan).

## Repository layout

Planned layout; directories appear as the plan's goals deliver them.

| Path | Content |
|---|---|
| `README.md`, [GOVERNANCE.md](GOVERNANCE.md), `CONTRIBUTING.md`, `SECURITY.md`, `CHANGELOG.md` | Project documents |
| `docs/vision.md` | Long-form vision |
| `spec/` | Normative specification, one file per area, plus the area and profile registries |
| `adr/` | Architecture decision records |
| `inventory/` | Machine-readable evidence of the pinned reference implementation (non-normative) |
| `conformance/` | Vector files, JSON Schemas, the runner contract, the porting guide, the list of untested requirements |
| `runner/` | Reference runner (Node) and the reference harness |
| `parity/` | Port registry, per-port declarations, generated parity matrix |
| `reports/` | Per-port conformance review reports and issue drafts |
| `security/` | Threat model, release checklist, release evidence |
| `tools/` | Linters and generators run by `npm run verify` and CI |
| [plan/plan.md](plan/plan.md), [plan/baseline.md](plan/baseline.md), `plan/memory/` | The agentic loop's plan, starting facts, loop memory and owner inputs |

## How ports conform

1. A port picks the **profiles** it implements: `core` plus optional modules such as `email`,
   `passwordless`, `two-factor`, `oauth`, `admin`, `api-keys`, `events`, `webhooks`, `tools`,
   `idp`, `oidc-provider` and `resource-server` for servers, and `client-core` with
   `client-cookie` or `client-bearer` for clients.
2. It implements the **harness** described by the runner contract (test builds only) and runs the
   conformance vectors of those profiles with a runner.
3. It publishes a **declaration** in `parity/ports/<id>.json`: one status per requirement (`pass`,
   `fail`, `deviation` with a public tracking link, `not-applicable`, `untested`).
4. When every MUST requirement of its profiles passes, it may state
   "conforms to awesome-lang-auth spec vX.Y, profiles: …" with a link to its report.

Details: [GOVERNANCE.md § Conformance claims](GOVERNANCE.md#conformance-claims).

## Versioning

The specification follows Semantic Versioning, independently of any port. Ports claim
conformance to a `MAJOR.MINOR`. A new MUST requirement, or a stricter one, is a major change; new
optional profiles, SHOULD or MAY requirements and new vectors for existing requirements are minor
or patch changes. See [GOVERNANCE.md § Spec versioning](GOVERNANCE.md#spec-versioning).

## Status

Bootstrapped on 2026-10-02. Version `0.1.0-draft`. The reference implementation is pinned at
awesome-node-auth 1.10.8. Progress is tracked in [plan/plan.md](plan/plan.md); the target is spec
1.0.0 with vectors for every requirement, a reference harness that runs them against the pinned
reference, and a reviewed declaration for each of the nine ports.

## Licence

Not yet chosen: the owner decides it in goal G02. Until then no licence is granted beyond what
the GitHub terms of service allow.
