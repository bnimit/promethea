# Stream 5 — EU regulatory landscape

*Snapshot dated 2026-04-26.*

## Digital Markets Act (DMA) — 2025-2026 enforcement state

Apple and Google were designated gatekeepers for mobile OS, browsers, and app stores in September 2023. Enforcement has materially advanced: in **April 2025 the Commission fined Apple €500M and Meta €200M** for non-compliance — Apple's anti-steering rules were ruled illegal. Apple is moving to a unified business model by **January 1, 2026**, eliminating per-install Core Technology Fees on alternative app stores and sideloaded apps.

Articles most relevant to a new mobile OS:

- **Art. 6(3)** — gatekeepers must show choice screens for browsers/search and allow easy default-app changes. By spring 2025 Apple expanded defaults to dialer, messaging, translation, navigation, keyboards, password managers, call-spam.
- **Art. 6(4)** — sideloading and alternative app stores on iOS.
- **Art. 6(7)** — gatekeepers must give third parties **free, equivalent API access** to OS / hardware features (NFC, secure element, etc.), subject only to strict, proportionate security conditions. **This is the lever that lets a sovereign OS demand interoperability with the iOS / Android ecosystem.**

The FSFE and developers continue filing complaints (Dec 2025) that Apple's compliance is hostile, keeping enforcement pressure live through 2026.

## Cyber Resilience Act (CRA) — compliance reality

Two binding milestones:

- **2026-09-11** — vulnerability + severe-incident reporting begins. Manufacturers must report **actively exploited vulnerabilities to ENISA + national CSIRTs within 24 hours**, with a final report at 14 days when a fix exists. Applies to legacy products too.
- **2027-12-11** — full compliance: security-by-default, secure update mechanisms, **SBOM in machine-readable format covering top-level dependencies**, 10-year documentation retention, CE marking.

For an OS vendor, compliance means: secure-boot / integrity by default, signed update channels, disclosed PSIRT, SBOM (CycloneDX or SPDX), automated vulnerability monitoring across the dependency graph, and conformity assessment. An OS will likely fall into the **"important"** Class II category (operating systems for servers / desktops / mobile devices are explicitly listed), requiring third-party conformity assessment — not self-assessment.

## EUCC + Common Criteria — the procurement gate

The **EUCC scheme entered application 2025-02-27**, replacing fragmented national CC schemes. It uses Common Criteria (ISO/IEC 15408) with two assurance levels: **"substantial" (AVA_VAN 1-2)** and **"high" (AVA_VAN 3-5)**. Government / defense buyers want "high".

For mobile OS the relevant Protection Profile is **ETSI's Consumer Mobile Device Protection Profile**, certified by ANSSI in 2023 — the world's first comprehensive smartphone PP under CC. This is the realistic baseline a sovereign OS variant would target.

