# Privacy & Permission Framework — Deferred-Features Roadmap

This document is the durable backlog for items deferred from the [Privacy & Permission Framework v0.1 spec](../superpowers/specs/2026-04-26-privacy-framework-design.md). Each item lists its target version, the vision-doc phase it lines up with, the dependencies that gate it, and the spec section that originally deferred it.

This file outlives the v0.1 spec — when v0.2 work begins, this is where contributors look first to know what's expected.

## How to use this file

- **Picking up an item:** open a GitHub issue, link the relevant section here, and write an RFC if the item changes a public interface. Move the item from "Pending" to "In progress" in this file in the same RFC.
- **Adding an item:** new deferred items should land here whenever a spec is updated to push something out a version. Each item must have a target version, a phase, and at least one dependency.
- **Removing an item:** when an item ships, move it to the "Shipped" section at the bottom, with the version it shipped in and a link to the merged RFC / PR.

## Item index

| # | Item | Target | Phase | Status |
| --- | --- | --- | --- | --- |
| 1 | [Android VM integration](#android-vm-integration) | v0.2 | 2 | Pending |
| 2 | [Hardware-attested gate](#hardware-attested-gate) | v0.2 | 2 | Pending |
| 3 | [Per-app DoH broker](#per-app-doh-broker) | v0.2 | 2 | Pending |
| 4 | [Regional service impls](#regional-service-impls) | v0.2-v0.4 | 2-4 | Pending |
| 5 | [Ledger inspection GUI](#ledger-inspection-gui) | v0.3 | 3 | Pending |
| 6 | [Multi-user profiles](#multi-user-profiles) | v0.3 | 3 | Pending |
| 7 | [Geofence profile switch](#geofence-profile-switch) | v0.3 | 3 | Pending |
| 8 | [Federated ledger](#federated-ledger) | v0.4 | 3 | Pending |
| 9 | [Formal verification](#formal-verification) | v0.5+ | 5+ | Pending |

---

## Android VM integration

**Target:** v0.2  •  **Phase:** 2 (M9-M18)  •  **Status:** Pending

In v0.1 the daemon runs on existing AOSP / postmarketOS as a userspace daemon next to a Linux host. In v0.2, the daemon mediates capability requests *from* the pKVM/crosvm-isolated Android runtime. The Android-side IPC client speaks our typed capability protocol over a virtio-vsock channel.

**Scope:**
- Virtio-vsock IPC bridge between the host daemon and an Android-side stub.
- Translation of Android `PermissionController` calls into our typed-capability protocol.
- Specific handling for FD passing across the VM boundary (SCM_RIGHTS doesn't cross VMs natively; we need a virtio-FD bridge or equivalent).
- Sandboxed Google Play interop: when SGP is installed inside the VM, its permission requests still flow through our broker.
- Stated-purpose contracts for Android apps installed via Indus Appstore / F-Droid / sideload.

**Dependencies:**
- `android-runtime/` subsystem reaching alpha (pKVM / crosvm container working on at least one reference device).
- `compositor/` reaching alpha so the Portal UI Coordinator can render system-trusted prompts that the Android-VM apps can't spoof.
- A virtio-FD bridge or alternative cross-VM resource handle mechanism.

**Spec section:** [Privacy spec § Non-goals → "No full pKVM / crosvm Android-VM integration in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Hardware-attested gate

**Target:** v0.2  •  **Phase:** 2 (M9-M18)  •  **Status:** Pending

Sensitive capabilities (decrypt user data, access keystore) require an attestation challenge that proves the daemon binary is unmodified — closing the userspace-tamper hole. v0.1 has a stub that records "attestation: not-yet-implemented" in the ledger.

**Scope:**
- `HardwareAttestor` trait in `frameworks/identity/`.
- Reference impl for at least one TEE (Pixel Titan M2 / StrongBox; later Samsung Knox; Bittium TEE for the EU defense variant).
- Attestation challenge protocol: nonce → daemon signs with HW-backed key → policy engine verifies before granting sensitive capabilities.
- Migration path for users who upgrade from v0.1 (locally-generated key) to v0.2 (HW-backed key).

**Dependencies:**
- `frameworks/identity/` shipping at least one `HardwareAttestor` impl.
- A reference device with HW-backed attestation available for testing.

**Spec section:** [Privacy spec § 5.10 Hardware-Attested Gate](../superpowers/specs/2026-04-26-privacy-framework-design.md#510-hardware-attested-gate).

---

## Per-app DoH broker

**Target:** v0.2  •  **Phase:** 2 (M9-M18)  •  **Status:** Pending

v0.1 enforces network egress at L4 only (nftables, per-uid). v0.2 adds an on-device DNS-over-HTTPS broker per app, so DNS resolution itself is bound to the capability's declared destinations — defeating side-channel resolution and leaking-via-DNS attacks.

**Scope:**
- Per-app DoH resolver instances, configured from the app's manifest.
- Logging of DNS queries to the ledger (with redaction in default privacy mode).
- Enforcement at L7: a query for an undeclared destination fails closed.

**Dependencies:**
- A working DoH client library (likely `trust-dns-resolver` or equivalent).
- `services/push/` may need to coordinate on the operator-run resolver infrastructure.

**Spec section:** [Privacy spec § Non-goals → "No per-app DNS-over-HTTPS broker in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Regional service impls

**Target:** v0.2 to v0.4  •  **Phase:** 2-4  •  **Status:** Pending

v0.1 defines the `IdentityProvider`, `PaymentService`, and `LanguageService` traits and ships one default impl per region (FOSS-base language, no-op payment, generic OAuth identity). Real impls land per region as the regional pillars come online.

**Sub-items:**

| Impl | Trait | Target | Dependencies |
| --- | --- | --- | --- |
| eIDAS 2.0 wallet | `IdentityProvider` | v0.3 | EU eIDAS wallet SDK availability; EU regional manifest signing in place |
| Aadhaar + DigiLocker | `IdentityProvider` | v0.3 | DigiLocker partner credentials; DPDP compliance review |
| UPI Intent | `PaymentService` | v0.3 | NPCI engagement; Indus Appstore distribution alignment |
| SEPA Instant + Wero/EPI | `PaymentService` | v0.4 | EU payment-rails SDK; legal review |
| On-device Bhashini ASR/TTS/MT | `LanguageService` | v0.3 | Bhashini SDK on-device packaging; storage for 22-language model |
| MapMyIndia / Mappls | `MapsProvider` | v0.3 | API agreement |
| Qwant / HERE | `MapsProvider` | v0.4 | API agreement |

**Spec section:** [Privacy spec § Non-goals → "No Bhashini / UPI Intent / eIDAS bindings in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Ledger inspection GUI

**Target:** v0.3  •  **Phase:** 3 (M18-M30)  •  **Status:** Pending

v0.1 ships a CLI for browsing the audit ledger. v0.3 adds a user-facing GUI in the system Settings app — timeline view, per-app capability history, denial reasons, "remove this grant" affordances.

**Scope:**
- Settings-app integration.
- Localized UI strings (24+ EU languages, 22 Indian scheduled languages).
- Export-to-OHTTP feature so privacy researchers can opt-in to share aggregated ledger data.

**Dependencies:**
- `shell/` and `apps/settings/` reaching alpha.
- `frameworks/language/` providing localization primitives.

**Spec section:** [Privacy spec § Non-goals → "No user-facing GUI for ledger inspection in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Multi-user profiles

**Target:** v0.3  •  **Phase:** 3 (M18-M30)  •  **Status:** Pending

v0.1 is single-user. v0.3 adds AOSP-style work-profile isolation: per-user ledger, per-user manifest, per-user capability state, end-session wipe support.

**Scope:**
- Per-user ledger storage paths and signing keys.
- Per-user lockdown profile state.
- Cross-user capability isolation (an app installed in profile A cannot see resources in profile B).
- "Switch profile" UX integrated into the system shell.

**Dependencies:**
- `init/` providing user-account primitives.
- `compositor/` supporting profile-aware session management.

**Spec section:** [Privacy spec § Non-goals → "No multi-tenant / work-profile isolation in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Geofence profile switch

**Target:** v0.3  •  **Phase:** 3 (M18-M30)  •  **Status:** Pending

v0.1 supports manual / hardware-key / MDM lockdown profile switches. v0.3 adds opt-in geofence-triggered switching ("activate Travel profile when leaving Berlin").

**Scope:**
- Privacy-respecting geofence primitive: regions defined as encrypted client-side state, evaluated only against a privacy-preserving location signal (not raw GPS).
- Profile-switch policy: per-geofence rules, time-of-day rules, transition smoothing.
- Audit log for every triggered switch.

**Dependencies:**
- A `LocationService` impl with a privacy-preserving geofence primitive (likely on-device only; no upload).
- User-facing affordances in the Settings app.

**Spec section:** [Privacy spec § Non-goals → "No geofence-triggered profile switching in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Federated ledger

**Target:** v0.4  •  **Phase:** 3 (late) / 4  •  **Status:** Pending

v0.1-v0.3 keep the ledger single-device. v0.4 adds cross-device federation: a user with a phone, tablet, and laptop sees one consistent capability ledger across devices.

**Scope:**
- Cross-device identity primitive (probably via the `IdentityProvider` impl: eIDAS / DigiLocker as the trust anchor for cross-device pairing).
- Conflict resolution for concurrent writes (CRDT-style merging? signed timestamps?).
- Distributed-trust model for the ledger's hash chain.
- An RFC for the federation protocol itself, before implementation.

**Dependencies:**
- Stable cross-device identity / pairing in `frameworks/identity/`.
- An RFC accepted for the federation protocol.

**Spec section:** [Privacy spec § Non-goals → "No federated cross-device ledger"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Formal verification

**Target:** v0.5+  •  **Phase:** 5+ (M36+)  •  **Status:** Pending

v0.1-v0.3 ship TLA+ / Alloy models of the Policy Engine only. v0.5+ extends to a full Verus / Creusot proof of the Capability Broker, IPC parser, and ledger.

**Scope:**
- Stable, frozen API surface for the components being verified.
- Verus or Creusot proofs alongside the Rust source.
- CI integration: proofs re-checked on every PR touching verified components.

**Dependencies:**
- API stability — no proof effort makes sense before the API converges.
- Verus / Creusot tooling reaching production maturity.
- One or more team members with formal-methods background hired.

**Spec section:** [Privacy spec § Non-goals → "No formal verification of the full daemon in v0.1"](../superpowers/specs/2026-04-26-privacy-framework-design.md#11-non-goals-deferred-to-future-versions).

---

## Shipped

*This section will be populated as items move from Pending → Shipped. Each entry will include the version, the merged RFC, and the PR series.*

| # | Item | Shipped in | RFC | PR series |
| --- | --- | --- | --- | --- |
| _none yet_ | | | | |
