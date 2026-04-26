# Stream 2 — Rust in OS kernels: 2025-2026 landscape

*Snapshot dated 2026-04-26.*

## Rust-for-Linux upstream status

The "experiment" label was officially dropped in 2025 and Linux 7.0 (April 12, 2026) ships Rust as a peer language with a stable Rust 1.93 toolchain anchor. Production Rust now sits at ~600 KLOC across drivers, FS abstractions, and core bindings. Working subsystems include the NVIDIA Nova GPU driver (Turing/RTX 20), Apple AGX (M1/M2), Android binder/ashmem, NVMe abstractions, Asahi-derived DRM components, and `alloc` infrastructure. Filesystems remain exploratory (PuzzleFS, tarfs prototypes); no full production FS is in-tree yet.

Politics: Torvalds publicly committed to override C-only maintainers; Greg KH posted an LKML note encouraging new drivers in Rust. Friction continues around DMA/VFS abstractions. Hector Martin resigned Feb 2025 and Asahi Lina paused Apple GPU work in March 2025 — a real loss but not blocking RfL momentum. Miguel Ojeda's Rust kernel policy is now de facto canonical.

**2026-2028 trajectory:** mostly-Rust new drivers expected as the default, full FS in Rust likely 2027+, networking and block layer abstractions filling in. Core C subsystems will remain C indefinitely.

## Rust microkernel projects

| Project | Maturity | Mobile/ARM | Drivers | License | Backer |
| --- | --- | --- | --- | --- | --- |
| **Redox** | Stable, self-hosting; ARM64 dynamic linking 2025; runs on Qualcomm phones (no touch driver) | AArch64 + RISC-V active | Native Intel GPU driver in dev; reading Linux DRM APIs | MIT | Jeremy Soller, community |
| **Hubris** | Production (Oxide servers) | Cortex-M only, NO Cortex-A | MCU peripherals only | MPL-2.0 | Oxide Computer |
| **Theseus** | Research (Rice University) | x86_64 mainly | Minimal | MIT | Academic |
| **KataOS / Sparrow** | Released 2022; low public activity since 2023 | RISC-V + ARM QEMU | Skeletal | Apache-2.0 | Google AmbiML |
| **Tock** | Production (embedded) | Cortex-M + RISC-V | MCU only | Apache / MIT | Stanford / community |
| **R3** | Experimental | Cortex-M | None | MIT | Solo |
| **Drone OS** | Hobby / research | Cortex-M | None | Custom | Solo |

Only Redox targets application-class ARM with a UNIX-like userspace; everything else is MCU or research. Upstream collaboration: Redox and Tock are openly collaborative; Hubris is open source but Oxide-driven; KataOS is Apache-licensed but effectively unstaffed.

## seL4 + Rust userspace

The seL4 Foundation funds and maintains `seL4/rust-sel4` (v3.0.0); Rust is officially supported in microkit protection domains. HAMR adds SysMLv2 / AADL → Rust codegen targeting microkit. Real deployments: aerospace and defense (DARPA HACMS lineage), DornerWorks, Cog Systems INTEGRITY-style phones (older). KataOS is the public reference for seL4+Rust userspace structure: rootserver in Rust, `sel4-sys` crate for syscalls, dynamic memory reclamation patches to seL4 itself. **No commercial mobile phone shipping today on seL4.** It is feasible but every driver, IPC contract, and userspace service must be built from scratch.

## Driver story for ARM mobile SoCs

This is the realistic blocker. Modems, GPUs, ISPs, and NPUs on Snapdragon / Tensor / Exynos / MediaTek depend on huge vendor binary blobs (QMI / RIL, Adreno KGSL, Mali, proprietary ISP firmware). On Linux, Qualcomm now does same-day upstreaming for Snapdragon 8 Elite Gen 5, and Adreno 800 patches are in-tree — but these are C drivers binding to Linux DMA / IOMMU / clk / regulator / PM frameworks. A Rust microkernel would need to either (a) reimplement equivalent frameworks and re-port each driver, or (b) run Linux in a VM / compat layer for drivers (KataOS-style isolation). For 2026-2028, only path (b) is realistic for modem / GPU / ISP / NPU on commodity SoCs. Redox's strategy of binding to Linux DRM ABIs is the most pragmatic precedent.

