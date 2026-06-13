# Roadmap

This roadmap stacks QuAuthz from the shipped RFC 8693 STS foundation through a
complete OAuth 2.1 + OIDC + DPoP + PQC Authorization Server. Each phase must keep
the same security boundary: QuAuthz composes standard OAuth, OIDC, JOSE, and
post-quantum cryptography rather than inventing new protocol primitives or
reimplementing mature platform components.

OAuth 2.1 is a draft as of June 2026 (draft-ietf-oauth-v2-1-15). All OAuth 2.1
claims in this roadmap mean `OAuth 2.1 draft-aligned` until the RFC is published.

## Current Status

Phase 0 is complete. The existing `sts-delegate-rs` STS is shipped at 87% RFC
conformance. Phase 1 is the next active phase.

## Milestone Table

| Phase | Name | Status | Target window | Conformance target |
|---|---|---|---|---|
| 0 | STS Foundation (done) | Complete | Shipped | 87% (70/80 audited requirements) |
| 1 | OAuth 2.1 AS MVP | Planning / next | 8 weeks from Tier 1 start | 90%+ RFC 7636, RFC 8414, OIDC Core mandatory clauses |
| 2 | Production Baseline | Not started | 8 weeks after Phase 1 | RFC 7009, RFC 7662, RFC 7591, RFC 8707 |
| 3 | High-Security Profile | Not started | 4 weeks after Phase 2 | RFC 9126, RFC 9101, RFC 8705, FAPI 2.0 alignment |
| 4 | Delegation / MCP Profile | Not started | 4 weeks after Phase 3 | RFC 8693 AS integration, RFC 9728 |
| 5 | PQC Profile | Not started | 8 weeks after Phase 4 | FIPS 204 ML-DSA-65 ID Token, ML-KEM-768 planned |

## Phase 0: STS Foundation (Complete)

Goal: ship a correct, PQC-first RFC 8693 token-exchange STS that is auditable and
conformant enough for production pilots.

Deliverables (shipped in `sts-delegate-rs`):

- `POST /token` — RFC 8693 token exchange with `act` claim.
- `POST /introspect` — token introspection.
- `POST /revoke` — token revocation.
- `GET /jwks` — public key publication.
- `GET /.well-known/oauth-authorization-server` — RFC 8414 metadata.
- ML-DSA-65 default signing (FIPS 204 / OpenSSL 3.x).
- DPoP proof validation (RFC 9449) and `cnf.jkt` sender-constraining.
- RFC 7523 `private_key_jwt` actor authentication.
- Replay detection via per-process JTI store.
- 87% RFC conformance (70/80 requirements audited in June 2026 audit).

Conformance target: 87% achieved.

Key RFCs: RFC 8693, RFC 8414, RFC 9449, RFC 7523, RFC 8725, RFC 9068.

Risks (residual):

- Remaining 13% of audited requirements are tracked in the `sts-delegate-rs`
  conformance ledger and open issues. They do not block Phase 1 AS work but
  should be closed before Phase 1 ships to avoid protocol inconsistency between
  the STS and AS layers.

## Phase 1: OAuth 2.1 AS MVP (Tier 1 — Building)

Goal: ship a local-first OAuth 2.1 Authorization Server with browser-based
authentication, Authorization Code + PKCE, ID Tokens, Access Tokens, UserInfo,
and OIDC Discovery.

Target window: 8 weeks from implementation start.

Deliverables:

- Rust workspace scaffolded: `sts-as-http`, `sts-as-core`, `sts-as-store`,
  `sts-as-config`, `sts-as-cli`.
- Shared crates extracted or vendored: `sts-jose`, `sts-dpop`, `sts-replay`,
  `sts-exchange`, `sts-verify`.
- `GET /authorize` — authorization endpoint; PKCE required; redirect URI exact match.
- `POST /token` — authorization-code grant; `refresh_token` grant.
- `GET /userinfo` — claims within authorized scope; no over-disclosure.
- `GET /.well-known/openid-configuration` — OIDC Discovery.
- `GET /.well-known/oauth-authorization-server` — RFC 8414 metadata (extended).
- `GET /jwks` — public keys for ID Token and Access Token verification.
- Client registry: `client_id`, `redirect_uris`, grant types, scope allowlist.
- User model: stable `sub`, TOTP credential, recovery codes, session state.
- TOTP login and consent flow in browser.
- Durable SQLite store behind `sts-as-store` abstraction.
- DPoP proof validation at `/token`; `cnf.jkt` binding when DPoP present.
- ML-DSA-65 default signing for Access Token; opt-in RS256.
- Signed ID Tokens per OIDC Core.
- `sts-as-cli bootstrap` quickstart: generates key, writes config, seeds client/user.
- Redacted audit events: no raw token values, codes, or credential material.
- Conformance ledger and negative-path test harness.

Negative paths required before Phase 1 ships:

