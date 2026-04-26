# Stream 7 — India sovereign mobile OS landscape

*Snapshot dated 2026-04-26.*

## India digital-sovereignty initiatives (status as of Q1 2026)

- **BharOS (JandKops, IIT Madras Pravartak):** Launched Jan 2023 with high-profile testing by Ministers Vaishnaw and Pradhan. Reality in 2025-26: roughly **500 Lava-built handsets** shipped, deployment confined to "organisations with stringent privacy requirements" — i.e., effectively a pilot for a few defence / PSU clients. No consumer SKU, no app store, no carrier integration. **PR-positive, adoption-anaemic.**
- **Indus OS / Indus Appstore:** OS layer **dead** post-PhonePe acquisition ($90M, 2022). PhonePe pivoted purely to Indus Appstore (launched Feb 2024, ~2M downloads, 350K apps; zero-commission third-party billing). **Lesson: the distribution layer survived, the OS layer didn't.**
- **Jio's OS plays:** **KaiOS** (16 % stake, 2018) drove ~100M JioPhone units but stalled as 4G feature-phones lost to sub-Rs.5K Androids. **Pragati OS** is Android-Go reskin co-developed with Google for JioPhone Next — not sovereign, Google-dependent. **Jio Bharat Platform** is the new minimal OS for the Rs.999 Jio Bharat V2; Karbonn licensed it.
- **Maya OS (DRDO / C-DAC / NIC):** Ubuntu-based desktop OS with Chakravyuh EDR, deployed across Defence Ministry endpoints from Aug 2023; expanded across services by 2025. **Sets a procurement precedent — but desktop, not mobile.**
- **"IndiaOS" advocacy:** Takshashila and others openly skeptical that another govt-led OS attempt will succeed.

## Indian regulatory landscape

- **DPDP Act 2023:** Substantive provisions effective ~May 2027 (18-month runway from notification). OS vendor is a **Data Fiduciary** for any first-party telemetry / account systems; needs consent UX in Indic languages, breach notification to DPB, and data-minimisation by design.
- **CERT-In 2022 directions (Sec 70B IT Act):** 6-hour incident reporting, **180-day log retention within Indian jurisdiction**, NTP sync to NIC / NPL. VPN / VPS / cloud subset has 5-year KYC retention — does NOT apply to a pure OS vendor, but does apply to any VPN you bundle.
- **MTCTE (TEC) + BIS CRS:** Phase VI mandatory from Aug 24, 2025 covers handsets and CPE; user manuals / QR docs in Indic languages mandatory from Sept 19, 2025. Affects OEM partner, not OS, but the OS must surface required Indic docs.
- **Public Procurement (Make in India) Order 2017/2020 + National Policy on Software Products 2019:** Class-I local-content suppliers (> 50 % local value-add) get purchase preference; Maya OS sets the precedent. **This is the procurement door to walk through.**

## IndiaStack — what to bake in as first-party

| Stack | API maturity | OS-native potential | Verdict |
| --- | --- | --- | --- |
| **UPI Intent** | Very mature; NPCI deprecating UPI Collect Nov 2025 in favour of Intent / QR | **System-level payment intent** (replace per-app PSP SDKs) | **Highest leverage — bake in** |
| **Aadhaar eKYC + DigiLocker** | OAuth-style partner APIs, ~70 doc types | System-level identity / wallet provider | **High — bake in as IdentityProvider** |
| **Bhashini** | REST APIs for ASR / TTS / MT in 22 languages; **Intel partnership for offline AI-PC inference (2026)** | On-device IME, system TTS, live captions | **High — clear differentiator vs Google** |
| **ABDM (ABHA Health)** | FHIR-based; ABHA address as health ID | Wallet card, not deep system integration | Medium — surface in identity wallet |
| **ONDC** | Buyer / seller protocol APIs | Shopping intent across apps | Medium — hard to win without merchant traction |
| **CoWIN / Jan-Dhan** | Largely closed / declining relevance | Skip | Low |

## Mobile market structure

- **2025 share** (Omdia / IDC / Canalys): Vivo ~19 %, Samsung 14-16 %, OPPO 13 %, Xiaomi 9-13 %, Realme ~10 %, Motorola ~8 %, Apple 7-10 %. Vivo leads full-year.
- **ASP** ~Rs.21K (~$250); the volume centre is Rs.10K-25K ($120-300). Apple's premium share grew, but **a $400+ sovereign-OS device only has a niche of < 5 % of the market** — primarily prosumer / government.
- **SoC:** MediaTek dominant in mass (5G entry), Qualcomm gaining 2025, UNISOC fading. Plan for **MediaTek Dimensity reference platform** first.
- **OEM partner candidates:** **Lava** (already BharOS-validated, "design in India", PLI beneficiary) is the obvious primary. **Karbonn** (Jio Bharat licensee) and **Micromax** (IIT-M ties) are secondary. **JioBharat brand** would require political alignment with Reliance — high reward, high lock-in.

