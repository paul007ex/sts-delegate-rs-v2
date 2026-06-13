# Requirements Governance

This document defines how QuAuthz (sts-authority) turns product ideas into
requirements before implementation. The goal is to stay in PM mode long enough
to avoid locking poor architecture or vague compliance claims into code.

Current status: planning artifact. No application code exists yet.

## Requirement Types

| Type | Meaning | Example |
|---|---|---|
| Functional | What the product must do | Issue an authorization code bound to a PKCE challenge |
| Non-functional | How well it must operate | Token endpoint responds within a defined latency budget under load |
| Security | What must fail closed or be protected | Reject PKCE verifiers that do not match the stored challenge |
| Operational | What makes it supportable | Discovery metadata stays honest when features are disabled |
| Standards | What must conform to an RFC or published specification | Authorization Code flow conforms to RFC 6749 §4.1 and RFC 7636 |
| Regulatory/control | What maps to audit or buyer controls | Audit events support encryption-in-transit and access-control narratives |
| Non-goal | What the product explicitly does not own | Do not implement SCIM provisioning or a password database by default |

## Capture Rules

- Every requirement must have an observable acceptance signal.
- Material architecture, use-case, or requirement decisions must update the
  relevant living spec artifact in the v2 docs tree.
- Every standards-sensitive requirement must link to a primary RFC source and
  the section number, not just the RFC number.
- Every compliance-sensitive requirement must state whether it is shipped,
  planned, customer-owned, or external-assessment-required.
- Every security requirement must include accepted and rejected test paths once
  implementation begins.
- Do not use "compliant" without scope, RFC version, evidence, and assessment
  boundary.
- Do not advertise metadata capabilities that runtime does not enforce.
- Do not conflate OpenID Provider (OP) obligations with Client/RP obligations
  in the same requirement row.

## Requirement Template

```text
ID:
Type:
Actor:
Scenario:
Requirement:
Tier:
Acceptance evidence:
RFC mapping:
Issues:
Status:
```

## PM Phase Gates

### Before Tier 1 Implementation

- Functional and non-functional requirements are separated from security
  requirements.
- Standards and RFC conformance matrix exists for the MVP surface.
- Failure-mode matrix exists for authorization and token endpoints.
- Auth code store and PKCE binding contract is specified.
- Audit and redaction policy is specified.

### Before Tier 2 Implementation

- Refresh token lifecycle and revocation requirements are testable.
- Client registry requirements are specified and cover confidential, public,
  and native app profiles.
- UserInfo scope-to-claim mapping is specified.
- Token introspection and revocation endpoint behavior is specified.
- Negative test plans exist for replay, verifier mismatch, redirect hijack,
  and mix-up attack vectors.
- Discovery metadata honesty requirements are linked to runtime enforcement,
  not just documentation.

### Before Tier 3 and Beyond

- DPoP sender-constraint requirements are separated from bearer token
  requirements.
- PQC signing requirements distinguish proven hybrid from classical fallback.
- STS delegation surface (RFC 8693 token exchange) is separated from
  Authorization Server issuance.
- Regulatory and control mappings distinguish product controls from customer
  deployment responsibilities.
- Conformance ledger rows have executable proof harness entries, not just
  planned status.

## Area 1: Authorization Endpoint Requirements

These requirements cover the `/authorize` surface: request validation, PKCE
S256 binding, redirect URI enforcement, state and nonce handling,
response_type gating, code one-time use, code PKCE binding, issuer response
parameter, and error redirect behavior.

