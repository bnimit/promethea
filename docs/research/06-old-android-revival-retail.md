# Stream 6 — Old-Android revival, retail / fleet market

*Snapshot dated 2026-04-26. The agent for this stream did not have live web tooling available; specific numbers are directional and should be verified against VDC Research, IDC Worldwide Rugged Mobile Computer Tracker, the postmarketOS device wiki, and Esper / Emteria public materials before being used externally.*

## Old-Android revival landscape

- **LineageOS** — broadest device tree, but support tracks AOSP versions and depends on volunteer maintainers. Devices typically get 2-4 years past OEM EOL; once a maintainer drops a tree, it bitrots fast.
- **DivestOS** — was a hardened LineageOS fork by Tad Wilkes that explicitly extended life of EOL devices with backported security patches and CVE patching for kernels Google / OEMs abandoned. Supported ~80+ devices, including Snapdragon 600/800-era hardware. Uniquely focused on past-EOL longevity — *but shut down December 2024* (see Stream 1).
- **/e/OS** — de-Googled LineageOS derivative with a managed update channel; ships on refurbished Murena / Fairphone hardware.
- **postmarketOS** — Alpine-based, targets mainline Linux kernel rather than vendor BSP; ~250+ devices listed but only a small subset are "daily-driver" tier. The right reference for a non-Android Linux future.
- **GrapheneOS / CalyxOS** — hardened but limited to Pixels.
- **AOSP custom builds** by integrators (Emteria, /e/OS for Business, Kyt) productize Android for kiosks.

**Maintenance burden / kernel reality:** Qualcomm / MediaTek / Exynos ship downstream BSP kernels (3.x-4.x) that never reach mainline. Mainlining work is done by volunteers (Linaro; Konrad Dybcio for Qualcomm SDM/MSM SoCs; Sumit Semwal; AngeloGioacchino Del Regno for MediaTek). Samsung Exynos has the worst story — almost no mainline GPU support. Qualcomm is the best of the three thanks to Freedreno (Rob Clark) for Adreno GPUs.

## Old-Android retail / fleet installed base

- **Rugged handhelds:** Zebra (TC5x/TC7x/TC8x lines, ~60% market share per VDC Research), Honeywell (CT40/CT60), Datalogic, Panasonic Toughbook, Bluebird, Point Mobile. Mostly Snapdragon 660/6xx-class running Android 8-11.
- **POS:** Verifone Engage / Carbon, Ingenico AXIUM, PAX A-series, Square Terminal, Toast (Elo), SumUp Solo. Mostly Android 7-10 with custom hardened forks.
- **Restaurant ordering / kiosks:** Toast, Square KDS, Lightspeed, Elo I-Series, Samsung Kiosk.
- **Healthcare:** Zebra TC52-HC, Honeywell CT40-HC on workstations-on-wheels.
- **Digital signage:** BrightSign (proprietary), Samsung Tizen and Android, LG webOS, generic Android sticks (Rockchip / Amlogic).
- **Logistics / fleet tablets:** Samsung Tab Active 2/3, Zebra ET4x, Geotab / Samsara in-cab.

VDC Research and IDC put the rugged Android handheld installed base at roughly 50-70 million units globally, with average device life of 4-6 years against an OEM software window of 3-4. Replacement triggers are typically: (a) Android Enterprise Recommended cert lapsing, (b) Google Mobile Services drift breaking apps, (c) battery wear, (d) Wi-Fi 6 / security mandate from the customer's IT.

## Pain points

- **GMS dependency:** Zebra and Honeywell devices ship GMS but lose certification at EOL; payment apps (Stripe Terminal SDK, Adyen) often require Play Integrity, which silently breaks on de-Googled forks.
- **Forced updates breaking integrations:** Android 11 → 12 scoped-storage and Bluetooth permission breakage hit fleet apps; many ISVs stayed on 10.
- **MDM cost:** SOTI MobiControl, VMware Workspace ONE, Ivanti Avalanche, Scalefusion, 42Gears SureMDM run $3-8/device/month — a real line item at fleet scale.
- **Telemetry:** GMS phones home regardless; for healthcare (HIPAA) and EU retail (GDPR), this is increasingly a procurement blocker.
- **App-store gatekeeping:** Play Protect can disable side-loaded fleet apps; managed Google Play requires a Google Workspace tenant.
- **EOL while mechanically fine:** the headline. Closing the Loop and Fairphone studies show > 60 % of retired enterprise devices are functionally healthy.

Concrete signals: Zebra's "Lifeguard for Android" (lifecycle extension to ~10 years on select devices) exists *because* customers complained loudly. Honeywell's Sentinel does the same. Both are paid add-ons.

## MDM / device-management table stakes

Any replacement OS must ship or integrate:

- **Kiosk / single-app lock** (equivalent to Android Lock Task Mode + COSU)
- **Zero-touch provisioning** (analog to Android Zero-Touch, Knox Mobile Enrollment, Apple ADE)
- **OTA with staged rollout, scheduling, rollback**
- **Remote attestation / device health** (the Play Integrity replacement)
- **Policy engine:** Wi-Fi / VPN / cert push, app whitelist, USB / peripheral lockdown
- **Compliance reporting** for PCI-DSS, HIPAA, SOC 2
- **AOSP / Android Management API compatibility shim** so existing MDM consoles (Intune, Workspace ONE, SOTI, Scalefusion, Hexnode) can manage it without writing a new console. **The single biggest distribution lever.**

