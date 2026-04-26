# Stream 4 — Privacy and permission framework state of the art

*Snapshot dated 2026-04-26.*

## GrapheneOS innovations

GrapheneOS extends AOSP's permission model with userspace-level interception that doesn't require kernel changes:

- **Storage Scopes** — replaces broad storage permission with per-app, per-directory/file allowlists. Lives in `PermissionController` and a `ContentResolver` shim — apps think they have full storage but receive a filtered view. Threat: apps demanding all-files access for legitimate single-folder use.
- **Contact Scopes** — same pattern for contacts; returns a curated subset rather than denying outright, defeating apps that refuse to run without contacts.
- **Network permission** — runtime permission via Android's app-id → uid → netd mapping, enforced through `INTERNET` toggling and per-uid iptables rules.
- **Sensors permission** — toggle for the sensors HAL beyond mic/camera (accelerometer, gyro, barometer) — used for motion-based fingerprinting and side-channel keystroke recovery.
- **Sandboxed Google Play** — runs GMS / Play Store / GSF as ordinary unprivileged apps in a user profile, shimming `GoogleServiceManager` calls.
- **Hardware attestation / Auditor** — uses Pixel Titan M's StrongBox-backed key attestation to remotely verify firmware/OS integrity, pinned to first-pairing TOFU.
- **hardened_malloc** — heap allocator hardening primitive — separate spec concern but underpins exploit resistance.
- **User / Work profiles** — AOSP's multi-user infra extended with end-session wipe, separate VPNs per profile.

Architectural takeaway: **GrapheneOS prefers shimming over blocking.** Telling apps "no" breaks them; returning empty / curated data keeps them functional while denying real data.

## Apple iOS privacy stack

- **App Tracking Transparency (ATT)** — mandatory consent prompt before `IDFA` access. Worth copying: the *required prompt* pattern. Marketing aspect: doesn't cover Apple's own SKAdNetwork or fingerprinting via device characteristics.
- **Privacy Manifests** — static declarations of data types collected and "required reason APIs". Genuinely useful as a *machine-checkable contract*; weakness is Apple-trusted enforcement only at App Store review.
- **Privacy Nutrition Labels** — self-attested by developer, not audited. Mostly marketing.
- **On-device intelligence** (Siri Suggestions, Photos ML) — architecturally sound; models run in Secure Neural Engine.
- **iCloud Private Relay** — two-hop architecture (Apple ingress + third-party CDN egress) so neither sees both client IP and destination. Limited to Safari and unencrypted HTTP.
- **Lockdown Mode** — profile-based attack-surface reduction — disables JIT, link previews, complex font parsing, message attachments. **Worth copying as a first-class OS profile, not an obscure toggle.**

## Android 14/15 privacy features

- **Privacy Indicators** — status-bar dot when mic/camera active, surfaced via `SensorPrivacyService`. Useful.
- **Photo Picker** — system UI returns user-chosen URIs without granting `READ_MEDIA_IMAGES`. The cleanest brokered-access pattern Google has shipped.
- **Partial photo access (Android 14)** — per-asset selection.
- **Health Connect** — per-data-type permissions with on-device store. Useful pattern, compromised by being a Google-controlled APK.
- **Ad ID controls** — resettable + deletable, but exists at all only because Google's ad business depends on it. Compromised by design.

## Permission frameworks in other OSes

- **Flatpak portals** — `xdg-desktop-portal` is a D-Bus daemon mediating filesystem, screenshot, location, camera access. App requests a portal method; the *portal* shows a system-trusted picker UI; only the chosen resource handle is returned to the sandboxed app. Where it falls short: portal coverage incomplete (notably GPU/audio), and apps with `--filesystem=home` bypass entirely.
- **macOS TCC** — SQLite-backed consent database in `/Library/Application Support/com.apple.TCC/`. Gated by `tccd`. Weakness: per-bundle-ID identity is forgeable until SIP+notarization checks; consent dialogs lack provenance.
- **SELinux / AppArmor** — kernel-LSM mandatory access control. Powerful but policy is static and admin-defined — not user-mediated runtime brokering. AOSP uses SELinux as a *floor* under the userspace permission system.
- **Snap confinement** — interfaces gated via AppArmor + seccomp profiles, with `snapd` as broker. Falls short: classic confinement escape hatch, slow auto-connect policy review.

**The key brokering insight:** portals win when the *broker UI itself* picks the resource (the user picks a file; the app never sees the path until selected). Granting capability and choosing instance happens in one user gesture.

## Hardware kill switches

- **Librem 5** — three physical switches cutting power to mic, camera, baseband/Wi-Fi — true hardware-cut, lines de-energized.
- **MNT Reform** — user-replaceable modules; switches and physical disconnects rather than firmware toggles.
- **ThinkPad webcam shutters / mic mute LED** — mic mute on modern X1 is firmware-arbitrated via Embedded Controller. Webcam shutter is mechanical (real).

