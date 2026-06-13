# UC-02: MCP OBO Gateway Delegation

Status: `planned`
Tier: 4 (Delegation / MCP Profile)

An AI agent (MCP client) holds a broad user token. An MCP gateway calls QuAuthz
`POST /token` with RFC 8693 token exchange to obtain a scoped OBO token carrying
both `sub` (the user) and `act` (the gateway). Downstream MCP tool servers verify
the scoped token and the `act` claim before executing any tool.

## Buyer Value

Eliminates the confused-deputy pattern where a gateway forwards the user's
god-token to every MCP server. Every downstream call is attributable to both the
user and the acting gateway, auditable without needing a central call log.

## Actors

- **Subject** — the authenticated user (`sub` claim).
- **Actor** — the MCP gateway acting on the user's behalf (`act.sub` claim).
- **QuAuthz AS** — performs RFC 8693 token exchange, mints scoped OBO token.
- **Resource server** — downstream MCP tool server; verifies token and `act`.

## Scenario

User logs in and obtains a broad access token from QuAuthz. An MCP gateway, holding
its own `private_key_jwt` actor credential, exchanges the user token for a token
scoped to a single MCP tool server audience and reduced scope set. The tool server
verifies the scoped token and logs both `sub` and `act`.

## Flow Steps

1. Subject authenticates; QuAuthz issues broad access token (scope: `openid mcp`).
2. MCP gateway calls `POST /token`:
   - `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
   - `subject_token=<broad token>`, `subject_token_type=access_token`
   - `actor_token=<gateway JWT>`, `actor_token_type=urn:ietf:params:oauth:token-type:jwt`
   - `audience=https://tool-server.example/`, `scope=tool:read`
3. QuAuthz validates subject token (signature, `exp`, `iss`, `aud`).
4. QuAuthz validates actor token (`private_key_jwt`; actor JWKS pinned).
5. QuAuthz mints OBO token: `sub=<user>`, `act={sub: <gateway>}`, `aud=<tool-server>`, `scope=tool:read`.
6. MCP gateway presents OBO token to tool server.
7. Tool server verifies signature via QuAuthz `GET /jwks` and checks `act` claim.

## Key Requirements

- RFC 8693 §4.1: `act` claim MUST be present; nested `act` supported for chained delegation.
- Downscoped `scope` MUST be a strict subset of the subject token's granted scope.
- Actor key pinned per `actor_id`; pooled JWKS without binding is not accepted.
- Token type in response is `urn:ietf:params:oauth:token-type:access_token`.
- `iss` in OBO token is QuAuthz, not the upstream IdP.

## Acceptance Evidence

- Exchange returns 200 with `act.sub` matching the gateway's `client_id`.
- Exchange with an unpinned actor key returns `invalid_client` + 401.
- Scope escalation attempt returns `invalid_scope`.
- Tool server can verify the OBO token using only QuAuthz `GET /jwks`.

## Status

`planned` — target: Phase 4 / Delegation profile milestone.