## SoC driver longevity

- **Qualcomm Snapdragon 600/800/801/805** — SDM / MSM mainline support is partial; Freedreno covers Adreno 3xx/4xx but modem, camera ISP, video are weak. Daily-driver-able on a few targets.
- **Snapdragon 820/835/845** — best-supported old Qualcomm tier in mainline thanks to Konrad Dybcio's work — OnePlus 6 (msm8998 / sdm845), Pixel 3/3a (sdm670/845), PocoF1 are postmarketOS reference platforms.
- **Snapdragon 6xx (660/665/670)** — partial; Pixel 3a is solid.
- **Samsung Exynos 7/8/9** — poor. Mali GPU via Panfrost helps but display / modem / camera require heavy reverse engineering. Galaxy S-series is largely unviable as a daily driver on mainline.
- **MediaTek MT67xx/MT68xx** — improving fast (Helio P/G mainlining by Collabora and AngeloGioacchino Del Regno), but enterprise rugged devices on MTK are rarer.

The cliff: anything pre-2017 (pre-sdm835) and anything Exynos. Sweet spot: 2018-2021 Qualcomm 6xx/8xx.

## Hardware revival candidates

Best mainline-Linux support today, ranked by retail relevance:

1. **Pixel 3a / 4a / 5** — postmarketOS daily-driver tier; minimal retail fleet but cheap on secondary market.
2. **OnePlus 6 / 6T** — flagship postmarketOS target; consumer, not fleet.
3. **Fairphone 4 / 5** — already shipping /e/OS; modular, repair-friendly, the obvious "certified hardware partner."
4. **Zebra TC52 / TC57 / TC72 / TC77** (Snapdragon 660) — *the* fleet target. Massive installed base. No mainline support today; would need investment.
5. **Honeywell CT40 / CT60** (Snapdragon 660 / 6490) — same story.
6. **Samsung Galaxy Tab Active 2/3** — Exynos, hard.
7. **Surface Duo** — sdm855 / sdm865, niche, has community Linux work.

Strategy implication: Pixel 3a and OnePlus 6 are the "we can demo today" devices. **Zebra TC5x is the "we must invest 6-12 months of mainlining" prize.**

## Business model precedents

- **Murena / e.foundation** — sells refurbished Fairphone, Pixel, Samsung pre-flashed with /e/OS plus a cloud subscription. Profitable at small scale.
- **Back Market** — refurb marketplace, $1B+ GMV, doesn't flash custom OS but is the obvious channel partner.
- **Closing the Loop** — circular-economy fleet recovery; pays customers per recovered device.
- **Emteria.OS** — sells AOSP-for-industrial as a subscription with MDM — closest direct analogue to the proposed product.
- **Esper.io** — Android-based "DevOps for devices" platform funded ~$100M; their pitch validates the market.
- **Zebra OneCare / Honeywell Edge Services** — incumbents already monetize lifecycle extension at $50-150 / device / year — that's the price umbrella.

## Synthesis: the strongest pitch

> "Take the Zebra TC52, Honeywell CT40, and Samsung Tab Active fleets that retailers and 3PLs already own — devices going EOL with 3+ years of mechanical life left — and give them a Rust-based, GMS-free, mainline-Linux second life with a managed-update SLA, drop-in MDM compatibility, and PCI / HIPAA-grade attestation. Cut device-refresh capex by 40-60 % and eliminate Google telemetry from the transaction path."

### Top 5 highest-leverage features to ship

1. **MDM compatibility shim** speaking the Android Management API and SOTI / Workspace ONE / Intune protocols so existing consoles "just work." Without this, enterprise distribution is dead.
2. **Hardened kiosk / single-app lock with verified boot and remote attestation** — Rust's memory-safety story is the differentiator vs. AOSP here; lean on it for PCI-DSS and HIPAA pitches.
3. **Certified peripheral stack:** Zebra / Honeywell barcode scanner SDKs (DataWedge equivalent), Verifone / Ingenico / PAX payment HALs, ESC/POS and ZPL label-printer drivers, Bluetooth LE beacons. Retail dies without these.
4. **OTA with staged rollout, A/B partitions, rollback, and a 7-10 year update SLA** — directly competes with Zebra Lifeguard and is the literal product promise.
5. **Zero-touch refurbishment pipeline:** a re-flashing service plus certified-refurbished SKUs (start with Pixel 3a/4a and OnePlus 6 for proof; invest in Zebra TC52 mainlining as the wedge into rugged fleets). Partner with Back Market / Closing the Loop for sourcing and Fairphone for new-device co-brand.

### Deployment models

- **Re-flash-as-a-service** for existing fleets (per-device fee + annual support).
- **Certified refurbished hardware** sold direct (Murena model).
- **OEM partnership** with a rugged vendor wanting a Google-free SKU for EU / healthcare buyers.

### Integration surface to commit to on day one

Android Management API shim, Stripe / Adyen / Square Terminal SDKs, Zebra DataWedge-compatible barcode intent API, ESC/POS + ZPL printing, Wi-Fi 6 / WPA3-Enterprise with 802.1X, MDM agents for Intune / Workspace ONE / SOTI / Scalefusion, attestation API for payment processors.

## Caveat

Web verification was not possible in this run; before publishing externally, re-pull current numbers from VDC Research, IDC Worldwide Rugged Mobile Computer Tracker, postmarketOS device wiki, DivestOS device list, and Esper / Emteria public materials.
