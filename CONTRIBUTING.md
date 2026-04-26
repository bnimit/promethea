# Contributing to Promethea

Thank you for your interest in contributing. This document explains how to participate in the project — the practical rules and the architectural expectations.

> **The short version.** Fork, branch, write tests, sign your commits with DCO and GPG, open a pull request, address review. Cross-subsystem or governance-affecting changes go through the [RFC process](GOVERNANCE.md#rfc-process) first.

## Before you start

1. Read [GOVERNANCE.md](GOVERNANCE.md) — particularly the merge policy and the RFC process.
2. Read [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md). All participation is governed by it.
3. Read [SECURITY.md](SECURITY.md) before reporting a vulnerability — *do not file a public issue for security bugs*.
4. Read the [vision doc](docs/superpowers/specs/2026-04-26-promethea-vision-design.md) to understand what this project is and what it is not. We say no to scope expansions that do not fit the four audience pillars.

## Developer Certificate of Origin (DCO)

We use the [Developer Certificate of Origin 1.1](https://developercertificate.org). We do not require a contributor license agreement (CLA) and we do not take copyright assignment.

Every commit must be signed off:

```
git commit -s -m "your commit message"
```

This adds `Signed-off-by: Your Name <you@example.com>` to the commit message and certifies that you have the right to submit the contribution under the project's license.

## GPG-signed commits

All commits merged to `main` must also be GPG-signed:

```
git commit -S -s -m "your commit message"
```

Configure your signing key in `git config user.signingkey` and enable `commit.gpgsign true`. CI rejects unsigned commits at merge time.

## License of contributions

By contributing, you license your code to the project under the same terms as the surrounding subtree:

- Most of the codebase: **Apache License 2.0** (see [`LICENSE`](LICENSE)). Your contribution carries an implicit Apache 2.0 patent grant per Section 3 of the license.
- `kernel-modules/`: **GPL-2.0** only, as required by the Linux kernel module ABI.
- Vendored dependencies under `third_party/`: their own upstream license, declared in each subdirectory's `LICENSE` and surfaced in the SBOM.

A `LICENSE` file in any subdirectory takes precedence over the repository default. If you are uncertain whether your contribution can be licensed Apache 2.0 (e.g., you are contributing on behalf of an employer with an unclear IP policy), please ask before submitting.

## Choosing what to work on

- **Good first issues** are tagged `good first issue` in GitHub.
- **Roadmap items** for each subsystem live in `docs/roadmaps/<subsystem>.md`. Pick one and check the linked spec.
- **Non-goals from the active spec**, when their target version arrives, are tracked under `docs/roadmaps/`. These are the v0.2 / v0.3 backlog; pick from here once their phase is open.
- **Larger work** — anything cross-subsystem or that changes a public interface — needs an RFC first. See [GOVERNANCE.md § RFC process](GOVERNANCE.md#rfc-process).

## Branch and PR conventions

- Branch from `main`; one logical change per PR.
- PR title prefix: subsystem in square brackets, e.g. `[privacy] Add hardware attestation gate`.
- Description must reference the relevant RFC, issue, or spec section.
- Keep PRs small. PRs over 800 lines need an RFC if they're not pure scaffolding.
- Tests required for behavior changes. See subsystem-specific test docs under `docs/architecture/`.
- Update the relevant `CHANGELOG.md` entry under the subsystem.

## Review process

| Path | Required reviewers |
| --- | --- |
| Security-critical: `services/privacy/`, `services/ota/`, `services/push/`, `kernel/`, `kernel-modules/`, `ipc/` | 2 of 3, from ≥ 2 distinct organizations |
| Other code paths | 1 maintainer, 1 contributor (or 2 maintainers) |
| RFCs, governance, licenses | RFC process; Board confirmation where applicable |

Reviewers are listed in [CODEOWNERS](CODEOWNERS). Maintainers are listed in [MAINTAINERS.md](MAINTAINERS.md).

## Testing expectations

- Unit tests for every new module; coverage target ≥ 80% lines, ≥ 90% on policy and security-critical paths.
- Property tests (via `proptest`) for stateful policy code.
- Integration tests for any IPC surface change.
- Fuzz harnesses (`cargo-fuzz`) for any parser, serializer, or untrusted-input path.
- The CI matrix runs: Linux x86_64, Linux aarch64, the active region matrix (EU, IN, global), and the active vertical matrix (consumer, retail, embedded). All must pass.

Performance regressions: any change touching the hot path must include benchmark results before merge.

## Style and tooling

- `cargo fmt --check` and `cargo clippy --deny warnings` must pass.
- Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/). When in doubt, follow the conventions in `services/privacy/` (the reference subsystem for this project).
- Public APIs require doc comments with examples. Examples are tested via `cargo test --doc`.
- One commit, one logical change. Squash-merge is the default; rebases are preferred over merge commits.

## Communication

- **Async / canonical:** GitHub issues and pull requests.
- **Real-time:** the project chat (TBD; will be a Matrix homeroom hosted on Foundation infrastructure once Phase 0 lands).
- **Synchronous meetings:** subsystem syncs are public, recorded, and minuted. Board meetings are public per [GOVERNANCE.md](GOVERNANCE.md).

## Reporting bugs that aren't security issues

File a GitHub issue using the bug template. Include: subsystem, OS variant (region × vertical), reproduction steps, expected vs. observed, logs (with sensitive data redacted).

## Reporting security issues

**Do not file a public issue.** See [SECURITY.md](SECURITY.md). The PSIRT roster handles vulnerabilities under the disclosure flow described there, which is designed to satisfy the EU CRA 24-hour reporting requirement that takes effect 2026-09-11.

## Dual-jurisdiction considerations

The project's audience pillars include both EU and India. When contributing region-specific code under `regions/<code>/` or compliance daemons under `services/compliance/`, consult the local subsystem lead — regulations differ across jurisdictions and incorrect placement can have legal consequences for downstream variant maintainers. When in doubt, ask first.

## Thank you

The project depends on contributors at every scale, from typo fixes to subsystems. We mean that. Welcome.
