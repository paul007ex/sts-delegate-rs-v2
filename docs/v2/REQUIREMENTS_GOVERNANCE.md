/Users/Shared/claude/sts-delegate-rs-v2/docs/v2/REQUIREMENTS_GOVERNANCE.md

The file is written. It follows the breachsafe-pqvpn-rs model exactly: same section order (Requirement Types table, Capture Rules, Requirement Template, PM Phase Gates, then requirement tables), same column shape (ID | Type | Requirement | Tier | Acceptance evidence | RFC | Issues | Status), and the same pattern of positive+negative test evidence per row.

Key choices made to match the repo's existing conventions (from `requirements/index.md`, `oidc-auth-code-pkce.md`, and `conformance-tiers.md`):

- Tier 1 for the core authz/token code path; Tier 2 for refresh tokens (matching the tier ladder in `conformance-tiers.md` and `roadmap.md`).
- RFC citations are section-exact: RFC 6749 §4.1.1, RFC 7636 §4.3, RFC 9207 §2, RFC 9068 §2, OIDC Core 1.0 §3.1.3.3, etc.
- REQ-AZ-002 and REQ-AZ-009 together encode the "bad redirect URI = error page, not error redirect" rule from RFC 6749 §4.1.2.1 — the most commonly botched authorization endpoint security requirement.
- REQ-AZ-008 wires RFC 9207 `iss` response parameter for mix-up defense, which the existing `conformance-tiers.md` explicitly lists as required.
- REQ-TOK-004 encodes the `invalid_client` + 401 rule and the `token_endpoint_auth_method` binding, consistent with the existing RFC 7523 surface in `sts-delegate`.
- REQ-TOK-005 includes the code-replay token revocation obligation with an explicit "or document the deferral" escape hatch, matching the repo's honesty-about-deferred-behavior convention.
