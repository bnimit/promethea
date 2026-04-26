# Promethea — Program-Level Vision Design

| | |
| --- | --- |
| **Status** | Draft, under review |
| **Date** | 2026-04-26 |
| **Author** | brainstorming session, project sponsor `nimitbhandari17@gmail.com` |
| **Type** | Vision / program-level design (not an implementation spec) |
| **Companion** | [Privacy & Permission Framework spec](2026-04-26-privacy-framework-design.md) — the focused first-component spec |
| **Research basis** | [`docs/research/`](../../research/) — eight synthesized streams |

## Executive summary

Promethea is an AOSP-class open Rust mobile and embedded operating system. It exists to give the regulatory tailwind currently building against US-headquartered mobile-OS gatekeepers (DMA in Europe, DPDP and Make-in-India procurement preference in India) somewhere credible to land. The product is a Rust-native, telemetry-free, regionally-customizable codebase that EU OEMs, Indian OEMs, retail and fleet integrators, and embedded vendors can all build downstream variants on. Four audience pillars — EU sovereignty, India sovereignty, retail / fleet revival, and embedded / IoT — are co-equal in the codebase from day one via a Regional Capability Layer (RCL) primitive that pluggably binds identity, payment, language, search, maps, and compliance services to a region. The implementation is Rust top-to-bottom; the production kernel is Linux + Rust-for-Linux; a parallel R&D track develops a Rust microkernel atop seL4 for embedded-first transition. Android-app compatibility is preserved via a pKVM/crosvm-isolated AOSP runtime with a microG sidecar. Push notifications never route through FCM from the host. Governance is foundation-led with a separate commercial integrator entity, ≥ 5 funded core engineers from day one, and explicit bus-factor reporting.

## 1. Problem statement

The mobile-OS market has two gatekeepers (Google, Apple). Both are US-headquartered. Both route telemetry, identity, payment, push, and app distribution through their infrastructure. EU and Indian regulators are actively legislating against this lock-in, but no credible Rust-native, jurisdictionally-independent mobile stack exists to fill the gap. Privacy-first projects do invaluable work but inherit AOSP's architecture and Google's center of gravity; sovereign-OS attempts (BharOS, Sailfish, EU OS) have stalled at hundreds of units. The opportunity in 2026 is to build the open base layer the next decade of mobile sovereignty needs, before incumbents consolidate further. See [`docs/research/05-eu-regulatory-landscape.md`](../../research/05-eu-regulatory-landscape.md) and [`docs/research/07-india-sovereign-mobile-os.md`](../../research/07-india-sovereign-mobile-os.md).

## 2. Positioning

**Strategic framing: AOSP, not Pixel.** The product is the *codebase, governance, and reference variants* — not a turnkey device. We compete with the Google-controlled AOSP + GMS + Play + FCM stack, not with GrapheneOS, /e/OS, or postmarketOS — those are collaborators and downstream beneficiaries.

### One-sentence pitch

A Rust-native, telemetry-free, regionally-customizable, AOSP-class open mobile and embedded operating system — the public base layer that EU OEMs, Indian OEMs, retail and fleet integrators, and embedded vendors all build downstream variants on, without depending on any single vendor's gatekeeping.

### Non-goals

- **Not a Pixel-class flagship competitor.** GrapheneOS owns that wedge; we partner, not compete.
- **Not a Linux desktop OS.** Mobile and embedded only.
- **Not a from-scratch microkernel as the production track.** The microkernel is R&D, not v1.0.
- **Not a closed or proprietary product.** Platform code is Apache-2.0; GPL-2.0 isolated to the `kernel-modules/` subtree only.
- **Not certification-led.** Per the AOSP-class reframe, certifications (EUCC, BIS, FIPS) happen in downstream variants on their own timelines; the upstream codebase is *certification-ready*, not *certification-gated*.

## 3. Audience pillars

Four pillars, co-equal in the codebase, staggered in commercial time-to-market.

