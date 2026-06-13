# UC-06: Enterprise AS Replacement

Status: `future`
Tier: 2–3 (Production Baseline + High-Security Profile)

An enterprise replaces a vendor IdP (Okta, Azure AD, Auth0) with QuAuthz as
the authoritative OAuth 2.1 / OIDC Authorization Server for internal applications.
QuAuthz handles authorization code, client credentials, token introspection,
revocation, and dynamic client registration for a fleet of internal relying parties.

## Buyer Value

Eliminates per-seat IdP licensing for internal tooling, removes telemetry leakage
to vendor platforms, and gives the security team full control over token policy,
key management, audit logs, and protocol behavior without vendor lock-in.

## Actors

- **Enterprise security team** — operates and configures QuAuthz.
- **Internal relying parties** — web apps, CLIs, daemons; previously registered with vendor IdP.
- **End users** — employees authenticating via browser SSO.
- **M2M clients** — CI pipelines, services; previously using vendor API keys.
- **Resource servers** — internal APIs verifying access tokens.

## Scenario

Enterprise migrates 12 internal applications off Okta. QuAuthz is deployed behind
an internal load balancer. Applications are re-registered via dynamic client
registration (RFC 7591) or static config. Users authenticate via browser SSO (UC-04).
M2M clients switch from Okta API keys to `private_key_jwt` (UC-03). Resource servers
update their JWKS endpoint to point at QuAuthz `GET /jwks`.

## Flow Steps

1. Security team deploys QuAuthz with persistent key store and external session store.
2. Applications registered via `POST /register` (RFC 7591) or static client config file.
3. DNS cutover: `https://auth.internal.example/` points to QuAuthz.
4. `GET /.well-known/oauth-authorization-server` returns correct issuer and endpoint URLs.
5. Browser SSO flows operate as in UC-04; existing session cookies are invalidated once (one-time re-login).
6. M2M clients switch to `private_key_jwt` (UC-03); old API keys revoked.
7. Token introspection (`POST /introspect`) and revocation (`POST /revoke`) operate for all issued tokens.
8. Security team runs token revocation on compromise; affected sessions require re-authentication.

## Key Requirements

- RFC 8414: discovery metadata accurate and complete before cutover.
- RFC 7591: dynamic client registration with `redirect_uris`, `grant_types`, `token_endpoint_auth_method`.
- RFC 7009: revocation endpoint handles both access and refresh tokens.
- RFC 7662: introspection returns `active`, `sub`, `scope`, `exp`, `client_id`.
- Multi-worker deployment MUST share replay store (no per-process JTI store).
- Key rotation MUST be zero-downtime (overlap window; see UC-05).

## Out of Scope for This Use Case

- SCIM provisioning or directory sync.
- SAML IdP bridging or federation.
- Social login (Google, GitHub OAuth apps).
- Full admin console UI (command-line and API management only).
- FAPI 2.0 certification (planned separately in Tier 3).

## Acceptance Evidence

- All 12 applications authenticate successfully post-cutover with no code changes beyond JWKS endpoint URL.
- Token revocation terminates active sessions within one `exp` cycle.
- `POST /introspect` returns correct `active` status for both valid and revoked tokens.
- Dynamic client registration returns `client_id` and `client_secret` (or JWKS URI) that work immediately.

## Status

`future` — depends on Phase 2 (Production Baseline: RFC 7009, RFC 7662, RFC 7591)
and Phase 3 (High-Security Profile: FAPI 2.0 alignment) completing first.
