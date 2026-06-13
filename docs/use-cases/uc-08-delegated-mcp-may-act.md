# UC-08: Delegated MCP may_act

Status: `future`
Tier: 4 (Delegation / MCP Profile)

The resource owner pre-authorizes a specific actor (an AI agent or MCP gateway) to
exchange tokens on their behalf by including a `may_act` claim in their access
token. The actor presents that token at QuAuthz without requiring the user to be
present at exchange time. QuAuthz validates the `may_act` binding and issues a
scoped OBO token. The user can revoke `may_act` authorization at any time.

## Buyer Value

Enables asynchronous and background AI agent workflows: the user authorizes an
agent once; the agent can act on the user's behalf for a defined window without
re-prompting the user at every invocation. The `may_act` claim makes the
pre-authorization explicit, auditable, and revocable — distinguishing it from
ambient credential forwarding.

## Actors

- **Subject** — the user; issues a token with `may_act` authorizing a specific actor.
- **Actor** — the AI agent or MCP gateway; listed in `may_act.sub` in the subject token.
- **QuAuthz AS** — validates `may_act` binding, mints OBO token with `act` claim, enforces revocation.
- **Resource server** — downstream MCP tool server; requires `act` claim; may also require `may_act` evidence.

## Scenario

User authorizes an AI coding agent to call the code-review MCP server on their
behalf for 24 hours. QuAuthz issues a broad token with `may_act={sub: <agent-id>}`.
The agent, acting without the user present, exchanges that token at QuAuthz for a
scoped OBO token with `scope=codereview:read` and `act={sub: <agent-id>}`. The
code-review server accepts the OBO token and logs both `sub` and `act`. Six hours
later the user revokes the `may_act` grant; subsequent exchange attempts fail.

## Flow Steps

1. User authenticates; requests broad token including `may_act` scope.
2. QuAuthz includes `may_act={sub: <agent-id>}` in the broad access token (RFC 8693 §4.1).
3. Agent calls QuAuthz `POST /token` at any later time (no user session required):
   - `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
   - `subject_token=<broad-token-with-may-act>`, `subject_token_type=access_token`
   - `actor_token=<agent-jwt>`, `actor_token_type=urn:ietf:params:oauth:token-type:jwt`
   - `audience=https://codereview.example/`, `scope=codereview:read`
4. QuAuthz validates: subject token signature, `may_act.sub == actor's client_id`, actor `private_key_jwt`, scope subset.
5. QuAuthz issues OBO token: `sub=<user>`, `act={sub: <agent>}`, `aud=<codereview>`, `scope=codereview:read`.
6. Agent calls code-review MCP server with OBO token.
7. User revokes `may_act` grant (`POST /revoke` or management API).
8. Agent's next exchange attempt returns `invalid_grant`; code-review server's next introspection returns `active=false`.

## Key Requirements

- `may_act` claim MUST match the presenting actor's `client_id`; mismatched actor returns `invalid_client` + 401.
- `may_act` grant is stored and independently revocable without revoking the subject's session.
- OBO token issued under `may_act` MUST carry `act` claim (RFC 8693 §4.1); `may_act` is not impersonation.
- Subject token `exp` MUST be honored; `may_act` does not extend the subject token lifetime.
- Revocation via `POST /revoke` MUST propagate to introspection within one TTL cycle.
- Nested delegation (`act` chain) is permitted but depth MUST be bounded (default: 3 levels).

## Out of Scope for This Use Case

- Autonomous consent UI for `may_act` grants (management API only for now).
- `may_act` across trust domains (cross-AS delegation).
- `may_act` with refresh tokens (access-token exchange only).

## Acceptance Evidence

- Agent exchanges `may_act`-bearing token successfully; OBO token has `act.sub == agent-id`.
- Agent with `client_id` not matching `may_act.sub` returns `invalid_client` + 401.
- After user revokes `may_act`, agent exchange returns `invalid_grant` + 400.
- OBO token introspected after revocation returns `active=false`.
- Depth-3 nested `act` chain accepted; depth-4 chain returns `invalid_request`.

## Status

`future` — highest-complexity delegation use case; depends on Phase 4 RFC 8693 AS
integration, revocation store (Phase 2), and may_act grant management API (not yet
scoped). Estimated after Phase 4 milestone ships.