- redirect URI mismatch;
- PKCE verifier failure;
- authorization code replay;
- issuer mix-up (RFC 9207);
- wrong audience;
- expired Access Token;
- expired ID Token;
- revoked session / logout;
- broad Access Token rejected at Resource Server;
- missing `nonce` in ID Token when `nonce` was in authorization request.

Conformance target: 90%+ of mandatory BCP14 requirements in RFC 7636, RFC 8414,
and OIDC Core mandatory clauses mapped to executable test cases with pass/fail
evidence.

Key RFCs: OAuth 2.1 draft-ietf-oauth-v2-1-15 (draft-aligned), RFC 7636, RFC 8414,
RFC 9207, RFC 9068, RFC 9449, OIDC Core (OpenID.Core), OIDC Discovery
(OpenID.Discovery).

Risks:

- Open decisions in the PRD (authentication method, storage abstraction, DPoP
  policy, PQC scope) must be resolved before implementation starts. Leaving them
  open produces architecture drift.
- `sts-exchange` extraction from `sts-delegate-rs` may uncover boundary issues;
  plan for one iteration cycle on the shared crate seam.
- TOTP enrollment UX in a browser requires a minimal HTML template surface;
  define the template boundary before writing Axum handlers so transport does not
  own session logic.

Mitigation:

- Close PRD open decisions table before writing the first `crates/` file.
- Write a failing negative-path test for each required rejection before writing
  the success path for the same endpoint.
- Keep `sts-as-http` route handlers thin: extract, validate, call `sts-as-core`,
  return. No policy in handlers.

Done when:

- Local quickstart: `sts-as-cli bootstrap` followed by a browser PKCE flow with
  TOTP, a signed ID Token, and a signed Access Token, in under 10 minutes on a
  fresh machine.
- `cargo test --workspace` passes with coverage over 75%.
- `cargo clippy --workspace --all-targets -- -D warnings` passes.
- All negative-path tests listed above pass.
- Conformance ledger rows are closed with test evidence.

## Phase 2: Production Baseline (Tier 2)

Goal: add the protocol surfaces required for a production-grade deployment:
revocation, introspection, Dynamic Client Registration, resource indicators, and
Postgres storage.

Target window: 8 weeks after Phase 1.

Deliverables:

- `POST /revoke` at the AS layer — RFC 7009 token revocation.
- `POST /introspect` at the AS layer — RFC 7662 token introspection.
- Dynamic Client Registration — RFC 7591 registration endpoint; RFC 7592
  management endpoint.
- Resource Indicators — RFC 8707 `resource` parameter at `/authorize` and `/token`.
- Issuer mix-up defense — RFC 9207 `iss` parameter in authorization response.
- Postgres backend behind `sts-as-store` abstraction; SQLite remains for local dev.
- Refresh Token rotation with family-based reuse detection and revocation cascade.
- Security event logging: structured events for every issuance, revocation,
  registration, and authentication event; no raw credential values.
- `cargo audit` and `cargo deny` in CI; dependency pinning for crypto crates.

Conformance target: RFC 7009 and RFC 7662 mandatory clauses closed; RFC 7591
registration requirements mapped to test cases.

Key RFCs: RFC 7009, RFC 7662, RFC 7591, RFC 7592, RFC 8707, RFC 9207, RFC 9700.

Risks:

- Postgres migration from SQLite-first schema requires careful abstraction boundary
  in `sts-as-store`; cost is lower if the abstraction is trait-based from Phase 1.
- RFC 7591 Dynamic Client Registration opens an attack surface (unauthenticated
  client registration); requires rate limiting and initial access token gating before
  exposing publicly.

Mitigation:

- Validate the `sts-as-store` abstraction boundary in Phase 1; add Postgres backend
  in Phase 2 without touching `sts-as-core`.
- Initial access token requirement for dynamic registration is mandatory, not optional.

Done when:

- Token revocation invalidates the Access Token and any associated Refresh Token
  family; a revoked token is rejected at the AS and at any Resource Server using
  introspection.
- Dynamic Client Registration issues a `client_id` and `registration_access_token`;
  registered client can complete a full PKCE flow.
- Postgres backend passes the same integration test suite as SQLite.

## Phase 3: High-Security Profile (Tier 3)

Goal: add the high-security protocol surfaces required for FAPI 2.0 alignment,
financial-grade clients, and DPoP enforcement by policy.

Target window: 4 weeks after Phase 2.

Deliverables:

- Pushed Authorization Requests (RFC 9126) — `POST /par`; redirect with `request_uri`.
- Request Objects (RFC 9101 / JAR) — signed authorization request objects.
- JARM — signed authorization response.
- DPoP required by policy — per-client or per-resource DPoP enforcement; Access
  Token issuance fails without a valid DPoP proof when policy requires it.
- mTLS sender-constraining (RFC 8705) — `cnf.x5t#S256` in token when mTLS used.
- FAPI 2.0 alignment — evaluate open FAPI 2.0 Security Profile requirements;
  file remaining gaps.

