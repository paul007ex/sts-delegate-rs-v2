# Competitive And Functional Landscape

Last reviewed: 2026-06-13.

This document tracks how QuAuthz compares to the Authorization Server products
a team would otherwise use, and where QuAuthz must differentiate. It is a planning
artifact, not a legal, procurement, or endorsement review. Competitor capabilities
change; re-verify before publishing any external claim.

## Comparison Table

| Dimension | QuAuthz | Okta | Auth0 | Keycloak | AWS Cognito |
|---|---|---|---|---|---|
| Cost / deploy model | OSS, self-hosted, zero per-token cost | SaaS, per-MAU pricing; can be expensive at scale | SaaS, per-MAU pricing; free tier limited | OSS, self-hosted; production ops burden is significant | AWS-managed SaaS; per-MAU after free tier |
| Local-first | Yes — single binary + Redis + Postgres; local dev uses in-memory store | No — SaaS; requires internet and Okta org | No — SaaS; requires internet and Auth0 tenant | Partial — self-hosted, but heavy; needs Postgres/MariaDB + JVM to run | No — AWS-managed; requires AWS account and region |
| Passwordless default | Yes — TOTP first; passkeys planned; no password column in MVP schema | Supported as an add-on policy; password DB present by default | Supported; password DB present by default | Supported via SPI; password DB enabled by default | Limited; password DB is the default flow |
| OAuth 2.1 draft-aligned | Yes — designed against draft-ietf-oauth-v2-1-15; PKCE required for all clients; no legacy implicit flow | Partial — PKCE available; legacy flows still supported; spec alignment varies by product tier | Partial — PKCE available; implicit flow can be enabled | Partial — PKCE available; older OIDC / OAuth 2.0 behavior enabled by default | Partial — PKCE available; legacy grants supported |
| DPoP support (RFC 9449) | Yes — shipped in Tier 0 STS; wired into AS token endpoint at Tier 1; `cnf.jkt` binding | Preview / beta as of 2025; Okta product timeline | Limited as of 2025 | Not present in mainline as of 2025; experimental extensions exist | Not present as of 2025 |
| PQC-ready signing | Yes — ML-DSA-65 (FIPS 204) default signing in Tier 0 STS; ID Token signing planned for Tier 5 | No — RSA/EC only; no announced PQC algorithm timeline | No — RSA/EC only | No — RSA/EC only | No — RSA/EC only |
| RFC 8693 delegation (act claim) | Yes — native; the original product; `act` claim is a first-class invariant | No — token exchange not RFC 8693; custom delegation or OIN patterns | No — native delegation with `act` is absent | No — STS-style delegation with `act` is absent | No — STS-style delegation with `act` is absent |
| MCP-aware | Yes — designed for MCP/OBO gateway authorization from the ground up | No — MCP is not an Okta design target | No | No | No |
| Open source | Yes — OSS, MIT or Apache 2.0 planned | No — proprietary SaaS | No — proprietary SaaS | Yes — Apache 2.0 | No — proprietary AWS service |
| Rust / performance | Yes — Rust/Axum; compiled binary; no JVM startup overhead | No — proprietary stack | No — Node.js backend | No — Java / Quarkus; JVM startup overhead |No |

## A/B Analysis

### What Okta does well

- Broad enterprise feature set: SAML, SCIM, lifecycle management, app catalog,
  adaptive MFA, hardware token support, and enterprise SLA.
- Turnkey SaaS: no infrastructure to operate; production-grade availability from
  day one.
- Deep enterprise integrations: AD/LDAP sync, HR system connectors, large partner
  ecosystem.
- Strong brand and compliance certifications: FedRAMP, HIPAA, ISO 27001, SOC 2.
- Okta Identity Engine (OIE) provides flexible policy composition.

What Okta does not do: native RFC 8693 delegation with `act` claim, local-first
deployment without an Okta org, PQC-signed tokens, or a meaningful DPoP enforcement
story as of mid-2025.

### What Keycloak does well

- Full-featured OSS: SAML 2.0, OIDC, SCIM, federation, social login, custom
  SPI extensions, and a broad enterprise feature set.
