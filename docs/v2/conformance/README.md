# Conformance Evidence

This directory holds durable conformance-planning artifacts for the future
Authorization Server / OpenID Provider product.

These files are planning and proof-control artifacts. They do not mean the
current `sts-delegate-rs` STS is a full OAuth Authorization Server. Current STS
behavior remains limited to the token-exchange product unless a row explicitly
names implemented current-product evidence.

## Artifacts

- [OAuth 2.1 draft BCP14 ledger](oauth21-draft15-bcp14-ledger.md) maps
  `draft-ietf-oauth-v2-1-15` normative language to current STS status, AS v2
  work, and proof shapes.

## Rules

- Do not link GitHub issues to `/tmp` as canonical evidence.
- Check in durable ledgers here, or copy concise safe summaries into issue
  comments.
- Keep raw local evidence, bearer tokens, private keys, and unredacted HTTP logs
  out of the repo.
- Label OAuth 2.1 as draft-tracked until it is an RFC.
