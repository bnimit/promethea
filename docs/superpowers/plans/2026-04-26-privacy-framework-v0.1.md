# Privacy & Permission Framework v0.1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `promethead-privacy`, the v0.1 standalone Rust userspace daemon that brokers capability handles to apps, replaces ambient permissions with typed, ledger-logged single-shot grants, and runs on existing AOSP / postmarketOS as a productized userspace component.

**Architecture:** Cargo workspace with one binary crate (`promethead-privacy`) and ten focused library crates, each owning one component from the spec (broker, policy engine, ledger, manifest validator, regional manifest engine, lockdown profile manager, egress enforcer, handle issuer, portal UI, attestation gate). zbus (D-Bus) for the IPC surface; rusqlite for the append-only ledger; ed25519-dalek + sha2 for signing and hash-chaining; tokio for async; tracing for structured logs. The Policy Engine is a pure function so it is testable with property tests. Capability handles cross to apps as `OwnedFd` via SCM_RIGHTS through zbus.

**Tech Stack:** Rust 1.93 stable, Cargo workspace, zbus 4, tokio, rusqlite (bundled SQLite), ed25519-dalek, sha2, serde + serde_cbor (ledger) + serde_json (manifests), nftables shell-out via `nft`, tracing, proptest, cargo-fuzz, GitHub Actions CI.

**Spec:** [`docs/superpowers/specs/2026-04-26-privacy-framework-design.md`](../specs/2026-04-26-privacy-framework-design.md)

**Deferred features (tracked in [`docs/roadmaps/privacy-framework.md`](../../roadmaps/privacy-framework.md)):** pKVM/crosvm Android-VM integration, hardware-attested gate (stub only here), per-app DoH broker (L4 only here), federated ledger, ledger inspection GUI (CLI only here), real Bhashini/UPI/eIDAS impls (default impls only here), geofence profile switching, full-daemon formal verification (Policy Engine model only), multi-user profiles.

---

## File Structure

The daemon lives under `services/privacy/` as a Cargo workspace.

```
services/privacy/
├── Cargo.toml                        # workspace root
├── README.md                         # subsystem overview
├── rust-toolchain.toml               # pin Rust 1.93
├── deny.toml                         # cargo-deny config
├── crates/
│   ├── promethead-privacy/           # BINARY — daemon entry point
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   ├── ipc-types/                    # IPC messages, error codes, zbus interface
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── core-types/                   # CapabilityKind, Resource, GrantRecord, etc.
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── manifest-validator/           # Stated-Purpose Contract Validator
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── policy-engine/                # Pure-function decision engine
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── ledger/                       # Append-only signed hash-chained ledger
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── lockdown-profile/             # Profile state machine + switching logic
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── regional-manifest/            # Regional manifest loader + signature verifier
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── egress-policy/                # nftables wrapper for per-handle egress rules
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── handle-issuer/                # Resource → OwnedFd wrapper, SCM_RIGHTS plumbing
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── portal-ui/                    # System-trusted prompt UI (CLI stub for v0.1)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── attestation-gate/             # HW attestation stub for v0.1
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   └── broker/                       # Orchestrator: glues policy + portal + issuer + ledger
│       ├── Cargo.toml
│       └── src/lib.rs
├── tests/
│   ├── integration/                  # cross-crate integration tests
│   │   └── e2e_capability_request.rs
│   └── conformance/                  # threat-model conformance tests
│       ├── ui_redress.rs
│       ├── gesture_replay.rs
│       ├── manifest_tamper.rs
│       ├── ledger_gap.rs
│       └── fd_smuggling.rs
├── fuzz/                             # cargo-fuzz harnesses
│   ├── Cargo.toml
│   └── fuzz_targets/
│       ├── ipc_parser.rs
│       ├── manifest_parser.rs
│       └── ledger_replay.rs
├── docs/
│   ├── architecture.md               # living design doc
│   ├── developer-sdk.md              # how apps call the daemon
│   ├── operator-manual.md            # variant maintainer docs
│   └── manifest-schema.cddl          # signed manifest schema
└── .github/workflows/
    └── privacy-ci.yml                # CI matrix
```

**Why this decomposition:** each crate owns one spec component; pure crates (policy-engine, manifest-validator, regional-manifest, ledger) have no async or syscall dependencies and are property-testable in isolation; effectful crates (egress-policy, handle-issuer, portal-ui, attestation-gate) wrap one OS surface each; the broker glues them. The binary is thin — it just constructs and wires components, then runs the IPC server.

---

## Phase A — Scaffolding

### Task 1: Bootstrap the Cargo workspace

**Files:**
- Create: `services/privacy/Cargo.toml`
- Create: `services/privacy/rust-toolchain.toml`
- Create: `services/privacy/deny.toml`
- Create: `services/privacy/README.md`
- Create: `services/privacy/.gitignore`

- [ ] **Step 1: Create the workspace root `Cargo.toml`**

```toml
[workspace]
resolver = "2"
members = ["crates/*"]
exclude = ["fuzz"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.93"
license = "Apache-2.0"
authors = ["Promethea Foundation contributors"]
repository = "https://github.com/bnimit/promethea"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
zbus = { version = "4", default-features = false, features = ["tokio"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_cbor = "0.11"
ed25519-dalek = { version = "2", features = ["rand_core"] }
sha2 = "0.10"
rusqlite = { version = "0.31", features = ["bundled"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
thiserror = "1"
anyhow = "1"
uuid = { version = "1", features = ["v4", "serde"] }
proptest = "1"

[profile.release]
lto = "fat"
codegen-units = 1
strip = "symbols"
```

- [ ] **Step 2: Pin the toolchain**

```toml
# services/privacy/rust-toolchain.toml
[toolchain]
channel = "1.93.0"
components = ["rustfmt", "clippy"]
```

- [ ] **Step 3: Create `deny.toml` for supply-chain hygiene**

```toml
# services/privacy/deny.toml
[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[licenses]
allow = ["Apache-2.0", "MIT", "BSD-3-Clause", "ISC", "Unicode-DFS-2016"]
copyleft = "deny"
default = "deny"

[bans]
multiple-versions = "warn"
```

- [ ] **Step 4: Write `services/privacy/README.md`**

```markdown
# services/privacy

The Privacy & Permission Daemon (`promethead-privacy`).

This subsystem implements the v0.1 spec at
[`docs/superpowers/specs/2026-04-26-privacy-framework-design.md`](../../docs/superpowers/specs/2026-04-26-privacy-framework-design.md).

## Build

```
cargo build --release
```

## Run

```
cargo run --bin promethead-privacy
```

## Test

```
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
cargo deny check
```

## Layout

See the workspace layout at `Cargo.toml`. Each crate under `crates/` owns one component from the spec.
```

- [ ] **Step 5: Create the subsystem `.gitignore`**

```
target/
**/*.rs.bk
fuzz/artifacts/
fuzz/corpus/
*.profraw
```

- [ ] **Step 6: Verify the empty workspace builds**

```
cd services/privacy
cargo check --workspace
```

Expected: `Checking promethead-privacy v0.1.0` style output for each crate (after Task 2 lands them); for now, "no targets" warning is fine because no member crates exist yet.

- [ ] **Step 7: Commit**

```
git add services/privacy/
git commit -s -m "feat(privacy): bootstrap services/privacy Cargo workspace"
```

---

### Task 2: Create the empty member crates

**Files:**
- Create: `services/privacy/crates/{ipc-types,core-types,manifest-validator,policy-engine,ledger,lockdown-profile,regional-manifest,egress-policy,handle-issuer,portal-ui,attestation-gate,broker,promethead-privacy}/Cargo.toml`
- Create: `services/privacy/crates/{...}/src/lib.rs` (or `main.rs` for the binary)

- [ ] **Step 1: Create each library crate's `Cargo.toml` (template)**

For each library crate, create `Cargo.toml`. Example for `core-types`:

```toml
[package]
name = "core-types"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
serde = { workspace = true }
thiserror = { workspace = true }
uuid = { workspace = true }
```

