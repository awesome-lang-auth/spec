# Survey baseline (2026-10-01)

Starting facts for the loop, from a read-only survey of the ten repositories of the
awesome-lang-auth organisation made on 2026-10-01 and 2026-10-02. The survey read the code; it ran
the test suite of awesome-flutter-auth only. These notes are evidence, not requirements: the
normative text follows the pinned inventory of the reference (`inventory/node-1.10.8/`, built by
G05 to G07) and the ADRs. When the inventory and this file disagree, the inventory wins.

**What is not here.** The survey also found security-sensitive weaknesses in several ports. They
are not listed in this public repository (plan rule 12), neither directly nor by omission: this
file records wire shapes, names, codes and defaults, plus behaviour that a port documents itself or
that a public issue already describes, and it says nothing else about the security properties of
any port. The owner holds the findings in the private directory and handles them through each
port's private advisory process. The spec's security baseline demands the safe behaviour without
naming who fails it.

## Ports

| Port id | Repository | Role | Language | Version surveyed | Commit | Contract it tracks |
|---|---|---|---|---|---|---|
| node | awesome-lang-auth/awesome-node-auth | server, **reference** | TypeScript (Express 5, Fastify adapter) | 1.10.8 (npm `@awesome-lang-auth/node`) | `7e640b4` (2026-09-29) | itself |
| go | awesome-lang-auth/awesome-go-auth | server | Go 1.25 (net/http, chi, gin, echo adapters) | v0.12.0 plus unreleased docs fix | `c9bc62c` (2026-10-01) | node 1.9.0 (`cc01e997`) |
| python | awesome-lang-auth/awesome-python-auth | server | Python ≥ 3.11, FastAPI | 1.1.0 (PyPI), main ahead of it | `1ee4ad3` (2026-09-28) | parity audit against node 1.10 closed as stale (#10) |
| rust | awesome-lang-auth/awesome-rust-auth | server toolkit | Rust 2024 | 1.9.0 in Cargo.toml, not published | `acb3d53` (2026-09-30) | node 1.10 audit (#13, closed) |
| dart | awesome-lang-auth/awesome-dart-auth | server | Dart 3.11 (Shelf, Dart Frog adapters) | 1.9.0 in pubspecs, not published | `5911c77` (2026-09-26) | node 1.10 events (#12) |
| lambda | awesome-lang-auth/awesome-lambda-auth | serverless deployment | Go 1.25, AWS SAM | unversioned | `4330384` (2026-09-30) | node 1.9.0 plus a later development line, through awesome-go-auth v0.11.0 |
| flutter | awesome-lang-auth/awesome-flutter-auth | client | Dart (pure Dart since 1.10.2) | 1.10.5 (pub.dev `awesome_flutter_auth`) | `7fc1001` (2026-09-29) | node 1.10 audit (#17, #19) |
| angular | awesome-lang-auth/awesome-angular-auth | client | TypeScript, Angular 21 | 1.10.0 on `develop` (npm still has `ng-awesome-node-auth` 1.9.0) | `c6d255a` (2026-09-28) | node 1.10.0 |
| react | awesome-lang-auth/awesome-react-auth | client | TypeScript, React 18/19 | 0.1.0 (npm `@awesome-lang-auth/react`) | `e343eed` (2026-09-25) | node wire protocol |

The tenth repository, awesome-lang-auth/awesome-lang-auth, is the documentation site (Docusaurus,
commit `d7f62fb`). It holds no auth code; its pages describe the node reference in prose, and its
weekly runtime check was red on 2026-09-28 because the Flutter entry still names the discontinued
`awesome_node_auth_flutter` package.

## Shared artefacts and prior art

- No repository shares test vectors with another. Each suite tests its own port against mocks or
  against its own reading of the reference.
- awesome-lambda-auth `test/contract`: a black-box HTTP suite (about 72 cases, standard library
  only) parametrised by `AWESOME_AUTH_CONTRACT_BASE_URL`; it runs against a deployed stack or the
  reference Express demo, probes capabilities, and `AWESOME_AUTH_CONTRACT_REQUIRE` turns a missing
  capability into a failure. Its `docs/spec/wire-contract.md` is a normative reading of node 1.9.0
  with file and line citations.
- awesome-go-auth: a cross-adapter wire suite (`adapter/internal/wiretest`) runs the same tests on
  four HTTP adapters; `CompatibilityNotes()` is an executable register of deliberate deviations
  that generates the README section; `TestCSRFExemptionsMatchTheReference` pins the CSRF
  exemption table.
- awesome-lambda-auth keeps three executable deviation registers (product, store, core) logged at
  cold start and pinned by tests.
- CI: go, lambda, python, rust, flutter and react run tests on pull requests; dart's workflow runs
  on pull requests but currently discovers no package and so tests nothing (PR #16 fixes it); node
  runs tests only in its publish workflow; angular's only workflow builds and publishes.

## Transport

- node: cookie transport by default; `X-Auth-Strategy: bearer` returns `accessToken` and
  `refreshToken` in the body instead of cookies; `Authorization: Bearer` takes precedence over the
  cookie. Cookies `accessToken`, `refreshToken`, `csrf-token`; with `cookieOptions.secure` and no
  domain or custom path the names become `__Host-*` with Path `/` (the refresh cookie included),
  otherwise `__Secure-*`; readers accept the three variants. Defaults: Secure `false`, SameSite
  `lax`, Path `/`, refresh cookie Path `<apiPrefix>/refresh` outside `__Host-`. Max-Age follows the
  token lifetime. CSRF is a double submit of the `csrf-token` cookie (15 minutes, 16 random bytes,
  hex) against `X-CSRF-Token`; `csrf.enabled` is off by default. CORS echoes allow-listed origins
  and allows `X-Auth-Strategy` (fixed in #5).
- node documentation drift: the detailed README says the refresh cookie is always scoped to
  `<apiPrefix>/refresh`; under `__Host-` it is Path `/`.
- go: same cookie names, prefixes and read priority; Secure defaults to `true`; bearer opt-in is an
  exact match of `X-Auth-Strategy: bearer`; CSRF exemption table pinned by test; no `/csrf`
  endpoint, the cookie is handed out by middleware.
- lambda: CSRF on by default and refused off in cookie mode (rule RS-3); cookie Max-Age derived
  from the configured TTLs; its CORS allow-list does not include `X-Auth-Strategy` (documented by
  awesome-react-auth and the site), so bearer mode from a browser on another origin fails the
  preflight.
- python: default API prefix `/api/auth`; cookie names `access-token` and `refresh-token`; optional
  `cookie_prefix`.
- dart: the router sets no auth cookie and protected routes read only `Authorization: Bearer`;
  refresh takes `{refreshToken}` in the body. Its README describes a cookie strategy.
- rust: a toolkit of services; its HTTP adapters serve static UI assets and a health check, and
  `POST /auth/login` is a stub. No auth HTTP surface yet.

## Errors

- node: `{error, code?}`; many 4xx answers carry no code; a missing or invalid access token answers
  403 without a code (`No access token provided`, `Invalid or expired access token`); only
  `SESSION_REVOKED` is a 401 from the auth middleware; the JWKS middleware answers 401
  `INVALID_TOKEN`; some "not supported by the store" cases answer 500 instead of 501.
- go: same envelope and the same code-less 403; codes include `OAUTH_STATE_INVALID` (node:
  `INVALID_OAUTH_STATE`), `INVALID_BODY` on an empty body (#34), `WEAK_PASSWORD`, `RATE_LIMITED`.
- lambda: as go, plus 429 `RATE_LIMITED` with `Retry-After` and `OFFSET_TOO_LARGE` on admin lists;
  its OIDC endpoints use RFC 6749 error bodies.
- python: FastAPI `{detail}` for most errors; `/refresh` answers
  `{success: false, code: "SESSION_REVOKED", message}`; validation errors are 422.
- dart: `{error}` with free text and no codes; some answers are plain text.
- rust: an error enum with no HTTP status mapping.
- Clients: flutter, angular and react refresh on 401 or 403 and stop on `SESSION_REVOKED`;
  awesome-flutter-auth #33 asks the reference for a machine-readable code on the missing-token
  answer.

## Tokens

- node: HS256 access and refresh tokens with the claims `sub`, `email`, `role`, `loginProvider`,
  `isEmailVerified`, `isTotpEnabled`, optional `sid`, plus `buildTokenPayload(user)`. Defaults:
  access 15 minutes, refresh 7 days. A `purpose` claim marks the 2FA temp token (5 minutes) and
  the admin console token (24 hours).
- go: reserved claims `sid`, `tid`, `jti`, `typ` (`access`, `refresh`, `temp`, `admin`), `iss`;
  one `Config.Secret` signs every token kind (#25, PR #97 adds `RefreshSecret`); refresh default
  30 days.
- lambda: typed tokens (`typ`), distinct secrets of at least 32 characters required at start;
  refresh default 7 days.
- dart: `typ` `access` and `refresh`; `iss` and `aud` written into tokens; refresh default 30 days.
- rust: refresh default 30 days.

## Sessions

- node: without a session store, one refresh token per user (single device); with one, each login
  creates a session (`sid` = session handle) storing the SHA-256 of the refresh token, and
  `/refresh` revokes the old session and creates a new one. `session.checkOn` is `allcalls`,
  `refresh` (default) or `none`; `allcalls` answers 401 `SESSION_REVOKED`. `singleSessionPerUser`
  revokes the other sessions at login. The device list is `GET /sessions` and
  `DELETE /sessions/:handle`. `POST /sessions/cleanup` is unauthenticated, as the site documents.
- go: rotation inside the same session (the `sid` is kept); `SessionCheckOn` default `refresh`.
- lambda: the session is a refresh family; replaying a rotated-out token revokes the whole session
  and later refreshes answer 401 `SESSION_REVOKED`; rotation is one conditional DynamoDB
  transaction; `POST /sessions/cleanup` always reports `{deleted: 0}` because expiry is by TTL.
- dart: `GET /sessions` always returns an empty list (its README says it lists sessions).
- Session list field names differ: python `lastActiveAt`; angular `SessionInfo.lastActive`;
  flutter reads `sessionHandle` with a `handle` fallback.

## Password and email

- node: `emailVerificationMode` `none` (default), `lazy` (blocked after
  `emailVerificationDeadline`) or `strict`, with `EMAIL_NOT_VERIFIED` and
  `EMAIL_VERIFICATION_REQUIRED`; `POST /forgot-password` always answers `{success: true}` with a
  1-hour token; verification token 24 hours, answered as JSON by `GET /verify-email` (the site
  describes a redirect); change-email token 1 hour with a notice to the old address;
  `POST /change-password` does not need `currentPassword` when the account has no password.
- go: applies the password policy on reset and change too (registered deviation); forgot-password
  answers 200 even when delivery fails; #30 (a passwordless account can never set a password) and
  #35 (values trimmed) are open.
- lambda: `WEAK_PASSWORD` on reset and change; `email.verification.mode` `none`, `lazy`, `strict`.
- python: no email-verification policy (stated in PR #14).

## Account lifecycle

- node: `POST /register` is mounted only with `onRegister` or `defaultRegister: true` and answers
  201 `{success, userId}` without a session. `DELETE /account` awaits `onBeforeDeleteUser(userId,
  {req, source: "self"})` (#20), then revokes sessions and removes roles, tenant memberships and
  metadata, and publishes `identity.user.deleted`.
- go: register issues a token pair (#21; PR #98 adds `IssueSessionOnRegister`, off by default);
  `DELETE /account` has no deletion hook.
- lambda: register is always mounted and issues a session (registered deviation).
- python PR #14, rust PR #15 and dart PR #15 add `issueSessionOnRegister` (default off), following
  the family decision that started in awesome-go-auth #21. dart register currently answers 200 with
  tokens.

## Passwordless and two-factor

- node: password login with 2FA answers `200 {requiresTwoFactor: true, tempToken,
  available2faMethods}` with methods among `totp`, `sms`, `magic-link`; forced enrolment answers
  `403 {requires2FASetup: true, tempToken, code: "2FA_SETUP_REQUIRED"}`. `POST /2fa/setup` returns
  `secret`, `otpauthUrl` and a `qrCode` data URL. Magic links last 15 minutes, SMS codes are 6 digits
  valid 10 minutes, both with a `mode: "2fa"` step-up variant. The site documents that magic-link
  and SMS logins skip TOTP when TOTP is enabled.
- go and lambda: `/2fa/setup` returns no `qrCode` (registered deviation); lambda's OAuth callback
  and admin login skip the second factor (registered deviations); lambda lists in
  `available2faMethods` only factors its stores can complete.
- flutter: `qrCode` optional since #22; forced enrolment comes back as a plain failure.

## OAuth

- node: `GET /oauth/google`, `/oauth/github`, `/oauth/:name` with `?return_path=` and their
  callbacks. State is HMAC-signed with a 10-minute expiry and binds a nonce cookie
  `oauth_nonce_<provider>` (scoped to the callback path, compared in constant time; #14), the
  origin and the return path (#22, #26). Return paths start with a single `/`, carry no backslash or
  control character and are at most 512 characters. Redirect origins come from the site URLs and
  CORS origins; production refuses OAuth start with an empty list (`OAUTH_ORIGIN_ALLOWLIST_EMPTY`).
  Account conflicts redirect to `account-conflict` (#15). The 2FA redirect after a callback is
  documented as `${origin}/auth/2fa?tempToken=…`.
- go: PKCE S256 always; a provisioning policy (auto-create, domain allowlist, verified email,
  `OnEmailMatch`); the default links by an unverified provider email (#36, PR #99 changes the
  default to the conflict flow).
- lambda: declarative `oauth.provisioning` policy inherited from go; start is refused without a
  redirect allowlist (RS-11).
- python and dart: OAuth routes delegate to application hooks; no built-in providers.
- rust: no OAuth flow.

## Admin

- node: admin router at `<apiPrefix>/admin` with access policies `first-user`, `is-admin-flag`,
  `open` or a predicate; `rootUser`.
- go: the admin router is not mounted without an `AccessPolicy`; port-only route
  `POST /users/{id}/promote`.
- lambda: `first-user` refused (RS-17); admin login has no second factor (registered deviation);
  `GET /admin/api/actions` always empty.
- python: the bundled admin SPA (copied from node) calls endpoints the Python admin router does not
  have.
- rust: admin API not implemented.

## Events and webhooks

- node: `identity.*` names, among them `identity.user.created`, `.deleted`, `.email.verified`,
  `.email.changed`, `.password.changed`, `.2fa.enabled`, `.2fa.disabled`, `.linked`, `.unlinked`,
  `identity.session.created`, `.revoked`, `.expired`, `.rotated`, `identity.auth.login.success`,
  `.login.failed`, `.logout`, `.oauth.success`, `.oauth.conflict`, `identity.tenant.*`,
  `identity.role.assigned`, `.revoked`, `identity.permission.granted`, `.revoked`. The session
  created/revoked/expired, user linked/unlinked, tenant and permission events are defined but not
  published. Payload: `ip`, `userAgent`, `correlationId`, `userId`, `sessionId`, `data`.
  Outgoing webhooks carry `X-Webhook-Event`, `X-Webhook-Delivery`, `X-Webhook-Timestamp` and
  `X-Webhook-Signature: sha256=<hex>`. Inbound webhook scripts run in `node:vm`, documented as not
  a security boundary (#10).
- go: 24 names, all published; the tools router is not served without an explicit access
  middleware; inbound scripts run out of process through a seam.
- lambda: events only when tools are enabled; `identity.session.rotated` reports
  `previousSessionId == sessionId`; outgoing webhooks optionally through SQS with the reference's
  1 s, 2 s, 4 s retry schedule; inbound scripts in a separate Lambda.
- python: publishes a subset since #13 (unreleased after 1.1.0).
- rust: PascalCase `EventType` values instead of `identity.*` names (#12).
- dart: five `identity.*` events (#12).

## Configuration

- API prefix default `/auth` (node, go, lambda, dart); `/api/auth` (python).
- Refresh lifetime default: 7 days (node, lambda, python), 30 days (go, dart, rust).
- Cookie Secure default: `false` (node), `true` (go, python).
- lambda refuses to start on unsafe configurations (rules RS-1 to RS-18): short or equal secrets,
  CSRF off in cookie mode, SameSite `none` without Secure, OAuth without allowlist, admin without
  policy, tools without an explicit auth posture, memory store in production, among others.

## Identity provider and resource server

- node: IdP mode publishes a single RSA key (`kid` `provisioner-key-1`) at
  `<apiPrefix>/.well-known/jwks.json` (the site documents a root path) with `Cache-Control:
  max-age=3600`; resource-server mode verifies RS256 tokens through a remote JWKS. #36 proposes an
  OIDC authorization server aligned with go and lambda.
- go: OIDC discovery, `/authorize`, `/token` (authorization code only, PKCE recorded but not
  verified) and `/userinfo` (#14, PR #97 adds PKCE verification, the refresh grant and
  `client_secret_basic`); JWKS client pins RS256 before key lookup.
- lambda: the same OIDC surface through go, with KMS or PEM keys, `kid` derived from the SPKI hash
  and additive rotation; #20 tracks the refresh grant after go PR #97.

## Clients

- flutter 1.10.5: cookies with CSRF on web, bearer with `X-Auth-Strategy: bearer` on native;
  single-flight refresh on 401 or 403 with one retry; `SESSION_REVOKED` signs out without a
  refresh; CSRF header only on same-origin requests; the `sessionExpired` event is declared but not
  emitted; on native the refresh token is kept in memory only; #33 (a late 401 after logout
  triggers a refresh without CSRF).
- angular 1.10.0: cookie transport only, no bearer mode; CSRF header only for the API origin;
  refresh exemptions matched with `url.includes`; a failed refresh surfaces as
  `Error("Session expired")`; `UiConfigService` defaults its prefix to `/api/auth` while the core
  defaults to `/auth`.
- react 0.1.0: cookie and bearer transports; refresh only when the error has no code or
  `INVALID_TOKEN`; a no-retry list of ten credential routes.
- All three take the user from `GET /me`; none ships offline token verification for the auth
  wire contract.

## Public issues and pull requests referenced by the plan

- awesome-node-auth: #36 (OIDC authorization server proposal); closed #5, #10, #13, #14, #15, #20,
  #22, #26.
- awesome-go-auth: #14, #21, #25, #30, #31, #32, #34, #35, #36, #37; PRs #93, #96, #97, #98, #99.
- awesome-python-auth: PR #14; closed #9, #10.
- awesome-rust-auth: #12; PR #15; closed #13.
- awesome-dart-auth: #12; PRs #15, #16.
- awesome-lambda-auth: #20; PRs #16 to #19.
- awesome-flutter-auth: #33; closed #17, #19, #22, #27, #29, #31.
- awesome-angular-auth: PR #10; closed #6, #7.