## Verified boot in Rust

No production Rust replacement for AVB 2.0 exists. Active work: rustBoot (MCU-to-SoC bootloader, can chainload Linux); oreboot (coreboot fork in Rust, x86/ARM/RISC-V, LinuxBoot payloads); Sprout (Rust UEFI bootloader, secure-boot in active dev); SentinelBoot (Rust RISC-V cryptographically secure bootloader).

For an Android-class device the realistic 18-24-month answer is to keep AVB 2.0 (libavb is small, audited, vendor-supported) and possibly wrap or rewrite libavb in Rust as a follow-on. A fully Rust-native verified-boot stack for a Snapdragon-class device is a multi-year effort.

## Synthesis for Promethea

**Production track (18-24 months):** Linux 7.x mainline + Rust-for-Linux, Android-derived userspace with AVB 2.0. Only path with working modem / GPU / ISP / NPU drivers on Snapdragon / Tensor / MediaTek today. New first-party drivers and HALs in Rust against RfL abstractions; ride the Nova / AGX / binder precedent. Risk is low because RfL is now policy-permanent and vendor SoC support lands upstream same-day.

**R&D track (5-7 years):** seL4 + Rust userspace, KataOS-style architecture, re-derived (KataOS itself is dormant). seL4 gives formal verification, the Foundation funds Rust support, the microkit + HAMR toolchain is the only credible path to a verified mobile TCB. Run vendor drivers in a Linux compat VM initially, peel off security-critical components (crypto, attestation, baseband isolation, biometrics) into native Rust protection domains first. Redox is a useful secondary reference for AArch64 userspace patterns but its monolithic-microkernel design lacks seL4's verification story. Hubris / Tock / R3 / Drone are wrong scale.

## Sources

- https://rust-for-linux.com/rust-kernel-policy
- https://www.phoronix.com/news/Rust-To-Stay-Linux-Kernel
- https://www.cnx-software.com/2026/04/13/linux-7-0-release-main-changes-arm-risc-v-and-mips-architectures/
- https://www.theregister.com/2025/02/21/linux_c_rust_debate_continues/
- https://lpc.events/event/19/contributions/2068/attachments/1859/3981/2025-12-11%20-%20LPC%202025%20-%20Rust%20for%20Linux.pdf
- https://www.theregister.com/2025/03/20/asahi_linux_asahi_lina/
- https://rust-for-linux.com/apple-agx-gpu-driver
- https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/
- https://www.qualcomm.com/developer/blog/2025/10/same-day-snapdragon-8-elite-gen-5-upstream-linux-support
- https://docs.sel4.systems/projects/rust/
- https://github.com/seL4/rust-sel4
- https://sel4.systems/Summit/2025/abstracts2025.html
- https://opensource.googleblog.com/2022/10/announcing-kataos-and-sparrow.html
- https://github.com/oxidecomputer/hubris/blob/master/FAQ.mkdn
- https://github.com/theseus-os/Theseus
- https://www.tockos.org/
- https://www.drone-os.com/
- https://www.osnews.com/story/143404/yes-redox-can-run-on-some-smartphones/
- https://www.webpronews.com/redox-os-2025-2026-roadmap-arm-support-security-boosts-and-variants/
- https://www.redox-os.org/news/this-month-251130/
- https://github.com/oreboot/oreboot
- https://github.com/nihalpasham/rustBoot
- https://securitypulse.tech/2025/11/13/sprout-open-source-uefi-bootloader-rust-secure-boot-roadmap/
- https://www.codethink.co.uk/articles/2024/secure_bootloader/
- https://source.android.com/docs/security/features/verifiedboot/avb
