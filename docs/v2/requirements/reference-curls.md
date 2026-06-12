# Reference Curl Pack

These are the real-tenant curl shapes used in the discovery/login walkthrough.
They are reference commands for the PM/dev ledger, not secrets-bearing artifacts.

Tenant issuer:

```text
https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698
```

## Discovery

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/.well-known/openid-configuration"
```

## JWKS

```bash
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/keys"
```

## Authorization Request

```bash
curl -i -sS -L --max-redirs 0 \
  "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/authorize?client_id=0oa13u4kntv2ka1al698&response_type=code&scope=openid+profile+email+offline_access&redirect_uri=http%3A%2F%2Flocalhost%3A8055%2Fcallback&state=<state>&code_challenge=<challenge>&code_challenge_method=S256&resource=https%3A%2F%2Fmcp-gateway&prompt=login"
```

Prereqs:

- client registered in the tenant;
- browser or user agent available;
- loopback callback listener on `http://localhost:8055/callback`;
- PKCE verifier generated locally;
- `state` and `nonce` freshly generated.

## Token Endpoint

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

Prereqs:

- fresh authorization code;
- client authentication material;
- original PKCE verifier;
- exact redirect URI;
- same resource indicator if the AS uses RFC 8707.

## UserInfo

```bash
USER_TOKEN=$(cat user_access_token.txt)
curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/userinfo" \
  -H "Authorization: Bearer $USER_TOKEN"
```

Prereqs:

- valid access token;
- token audience must be acceptable to UserInfo policy;
- scopes must authorize requested claims.

## Introspection

```bash
USER_TOKEN=$(cat user_access_token.txt)
curl -i -sS -X POST "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/introspect" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "<client_id>:<client_secret>" \
  --data-urlencode "token=$USER_TOKEN" \
  --data-urlencode "token_type_hint=access_token"
```

Prereqs:

- valid client authentication;
- access token to inspect.
