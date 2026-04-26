# Research dossier

This directory contains the synthesized research that informed the [Promethea vision design](../superpowers/specs/2026-04-26-promethea-vision-design.md) and the [Privacy & Permission Framework spec](../superpowers/specs/2026-04-26-privacy-framework-design.md). Each file summarizes one focused research stream conducted during the brainstorm phase on 2026-04-26.

These documents are *snapshots*, not living docs. They are dated and not maintained as the world changes — they exist so that anyone reading the design can see the evidence and reasoning behind each architectural decision. If you find a fact in here is now stale, the right action is to surface that in a new RFC or issue, not to update the snapshot.

## Files

| # | Stream | File |
| --- | --- | --- |
| 0 | Executive summary — top findings and decisions | [00-executive-summary.md](00-executive-summary.md) |
| 1 | Prior art on alternative / privacy mobile operating systems | [01-prior-art-mobile-os.md](01-prior-art-mobile-os.md) |
| 2 | Rust in OS kernels: Rust-for-Linux, Redox, seL4+Rust, microkernel landscape | [02-rust-kernel-landscape.md](02-rust-kernel-landscape.md) |
| 3 | Android-app compatibility (Waydroid, microG, Play Integrity) and push (UnifiedPush, ntfy) | [03-android-compat-and-push.md](03-android-compat-and-push.md) |
| 4 | Privacy and permission framework state of the art | [04-privacy-permission-framework.md](04-privacy-permission-framework.md) |
| 5 | EU regulatory landscape (DMA, CRA, EUCC, public-sector procurement, funding) | [05-eu-regulatory-landscape.md](05-eu-regulatory-landscape.md) |
| 6 | Old-Android revival, retail / fleet market, MDM compatibility | [06-old-android-revival-retail.md](06-old-android-revival-retail.md) |
| 7 | India sovereign mobile OS landscape (BharOS, IndiaStack, DPDP, market) | [07-india-sovereign-mobile-os.md](07-india-sovereign-mobile-os.md) |
| 8 | Rust vs Zig as the implementation language for this project | [08-rust-vs-zig.md](08-rust-vs-zig.md) |

## How this maps to the design

Each architectural decision in the vision doc cites the research stream that supports it. Highlights:

- **AOSP-class open base, not turnkey device** ← graveyard pattern in (1)
- **Linux + Rust-for-Linux production track, seL4+Rust R&D track** ← (2)
- **pKVM/crosvm Android-VM isolation, not LXC; UnifiedPush hybrid push fabric** ← (3)
- **Capability-handle primitive, ledger, lockdown profiles** ← (4)
- **DMA tailwind, CRA as unconditional baseline, EUCC as downstream variant concern** ← (5)
- **MDM compat shim and 7-10 year OTA SLA as retail wedge** ← (6)
- **Bhashini on-device, UPI Intent as system service, DigiLocker identity, India follows EU codebase milestones** ← (7)
- **Rust top-to-bottom; Zig is not a serious contender for *this* project** ← (8)

## Methodology and caveats

- Research was performed by parallel general-purpose research agents on 2026-04-26.
- Web search and web-fetch were the primary tools. One stream (06, retail) ran without live web access and explicitly flagged its findings as directional.
- Citations are URL-only; we did not have time to mirror sources locally. Link rot is a real risk; for any claim with a load-bearing role in a downstream decision, re-verify before acting.
- These summaries reflect the agents' synthesis, edited only for clarity and consistency. The agents were instructed to be opinionated. Their opinions are their own; the design's adopted positions are stated in the vision doc.
