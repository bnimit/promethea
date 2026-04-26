# Stream 8 — Rust vs Zig as the implementation language

*Snapshot dated 2026-04-26.*

**Recommendation: Rust. Unambiguously. Zig is not a serious contender for this project.**

## Kernel ecosystem maturity

Linux 7.0 (2026-04-12) shipped Rust as a stable, first-class kernel language. Real Rust drivers — NVIDIA Nova, Apple AGX, Android Binder, Google ashmem, NVMe abstractions — are running on hundreds of millions of devices. PCI, platform device, IRQ, DMA, and `dev_printk` APIs are now Rust-callable; DRM is roughly a year from *requiring* Rust for new drivers.

Zig in Linux: zero upstream presence. The only artifact is `Mic92/zig.ko`, a proof-of-concept printing "Hello kernel," and an internal Zig project tracker. There has been no RFC, no maintainer commitment, no analog to Miguel Ojeda's seven-year shepherding effort.

Microkernel landscape: Rust dominates production-credible work — Redox, Hubris, Theseus, KataOS on seL4, Tock. Zig has *hiillos* and *Pluto* — both hobby, neither shipping.

## Mobile / ARM driver realities

AGX, Nova, and the upcoming Adreno 800 driver path are Rust. There is no Zig GPU driver shipping on a real ARM SoC; Zig's GPU work is SPIR-V / PTX / AMDGCN compiler backends and the Mach engine, not kernel-mode drivers. Modem / baseband, ISP, NPU: zero Zig precedent. A Zig choice means writing every driver against C kernel headers via `@cImport` and shipping out-of-tree forever.

## Memory safety — honest delta

Microsoft and Google both cite ~70 % of CVEs as memory-safety bugs; Google specifically pegged Android at ~90 %. Rust's borrow checker categorically eliminates use-after-free, data races, and most aliasing bugs at compile time outside `unsafe` blocks. Zig's allocator discipline, `defer` / `errdefer`, slice bounds in safe build modes, and ReleaseSafe mode are real improvements over C — but it has no aliasing model, no Send / Sync analog, and use-after-free is undefined behavior in ReleaseFast. Zig moves the needle from C; it does not close the gap with Rust. For a privacy-first OS where memory bugs *are* the threat model, "better than C" is not the bar.

## Compile times, toolchain

Zig's marketed compile-time advantage is real for small / medium codebases and trivial cross-compilation (`-target aarch64-linux-musl` works out of the box). But at kernel scale Zig's single-threaded semantic analysis has caused multi-minute waits and stop-on-first-error friction. Rust's compile times are slower per crate but the model parallelizes, sccache / Cranelift dev builds are mature, and `rustup`'s ARM / RISC-V cross targets are one command. Build infrastructure for an AOSP-class project (Soong / Bazel-style) is a Rust-first world today.

## ABI / FFI / C interop

Zig's `@cImport` and "Zig as a better C" are genuinely elegant for binding small C libraries. But the Linux kernel header forest and AOSP HALs are not small libraries — they are macros, inline functions, and per-arch ifdef thickets. Rust-for-Linux solved that with `bindgen` plus hand-curated safe wrappers shipped in tree; Zig would need to redo that work, alone, out of tree, forever. `cbindgen` for emitting C-callable surfaces (for HAL stability) is mature; Zig has `extern` but no equivalent ABI-stability tooling.

## Embedded / no_std

Rust's `embedded-hal`, `probe-rs`, `defmt`, RTIC, and Embassy form the dominant MCU stack in 2026. Zig embedded is "throw a C lib in and hack" — fine for hobby boards, not for shipping a sensor hub or secure-element firmware on a phone.

## Talent / hiring

Stack Overflow 2025 Survey: Rust is most-admired (72 %), Zig fourth (64 %), but Zig usage is a small fraction of Rust's, and Rust usage itself grew +2pp YoY. Indeed / LinkedIn job-listing ratio is roughly 20-50× in Rust's favor. For an EU + India dual-pillar org hiring at scale, Zig dramatically narrows the talent funnel. The "Zig is simpler" claim breaks down at OS scale — when you must model resource ownership manually without compiler help, the apparent simplicity becomes the engineer's burden, multiplied by every contributor for the project's life.

## License / governance

Rust: dual MIT / Apache-2.0, Rust Foundation with AWS, Google, Microsoft, Huawei, Meta, and others — no single point of failure. Zig: MIT, ZSF is a 501(c)(3) recently boosted by a $512K Synadia / TigerBeetle pledge. ZSF is well-run but explicitly BDFL — Andrew Kelley is the language. **For a 10-year OS bet, the Kelley bus factor is a real institutional risk**; Rust has no comparable single-person dependency.

## Production OS-class shipping