- Self-hosted: complete data sovereignty; no vendor SaaS dependency.
- Large community: mature documentation, active contributor base, and broad
  deployment experience.
- Flexible realm and client model: suitable for multi-tenant or complex
  permission structures.

What Keycloak does not do: native RFC 8693 delegation with `act`, PQC-signed
tokens, a local quickstart in under 10 minutes (Keycloak requires a full Postgres/MariaDB
database and a JVM, adding significant operational overhead), or a
DPoP enforcement surface. Keycloak's operational complexity is frequently cited as a
reason teams prefer a managed IdP instead.

### What QuAuthz does differently

QuAuthz does not try to win on breadth. It wins on protocol depth, deployment
simplicity, and the specific delegation use case none of the incumbents address.

- Native RFC 8693 `act` claim delegation is the founding invariant. It is not an
  extension, a workaround, or a future roadmap item. Every token exchange produces
  a `sub` + `act` pair; there is no impersonation mode.
- Local-first means a developer can run a full OAuth 2.1 AS locally with a single
  binary (Redis + Postgres; local dev uses in-memory store), complete a PKCE flow in a browser, and inspect every token
  without an internet connection, an AWS account, or an Okta org.
- PQC-ready from Tier 0. ML-DSA-65 is the default signing algorithm on the STS
  today. It is not a roadmap item added to compete; it is the default.
- DPoP is a first-class protocol surface, not a preview flag.
- OAuth 2.1 draft alignment means no implicit flow, PKCE required for all clients
  including confidential clients, and no legacy grant types in the default config.
- The codebase is small enough to audit. The conformance ledger maps each shipped
  requirement to a specific RFC clause and a specific test.

## Positioning Statement

Keep your Okta. Add QuAuthz for local-first dev, quantum-ready tokens, and native
delegation the others cannot do.

QuAuthz is not a replacement for Okta, Auth0, Keycloak, or AWS Cognito in their
primary enterprise use cases — directory integration, SAML federation, social login,
and enterprise SLA management. Teams that need those features should keep using them.

QuAuthz is the AS for teams that need:

- a complete OAuth 2.1 delegation chain without a SaaS dependency;
- RFC 8693 `act` claim stamping on every delegated token;
- PQC-signed tokens today, not on a vendor roadmap;
- DPoP sender-constraining enforced at the AS and STS layer;
- a codebase small enough to read, audit, and run locally in under 10 minutes.

## Positioning Implications

Do not claim:

- "Nobody is building an OAuth 2.1 AS in Rust" — there are other OSS Rust AS projects.
- "Okta does not support PKCE" — Okta supports PKCE.
- "Keycloak is insecure" — Keycloak is a mature, widely-audited product.
- "QuAuthz replaces Keycloak for enterprise deployments" — it does not, in the current tier scope.
- "QuAuthz is FAPI 2.0 certified" — it is not; FAPI alignment is a future tier.

Do claim carefully, once implemented:

- QuAuthz issues delegation tokens with `act` stamped from an AS-native `POST /authorize`
  flow through to the STS exchange — a complete chain none of the incumbents provide.
- QuAuthz signs tokens with ML-DSA-65 by default and will verify them correctly
  via `/jwks` when an external verifier supports the algorithm.
- QuAuthz enforces DPoP at both the AS token endpoint and the STS exchange endpoint
  using the same shared `sts-dpop` crate, with no behavioral gap between layers.
- QuAuthz completes a local OAuth 2.1 PKCE flow with a real browser and a real
  signed ID Token in under 10 minutes from git clone.

## Open Research Tasks

- Track DPoP implementation state in Okta, Auth0, and Keycloak; re-verify before
  publishing comparisons.
- Track PQC signing roadmap announcements from major AS vendors.
- Re-run crates.io and GitHub searches for Rust AS projects before any "only"
  or "first" claim.
- Interview target users to confirm that local-first and PQC-first are real
  adoption drivers, not just technical preferences.
- Build a negative capability matrix: where QuAuthz explicitly refuses to go
  (SAML, SCIM, password DB, social login), so the boundary is clear to evaluators.
