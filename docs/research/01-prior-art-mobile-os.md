# Stream 1 — Prior art on alternative mobile operating systems

*Snapshot dated 2026-04-26. Findings are summarized from a research agent run; URLs cited but not mirrored. Re-verify before acting on any specific claim.*

## Project-by-project

**GrapheneOS.** Alive, growing. Small distributed core team after Daniel Micay stepped down in 2023; donation- and contract-funded foundation. ~400,000 active devices, ~1 in 25 Pixel users. Got right: ruthless focus on hardening AOSP on a single OEM's secure hardware (Pixel Titan M); credible threat-model marketing. Limit: Pixel-only lock-in; March 2026 Motorola partnership announcement is their first attempt to break out, devices not until late 2026/2027. **Lesson:** pick one secure-hardware partner and ship a hardened AOSP variant — do not invent a kernel/userland from scratch.

**/e/OS (Murena).** Alive, slow-growing EU privacy poster child. Foundation + Murena commercial arm (Gaël Duval), small-mid team. Install base undisclosed; Fairphone partner sold ~103k units in 2024 (only a fraction on /e/OS). Got right: turnkey commercial model — pre-installed phones, Murena Cloud, MVNO, Hiroh phone (Feb 2026). Limit: still an AOSP+microG fork; performance/UX consistently lags GrapheneOS; weak app-compat narrative. **Lesson:** productize the OS as a service (cloud + SIM + hardware bundle), don't just ship an image.

**CalyxOS.** Zombie-recovering. Founder Nicholas Merrill and tech lead Chirayu Desai both departed 2025; development paused, signing-key ceremony in Feb 2026, infra outage compounded delays. Now hiring under interim ED. Got right: non-profit governance, MicroG-friendly stance. Limit: organizational fragility — losing two key people nearly killed it. **Lesson:** bus-factor is existential; institutionalize signing/release infra before scaling features.

**DivestOS.** Dead. Final release 18 Dec 2024, shutdown announced 23 Dec 2024 after a decade. Solo maintainer "Tavi"; cited lack of funding. **Lesson:** solo-maintainer privacy OSes do not survive — fund or fold.

**postmarketOS.** Alive enthusiast platform, not a daily driver. ~723 devices supported as of Feb 2026; pivoted to a generic mainline-kernel architecture (huge maintenance win). **Lesson:** mainline-kernel-first dramatically reduces per-device debt — bake this in from day one.

**Sailfish OS / Jolla.** Zombie-revived. Filed bankruptcy; reborn as "Jollyboys" with original founders. New Jolla Phone Dec 2025 — first 2,000 sold out, second 2,000 on pre-order, ships H1 2026. EU/UK/CH/NO only. Some closed parts open-sourced since Sept 2025. Got right: explicitly EU-sovereignty positioning. Limit: 13 years and still ~tens of thousands of units; Android compat layer (Aliendalvik) is closed and proprietary. **Lesson:** EU-sovereignty messaging works for press but not scale; you need Android app compatibility from day one and it must not be a closed paid add-on.

**Ubuntu Touch / UBports.** Alive but tiny. Volunteer org, transitioning from 20.04 to 24.04-1.x branch through 2025-2026. Limit: Qt/Click app stack nobody develops for; near-zero install base. **Lesson:** a bespoke app framework with no developer economics is a death sentence — Android (or web) compat is mandatory.

**Tizen.** Mobile: dead. Watches: dead Sept 2025. TVs: being replaced by One UI. **Lesson:** even an OEM with global distribution + billions in cash cannot will a mobile OS into existence without developer pull.

**HarmonyOS / OpenHarmony.** Alive, the only credible challenger. ~900M devices in the broader ecosystem; ~51M "pure" HarmonyOS NEXT (no Android compat layer) by early 2026; 18% of China smartphone market, passed iOS in China. Got right: state-backed, OEM-backed, sanctions forced commitment, microkernel-ish multi-device story. Limit: Chinese-market only; no traction in EU/US. **Lesson:** a new mobile OS only works with (a) a guaranteed hardware channel and (b) a forced developer migration event. Without coercion, devs do not port.

**KaiOS.** Stalling. >100M JioPhones lifetime; new TCL Flip 4 5G launched May 2025. WhatsApp dropped support Feb 2025. **Lesson:** even huge unit numbers cannot save you when major apps walk; keep the OS open enough that the community can patch around vendor app departures.

**Fuchsia.** Alive but radically narrowed. Ships only on Nest Hub smart displays (Fuchsia v16 rolling out). Starnix (Linux ABI) and RISC-V work continues. **Lesson:** even Google with unlimited capital cannot replace Android on phones from a clean-slate kernel — start with a niche device class.

