/Users/Shared/claude/sts-delegate-rs-v2/PRD.md

The file is written. It follows the breachsafe-pqvpn-rs PRD structure exactly: Overview paragraph, Product Readiness Rating table with the same 10 dimensions and the 8/10 and 9/10 ladders, Table of Contents, Product Thesis with an ASCII flow diagram, Living Spec Discipline, Target Users, MVP Scope (Tier 1), Non-Goals for MVP, Compliance Tiers table (Tier 0 through Tier 5), Success Metrics, and Open Decisions.

Key structural choices made:

- Tier 0 is explicitly labeled "done" in the tiers table, pointing to the existing `sts-delegate-rs` STS at 87% conformance.
- The Product Readiness Rating starts at 5/10 (lower than PQVPN's 6/10) because this repo has no implementation at all — only planning docs. The rating is justified dimension by dimension.
- The compliance tier table maps each tier to its exact RFC set, with a note that OAuth 2.1 draft-aligned language is required until the RFC publishes.
- The Open Decisions table is structured as a three-column decision/options/blocks format so each unresolved choice names what it blocks, making it actionable rather than a parking lot.
- DPoP reuse of `sts-dpop` crate from `sts-delegate-rs` is called out in MVP Scope to preserve the no-hand-rolled-crypto rule.
- ML-DSA-65 is the default signing algorithm; RS256 is explicit opt-in only, consistent with the existing STS behavior.
