# OIDC And OAuth Endpoint Map

This file uses normative terminology first. Product examples are shown in
parentheses only.

## Authorization Endpoint

Normative role path:

```text
Resource Owner -> User Agent -> Authorization Endpoint -> User Agent -> Client/RP
```

Example mapping:

```text
human user -> browser -> /authorize at sts-authority -> browser -> MCP client callback
```

Required MVP request parameters:

- `response_type=code`
- `client_id`
- `redirect_uri`
- `scope`
- `state`
- `nonce` when OpenID Connect ID Token issuance is requested
- `code_challenge`
- `code_challenge_method=S256`

Required MVP response:

- `code`
- original `state`
- `iss` when RFC 9207 issuer identification is enabled

## Token Endpoint

Normative role path:

```text
Client/RP -> Token Endpoint
```

Required MVP grant:

- `grant_type=authorization_code`
- `code`
- `redirect_uri`
- `code_verifier`
- client authentication or client identification according to client type

Required response when `openid` was authorized:

- `access_token`
- `token_type=Bearer` or DPoP-bound profile when selected
- `expires_in`
- `id_token`
- optional `refresh_token` when policy allows it

## UserInfo Endpoint

Normative role path:

```text
Client/RP -> UserInfo Endpoint with Access Token -> Client/RP
```

UserInfo returns claims about the End-User. It is not the login endpoint. It
requires an Access Token whose scopes/claims policy authorizes those claims.

Minimum checks:

- Access Token signature or introspection result.
- issuer;
- audience/resource policy;
- expiration;
- scopes/claims;
- subject status;
- token revocation status where supported.

The returned `sub` must match the ID Token `sub` for the same authentication
transaction.

## JWKS

Normative role path:

```text
Client/RP or Resource Server -> JWKS Endpoint
```

JWKS publishes public verification keys only. It must never expose private key
members.

## Discovery

OIDC Provider metadata:

- `/.well-known/openid-configuration`

OAuth Authorization Server metadata:

- `/.well-known/oauth-authorization-server`

Discovery is a contract. Do not advertise endpoints, grants, algorithms, auth
methods, response types, or token types unless runtime behavior enforces them.

## Endpoint/Token Confusion Rules

- The ID Token authenticates the End-User to the Client/RP.
- The Access Token authorizes access to a Resource Server or UserInfo.
- The Authorization Code is a one-time credential for the Client/RP to redeem at
  the Token Endpoint.
- The Refresh Token is not sent to Resource Servers.
- A broad subject Access Token must not be accepted by a Resource Server that
  requires a narrowed delegated Access Token.