| ID | Type | Requirement | Tier | Acceptance evidence | RFC | Issues | Status |
|---|---|---|---|---|---|---|---|
| REQ-AZ-001 | Functional | The Authorization Endpoint MUST validate that `response_type=code` is present and reject all other response_type values with `unsupported_response_type`. | 1 | Positive: valid request with `response_type=code` reaches login. Negative: `response_type=token` and `response_type=id_token` return `unsupported_response_type` error redirect. | RFC 6749 §4.1.1; OAuth 2.1 draft §4.1.1 | — | planned |
| REQ-AZ-002 | Security | The Authorization Endpoint MUST validate that `client_id` is registered and that `redirect_uri` exactly matches a pre-registered URI character-for-character before doing anything else. If either check fails, the AS MUST NOT redirect; it MUST render an error page directly. | 1 | Positive: registered client and exact redirect URI proceeds. Negative: unknown `client_id` shows error page, not a redirect. Negative: redirect URI with trailing slash, different scheme, or extra path segment shows error page. | RFC 6749 §4.1.2.1; OAuth Security BCP RFC 9700 §4.1 | — | planned |
| REQ-AZ-003 | Security | The Authorization Endpoint MUST require PKCE and MUST reject requests where `code_challenge` is absent or where `code_challenge_method` is not `S256`. Plain method MUST NOT be accepted. | 1 | Positive: request with `code_challenge` and `code_challenge_method=S256` proceeds. Negative: missing `code_challenge` returns `invalid_request`. Negative: `code_challenge_method=plain` returns `invalid_request`. | RFC 7636 §4.3; OAuth 2.1 draft §4.1.1 | — | planned |
| REQ-AZ-004 | Security | If `state` is present in the Authorization Request, the AS MUST return the exact value unchanged in the Authorization Response. The AS MUST NOT truncate, transform, or omit `state`. | 1 | Positive: `state` in request appears unchanged in redirect response. Negative: modified or absent `state` in response causes Client to abort. | RFC 6749 §4.1.2; OAuth Security BCP RFC 9700 §4.11 | — | planned |
| REQ-AZ-005 | Security | If `nonce` is present in the Authorization Request for an OIDC flow, the AS MUST include the exact `nonce` value in the issued ID Token. | 1 | Positive: ID Token contains the exact `nonce` sent in the authorization request. Negative: ID Token without `nonce` when one was sent is rejected by the Client/RP. | OIDC Core 1.0 §3.1.2.1 §3.1.3.7 | — | planned |
| REQ-AZ-006 | Functional | Authorization codes MUST be one-time use. After a code is redeemed at the Token Endpoint, any attempt to redeem the same code again MUST return `invalid_grant` and SHOULD revoke tokens already issued for that code. | 1 | Positive: first redemption succeeds. Negative: second redemption of the same code returns `invalid_grant`. Negative: tokens issued on first redemption are revoked on replay attempt (or test that this is explicitly deferred). | RFC 6749 §4.1.2; OAuth 2.1 draft §4.1.3 | — | planned |
| REQ-AZ-007 | Security | The authorization code MUST be cryptographically bound to the `code_challenge` value presented in the Authorization Request. The binding MUST be stored server-side and MUST survive until the code expires or is redeemed. | 1 | Positive: code redeemed with correct verifier succeeds. Negative: code redeemed with wrong verifier returns `invalid_grant`. Negative: code redeemed without any verifier returns `invalid_request`. | RFC 7636 §4.4 §4.6 | — | planned |
| REQ-AZ-008 | Standards | The Authorization Response MUST include the `iss` parameter containing the AS issuer identifier when the client is registered for the `iss` response parameter or when AS-level mix-up defense is required. | 1 | Positive: `iss` appears in the authorization redirect response alongside `code` and `state`. Negative: Client rejects a response where `iss` does not match the expected issuer. | RFC 9207 §2 | — | planned |
| REQ-AZ-009 | Security | When the AS rejects an Authorization Request due to an invalid `redirect_uri` or unknown `client_id`, it MUST display an error page directly and MUST NOT redirect to any URI. All other Authorization Request errors MUST use the error redirect to the pre-validated `redirect_uri` with `error` and `error_description` parameters. | 1 | Positive: bad redirect URI shows error page, no redirect. Positive: bad scope returns error redirect to the registered URI. Negative: attacker-controlled redirect URI is never followed. | RFC 6749 §4.1.2.1; OAuth Security BCP RFC 9700 §4.1 | — | planned |
| REQ-AZ-010 | Operational | The AS MUST enforce a short maximum lifetime on authorization codes. Codes that have not been redeemed within the configured maximum lifetime MUST be rejected at the Token Endpoint with `invalid_grant`. | 1 | Positive: code redeemed within lifetime succeeds. Negative: code presented after expiry returns `invalid_grant`. Configuration: max lifetime is configurable and defaults to no more than 10 minutes. | RFC 6749 §4.1.2 (10-minute guidance); OAuth 2.1 draft §4.1.3 | — | planned |

## Area 2: Token Endpoint — Authorization Code Grant Requirements

These requirements cover the `/token` surface for `grant_type=authorization_code`:
grant_type validation, PKCE verifier check, redirect_uri echo match, client
authentication, code one-time enforcement, JWT access token issuance (RFC 9068),
ID token issuance (OIDC Core), and refresh token conditional issuance.

