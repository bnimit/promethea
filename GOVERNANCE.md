# Governance

Promethea is governed by the **Promethea OS Foundation** (the *Foundation*), a non-profit entity to be incorporated in the European Union. The Foundation owns the trademark, the signing keys, the public infrastructure, and the project's continuity commitments. A separate **commercial integrator entity** sells certified downstream variants, paid support, and managed services; the commercial entity has no ability to fork the trademark and depends on the Foundation for the upstream codebase.

This document is the source of truth for how decisions get made. It is itself amendable only via the RFC process described below.

## Principles

1. **Open by default.** All design happens in public. Private discussion is an exception, justified per case.
2. **No single point of failure.** Bus factor ≥ 5 across paid core engineers; ≥ 3 across signing-key holders; multi-organization representation enforced on security-critical reviews.
3. **Reproducible and verifiable.** Every release is reproducibly built, signed, and published to a public transparency log (Sigstore + sigsum).
4. **Foundation owns continuity; vendors own variants.** No vendor or commercial entity can fork the trademark, take the signing keys hostage, or unilaterally override an RFC outcome.
5. **Multi-jurisdiction by design.** Foundation board composition, signing-key custody, and infrastructure are deliberately distributed across at least two jurisdictions (EU and India at minimum) to prevent any single legal regime from acting as a kill switch.

## Board

The Foundation Board is the highest decision-making body. Composition:

| Seat | Held by | How filled |
| --- | --- | --- |
| Founders' seat × 2 | Original founders | Held while active; revert to community-elected |
| EU public-sector seat × 1 | EU government, regulator, or sovereign-tech body | Appointed by EU member-state agency consortium |
| Indian academic / govt seat × 1 | Indian academic institution or MeitY-recommended body | Appointed by Indian academic / govt consortium |
| Community-elected × 3 | Active project contributors | Elected annually by maintainers |
| Commercial integrator × 1 (non-voting) | Commercial entity CEO | Appointed by commercial entity |

Board meetings: minutes published; agenda public 14 days in advance; community comment period before binding votes.

## Roles

- **Maintainers** — committers with merge access to one or more subtrees. Listed in [MAINTAINERS.md](MAINTAINERS.md). Promoted by existing maintainers via consensus.
- **Subsystem leads** — accountable owners for top-level subtrees (`services/privacy/`, `kernel/`, `compositor/`, etc.). Named in [CODEOWNERS](CODEOWNERS).
- **Release managers** — one per release line; sign release artifacts; rotate quarterly.
- **Security response (PSIRT)** — handles disclosures per [SECURITY.md](SECURITY.md). Roster of ≥ 3 humans across ≥ 2 organizations.
- **Code of Conduct committee** — handles enforcement per [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md). Named, ≥ 3 humans, ≥ 2 organizations, ≥ 1 woman or non-binary member.
- **Contributors** — anyone submitting a DCO-signed pull request.

## RFC process

All cross-subsystem changes, license-touching changes, governance changes, and Regional Capability Layer changes require an RFC.

1. Author a numbered RFC document under `docs/rfcs/NNNN-<short-name>/` based on the template at `docs/rfcs/0000-template/`.
2. Open a pull request titled `[RFC NNNN] <name>`.
3. **Comment period:** ≥ 14 calendar days for normal RFCs; ≥ 30 for governance, license, or RCL-affecting RFCs.
4. **Disposition:** the relevant subsystem lead (or the Board for cross-cutting RFCs) records a decision: *Accepted*, *Rejected*, *Deferred*, *Postponed*.
5. **Implementation:** accepted RFCs are tracked as GitHub issues; the RFC document is updated when implementation lands or diverges.

RFCs may not be merged silently. Every accepted RFC has a recorded disposition with a named decider and rationale.

## Merge policy

| Path | Required reviewers |
| --- | --- |
| `services/privacy/`, `services/ota/`, `services/push/`, `kernel/`, `kernel-modules/`, `ipc/` | 2 of 3 reviewers, from at least 2 distinct organizations |
| Other code paths | 1 maintainer, 1 contributor (or 2 maintainers) |
| `docs/rfcs/`, `GOVERNANCE.md`, `LICENSE-*`, `MAINTAINERS.md`, `CODEOWNERS` | RFC required; Board confirmation for governance and license |

All commits must be DCO-signed. All commits to `main` must be GPG-signed.

## License policy

Platform code is licensed under the **Apache License 2.0**. The patent grant and patent-retaliation provisions are load-bearing for an OS project that touches hardware, codecs, modems, secure elements, and payment HALs. Linux kernel modules under `kernel-modules/` are GPL-2.0 where the kernel ABI requires it; this subtree is firewalled and may not import Apache-licensed platform code in a way that would create a license conflict. License changes require a license-affecting RFC and Board confirmation. The Foundation does not require copyright assignment; DCO sign-off only. See [`LICENSE`](LICENSE) for the full Apache 2.0 text and [`NOTICE`](NOTICE) for attribution requirements.

## Bus-factor commitment

The Foundation publishes quarterly:
- Number of paid core engineers across subtrees.
- Number and identity (org affiliation) of release-key holders.
- Number and identity of PSIRT roster.
- Largest single-org concentration in any subsystem.

Targets: paid core ≥ 5; release-key holders ≥ 3 across ≥ 2 organizations; PSIRT roster ≥ 3 across ≥ 2 organizations; no single organization holds > 50% of any subsystem's commits over any rolling 6-month window.

If any target is missed, the Board must publish a remediation plan within 30 days.

## Commercial entity

A separate commercial entity (legal form: GmbH / SAS / Pvt-Ltd, jurisdiction TBD) sells certified variants, paid support, and managed services. It is bound by:
- A trademark license from the Foundation, terminable for cause.
- A non-compete on the upstream codebase: the entity may not fork the trademark or build a closed-source competing platform.
- Disclosure of all paid contributors active in the upstream as a condition of trademark license.

The commercial entity holds a non-voting Board seat and otherwise participates in the project as any commercial contributor would.

## Amendments

This document is amended via RFC, ≥ 30-day comment period, and Board confirmation by ≥ 2/3 of voting members.
