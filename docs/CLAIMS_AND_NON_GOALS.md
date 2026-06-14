Now I have a complete picture. Let me write the file.

# Claims and Non-Goals — QuAuthz (sts-authority)

**Product family:** quauthz-as planning lane
**Working product name:** sts-authority
**Status as of 2026-06-13:** Tier 0 STS shipped in `sts-delegate-rs`; Tier 1 Authorization Server planned, no code started

---

## Current Claims

These statements are true of the **shipped** Tier 0 product (`sts-delegate-rs`) that sts-authority builds on. They are not claims about the Authorization Server layer, which has not been implemented.

### RFC 8693 Token Exchange (Tier 0 — shipped, `sts-delegate-rs`)

- The Token Endpoint at `/token` accepts `grant_type=urn:ietf:params:oauth:grant-type:token-exchange` and issues a delegation token carrying `sub` (the Resource Owner) and `act` (the actor) per RFC 8693 §4.1.
- The issued token narrows audience and scope from the subject token per RFC 8693 §2.1.
- The exchange validates `subject_token_type` and `actor_token_type` per RFC 8693 §2.1 and rejects unsupported token types.
- Error responses follow RFC 8693 §2.2.2 error codes (`invalid_request`, `invalid_client`, `invalid_grant`, `unauthorized_client`, `unsupported_grant_type`, `invalid_scope`).
- 87% RFC conformance verified by two-pass audit (70/80 BCP14 requirements; see `docs/conformance/RFC_CONFORMANCE_AUDIT_FINAL.md`): RFC 8693 at 67% full/partial, RFC 9449 at 88%, RFC 7523 at 92%.

Evidence: `crates/sts-http/src/lib.rs` token endpoint; `docs/conformance/RFC_CONFORMANCE_AUDIT_FINAL.md`.

### DPoP Sender Constraint (Tier 0 — shipped, `sts-delegate-rs`)

- DPoP proof validation is implemented per RFC 9449 when the `DPoP` header is present.
- Proof `jti` replay detection is active within the process lifetime.
- 88% RFC 9449 conformance (22 fully implemented, 2 partial, 1 not implemented of 25 requirements audited).

Evidence: `crates/sts-http/src/lib.rs` DPoP proof validation path; `docs/conformance/RFC_CONFORMANCE_AUDIT_FINAL.md`.

### Token Signing (Tier 0 — shipped, `sts-delegate-rs`)

- ML-DSA-65 (NIST FIPS 204 Module-Lattice-Based Digital Signature Standard) is the default signing algorithm for issued tokens.
- The signing key is generated locally; private key material is never included in JWKS responses.
- JWKS is served at `/jwks` and publishes public key material only.

Evidence: `crates/sts-jose/` ML-DSA key generation and JWS signing; `/jwks` route.

### Authorization Server Metadata (Tier 0 — shipped, `sts-delegate-rs`)

- `/.well-known/oauth-authorization-server` is served and reflects the issuer, token endpoint, JWKS URI, and the token-exchange grant type per RFC 8414.
- Metadata advertises only endpoints and grant types that the runtime enforces.

Evidence: `crates/sts-http/src/lib.rs` well-known route.

### Token Lifecycle Endpoints (Tier 0 — shipped, `sts-delegate-rs`)

- `/introspect` per RFC 7662 is implemented.
- `/revoke` per RFC 7009 is implemented.

Evidence: `crates/sts-http/src/lib.rs` introspection and revocation routes.

### RFC 7523 JWT Client Authentication (Tier 0 — shipped, `sts-delegate-rs`)

- `private_key_jwt` client authentication is implemented per RFC 7523.
- 92% RFC 7523 conformance (23 fully implemented, 2 partial of 25 requirements audited).

Evidence: `docs/conformance/RFC_CONFORMANCE_AUDIT_FINAL.md` RFC 7523 section.

---

## Not Yet Claimed

These are planned Tier 1 and later capabilities. None of the following endpoints, grants, token types, or behaviors exist in the current codebase. They are planning-lane items in this repo only.

### Tier 1 — OAuth/OIDC Authorization Server MVP (planned, no code)

- `/authorize` endpoint (Authorization Endpoint per RFC 6749 §3.1, OIDC Core 1.0 §3.1.2).
- Authorization Code grant with PKCE S256 (RFC 7636 §4.2, §4.3, §4.5, §4.6).
- Browser session and user model with stable `sub`.
- Client registry with redirect URI exact-match enforcement (RFC 6749 §3.1.2).
- OIDC ID Token issuance and validation (OIDC Core 1.0 §2, §3.1.3.7).
- UserInfo endpoint returning only scope-authorized claims with `sub` bound to the ID Token (OIDC Core 1.0 §5.3).
- OIDC Discovery at `/.well-known/openid-configuration` (OIDC Discovery 1.0 §4).
- Refresh tokens with rotation and reuse detection.
- Issuer identification in Authorization responses per RFC 9207.
- OAuth 2.1 draft-aligned baseline (spec is `draft-ietf-oauth-v2-1-15` as of 2026-06-13; claim must say draft-aligned until it is a published RFC).

### Tier 2 — Production OAuth Baseline (planned, no code)

