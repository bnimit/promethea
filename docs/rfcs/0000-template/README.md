# RFC NNNN: <Title>

- **Status:** Draft / In comment period / Accepted / Rejected / Deferred / Postponed
- **Author(s):** Name <email or @handle>
- **Subsystem(s):** e.g., `services/privacy/`, cross-cutting
- **Pillars affected:** EU / India / Retail / Embedded / cross-cutting
- **Decided by:** subsystem lead name (or Board)
- **Comment period:** YYYY-MM-DD → YYYY-MM-DD
- **Links:** spec section, related RFCs, prior discussion

## Summary

One paragraph. What is this RFC proposing, and why now?

## Motivation

Why are we doing this? What's the problem? Who is affected? What evidence do we have that this matters? Cite issues, user reports, regulatory drivers, or research where relevant.

## Detailed design

The substance. This section should be detailed enough that another engineer could implement it without further consultation.

Cover, where applicable:
- Public API surface (Rust trait signatures, IPC schemas, on-disk formats)
- Data flow
- Error modes
- Security considerations (threat model, attack surface added or removed)
- Privacy considerations (data collected, retained, transmitted)
- Performance considerations
- Compatibility (forward, backward, with downstream variants)
- Telemetry posture (must be no-op or OHTTP-bucketed by default)

## Variant matrix

How does this proposal interact with each pillar?

| Pillar | Effect |
| --- | --- |
| EU sovereignty | |
| India sovereignty | |
| Retail / fleet | |
| Embedded / IoT | |

## Regional Capability Layer interactions

If this RFC introduces or modifies an `IdentityProvider`, `PaymentService`, `LanguageService`, or any other regionally-pluggable trait, document the trait change here. Include the migration path for existing regional implementations.

## Drawbacks

What are the costs of accepting this proposal? Engineering cost, complexity cost, attack-surface increase, vendor-lock-in risk, regulatory risk?

## Alternatives considered

What other designs were considered, and why is this the right one? "Do nothing" is always an option to be evaluated.

## Prior art

What have other projects done? AOSP, GrapheneOS, /e/OS, postmarketOS, Fuchsia, seL4 ecosystem, Apple, anything relevant. Cite specifics.

## Unresolved questions

What's open that this RFC doesn't yet answer? These become tracking issues post-acceptance.

## Future work

What does this enable that comes after?

## Implementation plan

A rough decomposition into PRs. The RFC author is not required to implement, but should sketch what the implementation entails.

- [ ] PR 1: ...
- [ ] PR 2: ...
- [ ] PR 3: ...

## Testing strategy

How will this be tested? Unit, integration, property, fuzz, formal, hardware-in-loop?

## Disposition

To be filled in by the decider when the comment period closes:

- **Decision:** Accepted / Rejected / Deferred / Postponed
- **Decided by:** name, role
- **Date:** YYYY-MM-DD
- **Rationale:** brief explanation (longer rationales go below)
