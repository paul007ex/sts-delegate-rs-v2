# Conformance Tiers

This is the planning skeleton for executable conformance evidence. The durable
OAuth 2.1 draft BCP14 ledger now lives in
[`conformance/oauth21-draft15-bcp14-ledger.md`](conformance/oauth21-draft15-bcp14-ledger.md).

## Source Status Checked On 2026-06-12

| Spec/profile | Status | Planning rule |
| --- | --- | --- |
| OAuth 2.1 | `draft-ietf-oauth-v2-1-15` | Say draft-aligned until it is an RFC |
| OAuth Security BCP | RFC 9700 | Production security baseline |
| OIDC Core | OpenID Connect Core 1.0 errata set 2 | OIDC MVP baseline |
| Browser-Based Apps | `draft-ietf-oauth-browser-based-apps-26` | Draft-tracked browser guidance |
| RFC 9207 | Published RFC | Required for mix-up defense |
| RFC 7636 | Published RFC | PKCE S256 required for MVP |
| RFC 8414 | Published RFC | OAuth metadata |
| OIDC Discovery | Final OpenID spec | OP metadata |
| RFC 7009 | Published RFC | Revocation tier |
| RFC 7662 | Published RFC | Introspection tier |
| RFC 8707 | Published RFC | Resource Indicators |
| RFC 9126 | Published RFC | PAR high-security profile |
| RFC 9396 | Published RFC | RAR advanced authorization profile |
| RFC 9449 | Published RFC | DPoP sender constraint |
| RFC 9728 | Published RFC | Protected Resource Metadata |

## Ledger Row Shape

Each implemented row must eventually contain:

- requirement source and section;
- normative requirement text summary;
- implementation package/function;
- positive test;
- negative/adversarial test;
- endpoint/curl proof when applicable;
- docs claim;
- status: planned, implemented, proof-only, deployment-only, or out-of-scope.

## MVP Requirements To Break Down First

- OpenID Provider discovery issuer exact-match validation.
- Discovery metadata honesty for endpoints, grants, response types, scopes,
  claims, auth methods, and signing algorithms.
- JWKS public-key publication only.
- JWKS key selection by `kid`.
- JWKS fail-closed behavior for unknown keys, unsupported algorithms, malformed
  keys, and private key leakage.
- Authorization Endpoint request validation.
- Authorization Code one-time use and PKCE binding.
- Token Endpoint authorization-code grant.
- OIDC ID Token issuance and validation.
- UserInfo claim authorization.
- issuer/mix-up defense.
- redirect URI exact matching.
- state and nonce handling.
- Access Token validation for Resource Servers.
- audit/log redaction.
