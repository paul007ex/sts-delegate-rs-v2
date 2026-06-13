# UC-04: Browser SSO

Status: `planned`
Tier: 1 (OAuth 2.1 AS MVP)

A user opens a browser-based application, is redirected to QuAuthz for
authentication, logs in once, and is redirected back with an authorization code.
Subsequent applications in the same session skip the login prompt (SSO). The
session terminates on explicit logout or expiry.

## Buyer Value

Gives teams a self-hosted SSO layer they fully control. No vendor lock-in, no
per-seat IdP pricing for internal tooling, no telemetry leaking user activity to
a third-party IdP for dev/staging environments.

## Actors

- **User** — authenticates via browser.
- **Relying Party (RP)** — web application initiating the OIDC flow.
- **QuAuthz AS** — authenticates user, issues authorization code and tokens, manages session cookie.
- **Second RP** — a second application that benefits from the existing session.

## Scenario

User visits a web app. The app redirects to QuAuthz with `response_type=code` and
`scope=openid profile email`. User authenticates. QuAuthz issues a code; the app
exchanges it for an ID token and access token. The user then opens a second app;
QuAuthz detects the existing session and issues a code without re-prompting.

## Flow Steps

1. RP redirects browser to QuAuthz `GET /authorize`:
   - `response_type=code`, `client_id`, `redirect_uri`, `scope=openid profile email`
   - `state=<csrf>`, `code_challenge=<S256>`, `code_challenge_method=S256`
2. QuAuthz presents login UI; user authenticates.
3. QuAuthz creates session (encrypted cookie, configurable TTL).
4. QuAuthz redirects to `redirect_uri` with `code` and `state`.
5. RP server calls `POST /token` with `code`, `code_verifier`, `redirect_uri`.
6. QuAuthz returns `id_token`, `access_token`, `refresh_token` (if `offline_access` requested).
7. Second RP triggers `GET /authorize`; QuAuthz finds active session, skips login, issues new code immediately.
8. User triggers logout (`GET /logout`); QuAuthz clears session; subsequent RP flows re-prompt.

## Key Requirements

- PKCE (S256) MUST be enforced; `plain` rejected.
- `state` parameter MUST be reflected unchanged; mismatch returns `invalid_request`.
- Session cookie MUST be `HttpOnly`, `Secure`, `SameSite=Lax` at minimum.
- OIDC Core §3.1.3.3: ID token MUST include `iss`, `sub`, `aud`, `exp`, `iat`, `nonce` (if sent).
- `nonce` in authorization request MUST appear in ID token.
- Back-channel logout or session invalidation MUST terminate SSO session (RFC 7517 / OIDC Session §2.1).

## Acceptance Evidence

- Full browser redirect flow completes; ID token verifies against `GET /jwks`.
- Second RP receives code without re-prompting within session TTL.
- After `GET /logout`, third RP flow re-prompts for credentials.
- PKCE without `code_challenge` returns `invalid_request`.
- `state` mismatch in callback detection returns `invalid_request`.

## Status

`planned` — target: Phase 1 / OAuth 2.1 AS MVP milestone (browser login UI required).
