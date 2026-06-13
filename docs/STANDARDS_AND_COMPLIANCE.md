# Standards And Compliance Matrix

This document tracks standards, RFCs, guidance, and regulatory/control
frameworks relevant to QuAuthz (sts-delegate-rs-v2 / sts-authority).

Current status: planning artifact. This repo contains PRD and architecture
documents only. No production Authorization Server has shipped.

Last reviewed: 2026-06-13.

## Claim Classes

| Class | Meaning |
|---|---|
| Normative | Implementation must follow or explicitly reject the standard behavior |
| Product policy | QuAuthz chooses this behavior even if not mandated by a standard |
| Guidance | Informs architecture/security posture but is not a conformance claim |
| Draft/watch | Track status; do not market as RFC compliance |
| External assessment | Requires assessor, customer scope, or validated module boundary |

## Critical Status Notes

- OAuth 2.1 (`draft-ietf-oauth-v2-1-15`) is still a draft as of June 2026.
  All claims must say `OAuth 2.1 draft-aligned` until the document is published
  as an RFC. Do not claim OAuth 2.1 conformance.
- NIST FIPS 203 (ML-KEM) and FIPS 204 (ML-DSA) are final post-quantum
  cryptography standards as of 2024. FIPS 205 (SLH-DSA) is also final.
- FIPS 140-3 is a cryptographic module validation boundary. Using a FIPS
  algorithm or a FIPS-capable library does not make QuAuthz FIPS validated.
- OIDC Core 1.0 errata set 2 is the current baseline. The OpenID Foundation
  may publish errata set 3; track the upstream spec page.
- PQC JOSE drafts (`draft-ietf-jose-fully-specified-algorithms`,
  `draft-ietf-cose-dilithium`, and related) are in active IETF development.
  Do not claim PQC JOSE conformance or interoperability until the relevant
  draft is published as an RFC and at least one independent verifier is tested.
- Browser-Based Apps (`draft-ietf-oauth-browser-based-apps`) is still a draft.
  Use it as design direction, not a conformance reference.

