<!--
Thank you for contributing to Promethea. Please fill in this template so
reviewers can act on your PR efficiently.
-->

## Summary

<!-- One paragraph: what does this PR do, and why? -->

## Linked spec / RFC / issue

<!--
For non-trivial changes, link to the relevant RFC, design spec, or issue.
For changes affecting the Regional Capability Layer, identity/payment/language
traits, or any security-critical subtree, an RFC is required (see
GOVERNANCE.md).
-->

- RFC: <!-- RFC NNNN, or "n/a" -->
- Spec section: <!-- e.g., docs/superpowers/specs/2026-04-26-privacy-framework-design.md § Capability Broker, or "n/a" -->
- Issue: <!-- #NNN, or "n/a" -->

## Variant matrix

<!-- Which variants does this change affect? Mark all that apply. -->

- [ ] Region: EU
- [ ] Region: India
- [ ] Region: Global / region-agnostic
- [ ] Vertical: Consumer
- [ ] Vertical: Retail / fleet
- [ ] Vertical: Embedded
- [ ] Vertical: Defense / sovereign
- [ ] Architecture: aarch64
- [ ] Architecture: x86_64
- [ ] Architecture: riscv64

## Type of change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation only
- [ ] Refactor (no behavior change)
- [ ] Build / CI / tooling

## Testing

<!-- How was this change tested? Reviewers will ask if this section is empty. -->

- [ ] Unit tests added / updated
- [ ] Integration tests added / updated
- [ ] Property tests added / updated (if state machine or policy code)
- [ ] Fuzz harness added / updated (if untrusted-input parser)
- [ ] Manual testing on real hardware: <!-- which device, which variant -->

## Security checklist

<!--
Mandatory if this PR touches services/privacy/, services/ota/, services/push/,
services/telemetry/, services/compliance/, kernel/, kernel-modules/, ipc/,
or init/. Otherwise leave blank or write "n/a".
-->

- [ ] Threat model considered; no new attack surface introduced unintentionally
- [ ] No new `unsafe` blocks without justification comment
- [ ] No new dependencies on un-audited crates
- [ ] No telemetry / network egress added on the host side
- [ ] No regressions to capability-handle invariants

## Compliance / regulatory impact

<!--
Mark "n/a" if none. Otherwise describe.
-->

- [ ] EU CRA implications (SBOM, signed updates, vulnerability reporting)
- [ ] GDPR implications (data subject rights, lawful basis)
- [ ] DPDP / CERT-In implications (consent, log retention, incident reporting)
- [ ] Certification scope changes (EUCC, BIS, FIPS, ANSSI Qualification)
- [ ] n/a

## DCO and signing

- [ ] All commits are DCO-signed (`git commit -s`)
- [ ] All commits are GPG-signed (`git commit -S`)

## Reviewer guidance

<!-- Anything reviewers should look at first? Tricky bits? Out-of-scope notes? -->
