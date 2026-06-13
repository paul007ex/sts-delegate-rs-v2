# UC-07: API Gateway Token Narrowing

Status: `planned`
Tier: 4 (Delegation / MCP Profile)

An API gateway sits between the user-facing client and a set of downstream
microservices. The gateway receives a broad access token from the client, calls
QuAuthz `POST /token` (RFC 8693 token exchange) to obtain a narrowed token scoped
to a single downstream API audience, and forwards the narrowed token. Downstream
services never see the broad token.

## Buyer Value

Implements least-privilege at the network boundary without requiring every
downstream service to understand the upstream authorization model. A leaked
narrowed token is bounded to one audience and one reduced scope set. The original
broad token is never exposed to downstream services.

## Actors

- **Subject** — the authenticated user; holds the broad access token.
- **API gateway** — intercepts outbound calls; exchanges broad token for narrowed token; holds `private_key_jwt` actor credential.
- **QuAuthz AS** — validates broad token, validates gateway actor credential, mints narrowed OBO token.
- **Downstream microservice** — receives only the narrowed token; verifies it against QuAuthz `GET /jwks`.

## Scenario

A user's broad token has `scope=read write admin` and `aud=https://api.example/`.
The gateway intercepts a call to the payments microservice. It exchanges the broad
token for a narrowed token with `scope=payments:read` and `aud=https://payments.example/`.
The payments service receives only that narrow token and rejects any token with a
broader scope or a different audience.

## Flow Steps

1. Client sends request with `Authorization: Bearer <broad-token>` to the API gateway.
2. Gateway verifies the broad token (audience `https://api.example/` must include gateway as valid audience).
3. Gateway calls QuAuthz `POST /token`:
   - `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
   - `subject_token=<broad-token>`, `subject_token_type=access_token`
   - `actor_token=<gateway-jwt>`, `actor_token_type=urn:ietf:params:oauth:token-type:jwt`
   - `audience=https://payments.example/`, `scope=payments:read`
4. QuAuthz validates: broad token signature and `exp`, actor `private_key_jwt`, requested scope is a subset.
5. QuAuthz mints narrowed token: `sub=<user>`, `act={sub: <gateway>}`, `aud=https://payments.example/`, `scope=payments:read`.
6. Gateway forwards narrowed token to payments microservice.
7. Payments microservice verifies narrowed token via QuAuthz `GET /jwks`; checks `aud` and `scope`.

## Key Requirements

- Requested `audience` MUST be explicitly present in QuAuthz allowed-audience list (RFC 8693 §2.1).
- Requested `scope` MUST be a strict subset of the subject token's granted scope; no escalation.
- `act` claim MUST be present; downstream services MUST reject tokens without `act` if configured to require delegation.
- Gateway actor key pinned per `client_id`; pooled JWKS without binding rejected.
- Broad token MUST NOT be forwarded to downstream services; gateway MUST drop it after exchange.
- RFC 8707 Resource Indicators (`resource` parameter) used to specify downstream audience when supported.

## Acceptance Evidence

- Narrowed token `aud` matches the requested downstream microservice; broad token `aud` rejected by that service.
- Scope escalation attempt returns `invalid_scope` + 400.
- Unregistered downstream audience returns `invalid_target` + 400.
- Payments service rejects the broad token directly (wrong `aud`).
- `act.sub` in narrowed token matches the gateway's registered `client_id`.

## Status

`planned` — target: Phase 4 / Delegation profile milestone.
Depends on RFC 8693 AS integration and RFC 8707 Resource Indicators (Phase 2+).