**Firefox OS (post-mortem).** Dead 2016, lineage continues as KaiOS. Failure causes: performance gap vs cheap Android; weak app ecosystem; carriers/OEMs treated it as an experiment; targeted emerging markets where Android-cheap won on price; v1.0 shipped without a calculator. **Lesson:** "web apps are enough" was wrong then and is still wrong — native app-compat is non-negotiable.

**Librem 5 / PureOS.** Zombie. Purism finally finished shipping all backlog Librem 5 orders; price dropped $999→$699; PureOS Crimson still in beta Dec 2025. **Lesson:** custom hardware + custom OS + tiny team = years of delays; pick one moonshot, not three.

## Graveyard pattern

The dominant cause of death is **app-ecosystem starvation amplified by single-point organizational failure**. Every project that died (Firefox OS, DivestOS, Tizen mobile) or stalled (Ubuntu Touch, Librem 5, Sailfish, KaiOS) failed to make Android/iOS-class apps run well, leading to user attrition, which starved the project of revenue and credibility, which made the small team or sole maintainer either lose funding (DivestOS, Purism), get poached or burn out (CalyxOS leadership exodus), or get deprioritized by the parent (Mozilla, Samsung, Canonical). The bespoke-app-framework bet has a 0% success rate. The only project that broke this pattern, HarmonyOS NEXT, did so by forcing a state-backed, sanctions-driven mass developer migration — which is not a strategy you can copy.

## What's actually working

The survivors converge on a tight pattern: **fork AOSP or use mainline Linux, harden it, ship Android app compat unchanged, target a specific hardware partner, and run as a foundation with paid contributors plus a commercial arm.** GrapheneOS owns the security-maxi niche on Pixel hardware; Murena owns the EU-mainstream-privacy niche with bundled hardware/cloud/MVNO; CalyxOS is wounded but still fits the same mold. postmarketOS shows that a mainline-kernel-first architecture is the only sustainable way to support more than ~10 devices with a small team.

For a new EU + India sovereignty Rust-based entrant the actionable distillation is:

1. Do not write a new kernel — use Linux mainline or hardened AOSP and put Rust in userland/services.
2. Android app compatibility is non-negotiable from v0.1 and must be open and free.
3. Lock in OEM partner(s) (Fairphone or SHIFT for EU; Lava for India) before writing significant code.
4. Structure as a foundation + commercial entity from day one with at least a 5-person funded core to survive any single departure.
5. Ship a productized bundle (device + OS + cloud + SIM) for at least one downstream variant — flashable image alone has never reached scale.

## Sources (URL only)

- https://en.wikipedia.org/wiki/GrapheneOS
- https://www.androidheadlines.com/2025/10/grapheneos-may-break-pixel-exclusivity-with-a-new-phone-in-2026.html
- https://lemmy.world/post/39170839
- https://community.e.foundation/t/murena-roadmap-2026-and-beyond/78998
- https://techcrunch.com/2024/01/23/murena-android-google-mvno-mobile-plans/
- https://e.foundation/leaving-apple-google-wrapping-up-the-year-2025-with-e-os-and-murena/
- https://calyxos.org/news/2025/08/01/a-letter-to-our-community/
- https://calyxos.org/news/2025/12/17/calyxos-progress-update/
- https://alternativeto.net/news/2024/12/secure-mobile-operating-system-divestos-ends-after-a-decade-of-development/
- https://en.wikipedia.org/wiki/DivestOS
- https://www.sitepoint.com/postmarketos-fdroid-2026-status/
- https://postmarketos.org/
- https://www.androidpolice.com/sailfish-os-is-back-on-a-new-jolla-phone/
- https://forum.sailfishos.org/t/sailfish-community-news-2nd-april-2026-final-payment/28838
- https://en.wikipedia.org/wiki/Sailfish_OS
- https://forums.ubports.com/topic/10899/status-update-on-ubuntu-touch-24.04-1.x-march-april-2025
- https://samsung.gadgethacks.com/news/samsung-kills-tizen-smartwatches-by-september-2025/
- https://www.talkandroid.com/491688-samsung-tv-one-ui-tizen/
- https://en.wikipedia.org/wiki/HarmonyOS
- https://chinabizinsider.com/huaweis-harmonyos-hits-18-share-but-rivals-stay-away/
- https://en.wikipedia.org/wiki/KaiOS
- https://www.mymobileindia.com/whatsapp-to-end-support-for-kaios-devices-by-february-2025-impacting-feature-phone-users/
- https://en.wikipedia.org/wiki/Fuchsia_(operating_system)
- https://9to5google.com/2024/03/01/fuchsia-16-nest-hub-whats-new/
- https://en.wikipedia.org/wiki/Firefox_OS
- https://www.quora.com/Why-did-Firefox-OS-fail
- https://en.wikipedia.org/wiki/Librem_5
- https://puri.sm/posts/all-librem-5-smartphones-have-shipped/
- https://puri.sm/posts/pureos-crimson-development-report-december-2025/
