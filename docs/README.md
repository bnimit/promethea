# Documentation index

Welcome to the Promethea documentation tree. This directory contains the design specs, research dossier, RFCs, threat models, architecture documents, and per-subsystem roadmaps.

## Where to start

If you are new to the project, read in this order:

1. The top-level [`README.md`](../README.md) — what Promethea is.
2. [`docs/superpowers/specs/2026-04-26-promethea-vision-design.md`](superpowers/specs/2026-04-26-promethea-vision-design.md) — the program-level design.
3. [`docs/superpowers/specs/2026-04-26-privacy-framework-design.md`](superpowers/specs/2026-04-26-privacy-framework-design.md) — the focused first-component spec.
4. [`docs/research/00-executive-summary.md`](research/00-executive-summary.md) — the evidence base behind every architectural decision.
5. [`GOVERNANCE.md`](../GOVERNANCE.md) — how decisions get made.

## Directory map

| Path | Contents |
| --- | --- |
| [`docs/superpowers/specs/`](superpowers/specs/) | Design specs (vision doc, privacy framework, future component specs as Phase 0 begins) |
| [`docs/research/`](research/) | Synthesized research dossier — eight streams plus an executive summary |
| [`docs/rfcs/`](rfcs/) | Numbered Requests for Comments — substantial proposals; the [template](rfcs/0000-template/) lives at `0000-template/` |
| [`docs/architecture/`](architecture/) | Subsystem design documents (populated as subsystems land) |
| [`docs/threat-models/`](threat-models/) | Per-pillar and per-subsystem threat models (populated as subsystems land) |
| [`docs/roadmaps/`](roadmaps/) | Durable per-subsystem backlog files; deferred items from each spec live here so they survive past brainstorm |

## What goes where

- **A spec** describes a designed system. Specs are dated and snapshotted; they are not maintained as living docs. New design decisions land in new specs that supersede old ones, with a clear pointer.
- **An RFC** proposes a substantial change. RFCs are the unit of decision-making per [`GOVERNANCE.md § RFC process`](../GOVERNANCE.md#rfc-process). Every accepted RFC has a recorded disposition.
- **A research file** captures the empirical basis of a decision at a point in time. Research files are dated snapshots, not living docs.
- **An architecture doc** is the living description of a subsystem's current design. Updated as the subsystem evolves.
- **A threat model** is a living description of the adversary, attack surface, and mitigations for a subsystem or pillar.
- **A roadmap file** is the durable backlog for items deferred from a subsystem's spec. New deferred items land here whenever a spec is updated to push something out a version.

## Status of this tree

| Document | Status |
| --- | --- |
| Vision design spec | Draft, under review |
| Privacy framework spec | Draft, under review |
| Privacy framework roadmap | Live |
| Research dossier (8 streams + summary) | Snapshot, dated 2026-04-26 |
| RFC 0000 template | Live |
| Other RFCs | None yet |
| Architecture docs | Empty until Phase 0 begins |
| Threat models | Empty until subsystems land |

## Conventions

- **File naming:** specs are dated `YYYY-MM-DD-<short-name>-design.md`. Research files are numbered `NN-<topic>.md`. RFCs are numbered `NNNN-<short-name>/` (directories so multi-file RFCs work).
- **Markdown:** GitHub-flavored Markdown. ASCII-art diagrams are fine for now; we will adopt a richer diagram format (PlantUML or Mermaid) when Phase 0 makes a decision.
- **Citations:** URL only; we do not mirror sources locally. For load-bearing claims, re-verify before acting.
- **Language:** US English in technical docs, regardless of regional pillar. Localized end-user docs are a separate concern handled per regional manifest.
