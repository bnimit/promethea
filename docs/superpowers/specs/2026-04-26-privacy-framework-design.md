# Privacy & Permission Framework — Design Spec

| | |
| --- | --- |
| **Status** | Draft, under review |
| **Date** | 2026-04-26 |
| **Author** | brainstorming session, project sponsor `nimitbhandari17@gmail.com` |
| **Type** | Focused implementation spec for the first component |
| **Companion** | [Vision doc](2026-04-26-promethea-vision-design.md) — the program-level design |
| **Subsystem** | `services/privacy/` |
| **Daemon name (working)** | `promethead-privacy` |
| **Target version** | v0.1 — runs as standalone userspace daemon on AOSP / postmarketOS |
| **Deferred features** | tracked in [`docs/roadmaps/privacy-framework.md`](../../roadmaps/privacy-framework.md) |
| **Research basis** | [`docs/research/04-privacy-permission-framework.md`](../../research/04-privacy-permission-framework.md), [`docs/research/03-android-compat-and-push.md`](../../research/03-android-compat-and-push.md) |

## 1. Purpose

Replace ambient permissions ("the app has the camera permission") with **typed, single-shot, ledger-logged capability handles handed out by a system-trusted broker UI** — and act as the integration spine for the OS's other privacy guarantees (network egress policy, telemetry suppression, lockdown profiles, regional compliance).

The daemon is the first thing apps talk to before touching any sensitive resource and the only thing that can grant access to one. Its API is the OS's privacy contract.

## 2. Threat model

### In scope

- Malicious apps escalating ambient permissions.
- Apps fingerprinting via sensor side-channels.
- Apps exfiltrating data over network outside their declared purpose.
- UI-redress / clickjacking on grant prompts.
- Tampered or stale regional policy.
- Userspace tamper of the daemon itself.

### Out of scope (handled by other subsystems)

- Kernel exploits — kernel hardening (`kernel/`).
- TEE compromise — hardware-attestation chain (`frameworks/identity/`).
- Supply-chain attacks on third-party libs — CRA SBOM pipeline (`tools/`, `ci/`).
- Android-VM escape — pKVM / crosvm isolation (`android-runtime/`).
- Physical tamper — per-variant hardware concern.

## 3. Architecture overview

```
                ┌───────────────────────────────┐
   App ──IPC──▶ │   promethead-privacy         │ ◀── Regional Manifest
   (typed       │   (Rust daemon, single proc)  │     (signed; loaded at boot)
    capability  │                               │
    request)    │  ┌─────────────────────────┐  │
                │  │  Capability Broker      │  │
                │  │  + Stated-Purpose       │  │
                │  │    Contract Validator   │  │
                │  └────────────┬────────────┘  │
                │               │               │
                │  ┌────────────▼────────────┐  │
                │  │  Policy Engine          │  │     ┌──────────────────┐
                │  │  (default-deny;         │◀─┼─────│ Lockdown Profile │
                │  │   shim-deny; ask-user)  │  │     │ Manager          │
                │  └────────────┬────────────┘  │     └──────────────────┘
                │               │               │
                │  ┌────────────▼────────────┐  │     ┌──────────────────┐
                │  │  Portal UI Coordinator  │◀─┼─────│ User-Gesture     │
                │  │  (system-trusted UI)    │  │     │ Evidence Source  │
                │  └────────────┬────────────┘  │     │ (compositor)     │
                │               │               │     └──────────────────┘
                │  ┌────────────▼────────────┐  │
                │  │  Capability Handle      │  │
                │  │  Issuer (FD-based)      │  │
                │  └────────────┬────────────┘  │
                │               │               │
                │  ┌────────────▼────────────┐  │     ┌──────────────────┐
                │  │  Append-Only Ledger     │──┼────▶│ Audit Log        │
                │  │  (signed, hash-chained) │  │     │ (rotated, OHTTP) │
                │  └─────────────────────────┘  │     └──────────────────┘
                │                               │
                │  ┌─────────────────────────┐  │     ┌──────────────────┐
                │  │  Egress Policy Enforcer │──┼────▶│ netd / nftables  │
                │  └─────────────────────────┘  │     └──────────────────┘
                └───────────────────────────────┘
```

The daemon is a single Rust binary running as a privileged userspace service. It is the *only* component that can broker access to sensitive resources; system services and apps both must go through it.

## 4. Core abstractions

The Rust types below are sketches — final names and field details will be settled by the implementation plan, but the conceptual shape is fixed.

