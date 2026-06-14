# Repository Map

This document is the file and directory ownership map for QuAuthz
(`quauthz-as` / `sts-authority`). It complements the architecture and
workspace boundary docs by showing where things live today (planning phase) and
where new work should go as implementation starts.

## Current Top-Level Tree

This repository is planning-phase. The structure below shows what exists today
and what is planned for Tier 1 implementation.

```text
README.md                    Entry point and status disclaimer
FOUNDATION_SCORECARD.md      Readiness rating across dimensions
PRD.md                       Product thesis, MVP scope, compliance tiers
docs/
  ARCHITECTURE.md            Protocol flow, crate map, layer ownership, security boundary
  CLAIMS_AND_NON_GOALS.md    What is true today vs. planned vs. explicitly excluded
  COMPETITIVE_LANDSCAPE.md   Competitor analysis and positioning
  REPO_MAP.md                This file
  ROADMAP.md                 Phase-by-phase delivery plan
  REQUIREMENTS_GOVERNANCE.md Functional, non-functional, security requirements
  SPEC_EVOLUTION.md          Living spec discipline and decision recording
  STANDARDS_AND_COMPLIANCE.md  RFC, NIST, and control mapping
  TECH_STACK.md              Crate workspace, layer diagram, runtime flow, token shapes
  USE_CASES.md               Use-case index and priority order
  WORKSPACE_BOUNDARIES.md    Crate dependency rules and import purity
  use-cases/                 Detailed scenario docs per use case
  v2/
    architecture.md          AS architecture detail (planned)
    requirements/            Endpoint-level BCP14 requirement docs (started)
    conformance/             Conformance ledger templates and closed ledgers
```

## Planned Top-Level Tree (Tier 1 Implementation)

```text
README.md
SECURITY.md
CHANGELOG.md
CONTRIBUTING.md
Cargo.toml                   Rust workspace manifest (planned)
Cargo.lock
crates/
  sts-as-http/               Axum transport layer (planned)
  sts-as-core/               AS domain: grants, sessions, consent, PKCE (planned)
  sts-as-store/              Durable state: users, clients, sessions, codes (planned)
  sts-as-config/             TOML config resolution and startup validation (planned)
  sts-as-cli/                Operator CLI: bootstrap, local dev, admin ops (planned)
  --- shared with sts-delegate-rs ---
  sts-jose/                  JOSE/JWK/JWT/ML-DSA-65 (shared)
  sts-dpop/                  RFC 9449 DPoP proof (shared)
  sts-replay/                Replay detection: JTI + codes + nonces (shared, extended)
  sts-exchange/              RFC 8693 token exchange (shared, extracted)
  sts-verify/                Issuer / trust-anchor / JWKS fetch (shared)
docs/
  (as above, plus tier-specific additions)
tests/
  integration/               End-to-end PKCE + TOTP + token flow tests
  conformance/               Negative-path conformance harness
```

## Directory Ownership

| Path | Purpose | Owner / Notes |
|---|---|---|
| `README.md` | Entry point; status disclaimer; quickstart pointer | Short, honest about planning-phase status |
| `SECURITY.md` | Security contact and vulnerability reporting | Security process and disclosure contact |
| `CHANGELOG.md` | Release notes | User-visible changes only; no internal planning notes |
| `CONTRIBUTING.md` | Contribution guide | Workflow, testing, review expectations, RFC citation rules |
| `Cargo.toml` | Workspace manifest | Rust crate membership and shared metadata |
| `crates/` | Rust workspace crates | Separated by responsibility, not feature marketing |
| `docs/` | Product spec, architecture, planning, and conformance docs | Living product specification |
| `tests/` | Integration and conformance test harness | Real HTTP against a running AS; negative paths required |

## Doc Ownership