Software-arbitrated "kill" is theater if a kernel exploit can flip it. Hardware-cut is the real threat-model improvement, but adds BOM cost. Spec implication: if you only have software toggles, expose them as kernel-driven GPIO with a *visible LED tied to the same line* so the user can verify.

## Telemetry-free crash and analytics

- **Mozilla Socorro / crash-stats** — opt-in minidump submission; pings strip identifying fields server-side. Still uplinked to a Mozilla server.
- **Signal private metrics** — sealed-sender style aggregate metrics; uses oblivious HTTP / proxy aggregation for counts without per-user attribution.
- **Apple's differential privacy** — on-device noise injection prior to upload — useful for aggregate stats (emoji frequency), useless for crash diagnostics where you need the actual stack.
- **Google's Private Compute Core** + federated analytics — similar idea, federation server sees only noised gradients.

Production state-of-the-art: **Oblivious HTTP (RFC 9458)** is becoming the standard primitive — encrypts request to backend, proxy strips client IP. Apple Private Relay, Cloudflare, Fastly all support OHTTP. For crash data specifically, **on-device symbolication + bucketed reporting** (send "crash bucket X happened" not the stack) preserves utility for top crashes without raw upload.

## Synthesis: a novel permission framework for Promethea

GrapheneOS is the floor. To exceed it, anchor the daemon spec on these features:

1. **Capability-based portal broker as the *only* path to sensitive resources.** Adopt Flatpak / Photo Picker model wholesale: no app ever holds a "camera permission" — it requests a one-shot capability handle from a system-trusted UI. Default-deny; deny-by-shim (return empty) when an app refuses graceful handling. Implemented as a Rust daemon over a typed IPC (Cap'n Proto / D-Bus successor) with capability handles as unforgeable file descriptors.

2. **Permission ledger with cryptographic provenance.** Every grant is an append-only signed log entry: `(app-id, capability, scope, timestamp, user-gesture-evidence)`. User-gesture-evidence is the input event that authorized it. Prevents UI redress / clickjacking and gives forensic auditability — something neither GrapheneOS nor TCC ships.

3. **Stated-purpose contracts (Privacy Manifests done right).** Apps ship a signed, machine-checkable manifest of capabilities × purposes. The daemon enforces the *intersection* of manifest and runtime grant. Diverging behavior (manifest says "analytics," app calls contacts) triggers revocation. Apple ships the format; nobody enforces it at runtime.

4. **Per-capability network egress policy bound to purpose.** Network is gated not just by uid but by destination policy declared in manifest. Combine with on-device DNS-over-HTTPS broker that logs egress per-capability. Goes beyond GrapheneOS's binary INTERNET toggle.

5. **Lockdown profiles as first-class OS state, not a toggle.** Multiple named profiles ("Travel," "Border crossing," "Activist") each a declarative bundle of capability defaults, network rules, attestation requirements. User switches via a hardware-mapped key combo. Apple's Lockdown Mode is the seed; make it composable.

6. **Hardware-attested capability gates.** Sensitive capabilities (decrypt user data, access keystore) require StrongBox / SE-backed attestation that the *daemon itself* is unmodified — not just the OS image. Closes the userspace-tamper hole the Auditor app can't see.

7. **Privacy-preserving telemetry by construction.** No opt-out telemetry path exists in the build. Crash reporting uses on-device symbolication + bucket hashing + OHTTP transport to a relay run by a separate org. Aggregate metrics use local differential privacy. Zero free-form upload.

The throughline: **GrapheneOS hardens the existing permission model; this design replaces "the app has permission X" with "the app has been handed a typed capability handle for one specific resource, by the user, witnessed by a signed ledger, constrained by a manifest, transported only to declared destinations."** Permissions become artifacts of capability flow rather than ambient grants — which is what Rust's ownership model lets us encode in the daemon's API natively.

## Sources

- https://grapheneos.org/features
- https://grapheneos.org/usage#sandboxed-google-play
- https://attestation.app/about
- https://github.com/GrapheneOS/hardened_malloc
- https://developer.apple.com/documentation/apptrackingtransparency
- https://developer.apple.com/documentation/bundleresources/privacy_manifest_files
- https://support.apple.com/en-us/102602
- https://support.apple.com/en-us/105120
- https://source.android.com/docs/core/permissions/privacy-indicators
- https://developer.android.com/training/data-storage/shared/photopicker
- https://developer.android.com/health-and-fitness/guides/health-connect
- https://docs.flatpak.org/en/latest/portals.html
- https://www.rainforestqa.com/blog/macos-tcc-db-deep-dive
- https://snapcraft.io/docs/snap-confinement
- https://puri.sm/posts/librem-5-hardware-kill-switches-and-lockdown-mode/
- https://mntre.com/reform
- https://crash-stats.mozilla.org/documentation/
- https://signal.org/blog/private-app-metrics/
- https://www.apple.com/privacy/docs/Differential_Privacy_Overview.pdf