## GROUP 1: OAuth 2.1 Core (Tier 1 — Local OIDC AS MVP)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 6749](https://www.rfc-editor.org/info/rfc6749/) OAuth 2.0 | Normative | 1 | Base OAuth semantics; OAuth 2.1 supersedes several provisions — track which and apply the stricter rule |
| [draft-ietf-oauth-v2-1-15](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/) OAuth 2.1 | Draft/watch | 1 | Say `draft-aligned` until RFC; drop implicit grant, resource-owner password grant, and plain PKCE; require PKCE for all authorization-code clients |
| [RFC 7636](https://www.rfc-editor.org/info/rfc7636/) PKCE | Normative | 1 | S256 required for all authorization-code flows; plain not accepted; code verifier bound to authorization code; replay must be rejected |
| [RFC 8414](https://www.rfc-editor.org/info/rfc8414/) AS metadata | Normative | 1 | Discovery document must match enforced behavior exactly; do not advertise grants, response types, auth methods, or algorithms that are not implemented |
| [RFC 9207](https://www.rfc-editor.org/info/rfc9207/) OAuth mix-up defense | Normative | 1 | Issuer identifier must be included in authorization responses; clients must validate `iss`; required for MVP security baseline |
| [draft-ietf-oauth-browser-based-apps](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/) | Draft/watch | 1 | Apply BFF and PKCE patterns from this draft; do not claim conformance; track on promotion to RFC |

## GROUP 2: Token Types And JOSE (Tier 1 — required for ID Token and JWT Access Token issuance)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 7519](https://www.rfc-editor.org/info/rfc7519/) JWT | Normative | 1 | All issued JWTs must validate `iss`, `aud`, `exp`, `nbf`, `iat`, and algorithm; do not accept `alg: none` |
| [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515) JWS | Normative | 1 | Token signing must use a library implementation; do not hand-roll JOSE serialization or signature verification |
| [RFC 7517](https://www.rfc-editor.org/info/rfc7517/) JWK | Normative | 1 | JWKS endpoint must publish only public keys; private key leakage via JWKS is a critical defect |
| [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725) JWT BCP | Normative | 1 | Apply all MUST items: reject algorithm confusion, validate `typ`, require `aud` validation, do not log raw token values |
| [RFC 9068](https://www.rfc-editor.org/info/rfc9068/) JWT access token profile | Normative if JWT access tokens issued | 1 | If JWT Access Tokens are selected for MVP: include `iss`, `sub`, `aud`, `exp`, `iat`, `jti`, `client_id`, and scopes per profile |

## GROUP 3: Client Authentication (Tier 1–2)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 7521](https://www.rfc-editor.org/info/rfc7521/) JWT Bearer assertion framework | Normative | 1 | Generic assertion framework; `private_key_jwt` builds on this; apply `jti` replay defense |
| [RFC 7523](https://www.rfc-editor.org/info/rfc7523/) private_key_jwt | Normative | 1 | Client authentication at Token Endpoint; `iss` and `sub` must equal `client_id`; `aud` must equal token endpoint issuer; `jti` required and must reject replays |
| [RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705) OAuth mTLS | Draft/watch | 3 | Candidate for device and client binding in high-security profile; do not imply mTLS support until scoped; separate from DPoP sender constraint |

## GROUP 4: Token Exchange And Delegation (Tier 4 — existing STS, additive for sts-authority)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 8693](https://www.rfc-editor.org/info/rfc8693/) token exchange | Normative | 4 | Core delegation surface: `sub` is the user; `act` is the actor; `act` must never be omitted on delegation paths; do not conflate with impersonation |
| [RFC 8707](https://www.rfc-editor.org/info/rfc8707/) resource indicators | Normative | 4 | `resource` parameter binds token audience to a specific protected resource; required for least-privilege token narrowing in the MCP/OBO profile |
| [RFC 9728](https://www.rfc-editor.org/info/rfc9728/) protected resource metadata | Normative | 4 | Resource Servers publish their own metadata; AS discovery and resource metadata must be consistent; do not expose resource metadata from the AS endpoint if the RS has its own |

## GROUP 5: Security And Sender Constraint (Tier 2–3)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 9700](https://www.rfc-editor.org/info/rfc9700/) OAuth Security BCP | Normative | 2 | Production security baseline; drop implicit grant, ROPC, fragment-encoded tokens in BFF paths; validate redirect URI exactly; require PKCE; apply mix-up defense; never log tokens |
| [RFC 9449](https://www.rfc-editor.org/info/rfc9449/) DPoP | Normative | 3 | Sender-constrained access tokens; `jkt` thumbprint binds token to client key; replay defense via `jti` and `nonce`; proof window must be enforced; do not skip `ath` on protected-resource requests |
| [RFC 9126](https://www.rfc-editor.org/info/rfc9126/) PAR | Normative | 3 | Pushed Authorization Requests for high-security profile; `request_uri` returned to client; AS must bind PAR state to subsequent authorization; PAR-only policy must reject front-channel parameters |
| [RFC 9101](https://www.rfc-editor.org/info/rfc9101/) JAR | Normative | 3 | Signed authorization requests; `request` object must be verified before processing; JAR and PAR combination required for FAPI-like profiles |

## GROUP 6: Lifecycle Endpoints (Tier 2 — production OAuth baseline)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [RFC 7009](https://www.rfc-editor.org/info/rfc7009/) token revocation | Normative | 2 | Revocation endpoint must accept access and refresh tokens; must not reveal whether the token was valid; revocation must propagate to introspection within the same process boundary |
| [RFC 7662](https://www.rfc-editor.org/info/rfc7662/) token introspection | Normative | 2 | Introspection endpoint must require authenticated callers; `active: false` for expired, revoked, or unknown tokens; must not return claims for inactive tokens; do not expose introspection publicly |
| [RFC 7591](https://www.rfc-editor.org/info/rfc7591/) DCR | Draft/watch | 2 | Dynamic Client Registration is a future capability; do not implement without an explicit security review of open registration policies; initial deployment uses static client registry |

## GROUP 7: OIDC (Tier 1 — additive above OAuth 2.1 baseline)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [OIDC Core 1.0 errata set 2](https://openid.net/specs/openid-connect-core-1_0.html) | Normative | 1 | ID Token issuance: include `iss`, `sub`, `aud`, `exp`, `iat`, `nonce` where required; `nonce` must be bound to auth session and validated; `sub` must be stable and pairwise-safe by policy |
| [OIDC Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) | Normative | 1 | OP metadata at `/.well-known/openid-configuration`; issuer exact-match validation required; metadata must not advertise endpoints that are not implemented |
| [OIDC Session Management 1.0](https://openid.net/specs/openid-connect-session-1_0.html) | Draft/watch | 2 | Front-channel and back-channel logout are future scope; do not imply session-management conformance for MVP; track OpenID Foundation spec status |

## GROUP 8: PQC (Tier 5 — explicit opt-in, test-backed, no interoperability claims until RFC)

| Standard | Class | Tier | Requirement Impact |
|---|---|---|---|
| [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final) ML-KEM-768 | Normative for algorithm naming | 5 | Use NIST parameter set names exactly; ML-KEM-768 for key establishment baseline; do not advertise hybrid KEM support without a tested verifier |
| [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final) ML-DSA-65 | Normative for algorithm naming | 5 | ML-DSA-65 for PQC token signing baseline; algorithm identifier in JOSE `alg` header must match IANA-registered name when draft promotes; until then use the provisional identifier from the active IETF draft |
| [draft-ietf-jose-fully-specified-algorithms](https://datatracker.ietf.org/doc/draft-ietf-jose-fully-specified-algorithms/) PQC JOSE | Draft/watch | 5 | Track IANA JOSE registration for ML-DSA; do not advertise PQC JOSE until algorithm identifier is registered and at least one independent verifier is tested |
| [draft-ietf-cose-dilithium](https://datatracker.ietf.org/doc/draft-ietf-cose-dilithium/) COSE ML-DSA | Draft/watch | 5 | COSE/CBOR path; reference only; QuAuthz is JWT/JOSE-primary; track for future cross-format interop |
| [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) | Guidance | 5 | Key management guidance: key storage, rotation policy, compromise handling, and cryptoperiod planning; apply before PQC key lifecycle is scoped |

## Regulatory And Control Framework Starter Map

This section informs product planning only. It is not legal advice and is not a
certification claim.

| Framework / regulation | Class | Relevance |
|---|---|---|
| NIST SP 800-207 Zero Trust | Guidance | Avoid implicit trust based on network location; per-request authorization aligned with delegated token model |
| NIST CSF 2.0 | Guidance | Security program mapping for enterprise buyers |
| SOC 2 | External assessment | SaaS/control-plane trust services controls if hosted |
| ISO/IEC 27001/27002 | External assessment/guidance | ISMS and security controls |
| FedRAMP / NIST SP 800-53 | External assessment | Federal cloud-service authorization path; requires FIPS 140-3 validated cryptographic modules |
| FAPI 2.0 Security Profile | External assessment | Financial-grade API high-security profile; candidate for Tier 3 but requires third-party conformance testing before claiming |
| OpenSSF Scorecard / SLSA | Guidance | OSS release posture; track under supply-chain issue |

## Open Issues To Maintain Traceability

- [#44](https://github.com/paul007ex/sts-delegate-rs-v2/issues/44) OAuth 2.1 draft-15 and RFC 9700 baseline.
- [#45](https://github.com/paul007ex/sts-delegate-rs-v2/issues/45) Strict ID Token validation and UserInfo binding.
- [#46](https://github.com/paul007ex/sts-delegate-rs-v2/issues/46) Browser-based client and front-channel threat profile.

Future requirements and issues should reference this matrix when they introduce
standards, compliance, or regulatory claims.
