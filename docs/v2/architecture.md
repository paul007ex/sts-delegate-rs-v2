# Architecture

## Positioning

QuAuthz (`sts-authority`) is a local-first, standards-aligned OAuth 2.1 / OIDC
Authorization Server and OpenID Provider for teams that need browser-based user
authentication, signed ID Tokens, scoped Access Tokens, RFC 8693 STS delegation,
DPoP sender constraint, and PQC-ready signing without operating a full
Okta/Keycloak-scale IAM platform. It sits between the Resource Owner's browser
session and the Resource Servers that enforce access — it is not a VPN, not an
API gateway, and not a full IAM suite. `sts-delegate-rs` (the current STS) remains
the token-exchange product; QuAuthz is the upstream Authorization Server/OpenID
Provider that issues the tokens the STS later exchanges.

The product boundary is strict: QuAuthz owns interactive authentication, sessions,
grants, clients, consent, codes, refresh tokens, UserInfo, and identity lifecycle.
`sts-exchange` (shared with `sts-delegate-rs`) owns delegation and RFC 8693 token
exchange. The two products share JOSE/JWK/JWT, DPoP, replay, and trust-anchor
primitives through a common crate workspace.

## Left-to-Right Flow

```text
Resource Owner         User Agent          Client / RP           QuAuthz AS              Resource Server
(human user)           (browser)           (MCP client,          (sts-authority)         (MCP / API)
                                           web app, CLI)
      |                    |                    |                       |                      |
      | initiates login    |                    |                       |                      |
      |------------------->|                    |                       |                      |
      |                    | GET /authorize     |                       |                      |
      |                    | ?client_id         |                       |                      |
      |                    | &redirect_uri      |                       |                      |
      |                    | &scope             |                       |                      |
      |                    | &state&nonce       |                       |                      |
      |                    | &code_challenge    |                       |                      |
      |                    | &code_challenge_   |                       |                      |
      |                    | method=S256        |                       |                      |
      |                    |------------------------------------------->|                      |
      |                    | login + consent UI |                       |                      |
      |                    |<-------------------------------------------|                      |
      |                    | redirect_uri       |                       |                      |
      |                    | ?code&state&iss    |                       |                      |
      |                    |------------------->|                       |                      |
      |                    |                    | POST /token           |                      |
      |                    |                    | code+verifier         |                      |
      |                    |                    | +client auth          |                      |
      |                    |                    |----------------------->|                      |
      |                    |                    | access_token          |                      |
      |                    |                    | id_token              |                      |
      |                    |                    | [refresh_token]       |                      |
      |                    |                    |<-----------------------|                      |
      |                    |                    | Bearer / DPoP-bound access_token              |
      |                    |                    |------------------------------------------------------>|
      |                    |                    |                       | validate iss/aud/exp  |
      |                    |                    |                       | scope/cnf via JWKS    |
      |                    |                    |                       |<-------------------   |
      |                    |                    | [optional] POST /token exchange (RFC 8693)    |
      |                    |                    |----------------------->|                      |
      |                    |                    | delegated token        |                      |
      |                    |                    | (narrowed aud/scope,   |                      |
      |                    |                    |  act claim)            |                      |
      |                    |                    |<-----------------------|                      |
      |                    |                    | delegated token ------->  MCP / API           |
```

## Package / Crate Map

