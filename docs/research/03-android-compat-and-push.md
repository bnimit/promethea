# Stream 3 — Android compatibility and push notifications

*Snapshot dated 2026-04-26.*

## Waydroid

Waydroid runs a full Android system inside an LXC container with shared PID/network/mount/IPC/user namespaces, seccomp filters, AppArmor profiles, and direct GPU passthrough via Mesa for near-native graphics. The official image is still pinned to LineageOS 20 / Android 13; Lineage 23 (Android 16) builds exist only via the community `WayDroid-ATV/waydroid-builds` fork.

Integration gaps that matter for a product spec:

- **Clipboard** is broken or one-way; long-running unresolved issue.
- **File picker / share sheet / intent routing** are container-local — Android intents do not cross into Wayland, and host files only appear via a manually configured shared folder.
- **Notifications** surface inside Android's shade, not the host compositor.
- **Biometrics** are not bridged from host fingerprint readers.
- On real ARM phones (PinePhone/Pro, postmarketOS) Waydroid is **still flaky**: openrc service detection fails, container won't start, GPU passthrough is fragile.

**Security posture: the container does *not* contain in any meaningful VM sense.** Host device nodes are bind-mounted, kernel is shared, and any kernel exploit from inside Android is a host compromise. Treat it like a chroot with seccomp, not a hypervisor.

## microG

microG (Marvin Wißfeld / `mar-v-in`, financially backed by /e/ Foundation since 2020) reimplements Google Play Services APIs (FCM client, location, Maps, auth, Cast). Active in 2026 — recent releases add Device Integrity per-app blocking, Wi-Fi location fixes, 2FA fixes.

Works: FCM push (proxied through Google), Maps API v1, basic location via Mozilla / DejaVu backends, Google sign-in for many apps, in-app billing for some titles. Does not work: most banking apps, Google Pay, Snapchat, Pokémon Go, Netflix downloads, anything calling hardware-attested Play Integrity. **microG can pass *Basic* integrity at best — never *Device* or *Strong*.**

## Play Integrity in 2026

