# UC-03: CLI / M2M Service Authentication

Status: `planned`
Tier: 2 (Production Baseline)

A machine-to-machine client (CI pipeline, daemon, CLI tool) authenticates to
QuAuthz using `client_credentials` + `private_key_jwt` and receives a short-lived
access token scoped to a single downstream API audience. No browser, no user
session, no interactive prompt.

## Buyer Value

Replaces shared secrets and long-lived API keys with short-lived, auditable tokens
bound to a specific client identity. Every M2M call is traceable to a registered
client key; rotating the key does not require updating secrets in CI.

## Actors

- **Client** — CI pipeline, daemon, or CLI; holds an asymmetric private key.
- **QuAuthz AS** — authenticates the client via `private_key_jwt`, issues scoped token.
- **Resource server** — downstream API; verifies the access token.

## Scenario

A CI pipeline needs to push artifacts to an internal API. The pipeline holds a
private key registered with QuAuthz. It constructs a `private_key_jwt` assertion,
calls `POST /token`, and receives an access token scoped to the artifact API. The
access token expires in 5 minutes; the pipeline refreshes as needed.

## Flow Steps

1. Client constructs a JWT assertion:
   - `iss=client_id`, `sub=client_id`, `aud=https://quauthz.example/token`
   - `exp=now+60s`, `jti=<uuid>`
2. Client calls `POST /token`:
   - `grant_type=client_credentials`
   - `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`
   - `client_assertion=<JWT>`
   - `scope=artifact:push`, `audience=https://artifact-api.example/`
3. QuAuthz validates assertion: signature against registered JWKS, `iss=sub=client_id`, `aud`, `exp`, `jti` uniqueness.
4. QuAuthz issues access token: `sub=client_id`, `scope=artifact:push`, `aud=<artifact-api>`, `exp=now+300s`.
5. Client presents token to resource server via `Authorization: Bearer`.

## Key Requirements

- RFC 7523 §3: `iss`, `sub`, `aud`, `exp`, `jti` all required in assertion.
- `jti` replay detection enforced within assertion `exp` window.
- Client JWKS pinned per `client_id`; pooled JWKS without client binding rejected.
- `client_credentials` grant MUST NOT issue an ID token (no `openid` scope).
- Access token `exp` MUST be short (default ≤ 300 s; configurable per client).

## Acceptance Evidence

- Valid `private_key_jwt` assertion returns 200 with scoped access token.
- Replayed `jti` returns `invalid_grant` + 400.
- Assertion signed by unregistered key returns `invalid_client` + 401.
- Token verifies against QuAuthz `GET /jwks`; resource server accepts it.

## Status

`planned` — target: Phase 2 / Production Baseline milestone.