| Crate | Responsibility |
|---|---|
| `sts-as-core` | Authorization Server domain rules, grant state machines, session lifecycle, claims assembly, and policy decisions. No HTTP, no IO, no env reads. |
| `sts-as-store` | Durable state for users, registered clients, browser sessions, authorization codes, consent/grant records, and refresh-token families. Defines the store trait; SQLite implementation for MVP. |
| `sts-as-http` | `/authorize`, `/token` (AS grants), `/.well-known/openid-configuration`, `/.well-known/oauth-authorization-server`, `/userinfo`, `/jwks`, `/logout`, and OAuth error responses. Owns the Axum router only. |
| `sts-as-config` | AS configuration schema, environment resolution, and startup validation. Validates that advertised algorithms, grant types, and endpoints match enforced runtime behavior before the server accepts connections. |
| `sts-as-cli` | Bootstrap, client/user/admin registration operations, local dev quickstart, and key generation. Operator-facing binary; no runtime HTTP handling. |
| `sts-exchange` | Shared RFC 8693 token-exchange application service. Extracted and reused from `sts-delegate-rs`. Accepts subject and actor tokens, verifies trust anchors, narrows audience and scope, assembles `act` claim, and returns a delegated Access Token. |
| `sts-jose` | JOSE/JWK/JWKS/JWT signing and verification. Key generation, kid selection, algorithm policy, and fail-closed behavior for `alg=none`, unknown kid, and unsupported algorithm. Shared by `sts-as-core`, `sts-exchange`, `sts-dpop`, and `sts-verify`. |
| `sts-dpop` | DPoP proof parsing and validation per RFC 9449. Sender-constraint binding (`cnf.jkt`). Shared by the AS token endpoint and downstream Resource Server verification. |
| `sts-replay` | Replay-state policy and pluggable backend interface. MVP implementation: single-process in-memory store. Shared by `sts-dpop` (proof replay) and `sts-exchange` (jti replay). |
| `sts-verify` | Issuer/trust-anchor validation, OIDC discovery fetch, and JWKS retrieval with rotation-retry. Shared by `sts-as-core` (upstream IdP federation, future) and `sts-exchange` (subject token validation). |

## Layer Ownership

| Layer | Owns | Must not own |
|---|---|---|
| Domain (`sts-as-core`) | Grant rules, session state transitions, claims assembly, policy decisions | HTTP rendering, IO, env reads, key custody |
| Store (`sts-as-store`) | Durable state, query interface, migration | OAuth policy decisions, HTTP, crypto |
| Transport (`sts-as-http`) | Axum router, request parsing, response shaping, OAuth error codes | Key custody, policy decisions, store queries outside service calls |
| Config (`sts-as-config`) | Env parsing, schema validation, startup checks | Runtime policy decisions, store access |
| CLI (`sts-as-cli`) | Bootstrap operations, key generation, client/user admin | Runtime request handling, server lifecycle beyond start/stop |
| JOSE (`sts-jose`) | Key lifecycle, signing, verification, algorithm policy | OAuth grant logic, store access, HTTP |
| DPoP (`sts-dpop`) | Proof parsing, jkt binding, nonce issue | AS grant state, store queries, HTTP |
| Replay (`sts-replay`) | jti/nonce deduplication, TTL eviction | OAuth policy, crypto, HTTP |
| Verify (`sts-verify`) | OIDC discovery fetch, JWKS fetch, trust-anchor check | Grant policy, store, HTTP responses |
| Exchange (`sts-exchange`) | RFC 8693 exchange orchestration, act assembly, scope narrowing | AS session/grant ownership, user model |

## Security Boundary Checks

| Check | Rule |
|---|---|
| Key custody | `sts-as-http` (transport) must not hold or load signing keys. Key load lives in `sts-jose`; the transport receives a signer handle only. |
| Store / policy separation | `sts-as-store` must not evaluate OAuth policy. It stores and retrieves; `sts-as-core` decides. |
| Discovery honesty | `sts-as-http` must not advertise endpoints, grant types, response types, auth methods, scopes, or algorithms that `sts-as-core` does not enforce at runtime. |
| JOSE algorithm downgrade | `sts-jose` must never silently accept `alg=none`, an unknown `kid`, or an algorithm not in the configured allow-list. Fail closed. |
| JWKS publication | `/jwks` must expose public key material only. Private key members (`d`, `p`, `q`, `dp`, `dq`, `qi`) must cause a startup or serialization panic, not a filtered response. |
| UserInfo claim scope | UserInfo must return only claims authorized by the Access Token's granted scopes and the server's claims policy. It must not return claims from a different authentication transaction. |
| ID Token / Access Token confusion | The ID Token authenticates the End-User to the Client/RP. The Access Token authorizes Resource Server access. They must not be accepted interchangeably. The AS must not accept its own ID Token as a bearer credential at `/userinfo` without Access Token validation. |
| Refresh Token isolation | Refresh Tokens must not be sent to Resource Servers. Resource Servers must not accept Refresh Tokens as Access Tokens. |
| Session / grant separation | Browser session lifecycle (`sts-as-store`) and OAuth grant lifecycle are separate state machines. Ending one must not silently keep the other alive. |
| Import purity | `sts-as-core`, `sts-jose`, `sts-dpop`, `sts-replay`, and `sts-verify` must not import Axum, tower, or any HTTP framework. Cross-crate boundaries are trait interfaces, not concrete HTTP types. |

