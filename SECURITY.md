# Security Policy

Promethea is a privacy-and-security-first project. This document describes how to report a vulnerability, what to expect from us, and what versions we support.

## Reporting a vulnerability

**Do not file a public issue, pull request, or chat message for a vulnerability report.** Use one of the private channels below.

### Channels

- **Email:** `security@promethea-os.org` (mailbox provisioned in Phase 0). PGP-encrypted preferred. Public key fingerprint published at the project website and as `docs/security/psirt-pubkey.asc` once Phase 0 lands.
- **GitHub Security Advisory:** open a private advisory on the repository (Security tab → Report a vulnerability).

### What to include

- Affected subsystem(s) and version(s).
- Reproduction steps or proof-of-concept.
- Impact assessment from your perspective (data confidentiality, integrity, availability, what threat-model boundary is crossed).
- Whether the vulnerability is being actively exploited (this triggers expedited handling and may trigger CRA reporting; see below).
- Whether you intend to publicly disclose, and on what timeline.

## What to expect

| Stage | Timeline |
| --- | --- |
| Acknowledgement of receipt | Within 24 hours (business and non-business days) |
| Triage and severity assessment | Within 72 hours |
| Status update | Weekly until disposition |
| Patch availability | Severity-dependent (see below) |
| Public disclosure | Coordinated with reporter; default 90 days; expedited for active exploitation |

### Severity-driven patch SLAs

| Severity | Definition | Patch SLA |
| --- | --- | --- |
| Critical | Remote unauthenticated code execution; key compromise; cross-app or cross-VM data leak; permission daemon bypass | 7 days |
| High | Privilege escalation; permission downgrade; integrity bypass under default config | 30 days |
| Medium | DoS; information leak under non-default config | 90 days |
| Low | Minor issues, hardening opportunities | Next stable release |

## EU CRA compliance

The EU Cyber Resilience Act requires manufacturers to report **actively exploited vulnerabilities** to ENISA and national CSIRTs **within 24 hours** of becoming aware, with a final report within 14 days. This obligation begins **2026-09-11**.

When a downstream variant of Promethea is shipped commercially in the EU, the variant maintainer is responsible for that reporting. The Foundation will:

1. Maintain a published list of variant maintainers and their CRA points of contact.
2. Provide the variant maintainer with all information needed for the 24-hour and 14-day reports as soon as we become aware.
3. Run a coordinated disclosure window so variant maintainers can patch in lockstep with public disclosure.

If you are reporting a vulnerability you believe is being exploited in the wild, **say so explicitly in your report** so we can trigger the expedited path.

## Supported versions

Until v1.0 (Phase 3, ~M18-M30), only `main` is supported. Pre-1.0 releases receive best-effort fixes only.

After v1.0:

| Version | Status | Security patches |
| --- | --- | --- |
| Latest stable (`X.Y`) | Supported | Yes |
| Previous stable (`X.Y-1`) | Supported for 12 months after the next release | Yes |
| Older stable | Unsupported | Only critical, only by paid integrators |

Long-term-support (LTS) tiers will be defined per audience pillar:
- Retail/fleet variants: 7-10 year SLAs (paid).
- EU/India sovereign variants: certification-cycle aligned (typically 5+ years).
- Community consumer: 2 stable releases at any time.

## Hall of fame

Researchers who responsibly disclose are credited in `docs/security/hall-of-fame.md` (created in Phase 0) unless they prefer to remain anonymous.

## Bounty

The Foundation does not currently fund a bounty program. The commercial integrator entity may operate a bounty program for paid variants once incorporated; details will be announced at that time.

## PSIRT roster and bus factor

Per [GOVERNANCE.md](GOVERNANCE.md), the PSIRT roster is published in [MAINTAINERS.md](MAINTAINERS.md). Roster size: ≥ 3 humans across ≥ 2 organizations. Quarterly roster review; on-call rotation documented.

## Out of scope

The following are *not* in scope for this security policy:
- Vulnerabilities in third-party software not vendored into the repository (report to the upstream).
- Vulnerabilities in downstream variants ship by other parties (report to the variant maintainer).
- Issues that depend on a physically-compromised device (hardware tamper threat-model is a per-variant concern, not a platform concern).
- Bugs in unreleased experimental code paths (`tools/`, `microkernel/` R&D track) — though we still appreciate the heads-up.
