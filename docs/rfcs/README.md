# Requests for Comments (RFCs)

Substantial proposals — anything cross-subsystem, anything modifying a public API, anything affecting the Regional Capability Layer, governance, or licensing — go through the RFC process described in [`GOVERNANCE.md § RFC process`](../../GOVERNANCE.md#rfc-process).

## Index

| # | Title | Status | Author | Subsystem |
| --- | --- | --- | --- | --- |
| 0000 | RFC template (not a real RFC) | template | — | — |

## Template

Start a new RFC by copying [`0000-template/`](0000-template/) into a numbered directory: `NNNN-<short-name>/`. Fill in the README, then open a pull request titled `[RFC NNNN] <name>`.

## Status values

- **Draft** — author still iterating; not yet open for community comment.
- **In comment period** — open PR, comments solicited; ≥ 14 days for normal RFCs, ≥ 30 days for governance / license / RCL-affecting.
- **Accepted** — merged with a recorded disposition; implementation tracked as an issue.
- **Rejected** — not adopted; rationale recorded.
- **Deferred** — paused; revisit conditions recorded.
- **Postponed** — accepted in principle but blocked by other work.

Every RFC ends in one of those states. None merges silently.
