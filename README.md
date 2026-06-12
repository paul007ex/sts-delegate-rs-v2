# sts-delegate-rs-v2

Planning repository for the Authorization Server v2 track.

This repo is the PRD, roadmap, architecture, and issue-planning lane for a
future OAuth/OIDC Authorization Server built beside `sts-delegate-rs`.
Implementation work should not start here until the PRD and architecture
boundaries are clear enough to avoid turning the current STS into a mixed
Authorization Server.

## Product Direction

Working product name: `sts-authority`.

Target product promise:

> A local-first, passwordless OAuth/OIDC Authorization Server with native STS
> delegation, MCP/OBO awareness, DPoP, and PQC-ready token signing.

The current `sts-delegate-rs` product remains an STS/token-exchange server. This
v2 lane is where we plan the broader Authorization Server product before moving
code.

## Normative Role Language

Use OAuth/OIDC role names first, examples second:

| Normative role | Product examples |
| --- | --- |
| Resource Owner / End-User | Human user, us |
| User Agent | Browser, Apple system browser, desktop browser |
| Client / Relying Party | MCP client, Apple native app, web app, CLI |
| Authorization Server / OpenID Provider | `sts-authority` |
| Resource Server / Protected Resource | MCP server, API, Databricks MCP, ServiceNow MCP |

Do not use deployment examples as if they were protocol roles.

## Planning Docs

- [PRD](docs/v2/prd.md)
- [Roadmap](docs/v2/roadmap.md)
- [Architecture](docs/v2/architecture.md)
- [OIDC endpoints](docs/v2/oidc-endpoints.md)
- [Conformance tiers](docs/v2/conformance-tiers.md)
- [OIDC Authorization Code + PKCE requirements](docs/v2/requirements/oidc-auth-code-pkce.md)
- [OIDC requirements ledger](docs/v2/requirements/index.md)
- [Reference curl pack](docs/v2/requirements/reference-curls.md)
- [Issue migration index](docs/v2/issues/source-issue-index.md)

## Source Backlog

The initial v2 backlog was copied from
[`paul007ex/sts-delegate-rs`](https://github.com/paul007ex/sts-delegate-rs)
issues labeled `as-2.0`. The source issues are intentionally left in place until
we decide whether to transfer, close as migrated, or keep cross-repo tracking.