```rust
/// Unforgeable handle: a typed file descriptor returned by the broker.
/// Drop the FD, lose the access. No revocation problem.
pub struct CapabilityHandle<R: Resource> {
    fd: OwnedFd,
    _r: PhantomData<R>,
}

/// What kind of resource the handle grants access to.
pub trait Resource: sealed::Sealed {
    const KIND: CapabilityKind;
}

pub enum CapabilityKind {
    Camera,
    Microphone,
    Location { precision: LocationPrecision },
    Contacts,
    Photos,
    Storage,
    SensorReadings(SensorClass),
    Network { destinations: Vec<DestPolicy> },
    Keystore { key_id: KeyId },
    Telephony,
    // …extensible per subsystem RFC
}

/// What the app declared in its signed manifest.
pub struct StatedPurpose {
    pub capability: CapabilityKind,
    pub purpose: PurposeTag,           // Analytics, Messaging, MapsTile, …
    pub network_destinations: Vec<DestPolicy>,
    pub data_minimization: DataScope,  // Single, Curated, Full
}

/// Single grant decision.
pub struct GrantRecord {
    pub app_id: AppId,
    pub capability: CapabilityKind,
    pub scope: ResourceScope,                 // Photo(uuid), Folder(path)
    pub purpose: PurposeTag,
    pub timestamp_utc: i64,
    pub gesture_evidence: GestureProof,       // signed input event
    pub manifest_hash: [u8; 32],              // SHA-256 of app manifest
    pub ledger_seq: u64,
    pub ledger_prev_hash: [u8; 32],           // hash chain for tamper-evidence
    pub region: RegionCode,
    pub profile: LockdownProfileId,
}

/// Plug-in points for the Regional Capability Layer.
pub trait IdentityProvider { /* eIDAS / Aadhaar+DigiLocker / OAuth */ }
pub trait PaymentService    { /* UPI Intent / SEPA / no-op */ }
pub trait LanguageService   { /* on-device Bhashini / FOSS base */ }

/// Loaded at boot; signed by the Foundation or a downstream variant maintainer.
pub struct RegionalManifest {
    pub region: RegionCode,
    pub identity_provider: Box<dyn IdentityProvider>,
    pub payment_service:   Box<dyn PaymentService>,
    pub language_service:  Box<dyn LanguageService>,
    pub compliance_daemon: ComplianceTarget,   // GDPR / DPDP / CERT-In / …
    pub default_search_maps: ProviderRefs,
    pub allowed_app_stores: Vec<StoreId>,
    pub telemetry_policy: TelemetryPolicy,
    pub denial_default: DenialMode,            // HardDeny | ShimDeny
    pub signature: Ed25519Signature,
}
```

The **`CapabilityHandle<R>`** is the conceptual move that defines the design. The app never holds a permission *string*; it holds an unforgeable typed file descriptor that *is* its access. Permissions become artifacts of capability flow rather than ambient grants.

## 5. Components

### 5.1 Capability Broker

Receives typed IPC requests over Cap'n Proto / Unix socket. Validates the request against the app's signed manifest (Stated-Purpose Contract). Routes the validated request to the Policy Engine.

### 5.2 Policy Engine

Pure-function decision:

```rust
fn decide(
    req: &CapabilityRequest,
    app: &AppId,
    manifest: &RegionalManifest,
    profile: &LockdownProfile,
    ledger: &LedgerSnapshot,
) -> Decision;

pub enum Decision {
    Grant(ResourceScope),
    ShimDeny(ShimResource),       // app sees curated/empty result
    HardDeny(DenialReason),
    AskUser(PortalRequest),
    GrantWithEgressLimit(ResourceScope, EgressPolicy),
}
```

Pure functions are testable with property-based tests. State that affects decisions (ledger history, active profile, manifest) is passed in explicitly.

### 5.3 Portal UI Coordinator

Renders system-trusted picker UIs on the compositor — the *broker UI itself* picks the resource so the app receives only the chosen handle. Decoration is unspoofable; reads gesture evidence from the compositor (signed input event).

Pickers v0.1 includes:

- File picker
- Photo picker (per-asset selection)
- Contact picker (single contact or filtered subset)
- Location picker (one-shot vs always; precision selection)
- "Allow once" / "Allow while in use" / "Allow always" buttons distinguished by gesture evidence

### 5.4 Capability Handle Issuer

Wraps the resolved resource as an `OwnedFd` of the correct type and passes it via SCM_RIGHTS to the requesting app process. Resource backends are pluggable (file FD for files, anonymous FD for in-memory views, character device for camera, etc.).

### 5.5 Append-Only Ledger

Local SQLite (or sled) store; every grant signed and hash-chained. Never delete — rotate to encrypted cold storage. Schema:

| Column | Type | Notes |
| --- | --- | --- |
| `seq` | u64 | monotonic |
| `prev_hash` | [u8; 32] | SHA-256 of previous row |
| `record_blob` | bytes | CBOR-encoded `GrantRecord` |
| `signature` | bytes | Ed25519 over `(seq \|\| prev_hash \|\| record_blob)` |
| `wall_clock_utc` | i64 | for human inspection only |

Verification: walking the chain from row 0 must reproduce all hashes; any divergence indicates tamper.

### 5.6 Lockdown Profile Manager

Holds named profiles (`Default`, `Travel`, `BorderCrossing`, `Activist`, `KioskMode`, etc.) — each a declarative bundle of capability defaults, network rules, attestation requirements. Profile switch is itself an audited event in the ledger. Switch via:

- Hardware-mapped key combo (compositor-detected; user-configurable per device).
- MDM directive (for retail / fleet).
- User-initiated from Settings.
- Geofence trigger (off by default; opt-in per profile).

### 5.7 Egress Policy Enforcer

Translates per-capability network destinations into nftables rules; enforces them as long as the handle is alive. DNS-over-HTTPS broker per-app to defeat side-channel resolution. When a handle is dropped (FD closed), the kernel notifies the daemon; egress rules are removed.

### 5.8 Stated-Purpose Contract Validator

Loads the app's signed Privacy Manifest at install time and on every launch, verifies signature, computes hash. Surfaces conflicts (e.g., manifest declares "Analytics-only" but app requested Contacts → fail closed and log).

### 5.9 Regional Manifest Engine

Loads the active signed regional manifest at boot; exposes the region-specific provider traits (`IdentityProvider`, `PaymentService`, `LanguageService`) to other system services via a stable in-process API. Hot-reloads on signed update. Manifest updates are themselves audited events.

### 5.10 Hardware-Attested Gate

Sensitive capabilities (decrypt user data, access keystore) require an attestation challenge proving the daemon binary is unmodified. *Deferred to v0.2* — see [`docs/roadmaps/privacy-framework.md`](../../roadmaps/privacy-framework.md). v0.1 has a stub that records "attestation: not-yet-implemented" in the ledger.

## 6. Key data flow — capability request

1. App calls `broker.request_capability(StatedPurpose, scope_hint)` over IPC.
2. Capability Broker fetches the app's manifest hash from local cache; validates `StatedPurpose ⊆ Manifest`. If not, `HardDeny` and ledger-log.
3. Policy Engine consults: ledger history (`ledger.first_seen(app, capability)`, `ledger.recent_denials(app)`), active regional manifest, active lockdown profile, app trust tier. Emits `Decision`.
4. If `AskUser`: Portal UI Coordinator renders a system-trusted picker; user gesture is captured by the compositor and signed as `GestureProof`. User picks the *specific resource* (file, photo, contact). Decision becomes `Grant` over that resource only.
5. Capability Handle Issuer opens the resource as an FD, wraps in a typed `CapabilityHandle<R>`, passes to app via SCM_RIGHTS.
6. Ledger appends a `GrantRecord`, hash-chains to the previous entry, signs.
7. Egress Policy Enforcer activates nftables rules for the handle's lifetime.
8. App uses the FD. When the app closes / drops it, the kernel notifies the daemon; egress rules are removed; revocation is automatic.

## 7. Error handling — four denial modes

| Mode | When | App-visible result |
| --- | --- | --- |
| **HardDeny** | Manifest violation; lockdown profile forbids; hardware attestation fails | IPC error with stable error code; recommended UX is to fail loudly. |
| **ShimDeny** | App refuses graceful denial; regional policy says "don't break the app" | Returns curated / empty resource (empty contact list, dummy GPS, redacted file list). App thinks it succeeded. |
| **AskUser** | First-use; expired grant; sensitive scope | Portal UI; user gesture required. |
| **GrantWithEgressLimit** | Granted but the manifest declares no network destinations | Handle issued; nftables blocks all egress for the duration. |

**Default policy is `HardDeny`.** `ShimDeny` is opt-in per region. The EU regional manifest defaults to `HardDeny` for prosumer auditability and transparent failure. The India regional manifest defaults more towards `ShimDeny` because app breakage is more user-hostile in low-tech-literacy contexts. Retail / fleet defaults are MDM-controlled per deployment.

The default per region is declared in the `RegionalManifest::denial_default` field; per-capability overrides are allowed inside the manifest.

## 8. Regional manifest semantics

Manifests are loaded once at boot and on signed update. They:

- Bind the region's `IdentityProvider`, `PaymentService`, `LanguageService` impls.
- Declare allowed app store(s), default search and maps providers.
- Declare the compliance daemon target (GDPR / DPDP / CERT-In / global).
- Set the regional default denial mode and any per-capability overrides.
- Configure the telemetry policy (always OHTTP-bucketed; the policy controls whether telemetry is enabled at all and which buckets are allowed).
- Provide the trusted certificate roots specific to the region.

Manifest signing keys are held by the Foundation for the upstream `regions/global/` manifest, and by downstream variant maintainers for `regions/eu/`, `regions/in/`, etc. Variant maintainers are listed in `MAINTAINERS.md`. Manifest format: CBOR/COSE (preferred) or signed JSON. Schema lives at `frameworks/identity/manifest-schema.cddl` (or equivalent) and is version-tagged.

## 9. Telemetry / observability

**No opt-out telemetry path exists in the build.** Telemetry by construction:

- Crash reports use on-device symbolication, then bucket-hash the resulting stack to a bounded set of pre-known buckets, then submit `(bucket_id, count)` via OHTTP relay.
- Aggregate metrics use local differential privacy (Laplace noise injection at on-device aggregation time).
- The OHTTP relay is operated by the Foundation (or per region by the variant maintainer); it strips client IPs and forwards anonymized requests to the analytics endpoint.

The user-facing audit log (the ledger) is local-only by default. Optional opt-in OHTTP-relay export to a user-controlled endpoint is possible; default off.

## 10. Testing strategy

- **Unit:** every component as a pure-Rust crate; coverage target ≥ 90 % lines and ≥ 95 % on the Policy Engine. Property tests via `proptest` for the policy decision function.
- **Integration:** test harness simulates an app process making IPC calls; assert correct handle vs. denial vs. shim behavior across the **manifest × region × profile** matrix.
- **Fuzzing:** `cargo-fuzz` against the IPC parser, manifest parser, and ledger replay.
- **Formal modeling:** Policy Engine specified in a small TLA+ or Alloy model. Invariants:
    1. No app ever holds two handles to the same resource simultaneously.
    2. Ledger is hash-chain-consistent under concurrent writes.
    3. Lockdown profile changes never widen access.
    4. Regional manifest signature is verified before use.
    5. A `HardDeny` decision never leaks resource scope information back to the app.
- **Threat-model conformance:** scripted attacks (UI redress, gesture replay, manifest tamper, ledger gap, FD smuggling) — must all fail-closed.
- **End-to-end on real hardware:** Pixel 4a + Fairphone 5 daily soak in Phase 1 (M3-M9). Daily build runs the full integration matrix.

## 11. Non-goals (deferred to future versions)

Each non-goal is tagged with the version it lands in and the dependencies that gate it. The full backlog with milestone targets lives in [`docs/roadmaps/privacy-framework.md`](../../roadmaps/privacy-framework.md); each non-goal here cross-references that file.