## Carriers

- Jio ~470M / Airtel ~390M / Vi / BSNL. Jio uses VoLTE-only; Airtel / Vi support legacy fallback. Both maintain **IMS device whitelists** keyed off IMEI TAC + carrier config — non-Google ROMs frequently lose VoLTE until manual carrier-config push.
- **No precedent for carrier-bundled non-Google smartphone at scale** above feature-phone tier. JioPhone Next is Pragati-on-Android. A sovereign OS will need **direct TEC certification + per-carrier IMS profile negotiation** — budget 6-9 months per major carrier.

## Languages & accessibility

- **FOSS rendering already solid:** HarfBuzz handles Indic shaping (Pango delegates entirely to it since 1.31). Lohit (21 languages) and Samyak font families cover Devanagari conjuncts, Tamil ligatures, Bengali matras adequately.
- **IMEs:** Indic Keyboard (FOSS), Varnam (transliteration) cover most scheduled languages.
- **Bhashini on-device:** Vidyalekha already runs offline on PCs with Intel; Rest of World (Mar 2026) profiles an open-source handheld using Bhashini offline for 22 Indic languages — **proof that on-device Bhashini at phone-scale is viable in 2026**. This is strategically huge: a Rust OS can ship Bhashini as a system service (TTS, ASR, live caption, real-time translate) and beat Google Assistant in non-Hindi Indic.

## App ecosystem & Play Integrity risk

The **non-negotiable Indian app set** that hits Play Integrity hard:

- **Banking / finance:** HDFC, SBI YONO, ICICI iMobile, Paytm, PhonePe, GPay India, BHIM — all use Play Integrity DEVICE_INTEGRITY tier; **most break on GrapheneOS / microG today.**
- **Tolerable:** WhatsApp, Hotstar / JioCinema, Zomato / Swiggy, Ola / Uber, IRCTC — generally work without strong attestation.
- **Government:** ABHA, DigiLocker, mAadhaar — work; some require basic integrity only.

**This is the killer problem.** Mitigations: (a) ship Android **hardware-backed key attestation** with keys whitelisted by RBI / NPCI directly (precedent path via MeitY); (b) Indus Appstore as primary distribution (already commission-free, India-first); (c) work with NPCI to make UPI Intent the system-level payment surface so PhonePe / GPay's app-attestation matters less.

## Funding

- **PLI 2.0 IT Hardware:** Rs.17,000 cr; cumulative production Rs.14,463 cr by Sept 2025. Components PLI Apr 2025: Rs.22,919 cr.
- **DLI for chip design**, Semicon India (Rs.76,000 cr committed), MeitY TIDE 2.0, Startup India seed fund — all hardware-skewed; **no dedicated software / OS bucket.**
- **State appetite:** measurable post-Maya-OS, but BharOS underperformance has produced **fatigue, not enthusiasm**. Next entrant must show *traction first, ask for grants second.* Realistic path: anchor government procurement (Maya-mobile equivalent) → use that to unlock MeitY co-funding → then PLI for OEM partner.

## The single biggest lesson from prior plays

**Distribution is sovereignty; an OS without a forced distribution channel dies.** KaiOS got 100M units only because Jio bundled it; the moment Jio pivoted, KaiOS collapsed. Indus OS's *appstore* survived only because PhonePe forced it onto Xiaomi. BharOS has no carrier, no OEM volume mandate, no procurement quota — so it's stuck at 500 units. **You cannot win India on consumer pull. You win on a B2G / carrier mandate that forces 5-10M units, then expand.**

## EU + India dual-launch architecture

**Shared core (single Rust codebase):**
- Microkernel / capability model, Wayland compositor, Rust app framework, hardware attestation, sandboxing, update infra, Bluetooth / Wi-Fi / cellular stacks, web engine.

**Regional capability layer (compile-time + runtime feature flags):**
- **Identity:** EU = eIDAS 2.0 wallet / national eID; India = Aadhaar / DigiLocker.
- **Payments:** EU = SEPA Instant + Wero / EPI; India = UPI Intent as system-level PaymentService.
- **Languages / IME:** EU = 24 official; India = 22 scheduled with Bhashini on-device.
- **Default search / maps:** EU = Qwant / HERE; India = MapMyIndia / Mappls + Bhashini-aware search.
- **Compliance daemons:** EU = GDPR consent + DMA gatekeeper compliance; India = DPDP consent + CERT-In logging service + 180-day Indian-region log store.
- **Certification:** EU = CE / RED / Common Criteria; India = BIS CRS + TEC MTCTE + BIS local-content.

**Hardware tier:** EU launches at premium (€500-800 reference device, Snapdragon 8-class). India needs a **second SKU at Rs.18-25K ($220-300)** on Dimensity 7000-class with the same OS image — not a launch-day requirement but mandatory by month 12.

