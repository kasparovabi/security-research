# Security Research

Coordinated disclosures and security writeups by @kasparovabi

## Disclosures

| Date | Vendor / Product | Severity | Title | Status |
|---|---|---|---|---|
| 2026-04-29 | BasedHardware / Omi | CVSS 10.0 (Critical) | [Seventeen Vulnerabilities in Omi, Fourteen Days of Silence](2026-04-omi-disclosure.md) | Mostly fixed as of 2026-08-10; OMI-15 open ([update](2026-04-omi-disclosure.md#update--august-10-2026-day-117)) |

<!-- ADVISORIES:BASLA -->
## Published GitHub Security Advisories

Advisories published by the maintainers, with reporter credit accepted. Reports still in private triage are not listed here.

| Published | Repository | Severity | Summary | Advisory |
|---|---|---|---|---|
| 2026-10-09 | [embedded-society/altium-designer-mcp](https://github.com/embedded-society/altium-designer-mcp) | Medium | Unbounded allocation from file count fields crashes the server | [GHSA-p2qw-h8m7-5x69](https://github.com/embedded-society/altium-designer-mcp/security/advisories/GHSA-p2qw-h8m7-5x69) |
| 2026-09-30 | [clidey/whodb](https://github.com/clidey/whodb) | Medium | Read only mode bypassed by SELECT INTO | [GHSA-pvf2-6h4x-r24j](https://github.com/clidey/whodb/security/advisories/GHSA-pvf2-6h4x-r24j) |
| 2026-09-20 | [SylphxAI/pdf-reader-mcp](https://github.com/SylphxAI/pdf-reader-mcp) | Medium | Documented filesystem allowlist not implemented | [GHSA-qr3f-g5wf-225w](https://github.com/SylphxAI/pdf-reader-mcp/security/advisories/GHSA-qr3f-g5wf-225w) |
| 2026-09-17 | [Kronova-Intelligent-Systems/asset-intel-orchestration-engine](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine) | High | RLS policy exposed user email to the anon role | [GHSA-8h52-3cfj-cmrw](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine/security/advisories/GHSA-8h52-3cfj-cmrw) |

Last synced: 2026-10-10
<!-- ADVISORIES:BITIR -->

## Provenance

For the Omi disclosure, a separate page documents the full timeline with checkable artifacts: who reported what, when, and what the vendor's own commit history says about where the fix came from. Every claim there can be reproduced with a `git log` command.

- [Omi disclosure: provenance and timeline](omi-provenance.md)

## Methodology

Each writeup in this repository is published after:

1. A private report through the vendor's preferred coordinated-disclosure channel (typically GitHub Security Advisories).
2. A reasonable window for the vendor to acknowledge, triage, and patch.
3. A public-disclosure date communicated to the vendor in advance.

Where the vendor refuses to act, the disclosure window is shortened in proportion to the vendor's behavior, the public exposure of the affected code paths, and the regulatory clocks that may apply to affected users (HIPAA, GDPR, SOC 2 audit obligations).

No proof-of-concept exploit code is published in this repository. Technical detail is sufficient to reproduce findings against the source code; no payload, weaponized scanner, or live-target tooling is included.

## Contact

- GitHub: [@kasparovabi](https://github.com/kasparovabi)
- Email: kasparovabi@gmail.com
- For security reports about my own code, please open a private GitHub Security Advisory on the relevant repository.
