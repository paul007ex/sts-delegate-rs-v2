# Workspace Boundaries

This document defines the crate/package shape for QuAuthz (`sts-authority`, planned
in `sts-delegate-rs-v2`). The goal is to keep the Authorization Server product
layered and auditable before code moves from planning to implementation, and to
prevent the existing STS from being widened into an Authorization Server by accident.

Current status: planning repo. No implementation crates exist yet. This document
makes ownership and dependency direction visible so that implementation branches
start in the right shape.

## Design Rule

The workspace should use a ports-and-adapters shape with a hard vertical split
between the Authorization Server product and the STS product:

```text
operator surfaces and user-facing commands
  sts-as-cli        future admin-api        future admin-ui
        |                  |                     |
        +------------------+---------------------+
                           |
                           v
AS application and HTTP transport
  sts-as-http        sts-as-config
        |                  |
        v                  v
AS domain and durable state
  sts-as-core        sts-as-store
        |                  |
        +------------------+
                 |
                 v
shared protocol crates (owned jointly with sts-delegate-rs)
  sts-jose    sts-dpop    sts-replay    sts-verify    sts-exchange
```

No lower-level crate should depend on `sts-as-http` or `sts-as-cli`. The AS
domain layer must not depend on specific storage backends, HTTP renderers, or STS
HTTP transport. Shared protocol crates must not depend on any AS or STS
application crate.

## Product Boundary: What QuAuthz Owns

QuAuthz (`sts-authority`) owns the Authorization Server / OpenID Provider surface.
This includes:

- `/authorize` endpoint: interactive authentication, session creation, PKCE
  challenge acceptance, consent prompt, authorization code issuance.
- Browser session subsystem: session creation, binding, expiry, revocation on
  logout.
- Client registry: registered clients, redirect URIs, grant types, token
  endpoint auth methods, per-client policy.
- Authorization Code lifecycle: one-time use, PKCE binding, short expiry.
- Token Endpoint AS grants: `authorization_code`, `refresh_token`, and
  `client_credentials` if in scope for MVP.
- Refresh token lifecycle: issuance, rotation, reuse detection, revocation family.
- ID Token issuance: OIDC Core claims, `nonce`, `at_hash`, signing.
- UserInfo endpoint: claim authorization against scope/grant, Access Token
  validation, no claim leakage outside authorized scopes.
- OIDC Discovery and OAuth AS Metadata: `/.well-known/openid-configuration` and
  `/.well-known/oauth-authorization-server`; advertised only when the runtime
  enforces the advertised behavior.
- JWKS publication: public keys only; key selection by `kid`; fail-closed on
  unknown key or unsupported algorithm.
- Consent and grant model: recorded per user/client/scope, queryable for revocation.
- User model: stable `sub`, passwordless-first authentication (TOTP + recovery
  codes for MVP, WebAuthn/passkeys planned), no password database by default.
- Revocation endpoint (Tier 2): RFC 7009 token revocation.
- Introspection endpoint (Tier 2): RFC 7662, access-controlled, no unnecessary
  claim return.
- Issuer mix-up defense: RFC 9207 `iss` in authorization response, exact issuer
  matching in all endpoints.
- Redacted audit events: authentication events, grant events, token issuance,
  revocation — redacted, not raw token logging.

## Product Boundary: What QuAuthz Does NOT Own

### The STS / Token Exchange Layer

`sts-delegate-rs` (the existing STS) owns RFC 8693 token exchange. That means:

- `POST /token` with `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`.
- Consuming a subject token (broad Access Token from the AS) and an actor token.
- Minting a narrowed delegated token with `sub` = user, `act` = actor (RFC 8693
  §4.1 — delegation, not impersonation).
- STS discovery metadata advertising the token-exchange grant type.

QuAuthz does not duplicate this. The shared `sts-exchange` crate (see below) holds
the RFC 8693 application service; both products depend on it rather than forking.
The STS HTTP transport (`sts-http` in `sts-delegate-rs`) is not imported by any AS
crate.

### Resource Servers

QuAuthz issues Access Tokens and publishes JWKS. Resource Servers validate those
tokens by fetching `/.well-known/jwks.json` and verifying issuer, audience, expiry,
scopes, and sender constraint. The Resource Server validation path is out of scope
for QuAuthz code. QuAuthz does not embed Resource Server middleware, token
introspection wrappers for RS use, or MCP-server logic. RS authors use the JWKS
endpoint and standard JWT libraries.

### External Identity Providers (Tier 2+)

