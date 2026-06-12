# OIDC Requirements Ledger

This is the PM/dev accountability ledger for the v2 OIDC surface.

| ID | Tier | RFC / Section | Requirement | Prereqs | Curl proof | Code path | Test evidence | Issue | Status | Status detail | Done criteria | Owner |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| OIDC-DISC-001 | MVP | OIDC Discovery 1.0 §4.2 / §4.3 | Discovery response MUST be JSON and Client/RP MUST validate exact issuer match. | none | `curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/.well-known/openid-configuration"` | `sts-as-http.discovery` | `tests/test_discovery.py` | #44, #10 | planned | RFC-backed | exact JSON response + issuer mismatch rejection test | PM + Dev |
| OIDC-CONN-001 | MVP | OIDC Discovery 1.0 §4.3, OIDC Core 1.0 §3.1.2.1 / §3.1.3.7, RFC 7636 §4.2 / §4.3 / §4.5 / §4.6, RFC 6750 §2 | OIDC connector MUST discover issuer, build the authorization request, complete the PKCE exchange, validate the ID Token, and preserve the Access Token for downstream resource use. | real tenant configured, loopback callback listener, PKCE generator, client registration | see `oidc-auth-code-pkce.md#oidc-connector` | `sts-client.connector` | `tests/test_discovery.py`, `tests/test_authorize.py`, `tests/test_token.py`, `tests/test_id_token.py` | #44, #10, #9, #11, #45 | planned | RFC-backed | connector walk-through with discovery/authorize/token/ID Token validation evidence | PM + Dev |
| OIDC-JWKS-001 | MVP | RFC 7517 §5, RFC 7515 §4.1.4, RFC 8725 §3.1 | JWKS MUST publish public keys only; verifiers MUST fail closed on unknown kid/alg. | discovery issuer known | `curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/keys"` | `sts-jose.jwks` | `tests/test_jwks.py` | #10 | planned | RFC-backed | public-only JWKS + unknown kid/alg rejection | Dev |
| OIDC-AUTHZ-001 | MVP | OIDC Core 1.0 §3.1.2.1 / §3.1.2.2, RFC 6749 §4.1.1 / §4.1.2, RFC 7636 §4.2 / §4.3 | `/authorize` MUST validate `client_id`, `redirect_uri`, `state`, `openid` scope, and PKCE S256. | discovery issuer + client registration + loopback callback listener | `curl -i -sS -L --max-redirs 0 "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/authorize?...&prompt=login"` | `sts-as-http.authorize` | `tests/test_authorize.py` | #9, #46 | planned | RFC-backed | valid login flow + missing-openid/bad-redirect/bad-PKCE rejection tests | Dev |
| OIDC-AUTHZ-002 | MVP | RFC 6749 §4.1.2, RFC 8252 §7.3 | Authorization code MUST be one-time and bound to redirect URI and client; loopback redirect is allowed for native apps. | browser login completed, callback listener active | `loopback redirect -> http://localhost:8055/callback?code=...&state=...` | `sts-as-store.codes` | `tests/test_authorize.py::test_code_replay` | #9, #37 | planned | RFC-backed | code replay rejection + exact state echo | Dev |
| OIDC-TOKEN-001 | MVP | OIDC Core 1.0 §3.1.3.1 / §3.1.3.3, RFC 6749 §4.1.3, RFC 7636 §4.5 / §4.6 | `/token` MUST redeem auth code with client auth and PKCE verifier; response MUST include OIDC ID Token when `openid` scope was granted. | fresh authorization code, client auth, code_verifier | `curl -i -sS -X POST "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/token" ...` | `sts-as-http.token` | `tests/test_token.py` | #11, #10 | planned | RFC-backed | valid token exchange + bad verifier/replay/bad-client rejection tests | Dev |
| OIDC-IDT-001 | MVP | OIDC Core 1.0 §2 / §3.1.3.7 | ID Token MUST validate signature, `iss`, `aud`, `exp`, `iat`, `nonce`, and `kid`-based key selection. | JWKS known, token obtained | `id_token header/payload decoded from token response` | `sts-verify.id_token` | `tests/test_id_token.py` | #45, #10 | planned | RFC-backed | valid ID Token validation + issuer/audience/nonce failure cases | Dev |
| OIDC-USERINFO-001 | MVP | OIDC Core 1.0 §5.3 / §5.3.1 / §5.3.2, RFC 6750 §2.1 | UserInfo MUST require an Access Token and MUST return only authorized claims; `sub` MUST match the ID Token subject. | access token obtained | `curl -i -sS "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/userinfo" -H "Authorization: Bearer $USER_TOKEN"` | `sts-as-http.userinfo` | `tests/test_userinfo.py` | #10, #45 | planned | RFC-backed | token-required and subject-binding tests | Dev |
| OIDC-AT-001 | MVP | RFC 9068 §2, RFC 6750 §2 | JWT Access Tokens MUST have clear `iss`/`aud`/`scope`/`exp` semantics and Resource Servers MUST validate them. | access token obtained | `curl -i -sS -X POST "https://trial-5395738.okta.com/oauth2/aus13u46is3QfakEx698/v1/introspect" ...` | `sts-core.token` | `tests/test_access_token.py` | #10 | planned | RFC-backed | audience/scope/expiry validation + wrong-aud rejection | Dev |
| OIDC-LOG-001 | baseline | Security/doc policy | Logs/docs MUST not print raw bearer secrets, code verifiers, private keys, or Authorization headers. | none | `n/a` | `docs/`, `scripts/`, `tests/` | `tests/test_redaction.py` | #46 | planned | Policy | redaction tests + no-secret docs/output check | PM + Dev |

## How To Use

- PM owns the row definition and acceptance criteria.
- Dev fills the code path and test evidence.
- Add `Curl proof` only for rows that have an HTTP surface.
- Add `Prereqs` when a row needs a real tenant, a browser callback listener, a
  client secret, or a prior token.
- A row may move from `planned` to `implemented` only when code, tests, and docs
  all match the RFC requirement.
- If the row is only a policy choice or later-tier feature, mark it
  `proof-only`, `deployment-only`, or `out-of-scope` instead of claiming
  implementation.

## Relationship To The Narrative Requirements

The detailed walk-through lives in
[`oidc-auth-code-pkce.md`](oidc-auth-code-pkce.md). This ledger is the numbered
accountability layer on top of that narrative.