| Pillar | Buyer | Hardware tier | Differentiator | Reference SKU horizon |
| --- | --- | --- | --- | --- |
| **EU sovereignty / privacy** | EU public sector and privacy prosumers | Fairphone-class, ~€500-800 | EUCC-track downstream variant; eIDAS 2.0 wallet; GDPR-by-construction | Codebase v1.0 ~M18; downstream variants drive certification timeline |
| **India sovereignty + linguistic** | Indian public sector, then consumer | Lava / Karbonn-class Dimensity, ₹18-25K | UPI Intent as system service; Bhashini on-device; DigiLocker / Aadhaar identity provider; CERT-In compliance daemon | Procurement pilot ~M24; consumer SKU ~M30+ |
| **Retail / fleet revival** | EU+India retailers, 3PLs, healthcare, hospitality | Pixel 3a/4a/OnePlus 6 (demo); Zebra TC52 / Honeywell CT40 (prize) | Android Management API shim; certified peripheral stack; 7-10 year OTA SLA | Demo SKU ~M9 on Pixel 3a; fleet SKU ~M18 on revived rugged HW |
| **Embedded / IoT (long-horizon)** | Industrial, automotive, smart-home | ARM Cortex-A class boards, RISC-V dev kits | Rust microkernel R&D track (seL4+Rust); deterministic, formally-verifiable subsets | Microkernel preview ~M36; production embedded ~M60+ |

### Threat model is layered, not single

- Nation-state adversary — for the high-assurance EU defense / sovereign downstream variant.
- Corporate surveillance — for consumer EU + India.
- Supply-chain integrity — for retail / fleet.
- Physical tamper — for embedded.

The Privacy & Permission Framework's lockdown-profile mechanism is what allows one codebase to serve all four. See [`docs/superpowers/specs/2026-04-26-privacy-framework-design.md`](2026-04-26-privacy-framework-design.md).

