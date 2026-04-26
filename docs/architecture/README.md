# Architecture documents

Living descriptions of each subsystem's current design. Populated as subsystems land in code (Phase 0 onward).

## Conventions

- One file per subsystem, named `<subsystem>.md` (e.g. `privacy-daemon.md`, `compositor.md`).
- Each file is *living* — it is updated as the subsystem evolves. It is not a dated snapshot.
- Each file links back to the relevant spec under `../superpowers/specs/` and to any RFCs that have shaped the subsystem.
- Substantial design changes go through the RFC process; the architecture doc is updated when the RFC merges.

## Index

*(empty until Phase 0 begins)*

## Template

When you start an architecture doc, use this skeleton:

```markdown
# Subsystem: <name>

| | |
| --- | --- |
| **Path** | `services/<name>/` |
| **Lead** | @handle |
| **Spec** | [link to spec] |
| **Threat model** | [link to threat model] |
| **Status** | alpha / beta / stable |

## Purpose

What this subsystem does in one paragraph.

## Components

A box-diagram or list of internal components, with brief description of each.

## Public interface

The IPC surface, traits, or other public APIs other subsystems consume.

## Internal invariants

What must always be true of the subsystem's state.

## Failure modes

What happens when things go wrong, and how the subsystem fails closed.

## Dependencies

Which other subsystems this one depends on, and how.

## Open questions

Live design questions that don't yet have answers.
```
