# Authorization Server v2 Architecture

## Boundary Rule

Do not widen the current STS transport into a full Authorization Server. The AS
product owns interactive authentication, sessions, grants, clients, consent,
codes, refresh tokens, UserInfo, and identity lifecycle. The STS product owns
delegation/token exchange.

## Proposed Packages

| Package | Responsibility |
| --- | --- |
| `sts-as-core` | Authorization Server domain rules, grants, sessions, claims, policy decisions |
| `sts-as-store` | Durable state for users, clients, sessions, codes, grants, refresh-token families |
| `sts-as-http` | `/authorize`, `/token` AS grants, discovery, UserInfo, logout, errors |
| `sts-as-config` | AS configuration resolution and startup validation |
| `sts-as-cli` | bootstrap, client/user/admin operations, local dev flow |
| `sts-exchange` | extracted RFC 8693 exchange application service shared with the current STS |
| `sts-jose` | JOSE/JWK/JWKS/JWT signing and verification |
| `sts-dpop` | DPoP proof and sender constraint validation |
| `sts-replay` | replay-state policy and backend |
| `sts-verify` | issuer/trust-anchor validation |

## Critical Flow

```text
Resource Owner       User Agent       Client/RP        Authorization Server / OP       Resource Server
(human user)         (browser)        (MCP client)     (sts-authority)                 (MCP/API)
      |                  |                 |                         |                       |
      | uses             |                 |                         |                       |
      |----------------->|                 |                         |                       |
      |                  | GET /authorize?client_id&redirect_uri&scope&state&nonce&PKCE |
      |                  |------------------------------------------>|                       |
      |                  | login + consent                           |                       |
      |                  |<------------------------------------------|                       |
      |                  | redirect redirect_uri?code&state&iss      |                       |
      |                  |---------------->|                         |                       |
      |                  |                 | POST /token code+verifier+client auth             |
      |                  |                 |------------------------>|                       |
      |                  |                 | access_token + id_token |                       |
      |                  |                 |<------------------------|                       |
      |                  |                 | call API with access_token                        |
      |                  |                 |-------------------------------------------------->|
      |                  |                 |                         | validate iss/aud/exp/scope|
```

## Token Meaning

- ID Token: authentication result for the Client/Relying Party. It is not an API
  authorization token.
- Access Token: authorization credential for a Resource Server / Protected
  Resource or UserInfo endpoint.
- Refresh Token: credential used by a Client to obtain new Access Tokens, subject
  to rotation, revocation, and policy.
- Delegated STS token: narrowed Access Token produced by token exchange for a
  downstream Resource Server.

## Security Boundary Checks

- Transport must not own key custody.
- Store must not decide OAuth policy.
- Core must not perform HTTP rendering.
- JOSE must not silently downgrade algorithms.
- UserInfo must not return claims outside scope/claims policy.
- Discovery must advertise only enforced runtime behavior.