| ID | Type | Requirement | Tier | Acceptance evidence | RFC | Issues | Status |
|---|---|---|---|---|---|---|---|
| REQ-TOK-001 | Functional | The Token Endpoint MUST accept `grant_type=authorization_code` and MUST reject all other grant types that are not explicitly configured, returning `unsupported_grant_type`. | 1 | Positive: `grant_type=authorization_code` with valid inputs succeeds. Negative: `grant_type=implicit`, `grant_type=password`, and any unknown grant type return `unsupported_grant_type`. | RFC 6749 §4.1.3; OAuth 2.1 draft §4.1.3 | — | planned |
| REQ-TOK-002 | Security | The Token Endpoint MUST verify that the `code_verifier` in the Token Request transforms to a value that matches the stored `code_challenge` using S256. Any mismatch, missing verifier, or wrong encoding MUST return `invalid_grant`. | 1 | Positive: correct `code_verifier` produces matching S256 transform. Negative: wrong verifier returns `invalid_grant`. Negative: missing verifier returns `invalid_request`. Negative: verifier that is too short or too long returns `invalid_request`. | RFC 7636 §4.5 §4.6 | — | planned |
| REQ-TOK-003 | Security | The Token Endpoint MUST verify that the `redirect_uri` in the Token Request exactly matches the `redirect_uri` used in the Authorization Request that produced the code. Any mismatch MUST return `invalid_grant`. | 1 | Positive: same redirect URI as authorization request succeeds. Negative: different scheme, host, path, or port returns `invalid_grant`. Negative: absent redirect URI when one was required returns `invalid_request`. | RFC 6749 §4.1.3; OIDC Core 1.0 §3.1.3.1 | — | planned |
| REQ-TOK-004 | Security | Confidential clients MUST authenticate at the Token Endpoint. Missing or invalid client authentication MUST return `invalid_client` with HTTP 401. Public clients MUST present `client_id` and MUST NOT present a credential. Client authentication method is determined by the client's registered `token_endpoint_auth_method`. | 1 | Positive: confidential client with valid credential succeeds. Negative: confidential client with wrong credential returns `invalid_client` + 401. Negative: public client presenting a credential returns `invalid_client`. Negative: missing `client_id` for public client returns `invalid_request`. | RFC 6749 §4.1.3; OAuth 2.1 draft §4.1.3; RFC 7523 §3 | — | planned |
| REQ-TOK-005 | Security | The Token Endpoint MUST enforce one-time use of authorization codes. A code already redeemed or expired MUST return `invalid_grant`. If a replayed code is detected after tokens were already issued, the AS SHOULD revoke those tokens. | 1 | Positive: first redemption of a valid code succeeds. Negative: second redemption of the same code returns `invalid_grant`. Negative: revocation of previously issued tokens on replay is tested or the deferral is explicitly documented. | RFC 6749 §4.1.2; OAuth 2.1 draft §4.1.3 | — | planned |
| REQ-TOK-006 | Standards | When the Authorization Grant included `openid` scope, the Token Endpoint MUST issue a signed JWT Access Token conforming to RFC 9068 containing at minimum `iss`, `sub`, `aud`, `exp`, `iat`, `jti`, `client_id`, and `scope` claims. The `alg` MUST NOT be `none`. | 1 | Positive: access token decodes to correct claims. Positive: `aud` matches the resource indicator or the default audience. Negative: `alg=none` is never issued. Negative: expired access token rejected by Resource Server. Negative: access token with wrong `aud` rejected by Resource Server. | RFC 9068 §2 §2.2; RFC 8725 §3.1 | — | planned |
| REQ-TOK-007 | Standards | When `openid` scope is present in the grant, the Token Endpoint MUST issue a signed ID Token containing `iss`, `sub`, `aud`, `exp`, `iat`, and `nonce` (when nonce was in the Authorization Request). The ID Token `aud` MUST be the `client_id`. The ID Token MUST NOT be used as an Access Token. | 1 | Positive: ID Token issued when `openid` scope granted. Positive: ID Token contains `nonce` when nonce was sent in the authorization request. Negative: absent `openid` scope produces no ID Token. Negative: Client rejects ID Token with wrong `aud`. Negative: Resource Server rejects ID Token presented as Access Token. | OIDC Core 1.0 §2 §3.1.3.3 §3.1.3.7 | — | planned |
| REQ-TOK-008 | Functional | When `offline_access` scope is granted and the client is configured to receive refresh tokens, the Token Endpoint MUST issue a refresh token. Refresh tokens MUST be bound to the issuing client and MUST NOT be accepted by other clients. Refresh token issuance is optional per client configuration; clients not configured for refresh MUST NOT receive one. | 2 | Positive: `offline_access` scope + configured client returns a refresh token. Negative: client not configured for refresh does not receive one. Negative: refresh token presented by a different client returns `invalid_grant`. Positive: valid refresh token exchange returns new access token with equal or narrower scope. | RFC 6749 §6; OAuth 2.1 draft §4.3; OIDC Core 1.0 §12 | — | planned |
