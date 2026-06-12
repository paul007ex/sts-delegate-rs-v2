# Authorization Server v2 Roadmap

## Now: PRD And Architecture

Goal: make the product boundary clear before code starts.

- Decide product name and repo ownership.
- Finalize normative role glossary.
- Define crate/package boundaries.
- Copy or transfer AS 2.0 issues from `sts-delegate-rs`.
- Build the conformance ledger skeleton.
- Decide MVP token shapes and storage model.
- Track OAuth 2.1 draft-15, RFC 9700, OIDC Core, and browser/front-channel
  security issues before implementation branches start.

## Next: Local OIDC AS MVP

Goal: a real local OpenID Provider and OAuth Authorization Server.

- Client registry.
- User model and browser session.
- TOTP and recovery-code enrollment.
- Authorization Code + PKCE.
- Token Endpoint with authorization-code grant.
- JWKS.
- OIDC Discovery and OAuth AS metadata.
- ID Token issuance.
- UserInfo endpoint.
- Redacted audit events.

## Next+: Production OAuth Baseline

Goal: make the MVP safe enough for serious protected-resource testing.

- Revocation.
- Introspection.
- Refresh-token rotation and reuse detection.
- Resource Indicators.
- Issuer identification and mix-up defense.
- Endpoint rate limits and lockout.
- Admin bootstrap/RBAC.

## Later: Delegation, MCP, And High-Security Profile

Goal: differentiate from a generic Authorization Server.

- RFC 8693 token exchange integration.
- Resource Server / Protected Resource metadata.
- DPoP policy.
- PAR.
- RAR.
- Optional JAR/JARM decision.
- MCP/API demo that accepts scoped delegated token and rejects broad subject
  token.

## Later: PQC Profile

Goal: explicit, test-backed PQC signing and verifier story.

- ML-DSA key generation and inspection.
- PQC signing policy and downgrade rules.
- PQC token verification tests.
- KMS/HSM/provider custody path.
- Metadata advertisement only when verifiers and runtime enforce the selected
  algorithm.

## Later: Enterprise Surface

Goal: decide how much Okta/Keycloak-like surface belongs in this product.

- External IdP federation.
- SCIM.
- SAML.
- Tenant/realm administration.
- Admin API/UI.
- Log export and retention.
- HA storage and replay/state backend.