## 4. Technical architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Apps Layer                                                 │
│  • Native Rust apps (system + 3rd-party)                    │
│  • Android apps (in isolated AOSP runtime, opt-in)          │
│  • PWAs / web apps (WebView based on Servo or upstream WK)  │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  App Framework & Platform Services                          │
│  • Rust app SDK + UI toolkit (deferred — bake-off in P0)    │
│  • App Store client + reproducible-build signing            │
│  • IdentityProvider trait → eIDAS / Aadhaar+DigiLocker /…   │
│  • PaymentService trait    → UPI Intent / SEPA / …          │
│  • LanguageService trait   → on-device Bhashini / FOSS base │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  System Services (all Rust userspace)                       │
│  • Privacy & Permission Daemon  ← FIRST-COMPONENT SPEC      │
│  • Push fabric (UnifiedPush + ntfy operator + FCM bridge)   │
│  • OTA / update orchestrator (A/B partitions + rollback)    │
│  • MDM compat shim (Android Mgmt API + Intune/SOTI/WS1)     │
│  • Compliance daemon (regional: GDPR / DPDP / CERT-In)      │
│  • Telemetry pipe (OHTTP + bucketed; no free-form upload)   │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Compositor & UI Shell                                      │
│  • Wayland compositor in Rust (smithay-derived)             │
│  • System shell (Phosh-class but Rust)                      │
│  • HarfBuzz Indic-script rendering; Lohit/Samyak fonts      │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Android Runtime (isolated, OPT-IN per user)                │
│  • Waydroid-derived AOSP image                              │
│  • Running inside pKVM / crosvm VM, NOT LXC                 │
│  • microG sidecar for Google services replacement           │
│  • FCM-bridge UnifiedPush distributor inside the VM         │
│  • Sandboxed Google Play (GrapheneOS-pattern) for banking   │
│  • No-Android SKU is a first-class build target             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Init, Service Manager, IPC                                 │
│  • Rust init / service manager (systemd-class, simpler)     │
│  • Typed capability-passing IPC (Cap'n Proto over Unix)     │
│  • Capability handles as unforgeable file descriptors       │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  HAL & First-Party Drivers                                  │
│  • New HALs in Rust-for-Linux                               │
│  • Modem/GPU/ISP/NPU stay on vendor drivers initially       │
│  • Verified boot: AVB 2.0 in production; rustBoot R&D       │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Kernel — Production Track                                  │
│  • Linux 7.x mainline + Rust-for-Linux subsystems           │
│  • All new first-party kernel modules in Rust               │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Kernel — R&D Track (parallel, embedded-first)              │
│  • seL4 + Rust userspace (KataOS architectural template,    │
│    re-derived; KataOS itself is dormant)                    │
│  • Linux compat VM for legacy drivers during transition     │
│  • Targets formally-verifiable security-critical TCB        │
└─────────────────────────────────────────────────────────────┘
```

### 4.1 Implementation language

**Rust top-to-bottom for all first-party code.** Zig is not on the table for this project; see [`docs/research/08-rust-vs-zig.md`](../../research/08-rust-vs-zig.md). Possible narrow exception (deferred): Zig in a standalone bootloader or secure-element shim if the audit story demands it, but Rust `no_std` does the same job with the borrow checker as a bonus.

### 4.2 Two-track kernel

- **Production track (18-24 months → v1.0):** Linux 7.x mainline + Rust-for-Linux. All new first-party kernel modules and HALs in Rust. Risk is low because RfL is now policy-permanent and vendor SoC support lands upstream same-day. See [`docs/research/02-rust-kernel-landscape.md`](../../research/02-rust-kernel-landscape.md).
- **R&D track (5-7 years):** seL4 + Rust userspace, KataOS-derived (re-derived; KataOS itself is dormant). Embedded-first. Linux compat VM for vendor drivers during transition; security-critical components (crypto, attestation, baseband isolation, biometrics) migrate to native Rust protection domains first. Mobile migration optional per OEM after the embedded variant proves out.

### 4.3 Android-app compatibility

Waydroid-derived AOSP runtime + microG sidecar, **running inside a pKVM / crosvm VM** (not LXC). The container is **opt-in per user**; a **no-Android SKU is a first-class build target**. Sandboxed Google Play (GrapheneOS pattern) is available inside the VM for users who require banking apps; users who do not can run microG only, or no Google services at all.

The Android runtime is opt-in for two non-negotiable reasons:
- LXC-based Waydroid is *not* a security boundary on ARM; pKVM/crosvm is.
- Some downstream variants (high-assurance defense, fully-sovereign government) cannot ship AOSP code at all. Building the no-Android SKU as first-class avoids those variants forking.

See [`docs/research/03-android-compat-and-push.md`](../../research/03-android-compat-and-push.md).

### 4.4 Push fabric

Hybrid by design. **The host never speaks FCM directly.**

- **UnifiedPush native** for FOSS apps. Default distributor: a Foundation-operated **multi-region ntfy / autopush cluster** with RFC 8291 E2EE, battery-tuned 3-min keepalive.
- **FCM-bridge UnifiedPush distributor inside the Android VM** for closed-source apps that hard-depend on FCM. The bridge runs inside the AOSP runtime; the host never sees an FCM connection.
- **Signal-style hybrid** for adversarial-network scenarios: FCM-as-wakeup, our channel delivers payload.

A dedicated push-service spec is a follow-on deliverable, not part of v0.1. Documented as a deferred item under [`docs/roadmaps/`](../../roadmaps/).

### 4.5 Compliance, telemetry, regulatory awareness

- **CRA-readiness baked into core project hygiene**: SBOM (CycloneDX or SPDX) generated in CI; reproducible builds; signed releases published to a Sigstore + sigsum transparency log; named PSIRT with 24h-aligned disclosure flow per [`SECURITY.md`](../../../SECURITY.md).
- **GDPR / DPDP basics by construction**: no opt-out telemetry path exists in the build; crash reporting uses on-device symbolication + bucket hashing + OHTTP transport; aggregate metrics use local differential privacy.
- **Regional compliance daemons** under `services/compliance/` — separate `eu`, `in`, `global` impls; no compliance code scattered through general code paths.
- **EUCC, BSI VS-NfD, ANSSI Qualification, BIS CRS, FIPS 140-3** are *downstream variant concerns*, not upstream gates. The architecture is designed to make those certifications *possible* (no closed binaries on the critical path, deterministic builds, hardware attestation hooks); pursuing them is a per-variant decision.

## 5. Regional Capability Layer (RCL)

The RCL is the single architectural primitive that makes EU, India, retail, and embedded *variants of one codebase* rather than separate forks. Three parts.

### 5.1 Compile-time variant flags

`region/eu`, `region/in`, `region/global`, `vertical/consumer`, `vertical/retail`, `vertical/embedded`, `vertical/defense`, `cert/eucc-high`, `cert/in-bis`, etc. CI matrix builds the cross-product nightly. Variant code lives under `regions/<code>/` and `verticals/<code>/`, never inline in general code paths.

### 5.2 Runtime regional manifests

Cryptographically signed JSON or CBOR/COSE documents loaded by the privacy daemon at boot. Declares: identity provider, payment provider, default search/maps, language packs, compliance reporting endpoints, telemetry policy, certificate roots, allowed-app-store list. Manifests can be updated independently of OS releases. Schema lives in `frameworks/identity/manifest-schema.cddl` (or equivalent) — single source of truth.

### 5.3 Pluggable trait-based services

```rust
pub trait IdentityProvider { /* eIDAS / Aadhaar+DigiLocker / generic OAuth */ }
pub trait PaymentService    { /* UPI Intent / SEPA Instant / no-op */ }
pub trait LanguageService   { /* on-device Bhashini / FOSS base */ }
pub trait MapsProvider      { /* MapMyIndia / HERE / OpenStreetMap */ }
pub trait EmergencyService  { /* national 112 / 911 / 100 routing */ }
pub trait TelephonyAuxiliary{ /* VoLTE profiles, IMS configs */ }
```

Each trait has multiple implementations under `frameworks/<service>/impls/`; the active impl is selected by the regional manifest at runtime, after compile-time variant flag pre-filtering. No region-specific `if`s in general code.

## 6. Risk register

| Risk | Mitigation |
| --- | --- |
| **Play Integrity DEVICE_INTEGRITY for banking apps** | Container in pKVM/crosvm VM with hardware-backed keystore via virtio-RPC; Sandboxed Google Play (GrapheneOS pattern) inside the VM optional. Long-horizon: pursue eIDAS 2.0 wallet attestation (EU) and RBI / NPCI hardware-key whitelisting (India) as regulator-grade alternatives that don't depend on Google's permission. |
| **No-FCM push at scale** | Hybrid fabric: UnifiedPush native; FCM-bridge inside the Android VM; operator-run multi-region ntfy / autopush cluster as default distributor; Signal-pattern wakeup hybrid. RFC 8291 E2EE throughout. |
| **Bus factor / single-point organizational failure** (CalyxOS lesson) | ≥ 5 funded core engineers from day one; foundation governance with multi-Member-State + Indian academic / govt board representation; commercial integrator entity legally separate; signing and release ceremony documented and rehearsed quarterly; CODEOWNERS requires ≥ 2 reviewers from different organizations on any merge to security-critical paths. |
| **App-ecosystem starvation** (Firefox OS / Tizen / Sailfish lesson) | Android-app compatibility from v0.1 via pKVM Android runtime; native Rust app SDK in parallel for differentiated apps; Indus Appstore as India app distribution partner; explicit non-goal of inventing a bespoke app framework. |
| **Carrier IMS whitelisting in India** | Direct TEC certification + per-carrier IMS profile negotiation budgeted into the India SKU launch timeline; CoIMS-pattern fallback for community variants; Lava as launch OEM (BharOS-validated, has carrier relationships). |
| **EUCC / BSI certification cost** | Reframed as downstream-variant concern, not upstream gate; commercial integrator entity carries certification cost for the SKUs that need it; upstream codebase makes it *possible* via reproducible builds, signed releases, and hardware attestation hooks. |
| **Hardware root-of-trust supply chain** | OEM partner strategy (Fairphone or SHIFT for EU; Lava for India; Pixel 3a/4a + revived Zebra TC52 for retail); the platform is hardware-attestation-hook-aware but does not assume any specific TEE. |
| **"Sovereign OS fatigue"** in India after BharOS underperformance | Don't announce India until 100K committed units pipeline exists; lead with the EU codebase milestone; lean on Maya OS as the procurement precedent; partner with academic institutions for credibility. |

## 7. Feature pillars

Six top-level feature pillars; each becomes its own RFC stream once we move past the vision doc.

1. **Privacy & Permission Framework** — capability-broker daemon, lockdown profiles, regional manifests, append-only signed ledger. [Focused first-component spec.](2026-04-26-privacy-framework-design.md)
2. **App Store & Distribution** — reproducible builds, multi-store federation, F-Droid-grade transparency, paid-app support, sideload-with-warnings, no central kill-switch.
3. **OTA & Verified Boot** — A/B partitions, signed updates with transparency log (Sigstore + sigsum), 7-10 year SLA tier for retail / fleet, AVB 2.0 in production, rustBoot in R&D.
4. **Push Fabric** — UnifiedPush native; FCM-bridge inside the Android VM; operator-run ntfy / autopush cluster.
5. **MDM Compatibility** — Android Management API shim + Intune / SOTI / Workspace ONE protocol bridges; kiosk / single-app lock; remote attestation; certified peripheral stack (DataWedge-equivalent, Verifone / Ingenico / PAX HALs, ESC/POS, ZPL).
6. **Identity & Payment Wallets** — pluggable per region: eIDAS 2.0, Aadhaar + DigiLocker, generic OAuth/OIDC; UPI Intent as system payment service; SEPA Instant equivalent for EU; Bhashini on-device language service for India.

## 8. Roadmap

Phases are codebase milestones, not market launches. Reference SKU launches happen on each variant maintainer's own timeline.

| Phase | Window | Outcome |
| --- | --- | --- |
| **Phase 0 — Bootstrap** | M0–M3 | Foundation incorporated. ≥ 5 core hires. Repo + CI scaffolded. RFC process live. UI-toolkit and IPC bake-offs run. Threat model published. |
| **Phase 1 — Privacy Daemon v0.1** | M3–M9 | Privacy & Permission Daemon runs as standalone userspace on AOSP / postmarketOS. First public alpha. Real adopters; feedback loop active. **Demoable on Pixel 3a / 4a today.** |
| **Phase 2 — Production OS Alpha** | M9–M18 | Own init / services / HAL / compositor on Linux + RfL. Single reference device internal-use SKU (Pixel 4a or Fairphone 5). Push fabric, OTA, app store skeletons. |
| **Phase 3 — v1.0 Codebase Stable** | M18–M30 | Public 1.0 release. Android VM (pKVM) integrated. App store live. EU prosumer reference SKU available; retail demo SKU on Pixel 3a / 4a. EUCC track work begins downstream. |
| **Phase 4 — Regional Variants Ship** | M24–M36 | India procurement pilot on Lava (Bhashini, UPI Intent, DigiLocker). Retail fleet SKU on Zebra TC52 (after 6-12 mo mainlining investment). Indian consumer SKU follows after RBI / NPCI engagement. |
| **Phase 5 — Microkernel Preview** | M36–M60 | seL4 + Rust R&D track produces first credible mobile preview. Public migration plan. Embedded reference board kit. |
| **Phase 6 — Microkernel Production** | M60+ | Production-grade microkernel on embedded SKUs. Long-tail mobile migration optional per OEM. |

## 9. Governance

Foundation + commercial integrator entity model. See [`GOVERNANCE.md`](../../../GOVERNANCE.md) for the full document.

Key commitments (all measurable, all published quarterly):

- Bus factor ≥ 5 paid core engineers; ≥ 3 release-key holders across ≥ 2 organizations; ≥ 3 PSIRT roster across ≥ 2 organizations.
- No single organization holds > 50 % of any subsystem's commits over any rolling 6-month window.
- All code merged via 2-of-3 review on security-critical paths, with reviewers from at least 2 distinct organizations.
- All releases reproducibly built; release artifacts published to Sigstore + sigsum.

## 10. Repo layout

See the top-level directory tree maintained at [`README.md`](../../../README.md) and the in-tree docs.

Notable subtrees:

- `services/privacy/` — privacy & permission daemon (focused first-component subsystem).
- `regions/{eu,in,global}/` — regional capability impls; never `if region == "EU"` scattered through general code.
- `verticals/{consumer,retail,embedded,defense}/` — vertical-specific code paths.
- `kernel/` — Rust-for-Linux first-party modules.
- `kernel-modules/` — GPL-2.0 isolation subtree.
- `microkernel/` — R&D track, separate top-level.
- `android-runtime/` — pKVM image, microG, FCM bridge.
- `docs/superpowers/specs/` — design specs (this doc and the privacy framework spec live here).
- `docs/research/` — synthesized research dossier.
- `docs/rfcs/` — numbered RFCs.
- `docs/roadmaps/` — deferred-feature backlogs per subsystem.

## 11. Funding model

Tracked, not committed in the vision doc. The Foundation and the commercial integrator entity are funded separately.

- **Foundation funding sources:** Sovereign Tech Agency (DE), NLnet NGI Zero, Horizon Europe Cluster 3 / 4 consortium grants, MeitY co-funding once procurement traction exists in India, commercial-member dues.
- **Commercial integrator funding sources:** EIC Accelerator (€2.5M grant + €10M equity), national defense agencies (DGA / Cyberagentur) for certification-led variant work, paid certified-variant contracts, MDM SaaS subscriptions, refurbishment-pipeline service revenue.

See [`docs/research/05-eu-regulatory-landscape.md`](../../research/05-eu-regulatory-landscape.md) and [`docs/research/07-india-sovereign-mobile-os.md`](../../research/07-india-sovereign-mobile-os.md) for sourcing details.

## 12. Implementation strategy

The brainstorm produced *one focused implementation spec* alongside this vision: the [Privacy & Permission Framework](2026-04-26-privacy-framework-design.md). That spec is the v0.1 deliverable for Phase 1 — a Rust userspace daemon that runs on existing AOSP / postmarketOS as a standalone product, gets real adopters and feedback, and grows into the integration spine for the rest of the OS.

The vision-doc roadmap above tracks all other subsystems at phase level. As each phase opens, individual subsystem leads draft focused specs under `docs/superpowers/specs/` following the same convention as the privacy-framework spec, and implementation plans under `docs/rfcs/`.

## 13. Open questions

These are intentionally unresolved at vision-doc level. Each will be answered by an RFC during the relevant phase.

- **UI toolkit choice.** Slint vs Iced vs GTK4-rs vs Servo-based vs upstream Phosh-rust-rewrite — bake-off in Phase 0.
- **IPC framework.** Cap'n Proto vs zbus (D-Bus successor) vs custom — bake-off in Phase 0.
- **Web engine.** Servo (also Rust) is principled; we may need to ship WebKit or Chromium for parity in v1.0 and converge to Servo later. Marked as a known compromise.
- **Foundation jurisdiction.** EU-incorporated is the default expectation; specific Member State, legal form, and bylaws drafting are Phase 0 work.
- **Commercial integrator entity** — jurisdiction, structure, IP boundaries with the Foundation. Phase 0.
- **OEM partnerships.** Fairphone / SHIFT / Pine64 (EU); Lava / Karbonn / Micromax (India); Zebra / Honeywell (retail). Negotiations begin Phase 0.
- **Foundation funding sequence.** Sovereign Tech Agency first, EIC second, Horizon consortium third — but order depends on real-world grant cycles.

## 14. Appendix — how this design was produced

This vision doc and the companion privacy-framework spec were produced in a single brainstorm session on 2026-04-26 using the `superpowers:brainstorming` skill workflow. Eight parallel research streams (see [`docs/research/`](../../research/)) provided the empirical basis. Key decisions were made via multiple-choice questions answered by the project sponsor, with specific reframes recorded:

- Original framing was "EU sovereignty, certification-led, procurement-anchored." Reframed mid-session to "AOSP-class open base, certifications downstream, regional customization first-class" — see [`README.md`](../../../README.md) and the audience-pillars table for the resulting structure.
- India was added as a co-equal sovereignty pillar after research stream 7 surfaced the BharOS gap, the Bhashini-on-device viability, and the UPI Intent system-service opportunity.
- Implementation language was confirmed Rust (vs. Zig) by research stream 8.

The brainstorm's terminal state is invoking the `superpowers:writing-plans` skill to produce a detailed implementation plan from this spec and the privacy-framework spec.
