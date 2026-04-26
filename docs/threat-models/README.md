# Threat models

Per-pillar and per-subsystem threat models. Living docs.

The vision doc establishes that the threat model is *layered*, not single:

- Nation-state adversary — for the high-assurance EU defense / sovereign downstream variant.
- Corporate surveillance — for consumer EU + India.
- Supply-chain integrity — for retail / fleet.
- Physical tamper — for embedded.

The Privacy & Permission Framework's lockdown-profile mechanism is what allows one codebase to serve all four threat-model layers; specific protections are configured via regional manifest and active lockdown profile.

## Conventions

- Each threat model is a single Markdown file named for its scope: `pillar-<name>.md` or `subsystem-<name>.md`.
- Each file follows STRIDE or LINDDUN structure (project decision pending; default to STRIDE for engineering audiences and LINDDUN where privacy harms are the primary concern).
- Updates to a threat model trigger a security review per [`SECURITY.md`](../../SECURITY.md).

## Index

*(empty until subsystems land — first expected file is `subsystem-privacy-daemon.md` in Phase 1)*