**Cost / timeline reality:** A CC EAL4+ evaluation for a complex target like a mobile OS typically runs **18-36 months and €500k-€2M+** in lab fees alone (CESTI / BSI-licensed labs), excluding internal engineering cost of producing security target docs, ADV / AGD / ATE evidence, and sustaining maintenance for each release. CSPN (France's lighter alternative, ~25-35 person-days, ~2 months) is a faster on-ramp but only meets "moderate attacker" assurance — useful for restricted / RESTREINT but not classified procurement.

## Gaia-X, EU OS, sovereignty initiatives

- **Gaia-X** is in implementation phase 2025 with ~180 data spaces and a Berlin Sovereignty Summit (Nov 2025), but is widely critiqued for letting AWS / Google / Microsoft inside, diluting its sovereignty premise. **No mobile workstream.**
- **EU OS** (Robert Riemann, EU institutions context): Fedora-based, KDE Plasma, bootable-container immutable desktop. **Pure proof-of-concept, desktop only**, no mobile plans, presented at FOSDEM 2026. Lessons: (1) sovereignty narrative resonates politically; (2) no central EU procurement mandate exists — adoption is voluntary by Member State agencies; (3) immutable / bootc approach is the modern architecture pattern.
- A new EU mobile OS would have **no peer initiative** — meaning open positioning but also no political slipstream.

## Public-sector procurement today

- **Germany (BSI):** Procurement gated on BSI approval for **VS-NfD** (RESTRICTED). The dominant stack is **Samsung Galaxy + Secusmart SecuSUITE for Samsung Knox** (BlackBerry-owned). In **October 2025** BlackBerry UEM became the first MDM certified by BSI for managing both Apple Indigo (iOS) and Samsung Knox in classified deployments — confirming Samsung+BlackBerry as the de facto German government stack.
- **France (ANSSI):** Two tiers — **CSPN** (first-level) and **Qualification** (recommended for OIVs / state). ANSSI certified the ETSI smartphone PP, opening a structured CC path. Atos / Eviden Hoox (now Lifelink Hoox T40, 2024) is the domestic incumbent for defense / professional secure mobile.
- **Netherlands (NCSC) / EU Commission DG-DIGIT** broadly follow CC + national assessments and lean on Samsung Knox + BlackBerry stacks; Apple Business Manager for managed iOS fleets.

**Driving requirements:** CC EAL4+ / EUCC "high", FIPS 140-2/3 for crypto modules, supply-chain transparency, VS-NfD / RESTREINT / RESTRICTED clearances, MDM integration.

**Incumbents:** Samsung Knox (volume + cert), BlackBerry SecuSUITE / UEM (classified comms layer), Apple Business Manager (where allowed), **Bittium Tough Mobile 3** (Finnish, 5G, manufactured in Europe, certifying for EU / NATO classification, deliveries 2026), Atos Eviden Hoox.

## Funding sources — realistic map

- **EIC Accelerator (Horizon Europe)** — up to **€2.5M grant + €10M equity** for TRL 6-8. Competitive but the right size for an OS company. Strings: equity component, EU IP-stay-in-EU expectations.
- **Sovereign Tech Agency (Germany)** — €17M budget projected for 2025 (up from €3.5M in 2022). Grants typically **€50k+** for "open digital base technologies." Funded postmarketOS via NGI; perfect for upstream / library work, not full product.
- **NGI Zero (NLnet-managed)** — **€5k-€50k** per project, 50M total across programmes 2022-2025. Concerning: NGI was cut from the Horizon Europe 2025 Work Programme draft, future uncertain.
- **Horizon Europe cluster calls** (Cluster 3 - Civil Security/Cybersecurity, Cluster 4 - Digital) — consortium grants €3-15M, 24-36 months, require multi-Member-State partners.
- **National defense:** France (DGA, AID), Germany (Cyberagentur), each capable of €5-50M for sovereign-tech bets.

Realistic stack for a new OS: Sovereign Tech Agency for upstream components + EIC Accelerator for company + national defense agency for productisation.

## Existing EU mobile sovereignty plays — what worked, what didn't

- **Bittium Tough Mobile 3 (Finland, 2025):** Android-based, 5G, manufactured in Europe via HMD Secure, targeting EU / NATO classification certs, deliveries 2026. **Working model:** niche defense / critical-infra, hardware + software bundle, deep certification investment.
- **Atos / Eviden Hoox (France, since 2012):** Hardened Android, sold to defense / intelligence. Continuously refreshed (Hoox T40, 2024). **Survives** as France-aligned defense play but never broke into mainstream enterprise.
- **Sirin Labs Solarin (Israel, 2016):** **Failure.** $14-17k luxury positioning, obsolete chipset at launch, laid off 1/3 of staff. **Lesson:** privacy / security as luxury good doesn't scale; you need procurement-channel anchoring not consumer marketing.
- **Purism Librem 5, Murena/e-OS, /e/OS, GrapheneOS:** privacy-first communities exist but lack EU-government certification depth.

## Synthesis for Promethea

### 5 most leverage-worthy regulatory tailwinds

1. **DMA Art. 6(7) interoperability mandate** — forces Apple / Google to expose APIs (NFC, SE, sensors) to a competing OS or its apps, dissolving the historical lock-in moat.
2. **DMA Art. 6(3) default-app + choice-screen regime** — creates legal precedent that "you must let users pick" — extensible argument for "you must let governments pick the OS."
3. **CRA's hostile environment for legacy stacks** — incumbents shipping closed Android forks face SBOM / 24h-disclosure burdens that favor an OS designed-from-scratch with reproducible builds and transparent supply chain.
4. **EUCC unification (Feb 2025)** — single EU certification recognized across 27 Member States replaces 17 fragmented schemes, cutting go-to-market certification cost dramatically for a new entrant.
5. **Political tailwind from Gaia-X / EU OS / sovereignty discourse + the absence of a mobile sovereign initiative** — open political lane and likely receptive funders (Sovereign Tech Agency, DGA, BMWK).

### 3 hardest compliance / certification mountains

1. **CC EAL4+ / EUCC "high" against the ETSI Mobile PP** — €1-2M+, 18-36 months, must be repeated for material releases. Without it, no Germany VS-NfD, no France RESTREINT, no defense procurement. **Per the AOSP-class reframe, this is downstream variant work, not upstream gating.**
2. **CRA Sept 2026 deadline + ongoing 24h vulnerability reporting** — requires production-grade PSIRT, machine-readable SBOM across the entire OS, signed / attestable build pipeline. **Unconditional baseline; bake into core project hygiene.**
3. **Hardware root-of-trust + secure-element supply chain** — EUCC "high" and BSI VS-NfD effectively require a trusted boot chain anchored in certified silicon. Either you partner with a Bittium / HMD-class hardware OEM (loss of margin and control) or you credential a Qualcomm / MediaTek / SiPearl pathway (long, expensive). **Single hardest non-software barrier.**

## Sources

- https://secureprivacy.ai/blog/digital-markets-act-dma-explained-2025
- https://www.techpolicy.press/understanding-the-apple-and-meta-noncompliance-decisions-under-the-digital-markets-act/
- https://fsfe.org/news/2025/news-20250924-01.en.html
- https://www.theregister.com/2025/12/16/apple_dma_complaint/
- https://digital-markets-act.ec.europa.eu/questions-and-answers/interoperability_en
- https://www.keysight.com/blogs/en/tech/nwvs/2025/09/11/one-year-countdown-to-eu-cra-compliance-september-11-2026-changes-everything
- https://fossa.com/blog/sbom-requirements-cra-cyber-resilience-act/
- https://www.onekey.com/press-release/cyber-resilience-act-phase-1--reporting-requirements-for-manufacturers-begin-in-2026
- https://digital-strategy.ec.europa.eu/en/policies/cyber-resilience-act
- https://certification.enisa.europa.eu/certification-library/eucc-certification-scheme_en
- https://www.enisa.europa.eu/news/an-eu-prime-eu-adopts-first-cybersecurity-certification-scheme
- https://www.etsi.org/newsroom/press-releases/2308-etsi-protection-profile-for-securing-smartphones-gains-world-first-certification-from-french-cybersecurity-agency
- https://cds.thalesgroup.com/en/hot-topics/what-do-different-anssi-qualificationscertifications-mean
- https://gaia-x.eu/focus-on-digital-sovereignty-in-europe-gaia-x-as-a-central-topic-at-the-2025-digital-summit/
- https://www.theregister.com/2025/03/25/eu_os_free_govt_desktop/
- https://eu-os.eu/faq
- https://www.samsungknox.com/en/blog/german-government-security-approvals-for-solutions-with-samsung-galaxy-devices
- https://blogs.blackberry.com/en/2025/10/bsi-certification-blackberry-uem
- https://www.bittium.com/defense-security/bittium-tough-mobile-3/
- https://atos.net/en/2024/press-release_2024_05_16/eviden-introduces-the-lifelink-hoox-t40-its-next-generation-secure-phone-solution
- https://eic.ec.europa.eu/eic-funding-opportunities/eic-accelerator_en
- https://interoperable-europe.ec.europa.eu/collection/open-source-observatory-osor/document/funding-open-source-case-study-sovereign-tech-fund
- https://nlnet.nl/NGI0/
- https://edri.org/our-work/european-commission-cuts-funding-support-for-free-software-projects/