QuAuthz is the local OpenID Provider for local flows. Federating with upstream
external IdPs (Okta, Entra, Google, GitHub, etc.) is Tier 2 or later work. MVP does
not include external IdP discovery, token translation, attribute mapping, or
account linking. Until federation is explicitly scoped, do not add OAuth
`authorization_code` relay, OIDC backchannel, or social-login callback routes.

### MCP Servers

MCP servers are Resource Servers. QuAuthz provides tokens; MCP servers validate
them. QuAuthz does not own MCP protocol, tool dispatch, or MCP session logic. The
MCP/OBO delegation demo (a separate lab) calls the STS token exchange after the AS
issues the initial Access Token — that demo is not QuAuthz source.

## Crate Responsibilities

| Crate | Responsibility | Must not own |
|---|---|---|
| `sts-as-core` | AS domain: grant lifecycle, session rules, client validation, ID Token claims assembly, UserInfo claim policy, consent records, RFC/OIDC rule enforcement | HTTP rendering, SQL queries, storage backend specifics, STS delegation logic |
| `sts-as-store` | Durable AS state: user records, client registry, sessions, authorization codes, refresh-token families, grant records, revocation lists | OAuth policy decisions, HTTP layer, domain rule enforcement |
| `sts-as-http` | Axum/HTTP transport: `/authorize`, `/token` AS grants, `/userinfo`, `/jwks`, discovery, logout, error responses, redirect handling, CSRF/state binding | Key custody, AS domain logic, SQL, STS HTTP routes, replay state decisions |
| `sts-as-config` | AS configuration schema, validation, defaulting, startup checks, advertised-vs-enforced consistency | Runtime grant logic, HTTP routing, storage writes |
| `sts-as-cli` | Bootstrap, client registration, user enrollment, admin operations, local dev flow | Long-running server state, privileged key generation unless delegated to a key provider |
| `sts-exchange` | RFC 8693 exchange application service — shared with `sts-delegate-rs`; subject token validation, actor token validation, delegated token minting, `act` claim assembly | AS session/client state, HTTP transport, STS or AS discovery metadata |
| `sts-jose` | JOSE/JWK/JWKS/JWT signing and verification, key parsing, algorithm selection, `kid` rotation | Policy decisions, HTTP, grant state |
| `sts-dpop` | DPoP proof parsing and validation, `cnf.jkt` binding, replay check contract (not the state store) | Grant logic, session state, HTTP response shaping |
| `sts-replay` | Replay-state policy and backend abstraction: `jti` store contract, TTL, per-process default, future shared-store adapter | OAuth grant decisions, token signing, HTTP |
| `sts-verify` | Issuer/trust-anchor validation, OIDC discovery fetch, JWKS fetch and rotation, algorithm allowlist | Application business logic, session or grant state |

## Shared Crates: Joint Ownership With sts-delegate-rs

The crates `sts-jose`, `sts-dpop`, `sts-replay`, `sts-verify`, and `sts-exchange`
are protocol-level shared crates. They belong to neither the STS product nor the AS
product exclusively. The governing rules are:

1. No duplication. If a protocol behavior exists in one of these crates, neither
   product adds a parallel implementation in its own application crate.
2. The shared crates must not import `sts-http` (STS transport) or `sts-as-http`
   (AS transport). Protocol logic must not depend on a specific HTTP framework.
3. Changes to shared crates that break the API surface require coordinated review
   from both products before merge.
4. `sts-exchange` is the one place where RFC 8693 application-service logic lives.
   The STS product and the AS product both call it; neither owns it privately.

The current `sts-delegate-rs` crate graph already contains `sts-jose`, `sts-dpop`,
`sts-replay`, and `sts-verify`. When QuAuthz implementation begins, these crates
move to a shared workspace or are published as local path dependencies rather than
being copied.

## Import Rules

The following imports are forbidden by architecture:

| From | To | Reason |
|---|---|---|
| `sts-as-core` | `sts-as-store` | Domain must not depend on a concrete storage backend |
| `sts-as-core` | `sts-as-http` | Domain must not depend on HTTP transport |
| `sts-as-core` | `sts-http` (STS crate) | AS domain must not import STS HTTP transport |
| `sts-as-store` | `sts-as-core` | Store is a passive data layer; it must not make policy decisions |
| `sts-as-http` | `sts-core` (STS crate) | AS transport must not import STS application logic |
| `sts-jose` | `sts-as-core` or `sts-as-store` | Shared protocol crate must not depend on AS state |
| `sts-jose` | `sts-core` (STS crate) | Shared protocol crate must not depend on STS application logic |
| `sts-dpop` | `sts-as-core` or `sts-as-store` | DPoP validation is stateless; replay is via `sts-replay` only |
| `sts-exchange` | `sts-as-http` or `sts-http` | Exchange service must not depend on either HTTP transport |
| `sts-replay` | `sts-as-core` or `sts-as-store` | Replay backend must not make grant decisions |
| `sts-verify` | `sts-as-core` or `sts-as-store` | Trust-anchor validation must not depend on AS grant state |