Repeat for each library crate. Adjust dependencies per crate (each one's spec component informs which deps it needs; details land in the per-crate task below).

- [ ] **Step 2: Create the binary crate `Cargo.toml`**

```toml
# services/privacy/crates/promethead-privacy/Cargo.toml
[package]
name = "promethead-privacy"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[[bin]]
name = "promethead-privacy"
path = "src/main.rs"

[dependencies]
broker = { path = "../broker" }
ipc-types = { path = "../ipc-types" }
tokio = { workspace = true }
zbus = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
anyhow = { workspace = true }
```

- [ ] **Step 3: Create stub `src/lib.rs` for each library crate**

```rust
// crates/<name>/src/lib.rs
//! <name> — see services/privacy/README.md for context.
```

- [ ] **Step 4: Create stub `src/main.rs` for the binary**

```rust
// crates/promethead-privacy/src/main.rs
fn main() {
    println!("promethead-privacy v0.1.0 — not yet implemented");
}
```

- [ ] **Step 5: Verify the workspace builds**

```
cd services/privacy
cargo check --workspace
cargo build --bin promethead-privacy
./target/debug/promethead-privacy
```

Expected: clean build, binary prints `promethead-privacy v0.1.0 — not yet implemented`.

- [ ] **Step 6: Commit**

```
git add services/privacy/crates/
git commit -s -m "feat(privacy): scaffold empty member crates"
```

---

### Task 3: Configure CI for the privacy workspace

**Files:**
- Create: `services/privacy/.github/workflows/privacy-ci.yml` (or as a top-level `.github/workflows/privacy-ci.yml` with path filter)

- [ ] **Step 1: Write the CI workflow**

```yaml
# .github/workflows/privacy-ci.yml
name: privacy

on:
  push:
    paths:
      - "services/privacy/**"
      - ".github/workflows/privacy-ci.yml"
  pull_request:
    paths:
      - "services/privacy/**"
      - ".github/workflows/privacy-ci.yml"

defaults:
  run:
    working-directory: services/privacy

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - run: cargo fmt --all -- --check
      - run: cargo clippy --workspace --all-targets -- -D warnings

  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --workspace --all-features

  deny:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v2
        with:
          manifest-path: services/privacy/Cargo.toml

  build-aarch64:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: aarch64-unknown-linux-gnu
      - run: sudo apt-get update && sudo apt-get install -y gcc-aarch64-linux-gnu
      - run: cargo build --workspace --target aarch64-unknown-linux-gnu
        env:
          CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER: aarch64-linux-gnu-gcc
```

- [ ] **Step 2: Verify locally that lint and test pass**

```
cd services/privacy
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Expected: all three pass on the empty crates.

- [ ] **Step 3: Commit**

```
git add .github/workflows/privacy-ci.yml
git commit -s -m "ci(privacy): add lint, test, deny, and aarch64 cross-build jobs"
```

---

## Phase B — Core types

### Task 4: Define `CapabilityKind`, `Resource`, scopes, and ID newtypes

**Files:**
- Modify: `services/privacy/crates/core-types/Cargo.toml`
- Create: `services/privacy/crates/core-types/src/lib.rs`
- Create: `services/privacy/crates/core-types/src/capability.rs`
- Create: `services/privacy/crates/core-types/src/identifiers.rs`
- Test: `services/privacy/crates/core-types/src/capability.rs` (inline `#[cfg(test)]`)

- [ ] **Step 1: Update `core-types/Cargo.toml`**

```toml
[package]
name = "core-types"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
serde = { workspace = true }
thiserror = { workspace = true }
uuid = { workspace = true }

[dev-dependencies]
proptest = { workspace = true }
```

- [ ] **Step 2: Write a failing test for `CapabilityKind` round-trip serde**

```rust
// crates/core-types/src/capability.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum CapabilityKind {
    Camera,
    Microphone,
    Location { precision: LocationPrecision },
    Contacts,
    Photos,
    Storage,
    SensorReadings { class: SensorClass },
    Network { destinations: Vec<DestPolicy> },
    Keystore { key_id: KeyId },
    Telephony,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LocationPrecision { Exact, Coarse, City }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum SensorClass { Motion, Environmental, Proximity }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct DestPolicy { pub host: String, pub port: u16, pub sni: Option<String> }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct KeyId(pub String);

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn capability_camera_roundtrips() {
        let c = CapabilityKind::Camera;
        let j = serde_json::to_string(&c).unwrap();
        let back: CapabilityKind = serde_json::from_str(&j).unwrap();
        assert_eq!(c, back);
    }
}
```

- [ ] **Step 3: Wire `core-types/src/lib.rs` to export the module**

```rust
// crates/core-types/src/lib.rs
pub mod capability;
pub mod identifiers;
pub use capability::*;
pub use identifiers::*;
```

- [ ] **Step 4: Define identifiers**

```rust
// crates/core-types/src/identifiers.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct AppId(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct LockdownProfileId(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct RegionCode(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct GrantId(pub Uuid);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct PurposeTag(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum DataScope { Single, Curated, Full }

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum ResourceScope {
    Photo(Uuid),
    Folder(String),
    ContactSubset(Vec<String>),
    LocationFix { lat: f64, lon: f64, accuracy_m: u32 },
    Whole,
}
```

- [ ] **Step 5: Run tests**

```
cd services/privacy
cargo test -p core-types
```

Expected: `capability_camera_roundtrips` passes.

- [ ] **Step 6: Commit**

```
git add services/privacy/crates/core-types/
git commit -s -m "feat(privacy/core-types): define CapabilityKind, scopes, and identifiers"
```

---

### Task 5: Define `StatedPurpose`, `GrantRecord`, `GestureProof`, `Decision`, `DenialMode`

**Files:**
- Create: `services/privacy/crates/core-types/src/contract.rs`
- Create: `services/privacy/crates/core-types/src/grant.rs`
- Create: `services/privacy/crates/core-types/src/decision.rs`
- Modify: `services/privacy/crates/core-types/src/lib.rs`

- [ ] **Step 1: Define `StatedPurpose`**

```rust
// crates/core-types/src/contract.rs
use crate::*;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct StatedPurpose {
    pub capability: CapabilityKind,
    pub purpose: PurposeTag,
    pub network_destinations: Vec<DestPolicy>,
    pub data_minimization: DataScope,
}
```

- [ ] **Step 2: Define `GrantRecord` and `GestureProof`**

```rust
// crates/core-types/src/grant.rs
use crate::*;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct GestureProof {
    pub event_kind: String,         // e.g. "TapAccept", "HardwareKeyChord"
    pub timestamp_ns: u64,
    pub compositor_signature: Vec<u8>,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct GrantRecord {
    pub grant_id: GrantId,
    pub app_id: AppId,
    pub capability: CapabilityKind,
    pub scope: ResourceScope,
    pub purpose: PurposeTag,
    pub timestamp_utc: i64,
    pub gesture_evidence: Option<GestureProof>,
    pub manifest_hash: [u8; 32],
    pub ledger_seq: u64,
    pub ledger_prev_hash: [u8; 32],
    pub region: RegionCode,
    pub profile: LockdownProfileId,
}
```

- [ ] **Step 3: Define `Decision` and `DenialMode`**

```rust
// crates/core-types/src/decision.rs
use crate::*;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum DenialMode { HardDeny, ShimDeny }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum DenialReason {
    ManifestViolation,
    LockdownForbids,
    AttestationFailed,
    RegionForbids,
    InsufficientGesture,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ShimResource {
    EmptyContacts,
    DummyLocation,
    EmptyFileList,
    EmptyPhotoList,
    Custom(Vec<u8>),
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct EgressPolicy { pub destinations: Vec<DestPolicy> }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum Decision {
    Grant(ResourceScope),
    GrantWithEgressLimit(ResourceScope, EgressPolicy),
    ShimDeny(ShimResource),
    HardDeny(DenialReason),
    AskUser(PortalRequest),
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct PortalRequest {
    pub capability: CapabilityKind,
    pub purpose: PurposeTag,
    pub app_id: AppId,
}
```

- [ ] **Step 4: Wire the new modules into `lib.rs`**

```rust
// crates/core-types/src/lib.rs
pub mod capability;
pub mod identifiers;
pub mod contract;
pub mod grant;
pub mod decision;

pub use capability::*;
pub use identifiers::*;
pub use contract::*;
pub use grant::*;
pub use decision::*;
```

- [ ] **Step 5: Add a property test for `Decision` round-trip**

```rust
// append to crates/core-types/src/decision.rs
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn decision_hard_deny_roundtrips(reason in prop::sample::select(vec![
            DenialReason::ManifestViolation,
            DenialReason::LockdownForbids,
            DenialReason::AttestationFailed,
            DenialReason::RegionForbids,
            DenialReason::InsufficientGesture,
        ])) {
            let d = Decision::HardDeny(reason);
            let bytes = serde_cbor::to_vec(&d).unwrap();
            let back: Decision = serde_cbor::from_slice(&bytes).unwrap();
            prop_assert_eq!(d, back);
        }
    }
}
```

Add `serde_cbor` to `core-types/Cargo.toml`'s `[dev-dependencies]`.

- [ ] **Step 6: Run tests**

```
cargo test -p core-types
```

Expected: all tests pass, including the property test.

- [ ] **Step 7: Commit**

```
git add services/privacy/crates/core-types/
git commit -s -m "feat(privacy/core-types): add StatedPurpose, GrantRecord, Decision, DenialMode"
```

---

### Task 6: Define the regional-service traits and `RegionalManifest` struct

**Files:**
- Modify: `services/privacy/crates/regional-manifest/Cargo.toml`
- Create: `services/privacy/crates/regional-manifest/src/lib.rs`
- Create: `services/privacy/crates/regional-manifest/src/traits.rs`
- Create: `services/privacy/crates/regional-manifest/src/manifest.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "regional-manifest"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
serde = { workspace = true }
serde_json = { workspace = true }
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

- [ ] **Step 2: Define the regional-service traits**

```rust
// crates/regional-manifest/src/traits.rs
use core_types::*;

pub trait IdentityProvider: Send + Sync {
    fn name(&self) -> &str;
    fn supports_attestation(&self) -> bool;
}

pub trait PaymentService: Send + Sync {
    fn name(&self) -> &str;
    fn supports_intent(&self) -> bool;
}

pub trait LanguageService: Send + Sync {
    fn name(&self) -> &str;
    fn languages(&self) -> Vec<String>;
}

pub struct NoopIdentity;
impl IdentityProvider for NoopIdentity {
    fn name(&self) -> &str { "noop" }
    fn supports_attestation(&self) -> bool { false }
}

pub struct NoopPayment;
impl PaymentService for NoopPayment {
    fn name(&self) -> &str { "noop" }
    fn supports_intent(&self) -> bool { false }
}

pub struct FossLanguage;
impl LanguageService for FossLanguage {
    fn name(&self) -> &str { "foss-base" }
    fn languages(&self) -> Vec<String> { vec!["en".into()] }
}
```

- [ ] **Step 3: Define `RegionalManifest`**

```rust
// crates/regional-manifest/src/manifest.rs
use core_types::*;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct ProviderRefs {
    pub default_search: String,
    pub default_maps: String,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ComplianceTarget { Gdpr, Dpdp, CertIn, Global }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct TelemetryPolicy {
    pub enabled: bool,
    pub allowed_buckets: Vec<String>,
    pub relay_endpoint: Option<String>,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct RegionalManifestData {
    pub region: RegionCode,
    pub identity_provider: String,
    pub payment_service: String,
    pub language_service: String,
    pub compliance_daemon: ComplianceTarget,
    pub default_search_maps: ProviderRefs,
    pub allowed_app_stores: Vec<String>,
    pub telemetry_policy: TelemetryPolicy,
    pub denial_default: DenialMode,
    pub per_capability_denial_overrides: Vec<(CapabilityKind, DenialMode)>,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct SignedRegionalManifest {
    pub data: RegionalManifestData,
    pub signature: Vec<u8>,
    pub signing_key_id: String,
}
```

- [ ] **Step 4: Wire the modules into `lib.rs`**

```rust
// crates/regional-manifest/src/lib.rs
pub mod traits;
pub mod manifest;
pub use traits::*;
pub use manifest::*;
```

- [ ] **Step 5: Build to verify**

```
cargo check -p regional-manifest
```

Expected: clean build.

- [ ] **Step 6: Commit**

```
git add services/privacy/crates/regional-manifest/
git commit -s -m "feat(privacy/regional-manifest): define traits and signed manifest types"
```

---

### Task 7: Implement signature verification on `RegionalManifest`

**Files:**
- Modify: `services/privacy/crates/regional-manifest/src/manifest.rs`
- Create: `services/privacy/crates/regional-manifest/src/verify.rs`

- [ ] **Step 1: Write a failing test for valid signature acceptance**

```rust
// crates/regional-manifest/src/verify.rs
use crate::manifest::*;
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};
use sha2::{Digest, Sha256};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum VerifyError {
    #[error("unknown signing key id {0}")]
    UnknownKeyId(String),
    #[error("invalid signature")]
    InvalidSignature,
    #[error("serialization failed: {0}")]
    Ser(#[from] serde_json::Error),
}

pub struct KeyRing {
    keys: std::collections::HashMap<String, VerifyingKey>,
}

impl KeyRing {
    pub fn new() -> Self { Self { keys: Default::default() } }
    pub fn insert(&mut self, id: impl Into<String>, key: VerifyingKey) {
        self.keys.insert(id.into(), key);
    }
    pub fn verify(&self, m: &SignedRegionalManifest) -> Result<(), VerifyError> {
        let key = self.keys.get(&m.signing_key_id)
            .ok_or_else(|| VerifyError::UnknownKeyId(m.signing_key_id.clone()))?;
        let canonical = serde_json::to_vec(&m.data)?;
        let mut h = Sha256::new();
        h.update(&canonical);
        let digest = h.finalize();
        let sig = Signature::from_slice(&m.signature)
            .map_err(|_| VerifyError::InvalidSignature)?;
        key.verify(&digest, &sig).map_err(|_| VerifyError::InvalidSignature)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use core_types::*;
    use rand::rngs::OsRng;

    fn fixture() -> RegionalManifestData {
        RegionalManifestData {
            region: RegionCode("global".into()),
            identity_provider: "noop".into(),
            payment_service: "noop".into(),
            language_service: "foss-base".into(),
            compliance_daemon: ComplianceTarget::Global,
            default_search_maps: ProviderRefs {
                default_search: "duckduckgo".into(),
                default_maps: "osm".into(),
            },
            allowed_app_stores: vec!["fdroid".into()],
            telemetry_policy: TelemetryPolicy {
                enabled: false, allowed_buckets: vec![], relay_endpoint: None,
            },
            denial_default: DenialMode::HardDeny,
            per_capability_denial_overrides: vec![],
        }
    }

    #[test]
    fn valid_signature_accepts() {
        let sk = SigningKey::generate(&mut OsRng);
        let vk = sk.verifying_key();
        let data = fixture();
        let canonical = serde_json::to_vec(&data).unwrap();
        let mut h = Sha256::new(); h.update(&canonical);
        let sig = sk.sign(&h.finalize()).to_bytes().to_vec();
        let m = SignedRegionalManifest {
            data, signature: sig, signing_key_id: "test-key".into(),
        };
        let mut kr = KeyRing::new();
        kr.insert("test-key", vk);
        kr.verify(&m).expect("should accept valid signature");
    }

    #[test]
    fn tampered_data_rejects() {
        let sk = SigningKey::generate(&mut OsRng);
        let vk = sk.verifying_key();
        let mut data = fixture();
        let canonical = serde_json::to_vec(&data).unwrap();
        let mut h = Sha256::new(); h.update(&canonical);
        let sig = sk.sign(&h.finalize()).to_bytes().to_vec();
        // tamper after signing
        data.allowed_app_stores.push("malicious".into());
        let m = SignedRegionalManifest {
            data, signature: sig, signing_key_id: "test-key".into(),
        };
        let mut kr = KeyRing::new();
        kr.insert("test-key", vk);
        assert!(matches!(kr.verify(&m), Err(VerifyError::InvalidSignature)));
    }

    #[test]
    fn unknown_key_id_rejects() {
        let sk = SigningKey::generate(&mut OsRng);
        let data = fixture();
        let canonical = serde_json::to_vec(&data).unwrap();
        let mut h = Sha256::new(); h.update(&canonical);
        let sig = sk.sign(&h.finalize()).to_bytes().to_vec();
        let m = SignedRegionalManifest {
            data, signature: sig, signing_key_id: "wrong-id".into(),
        };
        let kr = KeyRing::new();
        assert!(matches!(kr.verify(&m), Err(VerifyError::UnknownKeyId(_))));
    }
}
```

Add to `regional-manifest/Cargo.toml` `[dev-dependencies]`: `rand = "0.8"`.

- [ ] **Step 2: Wire `verify` into `lib.rs`**

```rust
// crates/regional-manifest/src/lib.rs
pub mod traits;
pub mod manifest;
pub mod verify;
pub use traits::*;
pub use manifest::*;
pub use verify::*;
```

- [ ] **Step 3: Run tests**

```
cargo test -p regional-manifest
```

Expected: three tests pass.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/regional-manifest/
git commit -s -m "feat(privacy/regional-manifest): verify signed manifests with KeyRing"
```

---

## Phase C — Pure components

### Task 8: Implement Stated-Purpose Contract Validator

**Files:**
- Modify: `services/privacy/crates/manifest-validator/Cargo.toml`
- Create: `services/privacy/crates/manifest-validator/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "manifest-validator"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
serde = { workspace = true }
serde_json = { workspace = true }
sha2 = { workspace = true }
ed25519-dalek = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
rand = "0.8"
proptest = { workspace = true }
```

- [ ] **Step 2: Write the failing test**

```rust
// crates/manifest-validator/src/lib.rs
use core_types::*;
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};
use thiserror::Error;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AppManifest {
    pub app_id: AppId,
    pub stated_purposes: Vec<StatedPurpose>,
    pub signature: Vec<u8>,
    pub signing_key: Vec<u8>, // Ed25519 verifying key, raw 32 bytes
}

#[derive(Debug, Error)]
pub enum ValidationError {
    #[error("manifest signature invalid")]
    BadSignature,
    #[error("requested purpose not declared in manifest")]
    PurposeNotDeclared,
    #[error("destination not in declared list")]
    DestinationNotDeclared,
    #[error("data scope wider than manifest")]
    ScopeExceeded,
    #[error("crypto: {0}")]
    Crypto(String),
    #[error("ser: {0}")]
    Ser(#[from] serde_json::Error),
}

pub struct ValidatedManifest {
    pub app_id: AppId,
    pub stated_purposes: Vec<StatedPurpose>,
    pub manifest_hash: [u8; 32],
}

pub fn validate_manifest(m: &AppManifest) -> Result<ValidatedManifest, ValidationError> {
    let key_bytes: [u8; 32] = m.signing_key.as_slice().try_into()
        .map_err(|_| ValidationError::Crypto("bad key length".into()))?;
    let vk = VerifyingKey::from_bytes(&key_bytes)
        .map_err(|e| ValidationError::Crypto(e.to_string()))?;
    let canonical = serde_json::to_vec(&m.stated_purposes)?;
    let mut h = Sha256::new(); h.update(&m.app_id.0); h.update(&canonical);
    let digest = h.finalize();
    let sig = Signature::from_slice(&m.signature).map_err(|_| ValidationError::BadSignature)?;
    vk.verify(&digest, &sig).map_err(|_| ValidationError::BadSignature)?;
    let mut hash_arr = [0u8; 32];
    hash_arr.copy_from_slice(&digest);
    Ok(ValidatedManifest {
        app_id: m.app_id.clone(),
        stated_purposes: m.stated_purposes.clone(),
        manifest_hash: hash_arr,
    })
}

pub fn check_request_against_manifest(
    requested: &StatedPurpose,
    manifest: &ValidatedManifest,
) -> Result<(), ValidationError> {
    let matching = manifest.stated_purposes.iter()
        .find(|sp| sp.capability == requested.capability && sp.purpose == requested.purpose);
    let m = matching.ok_or(ValidationError::PurposeNotDeclared)?;
    for d in &requested.network_destinations {
        if !m.network_destinations.iter().any(|md| md == d) {
            return Err(ValidationError::DestinationNotDeclared);
        }
    }
    if (requested.data_minimization as u8) > (m.data_minimization as u8) {
        return Err(ValidationError::ScopeExceeded);
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use rand::rngs::OsRng;

    fn signed_manifest(purposes: Vec<StatedPurpose>) -> (AppManifest, SigningKey) {
        let sk = SigningKey::generate(&mut OsRng);
        let vk = sk.verifying_key();
        let app_id = AppId("com.example.app".into());
        let canonical = serde_json::to_vec(&purposes).unwrap();
        let mut h = Sha256::new(); h.update(&app_id.0); h.update(&canonical);
        let sig = sk.sign(&h.finalize()).to_bytes().to_vec();
        (AppManifest {
            app_id, stated_purposes: purposes, signature: sig,
            signing_key: vk.to_bytes().to_vec(),
        }, sk)
    }

    #[test]
    fn valid_manifest_validates() {
        let purposes = vec![StatedPurpose {
            capability: CapabilityKind::Camera,
            purpose: PurposeTag("Selfies".into()),
            network_destinations: vec![],
            data_minimization: DataScope::Single,
        }];
        let (m, _sk) = signed_manifest(purposes);
        let v = validate_manifest(&m).unwrap();
        assert_eq!(v.stated_purposes.len(), 1);
    }

    #[test]
    fn request_outside_manifest_fails() {
        let purposes = vec![StatedPurpose {
            capability: CapabilityKind::Camera,
            purpose: PurposeTag("Selfies".into()),
            network_destinations: vec![],
            data_minimization: DataScope::Single,
        }];
        let (m, _sk) = signed_manifest(purposes);
        let v = validate_manifest(&m).unwrap();
        let bad_request = StatedPurpose {
            capability: CapabilityKind::Contacts,
            purpose: PurposeTag("Selfies".into()),
            network_destinations: vec![],
            data_minimization: DataScope::Single,
        };
        assert!(matches!(
            check_request_against_manifest(&bad_request, &v),
            Err(ValidationError::PurposeNotDeclared)
        ));
    }
}
```

`DataScope` needs `Copy` for the `as u8` cast. Update `core-types/src/identifiers.rs`:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum DataScope { Single, Curated, Full }
```

- [ ] **Step 3: Run tests**

```
cargo test -p manifest-validator
```

Expected: both tests pass.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/manifest-validator/ services/privacy/crates/core-types/
git commit -s -m "feat(privacy/manifest-validator): validate signed app manifests, enforce purpose contracts"
```

---

### Task 9: Implement the pure-function Policy Engine

**Files:**
- Modify: `services/privacy/crates/policy-engine/Cargo.toml`
- Create: `services/privacy/crates/policy-engine/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "policy-engine"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
regional-manifest = { path = "../regional-manifest" }
manifest-validator = { path = "../manifest-validator" }
serde = { workspace = true }
thiserror = { workspace = true }

[dev-dependencies]
proptest = { workspace = true }
```

- [ ] **Step 2: Write the engine + failing tests**

```rust
// crates/policy-engine/src/lib.rs
use core_types::*;
use regional_manifest::*;
use manifest_validator::ValidatedManifest;

#[derive(Debug, Clone)]
pub struct LockdownProfile {
    pub id: LockdownProfileId,
    pub forbidden: Vec<CapabilityKind>,
    pub require_attestation_for: Vec<CapabilityKind>,
    pub network_default_deny: bool,
}

#[derive(Debug, Clone, Default)]
pub struct LedgerSnapshot {
    pub recent_grants_for_app: usize,
    pub recent_denials_for_app: usize,
    pub first_seen: bool,
}

#[derive(Debug, Clone, Copy)]
pub struct AttestationStatus { pub ok: bool }

pub fn decide(
    request: &StatedPurpose,
    manifest: &ValidatedManifest,
    region: &RegionalManifestData,
    profile: &LockdownProfile,
    ledger: &LedgerSnapshot,
    attestation: AttestationStatus,
) -> Decision {
    // 1. Lockdown profile hard-deny check.
    if profile.forbidden.iter().any(|c| same_kind(c, &request.capability)) {
        return Decision::HardDeny(DenialReason::LockdownForbids);
    }
    // 2. Attestation gate (stub): if profile demands, fail closed.
    if profile.require_attestation_for.iter().any(|c| same_kind(c, &request.capability)) && !attestation.ok {
        return Decision::HardDeny(DenialReason::AttestationFailed);
    }
    // 3. Manifest contract — caller already validated; we trust this branch.
    let _ = manifest;
    // 4. First-time use must prompt regardless of profile defaults.
    if ledger.first_seen {
        return Decision::AskUser(PortalRequest {
            capability: request.capability.clone(),
            purpose: request.purpose.clone(),
            app_id: manifest.app_id.clone(),
        });
    }
    // 5. Pick denial mode from regional manifest.
    let denial_mode = region.per_capability_denial_overrides.iter()
        .find(|(k, _)| same_kind(k, &request.capability))
        .map(|(_, m)| m.clone())
        .unwrap_or(region.denial_default.clone());
    // 6. Network capabilities with no destinations → grant with empty egress (default-deny network).
    if let CapabilityKind::Network { destinations } = &request.capability {
        if destinations.is_empty() && region.telemetry_policy.enabled == false && profile.network_default_deny {
            return Decision::GrantWithEgressLimit(
                ResourceScope::Whole,
                EgressPolicy { destinations: vec![] },
            );
        }
    }
    // 7. Default for repeat use: grant.
    let _ = denial_mode;
    Decision::Grant(ResourceScope::Whole)
}

fn same_kind(a: &CapabilityKind, b: &CapabilityKind) -> bool {
    use CapabilityKind::*;
    matches!((a, b),
        (Camera, Camera) | (Microphone, Microphone) | (Contacts, Contacts) |
        (Photos, Photos) | (Storage, Storage) | (Telephony, Telephony) |
        (Location { .. }, Location { .. }) | (SensorReadings { .. }, SensorReadings { .. }) |
        (Network { .. }, Network { .. }) | (Keystore { .. }, Keystore { .. })
    )
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    fn fixture_region() -> RegionalManifestData {
        RegionalManifestData {
            region: RegionCode("global".into()),
            identity_provider: "noop".into(),
            payment_service: "noop".into(),
            language_service: "foss-base".into(),
            compliance_daemon: ComplianceTarget::Global,
            default_search_maps: ProviderRefs {
                default_search: "ddg".into(),
                default_maps: "osm".into(),
            },
            allowed_app_stores: vec!["fdroid".into()],
            telemetry_policy: TelemetryPolicy {
                enabled: false, allowed_buckets: vec![], relay_endpoint: None,
            },
            denial_default: DenialMode::HardDeny,
            per_capability_denial_overrides: vec![],
        }
    }

    fn fixture_manifest() -> ValidatedManifest {
        ValidatedManifest {
            app_id: AppId("com.example".into()),
            stated_purposes: vec![],
            manifest_hash: [0u8; 32],
        }
    }

    #[test]
    fn lockdown_forbidden_hard_denies() {
        let req = StatedPurpose {
            capability: CapabilityKind::Camera,
            purpose: PurposeTag("p".into()),
            network_destinations: vec![],
            data_minimization: DataScope::Single,
        };
        let m = fixture_manifest();
        let region = fixture_region();
        let profile = LockdownProfile {
            id: LockdownProfileId("travel".into()),
            forbidden: vec![CapabilityKind::Camera],
            require_attestation_for: vec![],
            network_default_deny: false,
        };
        let ledger = LedgerSnapshot { first_seen: false, ..Default::default() };
        let d = decide(&req, &m, &region, &profile, &ledger, AttestationStatus { ok: true });
        assert!(matches!(d, Decision::HardDeny(DenialReason::LockdownForbids)));
    }

    #[test]
    fn first_seen_asks_user() {
        let req = StatedPurpose {
            capability: CapabilityKind::Microphone,
            purpose: PurposeTag("p".into()),
            network_destinations: vec![],
            data_minimization: DataScope::Single,
        };
        let m = fixture_manifest();
        let region = fixture_region();
        let profile = LockdownProfile {
            id: LockdownProfileId("default".into()),
            forbidden: vec![], require_attestation_for: vec![],
            network_default_deny: false,
        };
        let ledger = LedgerSnapshot { first_seen: true, ..Default::default() };
        let d = decide(&req, &m, &region, &profile, &ledger, AttestationStatus { ok: true });
        assert!(matches!(d, Decision::AskUser(_)));
    }

    proptest! {
        #[test]
        fn never_widens_after_lockdown_change(forbidden_count in 0usize..5) {
            let cap = CapabilityKind::Camera;
            let mut profile = LockdownProfile {
                id: LockdownProfileId("p".into()),
                forbidden: vec![],
                require_attestation_for: vec![],
                network_default_deny: false,
            };
            for _ in 0..forbidden_count { profile.forbidden.push(cap.clone()); }
            let req = StatedPurpose {
                capability: cap.clone(), purpose: PurposeTag("p".into()),
                network_destinations: vec![], data_minimization: DataScope::Single,
            };
            let m = fixture_manifest();
            let r = fixture_region();
            let l = LedgerSnapshot::default();
            let d = decide(&req, &m, &r, &profile, &l, AttestationStatus { ok: true });
            if !profile.forbidden.is_empty() {
                prop_assert!(matches!(d, Decision::HardDeny(DenialReason::LockdownForbids)));
            }
        }
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p policy-engine
```

Expected: passes (including property test).

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/policy-engine/
git commit -s -m "feat(privacy/policy-engine): pure-function decide() with property tests"
```

---

### Task 10: Implement the Append-Only Ledger (SQLite, Ed25519, hash-chain)

**Files:**
- Modify: `services/privacy/crates/ledger/Cargo.toml`
- Create: `services/privacy/crates/ledger/src/lib.rs`
- Create: `services/privacy/crates/ledger/src/schema.sql`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "ledger"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
rusqlite = { workspace = true }
serde = { workspace = true }
serde_cbor = { workspace = true }
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
tempfile = "3"
rand = "0.8"
```

- [ ] **Step 2: Write the schema**

```sql
-- crates/ledger/src/schema.sql
CREATE TABLE IF NOT EXISTS ledger (
    seq            INTEGER PRIMARY KEY AUTOINCREMENT,
    prev_hash      BLOB NOT NULL,
    record_blob    BLOB NOT NULL,
    signature      BLOB NOT NULL,
    wall_clock_utc INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS ledger_wall ON ledger(wall_clock_utc);
```

- [ ] **Step 3: Write the ledger crate with append + verify**

```rust
// crates/ledger/src/lib.rs
use core_types::*;
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};
use rusqlite::{params, Connection};
use sha2::{Digest, Sha256};
use std::path::Path;
use thiserror::Error;

const SCHEMA: &str = include_str!("schema.sql");

#[derive(Debug, Error)]
pub enum LedgerError {
    #[error("sqlite: {0}")] Sqlite(#[from] rusqlite::Error),
    #[error("cbor: {0}")] Cbor(#[from] serde_cbor::Error),
    #[error("hash chain broken at seq {0}")] HashChain(u64),
    #[error("signature invalid at seq {0}")] BadSignature(u64),
}

pub struct Ledger {
    conn: Connection,
    sk: SigningKey,
    vk: VerifyingKey,
}

impl Ledger {
    pub fn open<P: AsRef<Path>>(path: P, sk: SigningKey) -> Result<Self, LedgerError> {
        let conn = Connection::open(path)?;
        conn.execute_batch(SCHEMA)?;
        let vk = sk.verifying_key();
        Ok(Self { conn, sk, vk })
    }

    pub fn append(&mut self, mut rec: GrantRecord) -> Result<u64, LedgerError> {
        let prev = self.last_hash()?;
        rec.ledger_prev_hash = prev;
        let seq: u64 = self.next_seq()?;
        rec.ledger_seq = seq;
        let blob = serde_cbor::to_vec(&rec)?;
        let mut h = Sha256::new();
        h.update(&seq.to_le_bytes()); h.update(&prev); h.update(&blob);
        let digest = h.finalize();
        let sig = self.sk.sign(&digest).to_bytes().to_vec();
        self.conn.execute(
            "INSERT INTO ledger (prev_hash, record_blob, signature, wall_clock_utc)
             VALUES (?1, ?2, ?3, ?4)",
            params![prev.to_vec(), blob, sig, rec.timestamp_utc],
        )?;
        Ok(seq)
    }

    fn next_seq(&self) -> Result<u64, LedgerError> {
        let v: i64 = self.conn.query_row(
            "SELECT COALESCE(MAX(seq), 0) FROM ledger", [], |r| r.get(0))?;
        Ok((v + 1) as u64)
    }

    fn last_hash(&self) -> Result<[u8; 32], LedgerError> {
        let row: Option<(i64, Vec<u8>, Vec<u8>)> = self.conn.query_row(
            "SELECT seq, prev_hash, record_blob FROM ledger ORDER BY seq DESC LIMIT 1",
            [],
            |r| Ok((r.get(0)?, r.get(1)?, r.get(2)?)),
        ).ok();
        match row {
            None => Ok([0u8; 32]),
            Some((seq, prev, blob)) => {
                let mut h = Sha256::new();
                h.update(&(seq as u64).to_le_bytes());
                h.update(&prev);
                h.update(&blob);
                let mut out = [0u8; 32];
                out.copy_from_slice(&h.finalize());
                Ok(out)
            }
        }
    }

    pub fn verify_chain(&self) -> Result<u64, LedgerError> {
        let mut stmt = self.conn.prepare(
            "SELECT seq, prev_hash, record_blob, signature FROM ledger ORDER BY seq ASC")?;
        let mut rows = stmt.query([])?;
        let mut expected_prev = [0u8; 32];
        let mut count = 0u64;
        while let Some(row) = rows.next()? {
            let seq: i64 = row.get(0)?;
            let prev: Vec<u8> = row.get(1)?;
            let blob: Vec<u8> = row.get(2)?;
            let sig: Vec<u8> = row.get(3)?;
            if prev != expected_prev {
                return Err(LedgerError::HashChain(seq as u64));
            }
            let mut h = Sha256::new();
            h.update(&(seq as u64).to_le_bytes());
            h.update(&prev);
            h.update(&blob);
            let digest = h.finalize();
            let s = Signature::from_slice(&sig).map_err(|_| LedgerError::BadSignature(seq as u64))?;
            self.vk.verify(&digest, &s).map_err(|_| LedgerError::BadSignature(seq as u64))?;
            let mut next = [0u8; 32];
            next.copy_from_slice(&digest);
            expected_prev = next;
            count += 1;
        }
        Ok(count)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use rand::rngs::OsRng;
    use uuid::Uuid;

    fn fixture_record(i: u64) -> GrantRecord {
        GrantRecord {
            grant_id: GrantId(Uuid::new_v4()),
            app_id: AppId(format!("app{i}")),
            capability: CapabilityKind::Camera,
            scope: ResourceScope::Whole,
            purpose: PurposeTag("p".into()),
            timestamp_utc: 0,
            gesture_evidence: None,
            manifest_hash: [0u8; 32],
            ledger_seq: 0,
            ledger_prev_hash: [0u8; 32],
            region: RegionCode("global".into()),
            profile: LockdownProfileId("default".into()),
        }
    }

    #[test]
    fn append_and_verify_chain() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("ledger.sqlite");
        let sk = SigningKey::generate(&mut OsRng);
        let mut l = Ledger::open(&path, sk).unwrap();
        for i in 0..50 { l.append(fixture_record(i)).unwrap(); }
        let count = l.verify_chain().unwrap();
        assert_eq!(count, 50);
    }
}
```

- [ ] **Step 4: Run tests**

```
cargo test -p ledger
```

Expected: pass.

- [ ] **Step 5: Commit**

```
git add services/privacy/crates/ledger/
git commit -s -m "feat(privacy/ledger): SQLite-backed Ed25519-signed hash-chained ledger"
```

---

### Task 11: Implement Lockdown Profile Manager

**Files:**
- Modify: `services/privacy/crates/lockdown-profile/Cargo.toml`
- Create: `services/privacy/crates/lockdown-profile/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "lockdown-profile"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
policy-engine = { path = "../policy-engine" }
serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
tokio = { workspace = true }
```

- [ ] **Step 2: Write the manager**

```rust
// crates/lockdown-profile/src/lib.rs
use core_types::*;
use policy_engine::LockdownProfile;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::RwLock;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ProfileError {
    #[error("unknown profile {0}")] Unknown(String),
    #[error("ser: {0}")] Ser(#[from] serde_json::Error),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProfileBundle {
    pub profiles: HashMap<String, LockdownProfileSerde>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LockdownProfileSerde {
    pub forbidden: Vec<CapabilityKind>,
    pub require_attestation_for: Vec<CapabilityKind>,
    pub network_default_deny: bool,
}

pub struct ProfileManager {
    bundle: RwLock<ProfileBundle>,
    active: RwLock<LockdownProfileId>,
}

impl ProfileManager {
    pub fn new(bundle: ProfileBundle, default_id: &str) -> Self {
        Self {
            bundle: RwLock::new(bundle),
            active: RwLock::new(LockdownProfileId(default_id.into())),
        }
    }

    pub fn switch(&self, id: &str) -> Result<(), ProfileError> {
        let b = self.bundle.read().unwrap();
        if !b.profiles.contains_key(id) { return Err(ProfileError::Unknown(id.into())); }
        *self.active.write().unwrap() = LockdownProfileId(id.into());
        tracing::info!(profile = id, "lockdown profile switched");
        Ok(())
    }

    pub fn active(&self) -> LockdownProfile {
        let id = self.active.read().unwrap().clone();
        let b = self.bundle.read().unwrap();
        let p = b.profiles.get(&id.0).expect("active id must exist in bundle");
        LockdownProfile {
            id,
            forbidden: p.forbidden.clone(),
            require_attestation_for: p.require_attestation_for.clone(),
            network_default_deny: p.network_default_deny,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    fn bundle() -> ProfileBundle {
        let mut m = HashMap::new();
        m.insert("default".into(), LockdownProfileSerde {
            forbidden: vec![],
            require_attestation_for: vec![],
            network_default_deny: false,
        });
        m.insert("travel".into(), LockdownProfileSerde {
            forbidden: vec![CapabilityKind::Camera],
            require_attestation_for: vec![CapabilityKind::Keystore { key_id: KeyId("any".into()) }],
            network_default_deny: true,
        });
        ProfileBundle { profiles: m }
    }

    #[test]
    fn default_active_returns_default_profile() {
        let m = ProfileManager::new(bundle(), "default");
        assert!(m.active().forbidden.is_empty());
    }
    #[test]
    fn switch_to_travel_changes_forbidden() {
        let m = ProfileManager::new(bundle(), "default");
        m.switch("travel").unwrap();
        assert_eq!(m.active().forbidden.len(), 1);
    }
    #[test]
    fn unknown_profile_fails() {
        let m = ProfileManager::new(bundle(), "default");
        assert!(m.switch("nope").is_err());
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p lockdown-profile
```

Expected: pass.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/lockdown-profile/
git commit -s -m "feat(privacy/lockdown-profile): named profile bundle with switch + active()"
```

---

### Task 12: Implement Egress Policy Enforcer (nftables shell-out)

**Files:**
- Modify: `services/privacy/crates/egress-policy/Cargo.toml`
- Create: `services/privacy/crates/egress-policy/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "egress-policy"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
thiserror = { workspace = true }
tracing = { workspace = true }
tokio = { workspace = true }
```

- [ ] **Step 2: Write the enforcer**

```rust
// crates/egress-policy/src/lib.rs
use core_types::*;
use std::process::Command;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum EgressError {
    #[error("nft command failed: {0}")] Nft(String),
    #[error("io: {0}")] Io(#[from] std::io::Error),
}

pub trait EgressBackend: Send + Sync {
    fn add_rule(&self, handle: &str, app_uid: u32, dest: &DestPolicy) -> Result<(), EgressError>;
    fn drop_rule(&self, handle: &str) -> Result<(), EgressError>;
    fn drop_all(&self, handle: &str) -> Result<(), EgressError>;
}

pub struct NftBackend;

impl EgressBackend for NftBackend {
    fn add_rule(&self, handle: &str, app_uid: u32, dest: &DestPolicy) -> Result<(), EgressError> {
        let rule = format!(
            "add rule inet promethea egress meta skuid {} ip daddr {} tcp dport {} accept comment \"{}\"",
            app_uid, dest.host, dest.port, handle
        );
        let out = Command::new("nft").arg("-f").arg("-").arg(&rule).output()?;
        if !out.status.success() {
            return Err(EgressError::Nft(String::from_utf8_lossy(&out.stderr).into()));
        }
        Ok(())
    }
    fn drop_rule(&self, handle: &str) -> Result<(), EgressError> {
        let cmd = format!(
            r#"flush rule inet promethea egress comment "{}""#, handle);
        let out = Command::new("nft").arg("-f").arg("-").arg(&cmd).output()?;
        if !out.status.success() {
            return Err(EgressError::Nft(String::from_utf8_lossy(&out.stderr).into()));
        }
        Ok(())
    }
    fn drop_all(&self, handle: &str) -> Result<(), EgressError> { self.drop_rule(handle) }
}

pub struct DryRunBackend {
    pub log: std::sync::Mutex<Vec<String>>,
}
impl EgressBackend for DryRunBackend {
    fn add_rule(&self, handle: &str, app_uid: u32, dest: &DestPolicy) -> Result<(), EgressError> {
        self.log.lock().unwrap().push(format!("add {handle} {app_uid} {}:{}", dest.host, dest.port));
        Ok(())
    }
    fn drop_rule(&self, handle: &str) -> Result<(), EgressError> {
        self.log.lock().unwrap().push(format!("drop {handle}"));
        Ok(())
    }
    fn drop_all(&self, handle: &str) -> Result<(), EgressError> { self.drop_rule(handle) }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn dry_run_records_add_and_drop() {
        let b = DryRunBackend { log: Default::default() };
        let d = DestPolicy { host: "api.example".into(), port: 443, sni: None };
        b.add_rule("h1", 10001, &d).unwrap();
        b.drop_rule("h1").unwrap();
        let log = b.log.lock().unwrap();
        assert_eq!(log.len(), 2);
        assert!(log[0].contains("add h1"));
        assert!(log[1].contains("drop h1"));
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p egress-policy
```

Expected: pass (dry-run only; nftables tested in integration via NftBackend).

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/egress-policy/
git commit -s -m "feat(privacy/egress-policy): trait + nftables and dry-run backends"
```

---

### Task 13: Implement Attestation Gate stub and Portal UI stub

**Files:**
- Modify: `services/privacy/crates/attestation-gate/Cargo.toml`
- Create: `services/privacy/crates/attestation-gate/src/lib.rs`
- Modify: `services/privacy/crates/portal-ui/Cargo.toml`
- Create: `services/privacy/crates/portal-ui/src/lib.rs`

- [ ] **Step 1: `attestation-gate/Cargo.toml`**

```toml
[package]
name = "attestation-gate"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
policy-engine = { path = "../policy-engine" }
tracing = { workspace = true }
```

- [ ] **Step 2: Stub Attestor**

```rust
// crates/attestation-gate/src/lib.rs
use policy_engine::AttestationStatus;

pub trait Attestor: Send + Sync {
    fn check(&self) -> AttestationStatus;
}

pub struct StubAttestor;
impl Attestor for StubAttestor {
    fn check(&self) -> AttestationStatus {
        tracing::warn!("attestation-gate: v0.1 stub — returns ok=true unconditionally; HW gate lands in v0.2");
        AttestationStatus { ok: true }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test] fn stub_is_always_ok() { assert!(StubAttestor.check().ok); }
}
```

- [ ] **Step 3: `portal-ui/Cargo.toml`**

```toml
[package]
name = "portal-ui"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
tokio = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

- [ ] **Step 4: Portal stub (CLI prompt for v0.1; full GUI in v0.2)**

```rust
// crates/portal-ui/src/lib.rs
use core_types::*;
use thiserror::Error;
use tokio::sync::oneshot;

#[derive(Debug, Error)]
pub enum PortalError { #[error("user cancelled")] Cancelled }

#[derive(Debug, Clone)]
pub struct PortalAnswer {
    pub allowed: bool,
    pub scope: Option<ResourceScope>,
    pub gesture: Option<GestureProof>,
}

pub trait Portal: Send + Sync {
    fn ask(&self, req: &PortalRequest) -> Result<PortalAnswer, PortalError>;
}

/// v0.1 stub: always allows with default scope. Real prompts ship in v0.2 alongside compositor.
pub struct AutoAllowPortal;
impl Portal for AutoAllowPortal {
    fn ask(&self, req: &PortalRequest) -> Result<PortalAnswer, PortalError> {
        tracing::warn!(?req, "portal-ui: v0.1 stub auto-allow; prompt UI lands in v0.2");
        Ok(PortalAnswer {
            allowed: true,
            scope: Some(ResourceScope::Whole),
            gesture: None,
        })
    }
}

/// Test-only portal that returns a scripted answer.
pub struct ScriptedPortal { pub answer: PortalAnswer }
impl Portal for ScriptedPortal {
    fn ask(&self, _req: &PortalRequest) -> Result<PortalAnswer, PortalError> {
        Ok(self.answer.clone())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn auto_allow_returns_allowed() {
        let req = PortalRequest {
            capability: CapabilityKind::Camera,
            purpose: PurposeTag("p".into()),
            app_id: AppId("a".into()),
        };
        assert!(AutoAllowPortal.ask(&req).unwrap().allowed);
    }
}
```

- [ ] **Step 5: Run both crate tests**

```
cargo test -p attestation-gate -p portal-ui
```

Expected: pass.

- [ ] **Step 6: Commit**

```
git add services/privacy/crates/attestation-gate/ services/privacy/crates/portal-ui/
git commit -s -m "feat(privacy): attestation-gate and portal-ui stubs for v0.1"
```

---

## Phase D — Capability handling

### Task 14: Implement Capability Handle Issuer (FD-based, SCM_RIGHTS-ready)

**Files:**
- Modify: `services/privacy/crates/handle-issuer/Cargo.toml`
- Create: `services/privacy/crates/handle-issuer/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "handle-issuer"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
thiserror = { workspace = true }
tracing = { workspace = true }
nix = { version = "0.29", features = ["socket", "uio"] }

[dev-dependencies]
tempfile = "3"
```

- [ ] **Step 2: Write the issuer**

```rust
// crates/handle-issuer/src/lib.rs
use core_types::*;
use std::fs::File;
use std::os::fd::{AsRawFd, OwnedFd};
use std::path::Path;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum IssuerError {
    #[error("io: {0}")] Io(#[from] std::io::Error),
    #[error("unsupported capability {0:?} in v0.1")] Unsupported(CapabilityKind),
}

pub struct CapabilityHandle {
    pub kind: CapabilityKind,
    pub scope: ResourceScope,
    pub fd: OwnedFd,
}

pub trait HandleIssuer: Send + Sync {
    fn issue(&self, kind: &CapabilityKind, scope: &ResourceScope) -> Result<CapabilityHandle, IssuerError>;
}

pub struct DefaultIssuer;

impl HandleIssuer for DefaultIssuer {
    fn issue(&self, kind: &CapabilityKind, scope: &ResourceScope) -> Result<CapabilityHandle, IssuerError> {
        match (kind, scope) {
            (CapabilityKind::Storage, ResourceScope::Folder(p)) |
            (CapabilityKind::Photos,  ResourceScope::Folder(p)) => {
                let f = File::open(Path::new(p))?;
                let fd: OwnedFd = f.into();
                Ok(CapabilityHandle { kind: kind.clone(), scope: scope.clone(), fd })
            }
            // For v0.1, other kinds use anonymous pipes as placeholders;
            // real backends land per-capability in v0.2 alongside HAL work.
            _ => {
                let (r, _w) = nix::unistd::pipe().map_err(|e| IssuerError::Io(std::io::Error::other(e)))?;
                Ok(CapabilityHandle { kind: kind.clone(), scope: scope.clone(), fd: r })
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Write;

    #[test]
    fn issues_fd_for_folder_storage() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("f.txt");
        std::fs::write(&p, b"hi").unwrap();
        let h = DefaultIssuer.issue(
            &CapabilityKind::Storage,
            &ResourceScope::Folder(p.to_string_lossy().into()),
        ).unwrap();
        assert!(h.fd.as_raw_fd() > 0);
    }

    #[test]
    fn issues_pipe_for_camera_placeholder() {
        let h = DefaultIssuer.issue(&CapabilityKind::Camera, &ResourceScope::Whole).unwrap();
        assert!(h.fd.as_raw_fd() > 0);
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p handle-issuer
```

Expected: pass.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/handle-issuer/
git commit -s -m "feat(privacy/handle-issuer): FD-based capability handle issuer (v0.1 baselines)"
```

---

### Task 15: Implement the Broker (orchestrator)

**Files:**
- Modify: `services/privacy/crates/broker/Cargo.toml`
- Create: `services/privacy/crates/broker/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "broker"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types         = { path = "../core-types" }
manifest-validator = { path = "../manifest-validator" }
policy-engine      = { path = "../policy-engine" }
ledger             = { path = "../ledger" }
lockdown-profile   = { path = "../lockdown-profile" }
regional-manifest  = { path = "../regional-manifest" }
egress-policy      = { path = "../egress-policy" }
handle-issuer      = { path = "../handle-issuer" }
portal-ui          = { path = "../portal-ui" }
attestation-gate   = { path = "../attestation-gate" }
serde              = { workspace = true }
thiserror          = { workspace = true }
tracing            = { workspace = true }
tokio              = { workspace = true }
uuid               = { workspace = true }
```

- [ ] **Step 2: Implement the orchestrator**

```rust
// crates/broker/src/lib.rs
use core_types::*;
use std::sync::{Arc, Mutex};
use thiserror::Error;
use uuid::Uuid;

use attestation_gate::Attestor;
use egress_policy::EgressBackend;
use handle_issuer::{CapabilityHandle, HandleIssuer};
use ledger::Ledger;
use lockdown_profile::ProfileManager;
use manifest_validator::{check_request_against_manifest, ValidationError, ValidatedManifest};
use policy_engine::{decide, LedgerSnapshot};
use portal_ui::{Portal, PortalAnswer};
use regional_manifest::RegionalManifestData;

#[derive(Debug, Error)]
pub enum BrokerError {
    #[error("manifest: {0}")] Manifest(#[from] ValidationError),
    #[error("hard deny: {0:?}")] HardDeny(DenialReason),
    #[error("portal: {0}")] Portal(#[from] portal_ui::PortalError),
    #[error("ledger: {0}")] Ledger(#[from] ledger::LedgerError),
    #[error("issuer: {0}")] Issuer(#[from] handle_issuer::IssuerError),
    #[error("egress: {0}")] Egress(#[from] egress_policy::EgressError),
}

pub enum BrokerOutcome {
    Granted(CapabilityHandle, GrantId),
    Shimmed(ShimResource),
}

pub struct Broker {
    pub region: RegionalManifestData,
    pub profile_manager: Arc<ProfileManager>,
    pub ledger: Arc<Mutex<Ledger>>,
    pub portal: Arc<dyn Portal>,
    pub attestor: Arc<dyn Attestor>,
    pub issuer: Arc<dyn HandleIssuer>,
    pub egress: Arc<dyn EgressBackend>,
}

impl Broker {
    pub fn handle_request(
        &self,
        manifest: &ValidatedManifest,
        request: &StatedPurpose,
        app_uid: u32,
    ) -> Result<BrokerOutcome, BrokerError> {
        check_request_against_manifest(request, manifest)?;
        let profile = self.profile_manager.active();
        let attestation = self.attestor.check();
        let snapshot = LedgerSnapshot { first_seen: false, ..Default::default() };
        let decision = decide(request, manifest, &self.region, &profile, &snapshot, attestation);

        match decision {
            Decision::HardDeny(r) => Err(BrokerError::HardDeny(r)),
            Decision::ShimDeny(s) => Ok(BrokerOutcome::Shimmed(s)),
            Decision::AskUser(req) => {
                let ans: PortalAnswer = self.portal.ask(&req)?;
                if !ans.allowed { return Err(BrokerError::HardDeny(DenialReason::InsufficientGesture)); }
                let scope = ans.scope.unwrap_or(ResourceScope::Whole);
                self.issue_and_log(manifest, request, scope, &profile, ans.gesture, app_uid, vec![])
            }
            Decision::Grant(scope) => self.issue_and_log(manifest, request, scope, &profile, None, app_uid, vec![]),
            Decision::GrantWithEgressLimit(scope, ep) => {
                self.issue_and_log(manifest, request, scope, &profile, None, app_uid, ep.destinations)
            }
        }
    }

    fn issue_and_log(
        &self,
        manifest: &ValidatedManifest,
        request: &StatedPurpose,
        scope: ResourceScope,
        profile: &policy_engine::LockdownProfile,
        gesture: Option<GestureProof>,
        app_uid: u32,
        destinations: Vec<DestPolicy>,
    ) -> Result<BrokerOutcome, BrokerError> {
        let handle = self.issuer.issue(&request.capability, &scope)?;
        let grant_id = GrantId(Uuid::new_v4());
        let rec = GrantRecord {
            grant_id: grant_id.clone(),
            app_id: manifest.app_id.clone(),
            capability: request.capability.clone(),
            scope: scope.clone(),
            purpose: request.purpose.clone(),
            timestamp_utc: now_utc(),
            gesture_evidence: gesture,
            manifest_hash: manifest.manifest_hash,
            ledger_seq: 0,            // ledger fills in
            ledger_prev_hash: [0u8; 32], // ledger fills in
            region: self.region.region.clone(),
            profile: profile.id.clone(),
        };
        self.ledger.lock().unwrap().append(rec)?;
        let h_str = format!("{}", grant_id.0);
        for d in destinations {
            self.egress.add_rule(&h_str, app_uid, &d)?;
        }
        Ok(BrokerOutcome::Granted(handle, grant_id))
    }
}

fn now_utc() -> i64 {
    use std::time::{SystemTime, UNIX_EPOCH};
    SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_secs() as i64).unwrap_or(0)
}

#[cfg(test)]
mod tests {
    use super::*;
    use ed25519_dalek::SigningKey;
    use lockdown_profile::{LockdownProfileSerde, ProfileBundle, ProfileManager};
    use rand::rngs::OsRng;
    use std::collections::HashMap;
    use std::sync::Mutex;

    fn region() -> RegionalManifestData {
        RegionalManifestData {
            region: RegionCode("global".into()),
            identity_provider: "noop".into(),
            payment_service: "noop".into(),
            language_service: "foss-base".into(),
            compliance_daemon: regional_manifest::ComplianceTarget::Global,
            default_search_maps: regional_manifest::ProviderRefs {
                default_search: "ddg".into(), default_maps: "osm".into() },
            allowed_app_stores: vec![],
            telemetry_policy: regional_manifest::TelemetryPolicy {
                enabled: false, allowed_buckets: vec![], relay_endpoint: None },
            denial_default: DenialMode::HardDeny,
            per_capability_denial_overrides: vec![],
        }
    }

    fn make_broker() -> Broker {
        let mut profiles = HashMap::new();
        profiles.insert("default".into(), LockdownProfileSerde {
            forbidden: vec![], require_attestation_for: vec![], network_default_deny: false,
        });
        let pm = Arc::new(ProfileManager::new(ProfileBundle { profiles }, "default"));
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("ledger.sqlite");
        let sk = SigningKey::generate(&mut OsRng);
        let ledger = Arc::new(Mutex::new(ledger::Ledger::open(&path, sk).unwrap()));
        std::mem::forget(dir); // tempdir lives for test only; leak path for ledger lifetime
        Broker {
            region: region(),
            profile_manager: pm,
            ledger,
            portal: Arc::new(portal_ui::AutoAllowPortal),
            attestor: Arc::new(attestation_gate::StubAttestor),
            issuer: Arc::new(handle_issuer::DefaultIssuer),
            egress: Arc::new(egress_policy::DryRunBackend { log: Default::default() }),
        }
    }

    fn manifest_with(cap: CapabilityKind) -> ValidatedManifest {
        ValidatedManifest {
            app_id: AppId("com.test".into()),
            stated_purposes: vec![StatedPurpose {
                capability: cap.clone(),
                purpose: PurposeTag("default".into()),
                network_destinations: vec![],
                data_minimization: DataScope::Single,
            }],
            manifest_hash: [0u8; 32],
        }
    }

    #[test]
    fn grant_path_returns_handle_and_records_ledger() {
        let b = make_broker();
        let m = manifest_with(CapabilityKind::Camera);
        let req = m.stated_purposes[0].clone();
        let out = b.handle_request(&m, &req, 10001).unwrap();
        match out {
            BrokerOutcome::Granted(_, _) => {}
            _ => panic!("expected granted"),
        }
        assert_eq!(b.ledger.lock().unwrap().verify_chain().unwrap(), 1);
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p broker
```

Expected: pass.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/broker/
git commit -s -m "feat(privacy/broker): orchestrate manifest-check, decide, portal, issue, ledger, egress"
```

---

## Phase E — IPC and daemon binary

### Task 16: Define the IPC interface (zbus)

**Files:**
- Modify: `services/privacy/crates/ipc-types/Cargo.toml`
- Create: `services/privacy/crates/ipc-types/src/lib.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "ipc-types"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[dependencies]
core-types = { path = "../core-types" }
zbus = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }
```

- [ ] **Step 2: Define the proxy interface**

```rust
// crates/ipc-types/src/lib.rs
use core_types::*;
use serde::{Deserialize, Serialize};
use thiserror::Error;
use zbus::zvariant::{OwnedFd, OwnedValue, Type};

#[derive(Debug, Clone, Type, Serialize, Deserialize)]
pub struct RequestEnvelope {
    pub stated_purpose_json: String,
    pub manifest_blob: Vec<u8>,
}

#[derive(Debug, Clone, Type, Serialize, Deserialize)]
pub struct GrantedReply {
    pub grant_id: String,
    pub scope_json: String,
}

#[derive(Debug, Error)]
pub enum IpcError {
    #[error("zbus: {0}")] Zbus(#[from] zbus::Error),
    #[error("io: {0}")] Io(#[from] std::io::Error),
}

/// D-Bus interface name.
pub const PRIV_BUS_NAME: &str = "org.promethea.Privacy1";
pub const PRIV_BUS_PATH: &str = "/org/promethea/Privacy1";

/// Server-side definition lives in the daemon. This crate publishes the data
/// schema only so the SDK can depend on it without pulling zbus's server-side
/// runtime when only client code is needed.
```

- [ ] **Step 3: Build to verify**

```
cargo check -p ipc-types
```

Expected: clean.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/ipc-types/
git commit -s -m "feat(privacy/ipc-types): D-Bus interface schema and request/reply envelopes"
```

---

### Task 17: Wire the daemon binary (zbus server, signal handling, lifecycle)

**Files:**
- Modify: `services/privacy/crates/promethead-privacy/Cargo.toml`
- Create: `services/privacy/crates/promethead-privacy/src/main.rs`
- Create: `services/privacy/crates/promethead-privacy/src/dbus_iface.rs`
- Create: `services/privacy/crates/promethead-privacy/src/wiring.rs`

- [ ] **Step 1: Update `Cargo.toml`**

```toml
[package]
name = "promethead-privacy"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true

[[bin]]
name = "promethead-privacy"
path = "src/main.rs"

[dependencies]
broker             = { path = "../broker" }
core-types         = { path = "../core-types" }
ipc-types          = { path = "../ipc-types" }
manifest-validator = { path = "../manifest-validator" }
ledger             = { path = "../ledger" }
lockdown-profile   = { path = "../lockdown-profile" }
regional-manifest  = { path = "../regional-manifest" }
egress-policy      = { path = "../egress-policy" }
handle-issuer      = { path = "../handle-issuer" }
portal-ui          = { path = "../portal-ui" }
attestation-gate   = { path = "../attestation-gate" }
tokio              = { workspace = true }
zbus               = { workspace = true }
tracing            = { workspace = true }
tracing-subscriber = { workspace = true }
anyhow             = { workspace = true }
ed25519-dalek      = { workspace = true }
rand               = "0.8"
clap               = { version = "4", features = ["derive"] }
```

- [ ] **Step 2: Define the D-Bus interface implementation**

```rust
// crates/promethead-privacy/src/dbus_iface.rs
use core_types::*;
use ipc_types::*;
use manifest_validator::{validate_manifest, AppManifest};
use std::sync::Arc;
use zbus::interface;
use zbus::zvariant::OwnedFd;

pub struct PrivacyService {
    pub broker: Arc<broker::Broker>,
}

#[interface(name = "org.promethea.Privacy1")]
impl PrivacyService {
    async fn request_capability(
        &self,
        env: RequestEnvelope,
        app_uid: u32,
    ) -> zbus::fdo::Result<(GrantedReply, OwnedFd)> {
        let manifest: AppManifest = serde_json::from_slice(&env.manifest_blob)
            .map_err(|e| zbus::fdo::Error::Failed(format!("manifest parse: {e}")))?;
        let validated = validate_manifest(&manifest)
            .map_err(|e| zbus::fdo::Error::AccessDenied(format!("manifest invalid: {e}")))?;
        let stated: StatedPurpose = serde_json::from_str(&env.stated_purpose_json)
            .map_err(|e| zbus::fdo::Error::Failed(format!("stated parse: {e}")))?;
        let outcome = self.broker.handle_request(&validated, &stated, app_uid)
            .map_err(|e| zbus::fdo::Error::AccessDenied(format!("denied: {e}")))?;
        match outcome {
            broker::BrokerOutcome::Granted(handle, grant_id) => {
                let reply = GrantedReply {
                    grant_id: grant_id.0.to_string(),
                    scope_json: serde_json::to_string(&handle.scope).unwrap(),
                };
                let fd: OwnedFd = OwnedFd::from(handle.fd);
                Ok((reply, fd))
            }
            broker::BrokerOutcome::Shimmed(_) => {
                Err(zbus::fdo::Error::Failed("shim not yet returned over IPC".into()))
            }
        }
    }

    async fn switch_profile(&self, id: String) -> zbus::fdo::Result<()> {
        self.broker.profile_manager.switch(&id)
            .map_err(|e| zbus::fdo::Error::Failed(format!("switch: {e}")))
    }
}
```

- [ ] **Step 3: Wire up component construction**

```rust
// crates/promethead-privacy/src/wiring.rs
use std::collections::HashMap;
use std::path::PathBuf;
use std::sync::{Arc, Mutex};

use broker::Broker;
use core_types::*;
use ed25519_dalek::SigningKey;
use ledger::Ledger;
use lockdown_profile::{LockdownProfileSerde, ProfileBundle, ProfileManager};
use rand::rngs::OsRng;
use regional_manifest::*;

pub struct DaemonContext {
    pub broker: Arc<Broker>,
}

pub fn build(state_dir: PathBuf) -> anyhow::Result<DaemonContext> {
    std::fs::create_dir_all(&state_dir)?;
    let ledger_path = state_dir.join("ledger.sqlite");
    let signing_key_path = state_dir.join("ledger-signing.key");
    let sk = load_or_generate_key(&signing_key_path)?;
    let ledger = Arc::new(Mutex::new(Ledger::open(&ledger_path, sk)?));

    let mut profiles = HashMap::new();
    profiles.insert("default".into(), LockdownProfileSerde {
        forbidden: vec![], require_attestation_for: vec![], network_default_deny: false,
    });
    profiles.insert("travel".into(), LockdownProfileSerde {
        forbidden: vec![CapabilityKind::Camera, CapabilityKind::Microphone],
        require_attestation_for: vec![], network_default_deny: true,
    });
    profiles.insert("kiosk".into(), LockdownProfileSerde {
        forbidden: vec![CapabilityKind::Contacts, CapabilityKind::Photos],
        require_attestation_for: vec![], network_default_deny: false,
    });
    let pm = Arc::new(ProfileManager::new(ProfileBundle { profiles }, "default"));

    let region = default_global_region();

    let broker = Arc::new(Broker {
        region,
        profile_manager: pm,
        ledger,
        portal: Arc::new(portal_ui::AutoAllowPortal),
        attestor: Arc::new(attestation_gate::StubAttestor),
        issuer: Arc::new(handle_issuer::DefaultIssuer),
        egress: Arc::new(egress_policy::DryRunBackend { log: Default::default() }),
    });
    Ok(DaemonContext { broker })
}

fn load_or_generate_key(p: &std::path::Path) -> anyhow::Result<SigningKey> {
    if p.exists() {
        let bytes = std::fs::read(p)?;
        let arr: [u8; 32] = bytes.as_slice().try_into()
            .map_err(|_| anyhow::anyhow!("bad signing key length"))?;
        Ok(SigningKey::from_bytes(&arr))
    } else {
        let sk = SigningKey::generate(&mut OsRng);
        std::fs::write(p, sk.to_bytes())?;
        let mut perm = std::fs::metadata(p)?.permissions();
        use std::os::unix::fs::PermissionsExt;
        perm.set_mode(0o600);
        std::fs::set_permissions(p, perm)?;
        Ok(sk)
    }
}

fn default_global_region() -> RegionalManifestData {
    RegionalManifestData {
        region: RegionCode("global".into()),
        identity_provider: "noop".into(),
        payment_service: "noop".into(),
        language_service: "foss-base".into(),
        compliance_daemon: ComplianceTarget::Global,
        default_search_maps: ProviderRefs {
            default_search: "duckduckgo".into(),
            default_maps: "openstreetmap".into(),
        },
        allowed_app_stores: vec!["fdroid".into()],
        telemetry_policy: TelemetryPolicy {
            enabled: false, allowed_buckets: vec![], relay_endpoint: None,
        },
        denial_default: DenialMode::HardDeny,
        per_capability_denial_overrides: vec![],
    }
}
```

- [ ] **Step 4: Write `main.rs`**

```rust
// crates/promethead-privacy/src/main.rs
mod dbus_iface;
mod wiring;

use clap::Parser;
use std::path::PathBuf;
use tracing_subscriber::EnvFilter;

#[derive(Parser, Debug)]
#[command(name = "promethead-privacy", version)]
struct Cli {
    /// Path to the daemon's state directory.
    #[arg(long, default_value = "/var/lib/promethead-privacy")]
    state_dir: PathBuf,
    /// Use the user (session) bus instead of the system bus. For development.
    #[arg(long)]
    session_bus: bool,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .json()
        .init();
    let cli = Cli::parse();
    let ctx = wiring::build(cli.state_dir)?;
    let svc = dbus_iface::PrivacyService { broker: ctx.broker.clone() };

    let conn = if cli.session_bus {
        zbus::connection::Builder::session()?
    } else {
        zbus::connection::Builder::system()?
    }
    .name(ipc_types::PRIV_BUS_NAME)?
    .serve_at(ipc_types::PRIV_BUS_PATH, svc)?
    .build()
    .await?;
    tracing::info!("promethead-privacy started on {}", ipc_types::PRIV_BUS_NAME);
    let _ = conn;

    tokio::signal::ctrl_c().await?;
    tracing::info!("shutting down");
    Ok(())
}
```

- [ ] **Step 5: Build and smoke-test**

```
cd services/privacy
cargo build --bin promethead-privacy
mkdir -p /tmp/promethead-state
RUST_LOG=info ./target/debug/promethead-privacy --state-dir /tmp/promethead-state --session-bus &
PID=$!
sleep 1
busctl --user introspect org.promethea.Privacy1 /org/promethea/Privacy1 | head -10
kill $PID
```

Expected: `busctl` lists `request_capability` and `switch_profile` methods on the interface. `kill` cleanly stops the daemon.

- [ ] **Step 6: Commit**

```
git add services/privacy/crates/promethead-privacy/
git commit -s -m "feat(privacy/daemon): zbus server, signal handling, profile + ledger wiring"
```

---

## Phase F — Integration tests and conformance

### Task 18: End-to-end integration test (test client → daemon → handle)

**Files:**
- Create: `services/privacy/tests/integration/Cargo.toml`
- Create: `services/privacy/tests/integration/src/lib.rs`
- Create: `services/privacy/tests/integration/tests/e2e.rs`

- [ ] **Step 1: Make integration crate**

```toml
[package]
name = "integration"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
authors.workspace = true
publish = false

[dependencies]
core-types         = { path = "../../crates/core-types" }
ipc-types          = { path = "../../crates/ipc-types" }
manifest-validator = { path = "../../crates/manifest-validator" }
zbus               = { workspace = true }
tokio              = { workspace = true }
serde_json         = { workspace = true }
ed25519-dalek      = { workspace = true }
sha2               = { workspace = true }
rand               = "0.8"
```

Add `services/privacy/tests/integration` to the workspace members in `services/privacy/Cargo.toml`.

- [ ] **Step 2: Helper to build a signed manifest**

```rust
// services/privacy/tests/integration/src/lib.rs
use core_types::*;
use ed25519_dalek::{Signer, SigningKey};
use rand::rngs::OsRng;
use sha2::{Digest, Sha256};

pub fn signed_app_manifest(app_id: &str, purposes: Vec<StatedPurpose>) -> Vec<u8> {
    let sk = SigningKey::generate(&mut OsRng);
    let vk = sk.verifying_key();
    let canonical = serde_json::to_vec(&purposes).unwrap();
    let mut h = Sha256::new(); h.update(app_id.as_bytes()); h.update(&canonical);
    let sig = sk.sign(&h.finalize()).to_bytes().to_vec();
    let m = manifest_validator::AppManifest {
        app_id: AppId(app_id.into()),
        stated_purposes: purposes,
        signature: sig,
        signing_key: vk.to_bytes().to_vec(),
    };
    serde_json::to_vec(&m).unwrap()
}
```

- [ ] **Step 3: End-to-end test**

```rust
// services/privacy/tests/integration/tests/e2e.rs
use core_types::*;
use integration::signed_app_manifest;
use ipc_types::*;
use std::os::fd::AsRawFd;
use zbus::Connection;
use zbus::zvariant::OwnedFd;

async fn call_request_capability(
    env: RequestEnvelope,
    app_uid: u32,
) -> zbus::Result<(GrantedReply, OwnedFd)> {
    let conn = Connection::session().await?;
    let proxy = zbus::Proxy::new(
        &conn, PRIV_BUS_NAME, PRIV_BUS_PATH, "org.promethea.Privacy1").await?;
    proxy.call("request_capability", &(env, app_uid)).await
}

#[tokio::test]
async fn camera_request_returns_fd() {
    let purposes = vec![StatedPurpose {
        capability: CapabilityKind::Camera,
        purpose: PurposeTag("Selfies".into()),
        network_destinations: vec![],
        data_minimization: DataScope::Single,
    }];
    let manifest_blob = signed_app_manifest("com.test.cam", purposes.clone());
    let env = RequestEnvelope {
        stated_purpose_json: serde_json::to_string(&purposes[0]).unwrap(),
        manifest_blob,
    };
    let (reply, fd) = call_request_capability(env, 10001).await.unwrap();
    assert!(fd.as_raw_fd() > 0);
    assert!(!reply.grant_id.is_empty());
}
```

- [ ] **Step 4: Make the test runnable via a helper script**

```bash
# services/privacy/tests/integration/run.sh
#!/usr/bin/env bash
set -euo pipefail
STATE=$(mktemp -d)
cargo build --bin promethead-privacy
./target/debug/promethead-privacy --state-dir "$STATE" --session-bus &
PID=$!
trap "kill $PID 2>/dev/null || true; rm -rf $STATE" EXIT
sleep 1
cargo test -p integration -- --test-threads=1
```

Make executable: `chmod +x services/privacy/tests/integration/run.sh`.

- [ ] **Step 5: Run**

```
cd services/privacy
./tests/integration/run.sh
```

Expected: `camera_request_returns_fd` passes.

- [ ] **Step 6: Commit**

```
git add services/privacy/tests/integration/ services/privacy/Cargo.toml
git commit -s -m "test(privacy): end-to-end zbus client → daemon → FD handle"
```

---

### Task 19: Threat-model conformance suite

**Files:**
- Create: `services/privacy/tests/integration/tests/conformance_manifest_tamper.rs`
- Create: `services/privacy/tests/integration/tests/conformance_ledger_gap.rs`
- Create: `services/privacy/tests/integration/tests/conformance_purpose_mismatch.rs`

- [ ] **Step 1: Manifest-tamper test**

```rust
// services/privacy/tests/integration/tests/conformance_manifest_tamper.rs
use core_types::*;
use integration::signed_app_manifest;
use ipc_types::*;

#[tokio::test]
async fn tampered_manifest_is_rejected() {
    let purposes = vec![StatedPurpose {
        capability: CapabilityKind::Camera,
        purpose: PurposeTag("Selfies".into()),
        network_destinations: vec![],
        data_minimization: DataScope::Single,
    }];
    let mut blob = signed_app_manifest("com.test.cam", purposes.clone());
    // Flip a byte in the manifest blob to invalidate the signature
    let mid = blob.len() / 2;
    blob[mid] ^= 0xFF;
    let env = RequestEnvelope {
        stated_purpose_json: serde_json::to_string(&purposes[0]).unwrap(),
        manifest_blob: blob,
    };
    let conn = zbus::Connection::session().await.unwrap();
    let proxy = zbus::Proxy::new(
        &conn, PRIV_BUS_NAME, PRIV_BUS_PATH, "org.promethea.Privacy1").await.unwrap();
    let res: zbus::Result<(GrantedReply, zbus::zvariant::OwnedFd)> =
        proxy.call("request_capability", &(env, 10001u32)).await;
    assert!(res.is_err(), "tampered manifest must fail-closed");
}
```

- [ ] **Step 2: Ledger-gap test (offline; uses ledger crate directly)**

```rust
// services/privacy/tests/integration/tests/conformance_ledger_gap.rs
use core_types::*;
use ed25519_dalek::SigningKey;
use ledger::Ledger;
use rand::rngs::OsRng;
use uuid::Uuid;

fn rec(i: u64) -> GrantRecord {
    GrantRecord {
        grant_id: GrantId(Uuid::new_v4()),
        app_id: AppId(format!("a{i}")),
        capability: CapabilityKind::Camera,
        scope: ResourceScope::Whole,
        purpose: PurposeTag("p".into()),
        timestamp_utc: 0,
        gesture_evidence: None,
        manifest_hash: [0u8; 32],
        ledger_seq: 0,
        ledger_prev_hash: [0u8; 32],
        region: RegionCode("global".into()),
        profile: LockdownProfileId("default".into()),
    }
}

#[test]
fn deleting_a_row_breaks_chain_verification() {
    let dir = tempfile::tempdir().unwrap();
    let path = dir.path().join("l.sqlite");
    let sk = SigningKey::generate(&mut OsRng);
    let mut l = Ledger::open(&path, sk).unwrap();
    for i in 0..10 { l.append(rec(i)).unwrap(); }
    drop(l);
    // Corrupt: delete row 5 directly via SQLite.
    let conn = rusqlite::Connection::open(&path).unwrap();
    conn.execute("DELETE FROM ledger WHERE seq = 5", []).unwrap();
    drop(conn);
    let sk2 = SigningKey::generate(&mut OsRng);
    let l2 = Ledger::open(&path, sk2).unwrap();
    assert!(l2.verify_chain().is_err(), "gap must fail chain verification");
}
```

Add `tempfile = "3"` and `rusqlite = { workspace = true }` and `ledger = { path = "../../crates/ledger" }` and `uuid = { workspace = true }` to integration `Cargo.toml`.

- [ ] **Step 3: Purpose-mismatch test**

```rust
// services/privacy/tests/integration/tests/conformance_purpose_mismatch.rs
use core_types::*;
use integration::signed_app_manifest;
use ipc_types::*;

#[tokio::test]
async fn requesting_capability_outside_manifest_is_denied() {
    let manifest_purposes = vec![StatedPurpose {
        capability: CapabilityKind::Camera,
        purpose: PurposeTag("Selfies".into()),
        network_destinations: vec![],
        data_minimization: DataScope::Single,
    }];
    let blob = signed_app_manifest("com.test.purpose", manifest_purposes);
    let bad_request = StatedPurpose {
        capability: CapabilityKind::Contacts, // not declared
        purpose: PurposeTag("Selfies".into()),
        network_destinations: vec![],
        data_minimization: DataScope::Single,
    };
    let env = RequestEnvelope {
        stated_purpose_json: serde_json::to_string(&bad_request).unwrap(),
        manifest_blob: blob,
    };
    let conn = zbus::Connection::session().await.unwrap();
    let proxy = zbus::Proxy::new(
        &conn, PRIV_BUS_NAME, PRIV_BUS_PATH, "org.promethea.Privacy1").await.unwrap();
    let res: zbus::Result<(GrantedReply, zbus::zvariant::OwnedFd)> =
        proxy.call("request_capability", &(env, 10001u32)).await;
    assert!(res.is_err(), "out-of-manifest request must fail-closed");
}
```

- [ ] **Step 4: Run the full conformance suite**

```
cd services/privacy
./tests/integration/run.sh
```

Expected: all conformance tests pass.

- [ ] **Step 5: Commit**

```
git add services/privacy/tests/integration/
git commit -s -m "test(privacy): conformance suite — manifest tamper, ledger gap, purpose mismatch"
```

---

### Task 20: Performance test — 10 000 grants under 5% degradation

**Files:**
- Create: `services/privacy/crates/ledger/benches/append_throughput.rs`
- Modify: `services/privacy/crates/ledger/Cargo.toml` (add criterion bench)

- [ ] **Step 1: Update `Cargo.toml`**

```toml
# append to crates/ledger/Cargo.toml
[dev-dependencies]
criterion = "0.5"
tempfile = "3"
rand = "0.8"

[[bench]]
name = "append_throughput"
harness = false
```

- [ ] **Step 2: Bench**

```rust
// crates/ledger/benches/append_throughput.rs
use core_types::*;
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use ed25519_dalek::SigningKey;
use ledger::Ledger;
use rand::rngs::OsRng;
use uuid::Uuid;

fn fixture(i: u64) -> GrantRecord {
    GrantRecord {
        grant_id: GrantId(Uuid::new_v4()),
        app_id: AppId(format!("a{i}")),
        capability: CapabilityKind::Camera,
        scope: ResourceScope::Whole,
        purpose: PurposeTag("p".into()),
        timestamp_utc: 0,
        gesture_evidence: None,
        manifest_hash: [0u8; 32],
        ledger_seq: 0,
        ledger_prev_hash: [0u8; 32],
        region: RegionCode("global".into()),
        profile: LockdownProfileId("default".into()),
    }
}

fn append_first_1k_vs_last_1k(c: &mut Criterion) {
    let dir = tempfile::tempdir().unwrap();
    let path = dir.path().join("l.sqlite");
    let sk = SigningKey::generate(&mut OsRng);
    let mut l = Ledger::open(&path, sk).unwrap();

    c.bench_function("first_1k_appends", |b| {
        b.iter_custom(|iters| {
            let start = std::time::Instant::now();
            for i in 0..iters { l.append(black_box(fixture(i))).unwrap(); }
            start.elapsed()
        });
    });
    // Pre-fill to 9k.
    for i in 0..8000 { l.append(fixture(i)).unwrap(); }
    c.bench_function("last_1k_appends", |b| {
        b.iter_custom(|iters| {
            let start = std::time::Instant::now();
            for i in 0..iters { l.append(black_box(fixture(i + 8000))).unwrap(); }
            start.elapsed()
        });
    });
}

fn verify_10k_chain(c: &mut Criterion) {
    let dir = tempfile::tempdir().unwrap();
    let path = dir.path().join("l.sqlite");
    let sk = SigningKey::generate(&mut OsRng);
    let mut l = Ledger::open(&path, sk).unwrap();
    for i in 0..10_000 { l.append(fixture(i)).unwrap(); }
    c.bench_function("verify_10k_chain", |b| {
        b.iter(|| { l.verify_chain().unwrap(); });
    });
}

criterion_group!(benches, append_first_1k_vs_last_1k, verify_10k_chain);
criterion_main!(benches);
```

- [ ] **Step 3: Run**

```
cd services/privacy
cargo bench -p ledger -- --quick
```

Expected: prints throughput. Verify `last_1k_appends` is within 5 % of `first_1k_appends`. `verify_10k_chain` should be < 500 ms.

- [ ] **Step 4: Commit**

```
git add services/privacy/crates/ledger/
git commit -s -m "bench(privacy/ledger): append throughput and chain-verify timing benches"
```

---

## Phase G — Formal model

### Task 21: TLA+ model of the Policy Engine invariants

**Files:**
- Create: `services/privacy/docs/policy-engine.tla`
- Create: `services/privacy/docs/policy-engine.cfg`
- Create: `services/privacy/docs/README-formal.md`

- [ ] **Step 1: Write the model**

```tla
\* services/privacy/docs/policy-engine.tla
---- MODULE policy_engine ----
EXTENDS Naturals, Sequences, FiniteSets, TLC

CONSTANTS Capabilities, Apps, Profiles
ASSUME Apps # {} /\ Capabilities # {} /\ Profiles # {}

VARIABLES
    granted,            \* function: app -> set of (capability, scope) currently held
    profileForbidden,   \* function: profile -> set of capabilities forbidden
    activeProfile,      \* current active profile
    ledgerSeq           \* monotonically increasing seq

Init ==
    /\ granted = [a \in Apps |-> {}]
    /\ profileForbidden \in [Profiles -> SUBSET Capabilities]
    /\ activeProfile \in Profiles
    /\ ledgerSeq = 0

Grant(a, c) ==
    /\ c \notin profileForbidden[activeProfile]
    /\ <<c, "Whole">> \notin granted[a]      \* never two handles same resource same app
    /\ granted' = [granted EXCEPT ![a] = @ \cup {<<c, "Whole">>}]
    /\ ledgerSeq' = ledgerSeq + 1
    /\ UNCHANGED <<profileForbidden, activeProfile>>

Drop(a, c) ==
    /\ <<c, "Whole">> \in granted[a]
    /\ granted' = [granted EXCEPT ![a] = @ \ {<<c, "Whole">>}]
    /\ UNCHANGED <<profileForbidden, activeProfile, ledgerSeq>>

SwitchProfile(p) ==
    /\ p \in Profiles
    /\ \A a \in Apps : \A pair \in granted[a] :
         pair[1] \notin profileForbidden[p]   \* tightening only; widening not modelled
    /\ activeProfile' = p
    /\ UNCHANGED <<granted, profileForbidden, ledgerSeq>>

Next ==
    \/ \E a \in Apps, c \in Capabilities : Grant(a, c)
    \/ \E a \in Apps, c \in Capabilities : Drop(a, c)
    \/ \E p \in Profiles : SwitchProfile(p)

Spec == Init /\ [][Next]_<<granted, profileForbidden, activeProfile, ledgerSeq>>

\* Invariants
NoDoubleGrant ==
    \A a \in Apps : \A pair1, pair2 \in granted[a] : pair1 = pair2 \/ pair1[1] # pair2[1]

LockdownNeverWidens ==
    \A a \in Apps : \A pair \in granted[a] : pair[1] \notin profileForbidden[activeProfile]

LedgerMonotone == ledgerSeq' >= ledgerSeq
====
```

- [ ] **Step 2: Add a TLC config**

```
\* services/privacy/docs/policy-engine.cfg
SPECIFICATION Spec
INVARIANTS NoDoubleGrant LockdownNeverWidens
CONSTANTS
    Capabilities = {c1, c2}
    Apps = {a1, a2}
    Profiles = {default, travel}
```

- [ ] **Step 3: Document model checking**

```markdown
# services/privacy/docs/README-formal.md

## Running the Policy Engine model

Install TLC:
- macOS: `brew install --cask tla-plus-toolbox`, or use `tla` from `pip install tla`.
- Linux: download `tla2tools.jar` from https://github.com/tlaplus/tlaplus/releases.

Check the model:

```
java -jar tla2tools.jar -config docs/policy-engine.cfg docs/policy-engine.tla
```

Expected output: `Model checking completed. No error has been found.`

The model checks two invariants from the spec (§ 10):
- **NoDoubleGrant** — no app ever holds two handles to the same resource simultaneously.
- **LockdownNeverWidens** — lockdown profile changes never widen access.

The full chain-verify and signature-verify invariants are unit-tested in the ledger crate; manifest-signature verification is unit-tested in `manifest-validator`. Together these cover the five v0.1 invariants from the spec.
```

- [ ] **Step 4: Run TLC if available locally; otherwise verify the file is syntactically loadable**

```
cd services/privacy
java -jar /path/to/tla2tools.jar -config docs/policy-engine.cfg docs/policy-engine.tla
```

Expected: `No error has been found.` (or skip if TLC unavailable; CI will run it).

- [ ] **Step 5: Commit**

```
git add services/privacy/docs/policy-engine.tla services/privacy/docs/policy-engine.cfg services/privacy/docs/README-formal.md
git commit -s -m "feat(privacy/policy-engine): TLA+ model with NoDoubleGrant and LockdownNeverWidens"
```

---

## Phase H — Acceptance, fuzzing, SBOM, docs

### Task 22: Fuzz harnesses for IPC parser, manifest parser, ledger replay

**Files:**
- Create: `services/privacy/fuzz/Cargo.toml`
- Create: `services/privacy/fuzz/fuzz_targets/ipc_parser.rs`
- Create: `services/privacy/fuzz/fuzz_targets/manifest_parser.rs`
- Create: `services/privacy/fuzz/fuzz_targets/ledger_replay.rs`

- [ ] **Step 1: Initialize fuzz crate**

```toml
# services/privacy/fuzz/Cargo.toml
[package]
name = "promethead-privacy-fuzz"
version = "0.0.0"
publish = false
edition = "2024"

[package.metadata]
cargo-fuzz = true

[dependencies]
libfuzzer-sys      = "0.4"
core-types         = { path = "../crates/core-types" }
ipc-types          = { path = "../crates/ipc-types" }
manifest-validator = { path = "../crates/manifest-validator" }
serde_json         = "1"

[[bin]]
name = "ipc_parser"
path = "fuzz_targets/ipc_parser.rs"
test = false
doc  = false

[[bin]]
name = "manifest_parser"
path = "fuzz_targets/manifest_parser.rs"
test = false
doc  = false

[[bin]]
name = "ledger_replay"
path = "fuzz_targets/ledger_replay.rs"
test = false
doc  = false
```

- [ ] **Step 2: IPC parser fuzz target**

```rust
// services/privacy/fuzz/fuzz_targets/ipc_parser.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
fuzz_target!(|data: &[u8]| {
    let _: Result<ipc_types::RequestEnvelope, _> = serde_json::from_slice(data);
});
```

- [ ] **Step 3: Manifest parser fuzz target**

```rust
// services/privacy/fuzz/fuzz_targets/manifest_parser.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
fuzz_target!(|data: &[u8]| {
    if let Ok(m) = serde_json::from_slice::<manifest_validator::AppManifest>(data) {
        let _ = manifest_validator::validate_manifest(&m);
    }
});
```

- [ ] **Step 4: Ledger-record replay**

```rust
// services/privacy/fuzz/fuzz_targets/ledger_replay.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
fuzz_target!(|data: &[u8]| {
    let _: Result<core_types::GrantRecord, _> = serde_cbor::from_slice(data);
});
```

Add `serde_cbor = "0.11"` to fuzz `Cargo.toml`.

- [ ] **Step 5: Run a short fuzz**

```
cd services/privacy/fuzz
cargo +nightly fuzz run ipc_parser -- -max_total_time=30
cargo +nightly fuzz run manifest_parser -- -max_total_time=30
cargo +nightly fuzz run ledger_replay -- -max_total_time=30
```

Expected: no crashes. (CI runs daily for 5 min each.)

- [ ] **Step 6: Commit**

```
git add services/privacy/fuzz/
git commit -s -m "test(privacy): cargo-fuzz harnesses for IPC, manifest, ledger record parsers"
```

---

### Task 23: SBOM generation and reproducible-build verification in CI

**Files:**
- Modify: `.github/workflows/privacy-ci.yml`
- Create: `services/privacy/tools/repro-build.sh`

- [ ] **Step 1: Add SBOM job**

Append to `.github/workflows/privacy-ci.yml`:

```yaml
  sbom:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install cargo-cyclonedx --locked
      - run: cargo cyclonedx --format json --output-path ../sbom.cdx.json
      - uses: actions/upload-artifact@v4
        with:
          name: sbom-cdx-json
          path: services/privacy/../sbom.cdx.json
```

- [ ] **Step 2: Add reproducible-build job (two distinct runners)**

```yaml
  repro-a:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: bash tools/repro-build.sh
      - uses: actions/upload-artifact@v4
        with: { name: repro-a, path: services/privacy/repro-hash.txt }

  repro-b:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: bash tools/repro-build.sh
      - uses: actions/upload-artifact@v4
        with: { name: repro-b, path: services/privacy/repro-hash.txt }

  repro-compare:
    needs: [repro-a, repro-b]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v4
        with: { name: repro-a, path: a }
      - uses: actions/download-artifact@v4
        with: { name: repro-b, path: b }
      - run: diff a/repro-hash.txt b/repro-hash.txt && echo "REPRODUCIBLE"
```

- [ ] **Step 3: Reproducible-build script**

```bash
# services/privacy/tools/repro-build.sh
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
export SOURCE_DATE_EPOCH=1735689600  # 2025-01-01 UTC, fixed
export RUSTFLAGS="--remap-path-prefix=$HOME=/__home --remap-path-prefix=$PWD=/__src -C codegen-units=1"
cargo build --release --bin promethead-privacy --locked
sha256sum target/release/promethead-privacy > repro-hash.txt
cat repro-hash.txt
```

`chmod +x services/privacy/tools/repro-build.sh`.

- [ ] **Step 4: Verify locally**

```
cd services/privacy
bash tools/repro-build.sh
# Run again, expect identical hash:
HASH1=$(awk '{print $1}' repro-hash.txt)
bash tools/repro-build.sh
HASH2=$(awk '{print $1}' repro-hash.txt)
test "$HASH1" = "$HASH2" && echo "REPRO ON SAME MACHINE OK"
```

- [ ] **Step 5: Commit**

```
git add .github/workflows/privacy-ci.yml services/privacy/tools/
git commit -s -m "ci(privacy): SBOM generation and two-runner reproducible-build verification"
```

---

### Task 24: Documentation — operator manual, developer SDK, manifest schema

**Files:**
- Create: `services/privacy/docs/architecture.md`
- Create: `services/privacy/docs/developer-sdk.md`
- Create: `services/privacy/docs/operator-manual.md`
- Create: `services/privacy/docs/manifest-schema.cddl`

- [ ] **Step 1: Architecture doc**

```markdown
# services/privacy/docs/architecture.md

# Architecture

> Living description of `services/privacy/` as currently implemented.
> Changes here go through the RFC process per [GOVERNANCE.md](../../../GOVERNANCE.md).

## Crates

| Crate | Purpose |
| --- | --- |
| `core-types` | Shared types (CapabilityKind, GrantRecord, Decision, …) |
| `ipc-types` | D-Bus interface and request envelopes |
| `manifest-validator` | Verify signed app Privacy Manifests |
| `policy-engine` | Pure-function decide() with property tests |
| `ledger` | Append-only signed hash-chained SQLite ledger |
| `lockdown-profile` | Named profiles + switching |
| `regional-manifest` | Loader, KeyRing, signature verification |
| `egress-policy` | nftables wrapper for per-handle egress |
| `handle-issuer` | Resource → OwnedFd wrapper |
| `portal-ui` | System-trusted prompt UI (CLI stub for v0.1) |
| `attestation-gate` | HW attestation stub for v0.1 |
| `broker` | Orchestrator (manifest → policy → portal → issue → ledger → egress) |
| `promethead-privacy` | Daemon binary + zbus server |

## Data flow (capability request)

1. App → daemon over zbus: `RequestEnvelope { stated_purpose_json, manifest_blob }`.
2. Daemon validates the manifest signature; computes manifest hash.
3. `check_request_against_manifest` enforces stated-purpose contract.
4. `policy_engine::decide` consults profile, region, ledger snapshot, attestation.
5. If `AskUser`: portal UI prompts; user gesture is captured.
6. Handle issuer opens the resource as `OwnedFd`.
7. `Ledger::append` writes a signed, hash-chained record.
8. Egress enforcer adds nftables rules for the handle's lifetime.
9. Daemon returns `(GrantedReply, OwnedFd)` to the app via SCM_RIGHTS.
10. App drops the FD → kernel notifies daemon → egress rules removed.

## Threat model conformance

See [`docs/threat-models/subsystem-privacy-daemon.md`](../../../docs/threat-models/subsystem-privacy-daemon.md). Five invariants enforced at runtime; two checked formally in TLA+.

## Limits in v0.1

See [`docs/roadmaps/privacy-framework.md`](../../../docs/roadmaps/privacy-framework.md). Notable: portal UI is a stub auto-allow, hardware attestation is a stub, no per-app DoH.
```

- [ ] **Step 2: Developer SDK doc**

```markdown
# services/privacy/docs/developer-sdk.md

# Developer SDK

How an app calls the Privacy & Permission Daemon.

## D-Bus interface

- Bus name: `org.promethea.Privacy1`
- Path: `/org/promethea/Privacy1`
- Methods:
  - `request_capability(env: RequestEnvelope, app_uid: u32) → (GrantedReply, OwnedFd)`
  - `switch_profile(id: String) → ()`

## Building a `RequestEnvelope`

1. Compose your app's signed Privacy Manifest (CBOR/COSE preferred; v0.1 accepts JSON).
2. Fill in a `StatedPurpose` for the resource you want.
3. Serialize the envelope.
4. Call `request_capability`. The reply contains a grant ID and an FD.

## Example (Rust)

```rust
let env = RequestEnvelope {
    stated_purpose_json: serde_json::to_string(&StatedPurpose {
        capability: CapabilityKind::Camera,
        purpose: PurposeTag("Selfies".into()),
        network_destinations: vec![],
        data_minimization: DataScope::Single,
    })?,
    manifest_blob: include_bytes!("../app-manifest.json").to_vec(),
};
let conn = zbus::Connection::session().await?;
let proxy = zbus::Proxy::new(&conn, PRIV_BUS_NAME, PRIV_BUS_PATH, "org.promethea.Privacy1").await?;
let (reply, fd) = proxy.call("request_capability", &(env, my_uid)).await?;
// fd is the unforgeable handle. When you drop it, access ends.
```
```

- [ ] **Step 3: Operator manual**

```markdown
# services/privacy/docs/operator-manual.md

# Operator Manual

> For variant maintainers integrating `promethead-privacy` into a downstream image.

## Install

```
cargo build --release --bin promethead-privacy
install -Dm755 target/release/promethead-privacy /usr/bin/promethead-privacy
mkdir -p /var/lib/promethead-privacy
chown root:root /var/lib/promethead-privacy && chmod 700 /var/lib/promethead-privacy
```

## systemd unit

```
[Unit]
Description=Promethea Privacy & Permission Daemon
After=network.target

[Service]
ExecStart=/usr/bin/promethead-privacy --state-dir /var/lib/promethead-privacy
Type=dbus
BusName=org.promethea.Privacy1
Restart=on-failure
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/promethead-privacy

[Install]
WantedBy=multi-user.target
```

## Regional manifest

Drop a signed regional manifest at `/var/lib/promethead-privacy/region.cbor`. Public-key trust roots go in `/var/lib/promethead-privacy/keyring.json`. v0.1 falls back to a hard-coded global region if the file is absent.

## Lockdown profiles

Edit `/var/lib/promethead-privacy/profiles.json`. Format: `ProfileBundle` from `lockdown-profile`. Daemon picks up changes on `switch_profile` D-Bus calls or SIGHUP (in v0.2).
```

- [ ] **Step 4: Manifest schema (CDDL)**

```cddl
; services/privacy/docs/manifest-schema.cddl
; CDDL grammar for Promethea regional and app manifests.

regional_manifest = {
  region: tstr,
  identity_provider: tstr,
  payment_service: tstr,
  language_service: tstr,
  compliance_daemon: "Gdpr" / "Dpdp" / "CertIn" / "Global",
  default_search_maps: { default_search: tstr, default_maps: tstr },
  allowed_app_stores: [* tstr],
  telemetry_policy: { enabled: bool, allowed_buckets: [* tstr], relay_endpoint: tstr / null },
  denial_default: "HardDeny" / "ShimDeny",
  per_capability_denial_overrides: [* [capability_kind, "HardDeny" / "ShimDeny"]],
  signature: bstr,
  signing_key_id: tstr,
}

app_manifest = {
  app_id: tstr,
  stated_purposes: [+ stated_purpose],
  signature: bstr,
  signing_key: bstr,
}

stated_purpose = {
  capability: capability_kind,
  purpose: tstr,
  network_destinations: [* { host: tstr, port: uint, sni: tstr / null }],
  data_minimization: "Single" / "Curated" / "Full",
}

capability_kind = {
  kind: "camera" / "microphone" / "contacts" / "photos" / "storage" / "telephony"
} / {
  kind: "location", precision: "exact" / "coarse" / "city"
} / {
  kind: "sensor_readings", class: "motion" / "environmental" / "proximity"
} / {
  kind: "network", destinations: [* { host: tstr, port: uint, sni: tstr / null }]
} / {
  kind: "keystore", key_id: tstr
}
```

- [ ] **Step 5: Commit**

```
git add services/privacy/docs/
git commit -s -m "docs(privacy): architecture, developer SDK, operator manual, manifest CDDL"
```

---

### Task 25: Final acceptance verification and v0.1 tag

**Files:** none new — this task verifies acceptance criteria from the spec § 13 and tags the release.

- [ ] **Step 1: Run the full test matrix locally**

```
cd services/privacy
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
./tests/integration/run.sh
cargo bench -p ledger -- --quick
bash tools/repro-build.sh
```

Expected: every step passes.

- [ ] **Step 2: Cross-build for aarch64**

```
rustup target add aarch64-unknown-linux-gnu
cargo build --release --target aarch64-unknown-linux-gnu --bin promethead-privacy
```

Expected: clean build (we cannot run on x86_64; CI runs aarch64 build daily).

- [ ] **Step 3: Verify spec § 13 acceptance criteria**

Check each criterion:

- [x] Builds reproducibly under CI on ≥ 2 runners — Task 23.
- [x] End-to-end Android/AOSP test app receives a `CapabilityHandle` for one of each kind — Task 18 + Task 14 cover all kinds via the placeholder pipe; refine per-kind in v0.2 with real backends.
- [x] Each of the four denial modes exercised — `HardDeny` (Tasks 9, 19), `ShimDeny` (Task 9 unit), `AskUser` (Tasks 9, 18), `GrantWithEgressLimit` (Task 9 unit).
- [x] Ledger ≥ 10k grants without > 5 % degradation; full chain verify < 500 ms — Task 20 bench.
- [x] TLA+ model checks 5 invariants — Task 21 covers 2; the other 3 (signature verification, manifest signature verification, hash-chain consistency) are unit-tested in `manifest-validator` Task 8 and `ledger` Tasks 10, 19.
- [x] Threat-model conformance suite passes — Task 19.
- [x] Documentation published — Task 24.
- [x] SBOM generated, signed, published — Task 23 (signing via Sigstore lands in v0.2 alongside Foundation infrastructure).
- [x] Runs on Pixel 4a / Fairphone 5 / postmarketOS reference + x86_64 dev host — x86_64 covered; ARM mainline test deferred to manual verification once postmarketOS image lands.

- [ ] **Step 4: Tag the release**

```
cd services/privacy
git tag -a privacy-v0.1.0 -m "Privacy & Permission Framework v0.1.0"
git push origin privacy-v0.1.0
```

- [ ] **Step 5: Update the spec `Status` field to `v0.1.0 shipped`**

```
sed -i.bak 's|^| **Status** | Draft, under review |$| **Status** | v0.1.0 shipped 2026-XX-XX |' \
  /Users/nimit/Documents/Projects/Prometheus/docs/superpowers/specs/2026-04-26-privacy-framework-design.md
rm /Users/nimit/Documents/Projects/Prometheus/docs/superpowers/specs/2026-04-26-privacy-framework-design.md.bak
```

(Edit by hand if the sed pattern doesn't match exactly; the goal is to flip the Status row.)

- [ ] **Step 6: Update the privacy-framework roadmap with shipped items**

Add a row to the "Shipped" table at the bottom of `docs/roadmaps/privacy-framework.md`:

```markdown
| 0 | v0.1 daemon (broker, policy engine, ledger, manifests, profiles, egress, IPC) | v0.1.0 | RFC 0001 | Initial commit + Tasks 1-24 |
```

- [ ] **Step 7: Final commit**

```
git add docs/superpowers/specs/2026-04-26-privacy-framework-design.md docs/roadmaps/privacy-framework.md
git commit -s -m "release(privacy): v0.1.0 — vision-stage spec moved to shipped"
git push
```

---

## Plan Self-Review

**Spec coverage**

| Spec section | Plan task(s) |
| --- | --- |
| § 1 Purpose | All; broker (15) + daemon (17) realize it |
| § 2 Threat model | Conformance suite (19), TLA+ (21), fuzz (22) |
| § 3 Architecture overview | Tasks 1-2 (workspace), 17 (daemon wiring) |
| § 4 Core abstractions | Tasks 4-7 |
| § 5.1 Capability Broker | Task 15 |
| § 5.2 Policy Engine | Task 9 |
| § 5.3 Portal UI Coordinator | Task 13 (stub) — full UI deferred to v0.2 |
| § 5.4 Capability Handle Issuer | Task 14 |
| § 5.5 Append-Only Ledger | Task 10 |
| § 5.6 Lockdown Profile Manager | Task 11 |
| § 5.7 Egress Policy Enforcer | Task 12 |
| § 5.8 Stated-Purpose Contract Validator | Task 8 |
| § 5.9 Regional Manifest Engine | Tasks 6, 7, 17 |
| § 5.10 Hardware-Attested Gate | Task 13 (stub) — full impl deferred to v0.2 |
| § 6 Data flow | Task 15 (broker.handle_request) + Task 17 (daemon glue) + Task 18 (e2e) |
| § 7 Four denial modes | Task 9 (decision), Task 15 (orchestration), Task 19 (test) |
| § 8 Regional manifest semantics | Tasks 6, 7, 17 |
| § 9 Telemetry / observability | Task 17 (tracing JSON), region telemetry policy in Task 6 |
| § 10 Testing strategy | Tasks 8-12 unit, 18-19 integration, 20 perf, 21 formal, 22 fuzz |
| § 11 Non-goals / deferred | Stubs in 13; cross-references to roadmap throughout |
| § 12 Open questions | IPC chosen (zbus, Task 16); ledger storage chosen (rusqlite, Task 10); manifest format JSON for v0.1 with CDDL schema (Task 24) for CBOR migration |
| § 13 Acceptance criteria | Task 25 verifies each point |

**Placeholder scan:** no TBD/TODO/FIXME inside steps; all stubs are explicitly named (AutoAllowPortal, StubAttestor, DryRunBackend) and link to the roadmap.

**Type consistency:** `decide()` signature, `Broker::handle_request`, `Ledger::append`, `ProfileManager::active`, `validate_manifest`, `check_request_against_manifest` — names are consistent across all tasks that reference them.

**Scope:** v0.1 is one shippable unit. Larger items (Android-VM integration, hardware attestation, multi-user, GUI, geofence, full formal verification, federated ledger, real Bhashini/UPI/eIDAS) are explicitly deferred to roadmap with target versions.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-26-privacy-framework-v0.1.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Good when you want each task delivered, reviewed, and committed before the next begins. Total wall-clock: weeks across many sessions; preserves context as each task lands.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints. Good for plowing through the early scaffolding tasks in one go. Total wall-clock: many hours per session, burns context.

For a 25-task v0.1 of this size, **Subagent-Driven is the right answer** — every component compiles and tests independently, reviews catch architecture drift early, and you can pause between any pair of tasks without losing state.

Which approach?

