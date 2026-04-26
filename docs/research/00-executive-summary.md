# Executive summary — research dossier

A condensed view of the eight research streams. For each, the top finding, the load-bearing implication for design, and the URL of the full stream file.

---

## 1. Prior art on alternative mobile OSes

**Top finding:** the graveyard pattern is unambiguous. Every project that died (Firefox OS, Tizen mobile, DivestOS) or stalled (Ubuntu Touch, Sailfish, Librem 5, KaiOS post-2024) tried to invent a bespoke app framework or relied on a single maintainer / single-vendor channel. The survivors (GrapheneOS on Pixel, Murena on Fairphone) all converge on one pattern: hardened-AOSP-or-Linux base + Android-app compat + a single hardware partner + foundation-plus-commercial governance with ≥ 5 funded engineers.

**Implication for design:** AOSP-class open base; Android-app compat from v0.1; ≥ 5-engineer bus factor as a published KPI; OEM-partner strategy from day one.

[Full file →](01-prior-art-mobile-os.md)

---

## 2. Rust kernel landscape

**Top finding:** Linux 7.0 (April 2026) shipped Rust-for-Linux as a stable peer language with ~600 KLOC in tree, including the NVIDIA Nova GPU driver, Apple AGX, Android binder/ashmem, and NVMe abstractions. Qualcomm provides same-day upstream support for new Snapdragon SoCs. seL4 + Rust is Foundation-funded and is the credible long-horizon microkernel play; KataOS is the architectural template but is dormant. Hubris/Tock/R3/Drone are MCU-only.

**Implication for design:** two-track kernel — Linux + RfL as production, seL4+Rust (KataOS-derived, re-derived) as R&D. All new first-party kernel modules and HALs in Rust.

[Full file →](02-rust-kernel-landscape.md)

---

## 3. Android-app compatibility and push

**Top finding:** Waydroid + LXC is *not* a security boundary on ARM (shared kernel, bind-mounted device nodes). Production-grade isolation requires pKVM / crosvm-style VM. Google's Play Integrity API, with hardware-backed signals default-on as of May 2025, has effectively killed banking-app compatibility on non-Pixel non-Google distributions; microG passes only Basic verdict. UnifiedPush is the strongest FCM-replacement candidate but lacks scale-out infra and battery parity in adversarial network conditions.

**Implication for design:** Android runtime in pKVM/crosvm, opt-in per user, no-Android SKU as first-class build target. Hybrid push fabric: UnifiedPush native + FCM-bridge distributor *inside* the Android VM + operator-run ntfy/autopush cluster. Banking-app support is a known constraint requiring eIDAS / RBI / OEM-attestation engagement, *not* a v1.0 deliverable.

[Full file →](03-android-compat-and-push.md)

---

## 4. Privacy and permission framework prior art

**Top finding:** GrapheneOS is the floor (Storage Scopes, Sandboxed Google Play, hardware attestation via Auditor). The frontier move is replacing ambient permissions with **typed capability handles** brokered by a system-trusted UI — a synthesis of Flatpak portals, iOS PhotoPicker, and Apple's Privacy Manifests, but with runtime enforcement and cryptographic ledgering that no shipping OS provides.

**Implication for design:** the Privacy & Permission Daemon is the focused first-component spec. It anchors on capability handles, an append-only signed ledger with user-gesture-evidence, stated-purpose contracts, four-mode denial (HardDeny / ShimDeny / AskUser / GrantWithEgressLimit), per-capability network egress, lockdown profiles as first-class state, and OHTTP-bucketed telemetry by construction.

[Full file →](04-privacy-permission-framework.md)

---

## 5. EU regulatory landscape

**Top finding:** the DMA's interoperability mandate (Art 6(7)) and default-app rules (Art 6(3)) are genuine tailwinds for an alternative mobile OS. The CRA, with vulnerability reporting starting 2026-09-11 and full compliance 2027-12-11, is *unconditional* — anyone shipping in the EU must comply. EUCC certification is optional and downstream. Hardware root-of-trust supply chain is the single hardest non-software barrier.

**Implication for design:** CRA-readiness (SBOM, signed updates, transparency log, named PSIRT) is project hygiene baked into the core. EUCC, BSI VS-NfD, ANSSI Qualification are downstream variant concerns, not upstream gating. EU regulation reframed from "spine" to "constraint plus tailwind" per user direction during the brainstorm.

[Full file →](05-eu-regulatory-landscape.md)

---

## 6. Old-Android revival and retail / fleet market

**Top finding:** the rugged-Android handheld installed base is ~50-70 M units globally with a 4-6 year mechanical life and a 3-4 year OEM software window — meaning a large fleet of devices retired annually with usable hardware. The killer feature for the OS is an **MDM compatibility shim** that speaks the Android Management API plus Intune / Workspace ONE / SOTI protocols. Sweet spot for revival is 2018-2021 Snapdragon 6xx/8xx; the prize fleet target is Zebra TC52/TC57 (requires 6-12 months of mainlining).

**Implication for design:** retail/fleet is a co-equal pillar. MDM compat shim is a top-level feature pillar. 7-10 year OTA SLA tier exists for retail downstream variants. Demo SKU on Pixel 3a/4a is a Phase 1 milestone.

[Full file →](06-old-android-revival-retail.md)

---

## 7. India sovereign mobile OS landscape

**Top finding:** BharOS shipped ~500 units; Indus OS is dead; KaiOS-on-Jio collapsed when Jio pivoted; Maya OS (DRDO desktop) is the only sovereign-OS procurement precedent. The single biggest lesson is that **distribution is sovereignty** — India OS plays succeed only with a forced distribution channel (carrier mandate, state procurement, OEM bundling). On the technical side, on-device Bhashini at phone scale is now viable in 2026, UPI Intent is being repositioned by NPCI as the canonical payment surface, and Lava is the natural OEM partner.

**Implication for design:** India is co-equal in the codebase from day one (RCL primitive). Reference SKU launches stagger after EU's because of RBI/NPCI hardware-attestation engagement (~9-12 months) and BIS/MTCTE certification (~4-6 months). UPI Intent as `PaymentService` impl, DigiLocker+Aadhaar as `IdentityProvider` impl, Bhashini as `LanguageService` impl all live under `regions/in/`.

[Full file →](07-india-sovereign-mobile-os.md)

---

## 8. Rust vs Zig

**Top finding:** for an AOSP-class mobile/embedded OS in 2026, Rust is correct and Zig is not on the table. Rust ships in mainline Linux 7.0 with the exact drivers we need; Zig has zero upstream kernel presence and zero shipping OS-class production code at scale. Rust's talent pool is 20-50× larger; foundation governance is institutionally robust where Zig is BDFL with a real bus-factor risk over a 10-year horizon. Zig's wins (comptime, cross-compile, no hidden control flow) don't offset any of those for *this* product. Narrow possible exception: a standalone bootloader or secure-element shim — but Rust `no_std` does that with the borrow checker as a bonus.

**Implication for design:** Rust top-to-bottom for all first-party code. No Zig in the dependency tree.

[Full file →](08-rust-vs-zig.md)
