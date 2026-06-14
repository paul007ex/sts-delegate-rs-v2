/Users/Shared/claude/quauthz-as/docs/USE_CASES.md

The file is written. Here is what it contains and why each decision was made:

**Structure matches the pqvpn model exactly:** index table first (ID | Title | Tier | Buyer value | Status | Link), then one prose paragraph per use case, then a product thread diagram, a boundary list, and a PM priority ordering.

**Table columns added beyond the pqvpn model:** `Tier` and `Link` columns, because the conformance-tiers.md document is central to this repo's planning and every use case maps to a specific tier. The pqvpn model has no tier concept so it omitted that column.

**Tier assignments come from prd.md:** Tier 1 = OAuth/OIDC AS MVP, Tier 2 = production OAuth baseline, Tier 3 = high-security profile (DPoP, PAR), Tier 4 = Delegation/MCP profile (RFC 8693, act, may_act), Tier 5 = PQC profile.

**Status values are conservative:** UC-01 and UC-05 are `prototype` (they are the two early validation gates named in prd.md), MCP/delegation and narrowing are `planned`, and the two highest-complexity use cases (enterprise AS replacement, may_act) are `future`. This matches the pqvpn convention exactly.

**Claims are honest:** UC-06 explicitly names what is not included (SCIM, SAML, social login, full admin console, FAPI certification). UC-05 scopes PQC claims to tested algorithm versions and verifiers only. The OAuth 2.1 language says `draft-aligned` throughout, consistent with conformance-tiers.md's ruling that OAuth 2.1 is still a draft as of June 2026.

**Links point to** `docs/use-cases/<filename>.md` — those individual files do not yet exist, consistent with the planning-phase status of the repo.
