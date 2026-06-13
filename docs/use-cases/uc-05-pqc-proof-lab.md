# UC-05: PQC Proof Lab

Status: `prototype`
Tier: 5 (PQC Profile)

QuAuthz issues ID tokens and access tokens signed with ML-DSA-65 (FIPS 204).
A verifier — whether a resource server, test harness, or conformance tool — fetches
the QuAuthz JWKS and verifies the token signature using a post-quantum algorithm.
No classical-only fallback is available unless RS256 is explicitly re-enabled.

## Buyer Value

Demonstrates that the full OAuth / OIDC token flow can operate with PQC-only
signing today, without waiting for IdP vendors. Gives security teams a reproducible
lab to validate their verifier stack against real ML-DSA-65 tokens before the
migration window closes.

## Actors

- **Developer / security researcher** — runs QuAuthz in PQC mode, issues tokens, validates verifier behavior.
- **QuAuthz AS** — signs tokens with ML-DSA-65; publishes OKP-family JWKS entry.
- **Verifier** — any JOSE/JWT library or custom verifier that supports ML-DSA-65.

## Scenario

Researcher starts QuAuthz with `SIGNING_ALG=ML-DSA-65`. Completes a standard
Authorization Code + PKCE flow. Receives an ID token. Fetches JWKS. Verifies ID
token signature using the ML-DSA-65 public key. Confirms `alg` header, `kid`, and
signature bytes are correct. Attempts verification with an RS256 verifier; it fails
cleanly.

## Flow Steps

1. Researcher starts QuAuthz: `SIGNING_ALG=ML-DSA-65 quauthz serve --dev`.
2. `GET /.well-known/oauth-authorization-server` returns `id_token_signing_alg_values_supported: ["ML-DSA-65"]`.
3. PKCE Authorization Code flow completes; ID token returned with `alg=ML-DSA-65` in JOSE header.
4. Researcher fetches `GET /jwks`; response contains ML-DSA-65 public key entry with matching `kid`.
5. Researcher verifies ID token signature against JWKS using a PQC-capable JOSE library.
6. Researcher attempts RS256 verification: library returns signature failure (not an error from QuAuthz).
7. Researcher rotates signing key: new `kid` appears in JWKS; previous `kid` retained for overlap window.

## Key Requirements

- ML-DSA-65 (FIPS 204 / `sts-dpop` crate) is the default and the only enabled algorithm unless RS256 is explicitly opted in via config.
- `alg` JOSE header MUST match the `kid` in JWKS; no algorithm confusion path exists.
- JWKS MUST NOT expose the private key in any form.
- Key rotation MUST produce a new `kid` without dropping the previous key immediately (overlap window configurable).
- RS256 opt-in MUST be documented as a downgrade; the server MUST log a warning when RS256 is active.

## Acceptance Evidence

- ML-DSA-65 ID token signature verified by PQC JOSE verifier using `GET /jwks` (no hardcoded key).
- RS256 verifier rejects ML-DSA-65 token cleanly (algorithm mismatch, not panic or silent pass).
- Key rotation produces a new `kid`; old `kid` still present and valid during overlap window.
- `GET /.well-known/oauth-authorization-server` lists only `ML-DSA-65` by default.

## Status

`prototype` — this is one of the two early validation gates named in the PRD.
Target: Phase 5 / PQC Profile milestone, but an early proof is expected alongside Phase 1.
