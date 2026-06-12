# OIDC Curl Walkthrough - Okta Tenant

Date: 2026-06-12

Tenant issuer:

```text
https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698
```

## Current Swimlane Position

```text
Resource Owner      User Agent        Client / RP            OpenID Provider / AS          Resource Server
(End-User)          (Browser)         (MCP/web/native app)   (Okta)                        (API/MCP)
     |                  |                  |                         |                          |
     |                  |                  | GET /.well-known/openid-configuration                 |
     |                  |                  |------------------------>|                          |
     |                  |                  | issuer/endpoints/scopes/grants/JWKS URI                |
     |                  |                  |<------------------------|                          |
     |                  |                  |                         |                          |
     |                  |                  | GET /v1/keys            |                          |
     |                  |                  |------------------------>|                          |
     |                  |                  | public signing keys     |                          |
     |                  |                  |<------------------------|                          |
     |                  |                  |                         |                          |
     | uses app         |                  |                         |                          |
     |----------------->|                  |                         |                          |
     |                  | GET /authorize?response_type=code&scope=openid&state&nonce&PKCE |
     |                  |-------------------------------------------->|                          |
     |                  |                  |                         | returns login page        |
     |                  |<--------------------------------------------|                          |
     |                  |                  |                         |                          |
     |                  |                  |                         | WE ARE HERE               |
     |                  |                  |                         | No code yet. No JWT yet.  |
```

## Leg 0 - Discovery

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/.well-known/openid-configuration"
```

Key requirements for `sts-authority`:

- Return JSON OpenID Provider metadata.
- `issuer` must exactly match the issuer URL used to fetch discovery.
- Advertise only implemented endpoints, grants, response types, scopes, claims,
  auth methods, PKCE methods, and signing algorithms.
- Clients/RPs must fail closed if issuer validation fails.

## Leg 1 - JWKS

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/keys"
```

Observed key:

```json
{
  "kty": "RSA",
  "alg": "RS256",
  "kid": "YHalnhteh3lt6T5epRbCz9z9XI4Zov3D54nNEjG91YU",
  "use": "sig"
}
```

Key requirements for `sts-authority`:

- Publish public verification keys only.
- Never expose private JWK members.
- ID Tokens must include JOSE `kid`.
- Verifiers must reject unknown `kid`, unsupported `alg`, `alg=none`, malformed
  keys, and private key leakage.

## Leg 2 - Authorization Request

Exact curl run:

```bash
curl -i -sS -L --max-redirs 0 "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/authorize?response_type=code&client_id=0oa13u4kntv2ka1al698&redirect_uri=http%3A%2F%2Flocalhost%3A8055%2Fcallback&scope=openid%20profile%20email&state=U4uW1sjqDaDp_oxBLDyaOleOBjEp6eQ95tKp_gFJ1vQ&nonce=psuiJqS0fuX_oAPphFrJHsDHX3M5duJBzqbwfYBtre4&code_challenge=CdnmTPR8QW0KZTtncD6qe8r7Z4tj31EHqgQ4JxEDnrs&code_challenge_method=S256&prompt=login"
```

Every parameter:

```text
response_type=code
client_id=0oa13u4kntv2ka1al698
redirect_uri=http%3A%2F%2Flocalhost%3A8055%2Fcallback
scope=openid%20profile%20email
state=U4uW1sjqDaDp_oxBLDyaOleOBjEp6eQ95tKp_gFJ1vQ
nonce=psuiJqS0fuX_oAPphFrJHsDHX3M5duJBzqbwfYBtre4
code_challenge=CdnmTPR8QW0KZTtncD6qe8r7Z4tj31EHqgQ4JxEDnrs
code_challenge_method=S256
prompt=login
```

Observed response:

```http
HTTP/2 200
content-type: text/html;charset=utf-8
referrer-policy: no-referrer
cache-control: no-cache, no-store
```

Meaning:

- The Authorization Request is accepted.
- Okta returned the interactive End-User sign-in page.
- No authorization code exists yet.
- No ID Token or Access Token exists yet.

Key requirements for `sts-authority`:

- Validate `response_type=code`.
- Validate registered `client_id`.
- Exact-match `redirect_uri`.
- Require `scope` to contain `openid` for OIDC.
- Require high-entropy `state`.
- Require high-entropy `nonce` for ID Token replay protection.
- Require PKCE S256.
- `prompt=login` forces End-User reauthentication.
- Interactive pages must avoid token leakage, use no-store, and set browser
  security headers.

## JWT Review Area

No JWT is produced until the Token Endpoint leg.

When the browser completes login and redirects back with an authorization code,
the next command will be:

```bash
curl -i -sS -X POST "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "<client_id>:<client_secret>" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "code=<authorization_code>" \
  --data-urlencode "redirect_uri=http://localhost:8055/callback" \
  --data-urlencode "code_verifier=<original_code_verifier>"
```

Expected token response fields:

```json
{
  "access_token": "<JWT or opaque token>",
  "id_token": "<JWT>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "openid profile email"
}
```

ID Token validation checklist:

- signature validates with JWKS;
- `iss` equals issuer;
- `aud` contains Client ID;
- `exp` has not passed;
- `iat` is sane;
- `nonce` equals the original Authorization Request nonce;
- `sub` is stable End-User subject;
- `kid` selects a known JWKS key;
- `alg` is allowed by discovery and policy.

Access Token validation checklist:

- validate issuer;
- validate audience/resource;
- validate expiry;
- validate scopes;
- validate signature or introspection result;
- never treat Access Token as an ID Token.