Google flipped hardware-backed signals to **default-on for all integrators in May 2025**, and SafetyNet was fully retired January 2025. Banking, payment, streaming, dining, and gaming apps now routinely require *Device* (TEE-backed key attestation rooted in Google's hardware key provisioning) or *Strong* (StrongBox) verdicts.

Play Integrity Fix (Magisk module) keeps working only because volunteers rotate spoofed keyboxes when Google revokes them — this is a **cat-and-mouse game, not an architecture**. The Android root-CA rotation in February 2026 is expected to break attestation for many older devices. Realistic 2026 outlook: any non-Google distribution that lacks signed bootloader + Google-provisioned attestation keys is a second-class citizen for regulated apps. microG alone cannot close this gap.

## UnifiedPush

UnifiedPush (5-year retrospective Jan 2026) is a spec where apps register with a user-chosen *distributor* (ntfy, NextPush, Gotify, FCM-bridge, KDE Connect, Conversations XMPP) which holds the persistent connection. Battery: ntfy reports 0–1% over 17h with 3-min ping, ~2-3× higher on WebSocket vs JSON stream.

**Production-ready? For FOSS apps yes; as a general FCM replacement no.** Adoption is concentrated in Matrix (Element, FluffyChat), Mastodon (Tusky, Fedilab), Tuta, Jami, Conversations. No commercial app of consequence ships UnifiedPush; banking and streaming will not. Public ntfy.sh runs on a single DigitalOcean droplet with no scale-out — fine for hobbyists, not a tier-1 service.

## Push engineering — what's actually hard

- **One persistent socket per device, not per app.** FCM, APNs and ntfy all multiplex; UnifiedPush's "pick a distributor" achieves the same on Android. Mozilla's autopush does it at browser scope.
- **Doze / App Standby compliance.** Android kills background sockets aggressively; only FCM is whitelisted by default. Self-hosted distributors need user-toggled battery exemption — a UX cliff.
- **End-to-end encryption with offline storage.** RFC 8291 (ECDH P-256 + HKDF + AES-128-GCM) is the standard; Mozilla's autopush stores ciphertext in DynamoDB keyed by UAID until delivery.
- **Reliability under reconnection storms.** APNs-scale design (50M+ devices) requires sharded routing tables, TTL-bounded queues, and idempotent delivery.
- **Federation.** UnifiedPush's bet: each user picks a server; app servers POST to opaque endpoints. Hard parts are abuse prevention, rate limiting per endpoint, and key rotation without re-registering every app.
- **Signal's lesson:** even Signal falls back to FCM-as-wakeup for battery, then opens its own WebSocket for the real payload — pure self-hosted push always loses on battery on stock Android.

## Sandboxed Google Play (GrapheneOS)

GrapheneOS installs Play Services, Play Store and GSF as **ordinary unprivileged user apps** inside the standard app sandbox with no signature-spoofing, no platform UID, no special selinux domain. A small compatibility shim translates the privileged calls Play expects (e.g. account manager, geolocation) into AOSP equivalents. FCM works because Play Services itself maintains the socket — same battery profile as stock. Location requests are rerouted to AOSP network location. Play Integrity passes *Basic* and usually *Device* on Pixel hardware (since the bootloader is locked with a user key and the Titan M2 attestation chain is intact) — the only mainstream non-Google-OS that achieves this.

Tradeoffs vs microG: Play Services still runs and still phones home (now firewalled and permission-gated, not eliminated); Pixel-only hardware support; Play Integrity *Strong* still fails on some apps.

## Synthesis for Promethea

Best stack for "Android apps work, including banking, without ever calling Google":

1. **Linux mobile host** for native UX and Linux app ecosystem.
2. **Waydroid container with a GrapheneOS-style sandboxed-Play layer** as the Android runtime, with hardware-backed keystore exposed via virtio. **Crucially: container runs in a pKVM / crosvm VM, not LXC.** Plain Waydroid LXC is not a security boundary. Fall back to microG for users who don't need banking.
3. **UnifiedPush as the native push fabric** for FOSS apps, with an FCM-bridge distributor (running inside the Waydroid VM) for closed-source apps.
4. **Operator-run ntfy / autopush cluster** (multi-region, RFC 8291 E2EE, battery-tuned 3-min keepalive) as the default distributor.

Highest-risk technical bets:

- **Bet 1 — Play Integrity on a non-Pixel device.** Without Google-provisioned attestation keys baked at the factory, Device verdict is unreachable. This is the make-or-break for banking. Mitigation requires either Pixel-class hardware partnership or a regulator-grade alternative (eIDAS wallet attestation, RBI / NPCI hardware-key whitelisting), and hope that EU DMA pressure cracks the attestation monopoly before 2027.
- **Bet 2 — Waydroid as a security boundary on ARM.** Shared kernel + bind-mounted device nodes mean a hostile Android app can target host kernel bugs. Productizing requires moving to a KVM / crosvm-based VM model (à la Android's pKVM / ChromeOS ARCVM), which today is unproven on PinePhone-class SoCs.
- **Bet 3 — UnifiedPush battery and reliability parity with FCM** when Google Play Services isn't running. Doze whitelisting, 3G/5G NAT timeouts, and reconnection economics are unsolved at scale; ntfy.sh is a single droplet today.

## Sources

- https://waydro.id/
- https://thedocs.io/waydroid/core_concepts/security/
- https://wiki.archlinux.org/title/Waydroid
- https://github.com/waydroid/waydroid/issues/2229
- https://github.com/WayDroid-ATV/waydroid-builds
- https://github.com/waydroid/waydroid/issues/1113
- https://gitlab.com/postmarketOS/pmaports/-/issues/1944
- https://forum.pine64.org/showthread.php?tid=16644
- https://en.wikipedia.org/wiki/MicroG
- https://doc.e.foundation/support-topics/micro-g
- https://github.com/microg/GmsCore/releases
- https://factually.co/product-reviews/electronics-tech/microg-vs-grapheneos-sandboxed-play-services-privacy-app-compatibility-36e9cd
- https://www.androidauthority.com/google-play-integrity-hardware-attestation-3561592/
- https://android-developers.googleblog.com/2025/10/stronger-threat-detection-simpler.html
- https://approov.io/blog/limitations-of-google-play-integrity-api-ex-safetynet
- https://www.privacyportal.co.uk/blogs/free-rooting-tips-and-tricks/play-integrity-in-2026-basic-vs-device-vs-strong-what-actually-matters
- https://unifiedpush.org/users/distributors/
- https://unifiedpush.org/users/apps/
- https://f-droid.org/en/2026/01/08/unifiedpush-5-years.html
- https://docs.ntfy.sh/faq/
- https://github.com/binwiederhier/ntfy/issues/190
- https://datatracker.ietf.org/doc/html/rfc8030
- https://www.rfc-editor.org/rfc/rfc8291.html
- https://github.com/mozilla-services/autopush/blob/master/docs/architecture.rst
- https://designgurus.substack.com/p/push-notification-architecture-apns
- https://grapheneos.org/features
- https://grapheneos.social/@GrapheneOS/113459782313987260