## Token Meaning

| Token | Holder | Purpose | Must not be used as |
|---|---|---|---|
| ID Token | Client / Relying Party | Authenticates the End-User to the Client. Contains `iss`, `sub`, `aud` (client_id), `exp`, `iat`, `nonce`, and `at_hash` or `c_hash` when required. | API authorization credential at a Resource Server. Bearer token at `/userinfo` in place of an Access Token. |
| Access Token | Client / Relying Party, then Resource Server | Authorizes access to a protected resource or the UserInfo endpoint. Contains `iss`, `sub`, `aud` (resource), `exp`, `iat`, `scope`, and `cnf` when DPoP-bound. | Authentication proof for the Client/RP. Passed to the Authorization Endpoint. |
| Refresh Token | Client / Relying Party | Long-lived credential the Client presents to the Token Endpoint to obtain new Access Tokens without re-authentication, subject to rotation, revocation, and policy. | Sent to Resource Servers. Accepted by the UserInfo endpoint. |
| Delegated STS Token | Actor (MCP client, gateway, agent) | Narrowed Access Token produced by RFC 8693 token exchange. Contains the original `sub` (user), an `act` claim (actor identity), narrowed `aud` (single downstream resource), and narrowed `scope`. | Broad subject token. Passed back to the Authorization Endpoint. Used at a Resource Server without verifying the `act` claim. |

## Tier-to-Endpoint Mapping

| Tier | Tier label | Endpoints and behaviors |
|---|---|---|
| 0 | Existing STS (`sts-delegate-rs`) | `POST /token` (token exchange only, RFC 8693); `GET /jwks`; `GET /.well-known/oauth-authorization-server` (exchange-server metadata). Handled by `sts-exchange` + the existing `sts-http` transport. Not part of QuAuthz. |
| 1 | OAuth/OIDC AS MVP | `GET /authorize`; `POST /token` (authorization_code grant); `GET /userinfo`; `GET /jwks`; `GET /.well-known/openid-configuration`; `GET /.well-known/oauth-authorization-server`. Handled by `sts-as-http` + `sts-as-core` + `sts-as-store`. |
| 2 | Production OAuth baseline | `POST /revoke` (RFC 7009); `POST /introspect` (RFC 7662); `POST /token` adding refresh_token grant with rotation and reuse detection; resource indicator enforcement (RFC 8707); issuer identification and mix-up defense (RFC 9207). |
| 3 | High-security profile | `POST /par` (RFC 9126); DPoP enforcement per RFC 9449 at `/token` and `/userinfo`; optional JAR (RFC 9101); JARM response mode; optional mTLS (RFC 8705). Handled by `sts-as-http` + `sts-dpop` + `sts-jose`. |
| 4 | Delegation / MCP profile | `POST /token` token exchange (RFC 8693) accepting subject Access Tokens from Tier 1/2 and producing delegated tokens with `act` claim; Protected Resource Metadata (RFC 9728); RAR (RFC 9396). Handled by `sts-exchange` reused inside the QuAuthz workspace. |
| 5 | PQC profile | ML-DSA-65 signing and verification for Access Tokens and ID Tokens via `sts-jose`; downgrade-blocked algorithm policy; JOSE/JWK PQC metadata advertised only after runtime enforcement is verified; KMS/HSM custody path for signing keys. |
