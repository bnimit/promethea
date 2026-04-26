# Promethea

> A Rust-native, telemetry-free, regionally-customizable, AOSP-class open mobile and embedded operating system — the public base layer that EU OEMs, Indian OEMs, retail and fleet integrators, and embedded vendors all build downstream variants on, without depending on any single vendor's gatekeeping.

**Status:** vision-stage. No code yet. See [`docs/superpowers/specs/2026-04-26-promethea-vision-design.md`](docs/superpowers/specs/2026-04-26-promethea-vision-design.md) for the program-level design and [`docs/superpowers/specs/2026-04-26-privacy-framework-design.md`](docs/superpowers/specs/2026-04-26-privacy-framework-design.md) for the focused first-component spec.

## Why

Today's mobile OS landscape has two gatekeepers (Google, Apple) headquartered in a single jurisdiction, with telemetry, identity, payment, push, and app distribution all routed through their infrastructure. EU and Indian regulators have begun legislating against this lock-in (DMA, DPDP, CRA, Make-in-India procurement preference), but no credible Rust-native, jurisdictionally-independent mobile stack exists to fill the gap. Privacy-first projects (GrapheneOS, /e/OS, postmarketOS) do invaluable work but inherit AOSP's architecture and Google's centre of gravity. **Promethea is not another AOSP fork. It is a from-the-ground-up Rust open-base alternative — designed so that the regulatory tailwind has somewhere to land.**

## What

- **Implementation language:** Rust top-to-bottom for all first-party code.
- **Production kernel:** Linux + Rust-for-Linux (Linux 7.0 ships Rust as a peer language).
- **R&D kernel:** seL4 + Rust userspace (KataOS architectural template, re-derived) — embedded-first, mobile-later.
- **Android-app compatibility:** isolated AOSP runtime in a pKVM/crosvm VM (not LXC), with microG sidecar and a sandboxed Google Play option for users who need banking apps.
- **Push:** UnifiedPush + ntfy + an FCM-bridge distributor inside the Android VM. The host never speaks FCM.
- **Privacy primitive:** capability handles instead of ambient permissions — a typed file descriptor handed out by a system-trusted broker, ledger-logged, manifest-bound, network-egress-constrained.
- **Regional Capability Layer (RCL):** EU, India, retail, and embedded are first-class variants of one codebase, not separate forks. Identity providers, payment services, language services, and compliance daemons are pluggable per region.

## Audience pillars

| Pillar | Reference SKU horizon |
| --- | --- |
| EU sovereignty + privacy prosumer | Codebase v1.0 ~M18; downstream variants drive certification timeline |
| India sovereignty + linguistic-first | Procurement pilot ~M24; consumer SKU ~M30+ |
| Retail / fleet revival of old Android | Demo SKU ~M9 (Pixel 3a/4a); fleet SKU ~M18 (Zebra TC52) |
| Embedded / IoT (long-horizon) | Microkernel preview ~M36; production embedded ~M60+ |

## Non-goals

- Not a Pixel-class flagship competitor (GrapheneOS owns that wedge; we partner, not compete).
- Not a Linux desktop OS.
- Not a from-scratch microkernel as the production track — the microkernel is R&D, not v1.0.
- Not a closed or proprietary product. Platform code is Apache 2.0; GPL-2.0 isolated to the `kernel-modules/` subtree only.

## License

Platform code is licensed under the [Apache License 2.0](LICENSE). The patent grant and patent-retaliation provisions matter for an OS project, which is why we picked Apache 2.0 over MIT. Linux kernel modules under `kernel-modules/` are GPL-2.0 where the kernel ABI requires it. See individual subtree `LICENSE` files for exceptions and [`NOTICE`](NOTICE) for attribution requirements.

## Documents

- [GOVERNANCE.md](GOVERNANCE.md) — foundation structure, decision-making, RFC process
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to contribute, DCO, signed commits, review requirements
- [SECURITY.md](SECURITY.md) — PSIRT contact, disclosure flow, supported versions
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
- [MAINTAINERS.md](MAINTAINERS.md)
- [`docs/`](docs/) — architecture, threat models, RFCs, research, roadmaps

## Getting started

This repository currently contains design documents and a directory skeleton. Code lands in Phase 0 (M0–M3 of the roadmap). See the [vision-doc roadmap](docs/superpowers/specs/2026-04-26-promethea-vision-design.md#roadmap) for what's expected when.

To browse the design: start at the [vision doc](docs/superpowers/specs/2026-04-26-promethea-vision-design.md), then read the [privacy framework spec](docs/superpowers/specs/2026-04-26-privacy-framework-design.md), then dip into the [research dossier](docs/research/).
