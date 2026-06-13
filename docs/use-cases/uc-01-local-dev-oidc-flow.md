# UC-01: Local Dev OIDC Flow

Status: `prototype`
Tier: 1 (OAuth 2.1 AS MVP)

A developer runs QuAuthz locally, points a test application at it, and completes a
full Authorization Code + PKCE + OIDC login without any cloud dependency.

## Buyer Value

Eliminates the "I need an Okta dev tenant just to test auth" tax. Any developer
can spin up a conformant AS on localhost in one command, iterate on their
integration, and trust that the tokens are structurally correct before touching a
real IdP.

## Actors

- **Developer** — runs QuAuthz locally, operates the test client.
- **QuAuthz AS** — issues authorization code, access token, and ID token.
- **Test application** — any OAuth 2.1 / OIDC Relying Party (browser or CLI).

## Scenario

Developer starts QuAuthz, registers a client, opens a browser, authenticates with
a local credential, receives an ID token and access token, and calls a protected
local endpoint. All steps complete without network egress.

## Flow Steps

1. Developer runs `quauthz serve --dev`.
2. Developer registers client via `quauthz client register --redirect-uri http://localhost:8080/callback`.
3. Browser opens `GET /authorize` with `response_type=code`, `code_challenge`, `scope=openid profile`.
4. QuAuthz presents a local login form; developer submits credentials.
5. QuAuthz issues authorization code and redirects to callback URI.
6. Client exchanges code at `POST /token` with `code_verifier`.
7. QuAuthz returns `access_token`, `id_token`, `token_type=Bearer`.
8. Client calls `GET /userinfo` with the access token; QuAuthz returns subject claims.

## Key Requirements

- RFC 7636 PKCE enforced (plain not accepted; S256 required).
- RFC 8414 metadata at `/.well-known/oauth-authorization-server` correct on first request.
- ID token signed with ML-DSA-65 by default; RS256 explicit opt-in only.
- No network egress required during the dev flow.
- `GET /jwks` returns the public key; the test client can verify the ID token offline.

## Acceptance Evidence

- Full PKCE exchange completes in a single `curl` script with no stubs.
- ID token signature verifies against `GET /jwks`.
- Rotating the signing key produces a new `kid` without restarting the server.
- `/.well-known/oauth-authorization-server` matches RFC 8414 §2 required fields.

## Status

`prototype` — target: Phase 1 / 8 weeks from Tier 1 start.