- **No full pKVM / crosvm Android-VM integration in v0.1.** → **v0.2** (Phase 2). Daemon runs on existing AOSP / postmarketOS first; the VM-isolated Android container comes when the OS gains its own compositor and HAL. *Depends on:* `android-runtime/` subsystem reaching alpha; `compositor/` reaching alpha. See [`roadmaps/privacy-framework.md § Android VM integration`](../../roadmaps/privacy-framework.md#android-vm-integration).
- **No hardware-attested gate in v0.1.** → **v0.2** (Phase 2). Added once we have a reference device with TPM / StrongBox-class attestation and a working `frameworks/identity/` HW-attest impl. *Depends on:* `frameworks/identity/` shipping a `HardwareAttestor` trait impl. See [`roadmaps/privacy-framework.md § Hardware-attested gate`](../../roadmaps/privacy-framework.md#hardware-attested-gate).
- **No federated cross-device ledger.** → **v0.4** (Phase 3). Single-device scope only in v0.1-v0.3. Federation requires a separate RFC for cross-device identity, conflict resolution, and distributed-trust model. *Depends on:* identity / device-pairing primitives. See [`roadmaps/privacy-framework.md § Federated ledger`](../../roadmaps/privacy-framework.md#federated-ledger).
- **No user-facing GUI for ledger inspection in v0.1.** → **v0.3** (Phase 3). CLI tool only in v0.1. GUI ships when `shell/` reaches alpha and the system Settings app exists. *Depends on:* `shell/` and `apps/settings/` reaching alpha. See [`roadmaps/privacy-framework.md § Ledger inspection GUI`](../../roadmaps/privacy-framework.md#ledger-inspection-gui).
- **No Bhashini / UPI Intent / eIDAS bindings in v0.1.** → **v0.2-v0.4** (Phases 2-4). Define the *traits* in v0.1, ship one default impl per region in v0.1 (FOSS-base language, no-op payment, generic OAuth identity), leave full impls for the relevant regional phases. *Depends on:* per-impl, Bhashini SDK availability and on-device model packaging; NPCI / RBI engagement; eIDAS 2.0 wallet SDK release. See [`roadmaps/privacy-framework.md § Regional service impls`](../../roadmaps/privacy-framework.md#regional-service-impls).
- **No per-app DNS-over-HTTPS broker in v0.1.** → **v0.2**. v0.1 enforces egress at L4 (nftables) only. *Depends on:* `services/push/` and DoH broker library landing. See [`roadmaps/privacy-framework.md § Per-app DoH broker`](../../roadmaps/privacy-framework.md#per-app-doh-broker).
- **No geofence-triggered profile switching in v0.1.** → **v0.3**. Manual / hardware-key / MDM switching only in v0.1. *Depends on:* a `LocationService` impl with privacy-respecting geofence primitive. See [`roadmaps/privacy-framework.md § Geofence profile switch`](../../roadmaps/privacy-framework.md#geofence-profile-switch).
- **No formal verification of the full daemon in v0.1.** → **v0.5+** (Phases 5+). v0.1-v0.3 ship TLA+ / Alloy models of the Policy Engine only; full Verus / Creusot proof of the broker comes after the API stabilizes. See [`roadmaps/privacy-framework.md § Formal verification`](../../roadmaps/privacy-framework.md#formal-verification).
- **No multi-tenant / work-profile isolation in v0.1.** → **v0.3**. Single-user only in v0.1. *Depends on:* `init/` and user-account primitives. See [`roadmaps/privacy-framework.md § Multi-user profiles`](../../roadmaps/privacy-framework.md#multi-user-profiles).

## 12. Open questions (answered by implementation plan, not now)

- **IPC framework choice:** Cap'n Proto vs zbus (D-Bus successor) vs custom — bake-off in Phase 0.
- **Ledger storage:** SQLite vs sled vs flat append-only file with B-tree index — decision in Phase 0 prototype.
- **Manifest format:** signed JSON vs CBOR/COSE — leaning CBOR/COSE for size and crypto-native signing; final decision in Phase 1.
- **Where the daemon's own attestation key lives in v0.1** before HW root-of-trust is available — likely a long-lived locally-generated key with a clear migration path to HW-backed in v0.2.
- **Default policy on first-launch.** Do we ask user once for "all sensors disabled by default"-type defaults at provisioning, or rely on per-capability `AskUser`? Decision in Phase 1 UX bake-off.

## 13. Acceptance criteria for v0.1

The v0.1 release is considered complete when:

1. The daemon builds reproducibly under CI and is reproducible across at least two distinct CI runners.
2. End-to-end: an unmodified third-party Android-on-AOSP app runs through the daemon and receives a `CapabilityHandle` for one resource of each kind (camera, mic, location, contacts, photos, storage, network).
3. Each of the four denial modes (`HardDeny`, `ShimDeny`, `AskUser`, `GrantWithEgressLimit`) is exercised by integration tests on real hardware.
4. The ledger survives 10,000 grants without performance degradation > 5 % over the empty baseline; verification of the full chain is < 500 ms.
5. The policy engine's TLA+ / Alloy model checks all 5 stated invariants with no counterexamples.
6. The threat-model conformance suite (UI redress, gesture replay, manifest tamper, ledger gap, FD smuggling) passes 100 %.
7. Documentation is published: end-user docs, app-developer SDK, and integrator (downstream variant maintainer) docs.
8. SBOM is generated, signed, and published for every release artifact.
9. The daemon runs on at least Pixel 4a / Fairphone 5 / postmarketOS reference and an x86_64 dev host, with the same source.

## 14. Out-of-scope risks

- This spec does *not* close the Play Integrity gap for Indian banking apps; that remains a regulatory engagement (RBI / NPCI hardware-attestation key whitelisting) tracked by the vision doc. The privacy framework provides the *substrate* for hardware-attestation hooks but cannot, by itself, satisfy banking attestation.
- This spec does *not* solve push-delivery; that is a separate subsystem under `services/push/`. The daemon only mediates *which* apps may use push and under what egress policy.
- This spec does *not* implement OTA, app installation, MDM, or compositor work — those are separate subsystems that integrate with the daemon's IPC surface.