## Synthesis

### 5 highest-leverage advantages for an India sovereign Rust OS

1. **UPI Intent as a first-class system payment API** — NPCI's Nov 2025 Collect-deprecation hands you a clean integration moment Google / Apple won't replicate.
2. **On-device Bhashini** — 22-language ASR / TTS / MT running offline beats Google Assistant in tier-2/3, where Indic-first matters most. Differentiation Apple can't match.
3. **Make-in-India procurement preference** — Maya OS proves the pathway. Anchor 1-2M govt / PSU / defence devices via Class-I local-content status before chasing consumers.
4. **Lava partnership** — only OEM with full design-in-India capability, BharOS-experienced, PLI-funded, willing to ship low volumes.
5. **DPDP / CERT-In tailwind** — privacy-first Rust OS is the *natural* architecture for compliance; Google / Apple are bolting it on.

### 3 most dangerous India-specific risks

1. **Play Integrity / banking attestation** — if HDFC, SBI, Paytm, PhonePe don't run, the OS is dead on arrival regardless of how good UPI Intent is. **Mitigation must be locked before any consumer launch.**
2. **Carrier IMS whitelisting** — Jio and Airtel can silently kill VoLTE; without voice, you have a Wi-Fi tablet. Treat as a gating dependency.
3. **"Sovereign OS fatigue"** — BharOS's underperformance has poisoned the well with MeitY and the press. You will be benchmarked against it from day one. **Don't announce until you have 100K committed units.**

### Phasing recommendation

**India follows EU codebase milestones by 12 months at the SKU launch level, not co-equal at launch, not a separate program.** Co-equal launch fails because: (a) banking attestation needs RBI / NPCI engagement that takes 9-12 months *after* you have a shippable build; (b) MTCTE Phase VI + BIS adds 4-6 months of certification on top of CE; (c) Lava OEM tooling-up needs the EU launch as proof-of-life. A separate program duplicates engineering and dilutes the Rust codebase thesis.

**However**, per the AOSP-class reframe applied to the design, *both EU and India are co-equal in the codebase from day one* — the Regional Capability Layer ensures a single repository serves both markets. The phasing concerns described above apply to *reference device SKU launches*, which are a downstream commercial decision, not an upstream gating event.

## Sources

- https://www.iitm.ac.in/happenings/press-releases-and-coverages/iit-madras-incubated-firm-develops-indigenous-atmanirbhar
- https://mysuruinfrahub.com/bharos-indias-very-own-mobile-os-will-be-launched-on-lava-indias-indigenously-developed-smartphone/
- https://www.business-standard.com/companies/start-ups/fintech-major-phonepe-takes-on-google-apple-with-homegrown-indus-appstore-124022101074_1.html
- https://inc42.com/buzz/jiophone-next-to-run-on-pragati-os-designed-for-india/
- https://kaios.dev/2023/07/jio-bharat-4g-v2-and-kaios/
- https://en.wikipedia.org/wiki/Maya_(operating_system)
- https://takshashila.org.in/research/developing-an-indigenous-mobile-operating-system
- https://www.ey.com/en_in/insights/cybersecurity/decoding-the-digital-personal-data-protection-act-2023
- https://www.cert-in.org.in/PDF/CERT-In_Directions_70B_28.04.2022.pdf
- https://www.mtcte.tec.gov.in/aboutMTCTE
- https://alephindia.in/dot-enforces-mtcte-certification-for-telecom-equipment-from-august-2025.php
- https://www.dpiit.gov.in/static/uploads/2025/07/91bede654c9ad6fca528dcdd24554980.pdf
- https://nishithdesai.com/default.aspx?id=6230
- https://www.roro.io/post/seamless-payment-flow-using-the-upi-intent-mechanism
- https://www.frslabs.com/frsblog/2023/10/12/digilocker-how-to-integrate-digilocker-api-into-your-web-or-mobile-app-for-kyc/
- https://munsifdaily.com/powerful-leap-intel-digital-india/
- https://restofworld.org/2026/current-ai-bhashini-open-source-handheld-device-2/
- https://my.idc.com/getdoc.jsp?containerId=prAP53921425
- https://omdia.tech.informa.com/pr/2026/jan/indias-smartphone-shipments-fell-by-1percent-in-2025-amid-softer-demand-and-cost-pressures-vivo-remains-market-leader
- https://discuss.grapheneos.org/d/29413-banking-without-the-play-integrity-api
- https://en.wikipedia.org/wiki/HarfBuzz
- https://github.com/herlesupreeth/CoIMS_Wiki
- https://www.meity.gov.in/static/uploads/2024/02/PLI-booklet__english.pdf
- https://kpmg.com/in/en/blogs/2025/05/from-assemblers-to-innovators-indias-22919-cr-push-to-dominate-electronics-components.html
