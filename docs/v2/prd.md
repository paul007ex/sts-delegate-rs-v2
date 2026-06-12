# Authorization Server v2 PRD

## Executive Summary

We are planning `sts-authority`, a local-first OAuth/OIDC Authorization Server
for developers and platform/security teams that need browser-based user
authentication, OIDC identity tokens, OAuth access tokens, resource-specific
authorization, STS delegation, DPoP, and PQC-ready signing without immediately
operating a full Okta/Keycloak-scale IAM platform.

The current Rust product, `sts-delegate-rs`, remains an RFC 8693 token-exchange
STS. This v2 product is a separate Authorization Server/OpenID Provider lane.

## Problem Statement

Teams building MCP/OBO systems need to understand and prove the full
OAuth/OIDC chain:

1. The Resource Owner / End-User authenticates through a User Agent.
2. The Client / Relying Party receives an authorization code.
3. The Client redeems that code at the Token Endpoint.
4. The OpenID Provider returns an ID Token for authentication to the Client and
   an Access Token for protected-resource access.
5. Resource Servers validate Access Tokens by issuer, audience, expiry, scopes,
   sender constraint, and policy.
6. STS token exchange can later narrow that Access Token into a delegated token
   for an MCP/API hop.

The current STS can handle delegation, but it cannot yet be the upstream
Authorization Server/OpenID Provider.

## Target Users

- Local developer proving OAuth/OIDC and STS flows end to end.
- Platform/security engineer building MCP/OBO gateway authorization.
- Security lab evaluating DPoP and PQC-signed tokens.
- Product team embedding an Authorization Server into a security product.

## Product Promise

`sts-authority` should be a focused, understandable Authorization Server:

- local-first;
- passwordless-first;
- OIDC Core and OAuth 2.1-aligned;
- resource/audience-aware;
- DPoP-capable;
- native STS delegation-aware;
- PQC-ready where explicitly enabled and proven.

It should not claim to be a full Okta or Keycloak replacement until enterprise
features, conformance proof, operations, identity lifecycle, and admin UX exist.

## MVP Scope

MVP must include:

- durable Authorization Server state;
- user model with stable `sub`;
- client registry;
- browser session subsystem;
- TOTP plus recovery codes for first local proof, with WebAuthn/passkeys planned;
- Authorization Code + PKCE;
- Token Endpoint support for `authorization_code`;
- OIDC Discovery;
- Authorization Server metadata;
- JWKS;
- signed ID Tokens;
- JWT Access Tokens if selected for MVP;
- UserInfo;
- consent/grant model;
- redacted audit events;
- conformance ledger and executable proof harness.

## Non-Goals For MVP

- Full SaaS IAM replacement.
- Password database by default.
- SCIM provisioning.
- SAML app catalog.
- Social-login provider catalog.
- Full admin web console.
- FAPI certification claims.
- PQC interoperability claims beyond tested algorithms and verifiers.

## Compliance Tiers

| Tier | Claim | Core standards |
| --- | --- | --- |
| 0 | Existing STS | RFC 8693, RFC 8414 as applicable, JOSE/JWT, RFC 9449 where enabled |
| 1 | OAuth/OIDC AS MVP | OAuth 2.1 draft baseline, RFC 7636, RFC 8414, RFC 9207, OIDC Core, OIDC Discovery |
| 2 | Production OAuth baseline | RFC 9700, RFC 7009, RFC 7662, RFC 8707, RFC 7591/7592 if DCR is enabled |
| 3 | High-security profile | RFC 9449, RFC 9126, RFC 9101, JARM, RFC 8705 if mTLS is selected, FAPI 2.0 later |
| 4 | Delegation/MCP profile | RFC 8693, RFC 8707, RFC 9068, RFC 9449, RFC 9728 |
| 5 | PQC profile | JOSE/JWK/JWT, RFC 8725, current PQC JOSE/COSE/IANA/NIST references after selection |

OAuth 2.1 is still a draft as of June 12, 2026, so claims must say
`OAuth 2.1 draft-aligned` until it becomes an RFC.

Tracking issues:

- OAuth 2.1 draft-15 and RFC 9700 baseline:
  [#44](https://github.com/paul007ex/sts-delegate-rs-v2/issues/44).
- Strict ID Token validation and UserInfo binding:
  [#45](https://github.com/paul007ex/sts-delegate-rs-v2/issues/45).
- Browser-based client and front-channel threat profile:
  [#46](https://github.com/paul007ex/sts-delegate-rs-v2/issues/46).

## Success Metrics

- Local quickstart completes an OIDC Authorization Code + PKCE flow in under 10
  minutes.
- ID Token and Access Token are decoded and validated without raw token logging.
- UserInfo returns only authorized claims and requires a valid Access Token.
- Negative tests cover redirect URI mismatch, missing PKCE, wrong verifier, code
  replay, issuer mix-up, wrong audience, expired token, revoked token, and broad
  token rejection at Resource Servers.
- STS delegation demo preserves `sub`, represents actor/delegation with `act`
  where applicable, narrows `aud`, and narrows scopes.

## Open Decisions

- Product name: `sts-authority` vs `sts-delegate-rs-v2`.
- MVP authentication method: TOTP-first, passkey-first, or staged.
- JWT Access Token profile vs opaque tokens plus introspection for MVP.
- SQLite/file store first, or Postgres abstraction from day one.
- DPoP required by default or per-client/resource policy.
- PQC signing surfaces for MVP: Access Token, ID Token, client assertion, JARM,
  or later.