- Revocation with cross-session state (RFC 7009 server-wide, not just per-process).
- Introspection with durable grant state (RFC 7662 tied to store).
- Refresh-token family revocation cascade.
- Resource Indicators endpoint binding (RFC 8707).
- Dynamic Client Registration (RFC 7591 / RFC 7592), if enabled.

### Tier 3 — High-Security Profile (planned, no code)

- Pushed Authorization Requests (RFC 9126).
- Rich Authorization Requests (RFC 9396).
- mTLS sender constraint (RFC 8705), if selected.
- JAR / JARM, if decided.
- FAPI 2.0 profile, if decided.

### Tier 4 — Delegation and MCP Profile (planned, no code)

- Full RFC 8693 integration as an Authorization Server upstream (subject token minted by this AS, then exchanged at the STS).
- Protected Resource Metadata (RFC 9728).
- DPoP required-by-default policy at the AS layer (current: opt-in at STS).
- MCP/OBO demo chain with narrow delegated token validated at a Resource Server that rejects broad subject tokens.

### Tier 5 — PQC Profile (planned, no code)

- ML-DSA signing for ID Tokens and client assertions (not only STS access tokens).
- PQC signing policy governance and downgrade-refused behavior.
- Algorithm advertisement in discovery only when runtime enforcement is verified.
- KMS / HSM key custody path.

---

## Non-Goals

### MVP Non-Goals

- **Password database by default.** The first authentication factor is TOTP plus recovery codes. A password table is not in the MVP schema.
- **Full admin web console.** Bootstrap and client/user operations are CLI-driven for MVP.
- **FAPI certification claims.** The FAPI 2.0 profile is deferred to Tier 3 and requires a certification body. Do not claim FAPI alignment on the basis of implementing individual RFCs.
- **PQC interoperability claims beyond tested algorithms.** ML-DSA-65 signing is implemented at the STS layer. No cross-vendor PQC interoperability has been tested. Do not claim quantum-safe status for the AS layer before the Tier 5 evidence exists.
- **OAuth 2.1 published RFC conformance.** OAuth 2.1 is a draft. Any claim must say `draft-aligned` with the draft version (`draft-ietf-oauth-v2-1-15` or later) until the IETF publishes a final RFC.

### Permanent Non-Goals

- **Full Okta or Keycloak replacement.** sts-authority is a focused Authorization Server for delegation-aware, MCP/OBO, and DPoP use cases. Enterprise identity lifecycle management, high-availability tenant operations, and support SLAs are out of scope unless the product decision changes and replaces this section.
- **SAML.** No SAML SP, IdP, or metadata exchange. The protocol surface is OAuth 2.1 / OIDC.
- **SCIM provisioning.** No SCIM 2.0 API, directory sync, or user-import pipeline.
- **Social-login provider catalog.** No built-in connectors for Google, GitHub, Apple, Facebook, or other external IdPs. External IdP federation at the AS layer is a later-tier decision, not MVP.
- **Implementing new cryptographic primitives.** All JOSE, JWT, JWK, and PQC operations use audited crates (e.g., `ml-dsa`, `ring`, `rustls`). No hand-rolled signature, hash, or KEM code.
- **Forklift replacement of the current STS.** `sts-delegate-rs` remains the RFC 8693 token-exchange product. sts-authority is the Authorization Server that sits upstream. The boundary rule is: AS owns interactive authentication, sessions, grants, clients, consent, codes, refresh tokens, UserInfo, and identity lifecycle; the STS owns delegation and token exchange. Do not merge these surfaces.
- **Mixing VPN data-plane or PQ network tunnel logic from `breachsafe-pqvpn-rs` into the AS or STS.** Those are separate products with separate protocol surfaces.

---

## Required Standard for Any Compliance Claim

Any statement that this product conforms to, implements, or is aligned with an OAuth, OIDC, or related RFC requirement must satisfy all four of the following before it appears in documentation, release notes, README, or discovery metadata:

1. **RFC identity.** Name the exact RFC number and section (e.g., RFC 9449 §4.3, OIDC Core 1.0 §3.1.3.7). For draft specs, include the draft version (e.g., `draft-ietf-oauth-v2-1-15 §4.1`). Do not write "OAuth 2.1 compliant" for a draft.

2. **Tier.** State which conformance tier the claim belongs to (Tier 0 through 5 per the table in `docs/v2/prd.md`). A Tier 1 claim is not a Tier 0 claim, and a planned item is not a shipped item.

3. **Evidence.** Provide at least one of:
   - A passing test name with the assertion that exercises the requirement (positive case and at least one negative/adversarial case for MUST requirements).
   - A curl command that reproduces the behavior against a running instance, with the expected response shape shown.
   - A conformance ledger row (`docs/v2/conformance/`) with `status: implemented` and the code path named.

4. **Shipped vs. planned.** Label the claim as **shipped** (code merged, CI green, evidence present) or **planned** (issue filed, no code). Do not promote a planned item to shipped in docs, metadata, or release text before the evidence row is complete. Discovery metadata MUST NOT advertise endpoints, grant types, algorithms, response types, or token types unless the runtime enforces them.

Any claim that cannot satisfy all four criteria must be removed or reclassified as planned until evidence exists.