The repo will enforce this with a `scripts/check_workspace_boundaries.py` script
modeled on the equivalent in `breachsafe-pqvpn-rs`. A new crate or edge may only
be added when this document is updated and the boundary check passes.

## Allowed Crate Graph (Planned)

```text
sts-as-cli
  -> sts-as-config
  -> sts-as-core
  -> sts-as-store (bootstrap/admin path only)

sts-as-http
  -> sts-as-core
  -> sts-as-config
  -> sts-jose
  -> sts-dpop
  -> sts-replay
  -> sts-verify

sts-as-core
  -> sts-jose
  -> sts-exchange
  -> sts-verify

sts-as-store
  -> no local crates (pure data; depends only on DB driver externals)

sts-as-config
  -> no local crates

sts-exchange
  -> sts-jose
  -> sts-verify
  -> sts-dpop

sts-jose
  -> no local crates

sts-dpop
  -> sts-jose
  -> sts-replay (replay-check contract only)

sts-replay
  -> no local crates

sts-verify
  -> sts-jose
```

## What Is Explicitly Not In Scope

The following are out of scope for MVP and must not be silently introduced:

- **SAML**: no SP metadata, no AuthnRequest/Response parsing, no SAML token bridge.
  SAML is Tier 2+ enterprise surface and requires a separate scoping decision.
- **SCIM**: no SCIM 2.0 provisioning endpoints, no SCIM schema, no SCIM filter
  parsing. Provisioning is enterprise surface.
- **Social login / external IdP catalog**: no GitHub OAuth relay, no Google OIDC
  relay, no Okta upstream federation for MVP. Upstream federation is Tier 2+.
- **Password database by default**: the MVP authentication model is passwordless-
  first (TOTP + recovery codes). A password column must not appear in the user
  store schema as a default-on feature.
- **FAPI certification**: FAPI 2.0 Security Profile and certification claims are
  explicitly deferred. High-security profile work (PAR, JAR, JARM, mTLS, FAPI) is
  Tier 3 and must not be claimed in metadata or README until it is implemented and
  tested.
- **Full admin web console**: admin bootstrap and CLI tools are in scope. A
  production-quality web admin UX is enterprise surface.
- **Multi-tenant SaaS**: realm/tenant isolation, tenant-scoped JWKS, and
  per-tenant billing are not in MVP. The product is local-first.
- **Token introspection relay**: QuAuthz must not proxy introspection requests to
  an upstream IdP. Introspection (Tier 2) is local only.
- **MCP server implementation**: MCP tool dispatch, session management, and
  protocol framing are out of scope. QuAuthz provides tokens that MCP servers
  consume; it does not embed an MCP server.
- **PQC interoperability claims in MVP**: PQC signing is planned (Tier 5) but must
  not appear in conformance ledgers, metadata, or README as a supported feature
  until it is implemented, tested against verifiers, and advertised only at
  runtime when the selected algorithm is enforced.

## Security Review Checklist For New Crates Or Edges

Before adding a new crate or a new local dependency edge, answer:

- Does this make the dependency graph more layered or more tangled?
- Could this edge let HTTP transport make domain or policy decisions?
- Could this edge let the store make OAuth grant decisions?
- Could this edge let a shared protocol crate import AS or STS application state?
- Could this edge cause session secrets, raw tokens, or private key material to
  leak into CLI output or audit logs?
- Could this edge introduce a silent feature (social login, password column, SAML
  route) that is explicitly out of scope?
- Can the affected behavior be tested with accepted and rejected paths?

If any answer is unclear, open a planning issue before adding the edge.

## Comment Policy

Use comments where they preserve security or architecture intent:

- explain why a code path fails closed;
- explain trust boundaries, session or key custody boundaries, and secret handling;
- cite the RFC section or planning issue when behavior is non-obvious;
- document why a dependency is intentionally absent;
- mark planned scaffold as scaffold when it is not production behavior.

Do not comment to restate simple Rust mechanics, variable assignments, or function
return values.

## Python Tooling Policy

The product runtime is Rust. Python is acceptable for repository tooling when it is
isolated from shipped runtime behavior, uses the standard library where practical,
and is invoked explicitly from CI or developer commands.

Planned Python tooling:

- `scripts/check_workspace_boundaries.py`: validates that no crate imports a
  forbidden local dependency as defined in this document.

For the reference template see `breachsafe-pqvpn-rs/scripts/check_workspace_boundaries.py`.