Conformance target: RFC 9126 mandatory clauses; RFC 9101 signed request validation;
FAPI 2.0 Security Profile gap analysis complete.

Key RFCs: RFC 9126, RFC 9101, JARM, RFC 8705, FAPI 2.0 (openid-financial-api).

Risks:

- FAPI 2.0 certification requires a formal conformance test suite run; that is a
  separate effort from implementation. Phase 3 targets alignment, not certification.
- mTLS at the AS layer requires TLS termination to pass client certificate material
  to the AS; deployment topology (TLS-terminating proxy vs. native Axum TLS) must
  be resolved before implementation.

Done when:

- PAR request completes a full PKCE flow where the client does not send query
  parameters directly to `/authorize`.
- DPoP-required policy rejects a token request that lacks a valid DPoP proof.
- mTLS client certificate fingerprint appears as `cnf.x5t#S256` in the issued
  Access Token.

## Phase 4: Delegation / MCP Profile (Tier 4)

Goal: wire RFC 8693 delegation into the AS-issued token response and expose the
Protected Resource Metadata surface; build the MCP operator interface.

Target window: 4 weeks after Phase 3.

Deliverables:

- `sts-exchange` fully integrated at the AS token endpoint for `urn:ietf:params:oauth:grant-type:token-exchange` grant.
- `act` claim surface in AS-issued delegation tokens; no delegation path without `act`.
- RFC 9728 Protected Resource Metadata endpoint.
- MCP server surface for read-only posture and policy queries; no write actions
  without explicit operator approval and audit log entry.
- End-to-end delegation demo: browser PKCE login through QuAuthz AS → Access Token
  → RFC 8693 exchange at `sts-delegate-rs` → narrowed delegation token with
  `act` → downstream MCP tool.

Conformance target: RFC 8693 full delegation path; RFC 9728 RS metadata; `act` claim
present and auditable on every delegation.

Key RFCs: RFC 8693, RFC 8707, RFC 9068, RFC 9449, RFC 9728.

Risks:

- Wiring `sts-exchange` into the AS token endpoint must not collapse the AS/STS
  boundary; the AS issues upstream tokens; the STS narrows and stamps `act`.
  Coupling them in a single handler breaks the architecture invariant.
- MCP read-only surface must enforce the same policy engine as the CLI and API;
  no bypass path.

Done when:

- A single test harness can prove the complete chain: browser login → AS code grant
  → access token → STS exchange with `act` → downstream MCP tool validation.
- `act` is always present on delegation tokens; no impersonation mode exists.
- RFC 9728 metadata accurately describes the resource's accepted token formats.

## Phase 5: PQC Profile (Tier 5)

Goal: extend PQC signing to ID Tokens; verify ML-DSA-65 signed tokens in a separate
verifier process; plan ML-KEM-768 key exchange.

Target window: 8 weeks after Phase 4.

Deliverables:

- ML-DSA-65 ID Token signing verified by an external verifier process (not the
  binary that produced the token) using the `/jwks` published key.
- RS256 is explicit opt-in only; any attempt to default to RS256 or `none` is a
  startup error.
- JOSE PQC algorithm registration tracked against current IANA COSE/JOSE
  PQC registrations after IETF publication.
- ML-KEM-768 key exchange planning document; no implementation until IETF
  registration is stable.
- PQC policy surface: config option to enforce PQC-only algorithm for token signing;
  reject clients that cannot verify ML-DSA-65.
- PQC interoperability test pair: a separate consumer (such as `obo-lab` updated
  to use `sts-authority` as its upstream AS) verifies ML-DSA-65 signed tokens end
  to end.

Conformance target: FIPS 204 ML-DSA-65 signing and verification; current JOSE/COSE
PQC IANA registration state documented and tracked.

Key standards: FIPS 204 (ML-DSA), FIPS 203 (ML-KEM), RFC 8725, current JOSE/COSE
PQC IANA registrations.

Risks:

- External client support for ML-DSA-65 JWT verification is immature as of
  mid-2026. Phase 5 claims must name the specific verifier library and version
  used in the interoperability test.
- JOSE PQC algorithm identifiers may not be finalized at IANA before Phase 5
  starts; track the IETF COSE/JOSE working group and update algorithm identifiers
  when registration is stable.
- ML-KEM-768 key exchange is a different surface from token signing; do not conflate
  them in claims or docs.

Mitigation:

- Test with a real verifier, not the same binary that signed.
- Freeze algorithm identifiers at the IANA registration state at the time Phase 5
  ships; note the date in the conformance ledger.
- ML-KEM is a roadmap planning item only; do not ship a half-implemented KEM crate.

Done when:

- An external verifier process (separate binary or separate crate integration test)
  fetches `/jwks`, obtains the ML-DSA-65 public key, and verifies a signed ID Token
  produced by `sts-authority`.
- RS256 cannot be enabled as a default without an explicit config flag and a
  startup warning.
- Algorithm identifiers in token headers match current IANA registration state.