Rust: Linux kernel modules in mainline, Firecracker (AWS), Cloudflare workerd, Fuchsia components, Asahi GPU stack, Android crosvm, Windows kernel components. Zig at production scale: TigerBeetle (DB), Bun (runtime), Ghostty (terminal), parts of Uber. None are OS-class — no kernel, no driver-on-real-SoC, no init system, no compositor in production. **That gap is the entire decision.**

## What Zig genuinely does better

Comptime as unified meta-programming (cleaner than proc-macros); no hidden control flow (no Drop, no overload — auditable); explicit allocator-as-parameter (excellent for embedded); cross-compilation as a default; "read source = know everything" (genuine security-audit advantage on small codebases). These are real wins, not memes — but none is decisive for an OS that will live or die by driver count and CVE count.

## What Rust does better (additions)

Beyond the obvious: tooling (rust-analyzer, miri, kani, loom, cargo-fuzz, cargo-deny SBOM); supply-chain (crates.io with Sigstore, RustSec advisory DB); async story (tokio / hyper / axum) maps directly to a privacy daemon, push service, OTA orchestrator; formal verification adjacencies (Verus, Creusot, Prusti, Aeneas); type-state and sealed traits for HAL design; const generics for hardware register modeling; Rust 2024 edition stable and 1.0-stable for a decade.

## Hybrid option

Mixing Rust and Zig in one OS is operationally tenable only at clean ABI seams (FFI'd library, separate process). Examples: Lukarnel pairs Zig microkernel with Rust services. For an AOSP-class shipping product it is a net negative — two toolchains, two CI matrices, two security-review pipelines, two hiring funnels, two unsafe-code review cultures. Don't.

## Synthesis

For an EU + India dual-pillar, telemetry-free, privacy-first, AOSP-compatible mobile OS in 2026: **Rust is correct, Zig is not on the table.** The decisive facts are (a) Rust shipping in mainline Linux 7.0 with the exact drivers we need (Nova, AGX, Adreno-class, Binder), (b) ~10-50× talent pool, (c) compile-time memory safety closing the bug class that produces 70-90 % of mobile CVEs, and (d) institutional governance robustness over a 10-year horizon. Zig's wins (comptime, cross-compile, no hidden control flow) do not offset any of those for *this* product.

**Where Zig could earn a narrow seat (optional, not recommended for v1):**

- A standalone, audited **bootloader** or **secure-element shim** where "no hidden control flow + read-everything" is the security property. Even here, Rust + `no_std` + a minimal allocator gets you the same audit story with the borrow checker as a bonus.
- A **build-time code generator** (HAL register definitions, IDL) using Zig comptime — but `build.rs` plus `syn` / `quote` already does this in-tree.

**Net: ship Rust top-to-bottom. Revisit Zig in 2030 if a shipping ARM-SoC driver, a kernel upstream RFC, and a Foundation with non-Kelley succession all materialize.**

## Sources

- https://www.phoronix.com/news/Linux-7.0-Driver-Core
- https://www.ghacks.net/2026/04/13/linux-7-0-released-with-official-rust-support-and-new-code-for-sparc-and-alpha-cpus/
- https://botmonster.com/posts/rust-stable-linux-kernel-7/
- https://rust-for-linux.com/
- https://rust-for-linux.com/apple-agx-gpu-driver
- https://rust-for-linux.com/nova-gpu-driver
- https://github.com/Mic92/zig.ko
- https://github.com/ziglang/zig/projects/5
- https://www.osnews.com/story/143343/writing-an-operating-system-kernel-from-scratch-in-zig/
- https://github.com/xor-bits/hiillos
- https://github.com/ZystemOS/pluto
- https://alichraghi.github.io/blog/zig-gpu/
- https://rustmagazine.org/issue-3/is-zig-safer-than-unsafe-rust/
- https://biggo.com/news/202510051332_Zig_Memory_Safety_Debate_Book_Launch
- https://zackoverflow.dev/writing/i-spent-181-minutes-waiting-for-the-zig-compiler-this-week/
- https://msrc.microsoft.com/blog/2019/07/a-proactive-approach-to-more-secure-code/
- https://www.chromium.org/Home/chromium-security/memory-safety/
- https://github.com/rust-embedded/awesome-embedded-rust
- https://survey.stackoverflow.co/2025/technology/
- https://tigerbeetle.com/blog/2025-10-25-synadia-and-tigerbeetle-pledge-512k-to-the-zig-software-foundation/
- https://www.prnewswire.com/news-releases/synadia-and-tigerbeetle-pledge-512k-to-the-zig-software-foundation-302596839.html
- https://www.techzine.eu/news/devops/140394/rust-enters-the-linux-kernel-but-its-adoption-is-leveling-off/
