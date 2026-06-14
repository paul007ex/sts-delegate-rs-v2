/Users/Shared/claude/quauthz-as/docs/v2/ARCHITECTURE.md

The file is written. Here is what each section contains and why it was shaped that way:

**Positioning** — one paragraph that names the product, its place in the OAuth chain (AS/OP upstream of the STS), and what it does not own. Explicitly anchors `sts-delegate-rs` as the continuing STS product so the boundary is unambiguous.

**Left-to-Right Flow** — five-column ASCII diagram (Resource Owner / User Agent / Client-RP / QuAuthz AS / Resource Server) tracing the PKCE Authorization Code flow, then the Resource Server validation callback, then the optional RFC 8693 exchange leg that produces the delegated token with `act` claim. Matches the sequence documented in `docs/v2/architecture.md` and `oidc-endpoints.md` but rendered as a proper swim-lane diagram.

**Package / Crate Map** — ten rows derived exactly from the planning repo's package list, with expanded responsibility text pulled from the PRD and architecture docs. `sts-exchange` row explicitly calls out shared reuse from `sts-delegate-rs`. Each row includes a "no" boundary in the description so ownership is not ambiguous.

**Layer Ownership** — two-column table (owns / must not own) for each of the ten layers, mirroring the `breachsafe-pqvpn-rs` format but adapted to the OAuth/AS seams: domain, store, transport, config, CLI, JOSE, DPoP, replay, verify, exchange.

**Security Boundary Checks** — ten rows covering key custody, store/policy separation, discovery honesty, JOSE algorithm downgrade, JWKS private key leakage, UserInfo claim scope, ID Token/Access Token confusion, Refresh Token isolation, session/grant separation, and import purity. Sourced from the planning repo's existing bullet list and expanded to table format.

**Token Meaning** — four-row table (ID Token, Access Token, Refresh Token, Delegated STS Token) with holder, purpose, and "must not be used as" columns. Exact definitions from `docs/v2/architecture.md` and `oidc-endpoints.md`.

**Tier-to-Endpoint Mapping** — six tiers (0–5) derived from `docs/v2/prd.md` compliance tier table and `roadmap.md`, mapping each tier to its concrete endpoints and the crates that handle them.