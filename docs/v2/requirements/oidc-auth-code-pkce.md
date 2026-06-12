# OIDC Authorization Code + PKCE Requirements

Source tenant used for walkthrough:

```text
https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698
```

This file captures requirements as we walk the real Okta curl/browser flow. It
uses normative OAuth/OIDC role names first and product examples second.

## Discovery Normative Requirements

These are the direct requirements that apply before any browser login starts.

| Source | Normative requirement |
| --- | --- |
| OIDC Discovery 1.0 Section 4.2 | The OpenID Provider configuration response MUST be JSON and MUST include the issuer and endpoint metadata needed by the Client/RP. |
| OIDC Discovery 1.0 Section 4.2 | The OpenID Provider MUST publish `issuer`, `authorization_endpoint`, `jwks_uri`, `response_types_supported`, `subject_types_supported`, and `id_token_signing_alg_values_supported`. |
| OIDC Discovery 1.0 Section 4.3 | The Client/RP MUST validate that the discovered `issuer` exactly matches the issuer value it expects. |
| RFC 8414 Section 2 | The Authorization Server metadata MUST honestly describe supported endpoints and capabilities. |
| RFC 8414 Section 2 | If a metadata value is unsupported by runtime behavior, it MUST NOT be advertised as supported. |
| RFC 8414 Section 2 | Discovery metadata MAY be cached, but cache behavior MUST remain compatible with endpoint rollout and key rotation. |

## Role Mapping

| Normative role | Product examples |
| --- | --- |
| Resource Owner / End-User | Human user |
| User Agent | Browser, Apple system browser |
| Client / Relying Party | MCP login client, web app, native app |
| Authorization Server / OpenID Provider | Okta now, `sts-authority` later |
| Resource Server / Protected Resource | UserInfo, MCP server, API |

## Leg 0 - OpenID Provider Discovery

Command:

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/.well-known/openid-configuration"
```

Observed:

- `issuer=https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698`
- `authorization_endpoint=.../v1/authorize`
- `token_endpoint=.../v1/token`
- `userinfo_endpoint=.../v1/userinfo`
- `jwks_uri=.../v1/keys`
- `code_challenge_methods_supported=["S256"]`
- `id_token_signing_alg_values_supported=["RS256"]`

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| OIDC Discovery 1.0 Section 4.2 | Configuration response must publish OP metadata as JSON. |
| OIDC Discovery 1.0 Section 4.2 | `issuer`, `authorization_endpoint`, `jwks_uri`, `response_types_supported`, `subject_types_supported`, and `id_token_signing_alg_values_supported` are required OP metadata fields. |
| OIDC Discovery 1.0 Section 4.3 | The Client/RP must validate that discovered `issuer` exactly matches the issuer value it expected. |
| RFC 8414 Section 2 | OAuth AS metadata must be honest about endpoints, grants, auth methods, response types, and capabilities. |

Product tests:

- positive: discovery returns JSON and exact issuer.
- negative: Client/RP rejects discovery if issuer differs.
- negative: metadata must not advertise endpoints or algorithms that runtime does
  not enforce.

## Leg 1 - JWKS

Command:

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/keys"
```

Observed public key snippet:

```json
{
  "kty": "RSA",
  "alg": "RS256",
  "kid": "YHalnhteh3lt6T5epRbCz9z9XI4Zov3D54nNEjG91YU",
  "use": "sig"
}
```

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| RFC 7517 Section 5 | JWKS is a JSON object containing a `keys` array of JWK values. |
| RFC 7517 Section 4 | Public key parameters must be sufficient for verifiers to validate signatures. |
| RFC 7515 Section 4.1.4 | JOSE `kid` is used to match the signing key; verifiers must fail closed when key selection is ambiguous or missing. |
| RFC 8725 Section 3.1 | JWT implementations must perform algorithm verification and not trust attacker-controlled `alg` values. |

Product tests:

- positive: ID Token signed with active key validates through JWKS.
- negative: unknown `kid` rejected.
- negative: `alg=none` rejected.
- negative: private JWK members never appear in JWKS.
- negative: advertised `alg` must match signing and validation policy.

## Leg 2 - Authorization Request

Command shape:

```bash
curl -i -sS -L --max-redirs 0 \
  "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/authorize?client_id=0oa13u4kntv2ka1al698&response_type=code&scope=openid+profile+email+offline_access&redirect_uri=http%3A%2F%2Flocalhost%3A8055%2Fcallback&state=<state>&code_challenge=<challenge>&code_challenge_method=S256&resource=https%3A%2F%2Fmcp-gateway"
```

Observed:

- Okta accepted the Authorization Request.
- Browser login completed.
- Okta redirected to the loopback Client redirect URI with `code` and `state`.

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| OIDC Core 1.0 Section 3.1.2.1 | Authentication Request uses OAuth Authorization Request parameters plus OIDC parameters; `scope` must contain `openid`. |
| OIDC Core 1.0 Section 3.1.2.1 | `response_type`, `client_id`, `scope`, and `redirect_uri` are part of the Authorization Request validation surface. |
| OIDC Core 1.0 Section 3.1.2.2 | OP must validate all required parameters and redirect URI/client binding before continuing. |
| RFC 6749 Section 4.1.1 | Authorization Code flow starts at the Authorization Endpoint with `response_type=code`, `client_id`, optional `state`, and redirect URI when required. |
| RFC 6749 Section 4.1.2 | If `state` was present in the request, the Authorization Server includes the exact value in the response. |
| RFC 7636 Sections 4.2 and 4.3 | Client creates `code_challenge` from `code_verifier` and sends `code_challenge` plus `code_challenge_method` in the Authorization Request. |
| RFC 8707 Section 2 | Client may use `resource` to indicate the target protected resource; AS must use it when determining audience/resource binding if supported. |

Product tests:

- positive: valid Authorization Request reaches login/consent and issues a code.
- negative: missing `openid` rejected for OIDC.
- negative: bad redirect URI rejected without redirecting to attacker URI.
- negative: missing PKCE rejected for MVP.
- negative: returned `state` mismatch aborts the Client.

## Leg 3 - Authorization Code Callback

Observed:

```text
code=SPyBE6KVE0xU81ZXtuIcQ1ibMMeIGFfGFJ80QCQYmgI
state=qifw9XeysKmUhGScwS_Dog
```

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| RFC 6749 Section 4.1.2 | Authorization Server returns `code`; if `state` was present, returns identical `state`. |
| RFC 6749 Section 4.1.2 | Authorization code must expire shortly after issuance; the RFC recommends a maximum lifetime of 10 minutes. |
| RFC 6749 Section 4.1.2 | Client must not use an authorization code more than once; AS must deny replay. |
| RFC 8252 Section 7.3 | Native apps may use loopback redirect URIs on localhost with dynamic ports; local receiver captures the code. |

Product tests:

- positive: callback captures code and exact state.
- negative: wrong state aborts Client.
- negative: code replay is rejected at `/token`.
- negative: expired code rejected at `/token`.

## Leg 4 - Token Endpoint

Command shape:

```bash
curl -i -sS -X POST "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "<client_id>:<client_secret>" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "code=<authorization_code>" \
  --data-urlencode "redirect_uri=http://localhost:8055/callback" \
  --data-urlencode "code_verifier=<original_code_verifier>" \
  --data-urlencode "resource=https://mcp-gateway"
```

Observed redacted response:

```json
{
  "token_type": "Bearer",
  "expires_in": 86400,
  "access_token": "<924 chars>",
  "scope": "openid profile offline_access email",
  "refresh_token": "<43 chars>",
  "id_token": "<1033 chars>"
}
```

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| OIDC Core 1.0 Section 3.1.3.1 | Client redeems the Authorization Code at the Token Endpoint. |
| OIDC Core 1.0 Section 3.1.3.1 | Token Request includes `grant_type=authorization_code`, `code`, and `redirect_uri` when required. |
| OIDC Core 1.0 Section 3.1.3.3 | Successful Token Response returns OAuth token response fields and an ID Token for OIDC. |
| RFC 6749 Section 4.1.3 | Confidential Clients authenticate to the Token Endpoint. |
| RFC 7636 Sections 4.5 and 4.6 | Client sends `code_verifier`; AS transforms it and compares it to the stored `code_challenge`. |

Product tests:

- positive: valid code, redirect URI, client auth, and verifier returns tokens.
- negative: wrong verifier rejected.
- negative: wrong redirect URI rejected.
- negative: wrong client rejected.
- negative: reused code rejected.
- negative: missing client authentication rejected for confidential clients.

## Leg 5 - Real Token Claim Snippets

Raw tokens are bearer secrets and are not stored in this requirements file.

Observed Access Token header:

```json
{
  "kid": "YHalnhteh3lt6T5epRbCz9z9XI4Zov3D54nNEjG91YU",
  "alg": "RS256"
}
```

Observed Access Token payload snippet:

```json
{
  "iss": "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698",
  "sub": "pavel.volosen@hpe.com",
  "aud": "api://obo",
  "cid": "0oa13u4kntv2ka1al698",
  "scp": ["openid", "profile", "offline_access", "email"],
  "auth_time": 1781276008
}
```

Observed ID Token summary:

```text
id_token returned
aud=0oa13u4kntv2ka1al698
meaning=proof of End-User authentication for the Client/RP, not an API token
```

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| OIDC Core 1.0 Section 2 | ID Token is a security token containing claims about authentication of an End-User by an Authorization Server when using a Client. |
| OIDC Core 1.0 Section 3.1.3.7 | Client validates ID Token issuer, audience, signature, expiration, issued-at, and nonce where applicable. |
| RFC 9068 Section 2 | If JWT Access Tokens are used, issuer, subject, audience, expiration, issued-at, client, and scope semantics must be clear and validated by Resource Servers. |
| RFC 6750 Section 2 | Bearer Access Tokens are presented to protected resources; possession is sufficient unless sender constraints such as DPoP are added. |

Product tests:

- positive: Client validates ID Token using JWKS, issuer, audience, and nonce.
- negative: ID Token sent to Resource Server is rejected.
- positive: Resource Server validates Access Token issuer, audience, expiry, and scope.
- negative: Access Token with wrong audience rejected.
- negative: Access Token with missing scope rejected.
- negative: expired token rejected.

## PKCE Code We Need

Client-side example code is useful for demos and tests:

- generate high-entropy `code_verifier`;
- derive `code_challenge=base64url(SHA256(code_verifier))`;
- send only the challenge in the Authorization Request;
- keep verifier local until `/token`;
- verify returned `state`.

Server-side product code is mandatory:

- store `code_challenge` and method with the authorization code;
- require `S256` for MVP;
- compare transformed `code_verifier` to stored challenge;
- reject missing, wrong, replayed, or expired values.

## Leg 6 - UserInfo

Command:

```bash
USER_TOKEN=$(cat user_access_token.txt)
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/userinfo" \
  -H "Authorization: Bearer $USER_TOKEN"
```

Observed response:

```json
{
  "sub": "00u13u43lrh3W6KX5698",
  "name": "Paul Volosen",
  "locale": "en_US",
  "email": "pavel.volosen@hpe.com",
  "preferred_username": "pavel.volosen@hpe.com",
  "given_name": "Paul",
  "family_name": "Volosen",
  "zoneinfo": "America/Los_Angeles",
  "email_verified": true
}
```

Requirements:

| Source | Requirement for `sts-authority` |
| --- | --- |
| OIDC Core 1.0 Section 5.3 | UserInfo Endpoint is an OAuth 2.0 Protected Resource that returns claims about the authenticated End-User. |
| OIDC Core 1.0 Section 5.3.1 | Client sends an Access Token to UserInfo using OAuth Bearer Token usage. |
| OIDC Core 1.0 Section 5.3.2 | UserInfo response must return claims as JSON unless a signed/encrypted response format is selected. |
| OIDC Core 1.0 Section 5.3.2 | `sub` in UserInfo must match the `sub` in the ID Token for the same End-User; mismatch is an error condition for the Client/RP. |
| RFC 6750 Section 2.1 | Bearer token can be sent in the `Authorization: Bearer` header. |

Product tests:

- positive: valid Access Token with `profile email` returns authorized claims.
- negative: missing Access Token rejected.
- negative: expired Access Token rejected.
- negative: wrong audience/resource rejected, if UserInfo audience policy is used.
- negative: insufficient scopes return no unauthorized claims or an OAuth error.
- negative: UserInfo `sub` mismatch with ID Token `sub` causes Client/RP rejection.