| File | Owns |
|---|---|
| `PRD.md` | Product thesis, MVP scope, non-goals, compliance tiers, open decisions |
| `FOUNDATION_SCORECARD.md` | Readiness ratings across dimensions; gaps blocking 8/10 |
| `docs/ARCHITECTURE.md` | Protocol flow, crate map, layer ownership, security boundary, tier-to-endpoint table |
| `docs/CLAIMS_AND_NON_GOALS.md` | What is claimed today, what is not yet claimed, and what is explicitly excluded |
| `docs/COMPETITIVE_LANDSCAPE.md` | Competitor analysis, positioning, what not to claim |
| `docs/REPO_MAP.md` | This file; directory shape and file placement rules |
| `docs/ROADMAP.md` | Phase-by-phase delivery plan; milestones; current phase status |
| `docs/TECH_STACK.md` | Crate workspace, layered architecture diagram, runtime flow, token shapes, deployment shapes |
| `docs/SPEC_EVOLUTION.md` | Living spec discipline; how decisions are recorded and who updates what |
| `docs/REQUIREMENTS_GOVERNANCE.md` | Functional, non-functional, security, operational, and platform requirements |
| `docs/STANDARDS_AND_COMPLIANCE.md` | RFC, NIST, and control mapping per tier |
| `docs/WORKSPACE_BOUNDARIES.md` | Crate dependency direction; import purity rules; what shared crates may not import |
| `docs/USE_CASES.md` | Use-case index with priority order |
| `docs/use-cases/*.md` | Detailed scenario docs for each use case |
| `docs/v2/architecture.md` | AS architecture detail; crate boundary decisions; ADRs |
| `docs/v2/requirements/` | Endpoint-level BCP14 requirement docs (one file per endpoint) |
| `docs/v2/conformance/` | Conformance ledger templates and closed rows per RFC |

## Where New Files Go

| If you are adding... | Put it in... |
|---|---|
| A product thesis or MVP scope change | `PRD.md` |
| A new claim (shipped or planned) | `docs/CLAIMS_AND_NON_GOALS.md` |
| A new architecture diagram or layer boundary | `docs/ARCHITECTURE.md` or `docs/TECH_STACK.md` |
| A new protocol or security requirement | `docs/REQUIREMENTS_GOVERNANCE.md` |
| A new use case | `docs/USE_CASES.md` plus a file under `docs/use-cases/` |
| A new standards mapping | `docs/STANDARDS_AND_COMPLIANCE.md` |
| A new crate boundary or import purity rule | `docs/WORKSPACE_BOUNDARIES.md` |
| A planning decision or open question | `PRD.md` open decisions table + a GitHub issue |
| An endpoint-level BCP14 ledger | `docs/v2/requirements/<endpoint>.md` |
| A conformance ledger | `docs/v2/conformance/<rfc-number>.md` |
| A new AS-specific crate | `crates/sts-as-<name>/` |
| A shared crate (shared with sts-delegate-rs) | discuss in the `sts-delegate-rs` repo first; then add under `crates/` here |
| End-to-end integration tests | `tests/integration/` |
| Negative-path conformance tests | `tests/conformance/` |

Do not add new top-level directories without updating this file and the living spec first.

## Cross-Repo References

| Repo | Relationship | What this repo takes from it |
|---|---|---|
| `sts-delegate-rs` (sibling) | Tier 0 STS; ships the existing RFC 8693 token exchange | `sts-jose`, `sts-dpop`, `sts-replay`, `sts-exchange`, `sts-verify` are planned as shared crates extracted from this workspace |
| `obo-lab` (sibling) | Reference consumer of `sts-delegate-rs`; real Okta PKCE login + MCP OBO flow | Primary downstream integration target; QuAuthz AS should be droppable in place of Okta as the upstream IdP in the obo-lab flow |
| `sts-delegate` (Python, sibling) | Original Python STS reference implementation | Protocol reference only; not a build dependency; used for cross-checking RFC 8693 claim handling and error shapes |

Import rule: this repo does not import from `obo-lab` or `sts-delegate` at build
time. Protocol and conformance reference is by reading, not by dependency. The lab
imports the library; the library does not import from the lab.

## Current File Plan

Keep the repo shallow and obvious through the planning phase:

- product and architecture decisions stay in `docs/`;
- endpoint-level requirements and conformance ledgers stay under `docs/v2/`;
- implementation crates go under `crates/` when Tier 1 starts;
- integration and conformance tests go under `tests/`;
- no generated docs, build artifacts, or vendor directories committed to the root.
