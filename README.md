# Security Research

Vulnerability reports I have made to open source projects, and the fixes that came out of them.

Reports go to the maintainers privately first, through GitHub private vulnerability reporting or the channel named in the project's SECURITY.md. An advisory is listed here only after the maintainers have published it. Recent reports also came with a proposed patch in the advisory's temporary private fork.

Contact: kasparovabi@gmail.com

## Disclosures

| Date | Vendor / Product | Severity | Title | Status |
|---|---|---|---|---|
| 2026-10-27 | GuyMannDude / mnemo-cortex | CVSS 3.1 High | [A wildcard CORS default opened the unauthenticated loopback API to web pages](2026-10-mnemo-cortex-cors-loopback.md) | Fixed in 4.27.0 ([GHSA-682p-grrx-wrwg](https://github.com/GuyMannDude/mnemo-cortex/security/advisories/GHSA-682p-grrx-wrwg)) |
| 2026-10-27 | devantler-tech / KSail | CVSS 3.1 Medium | [The local web UI checked Host only on writes, so a rebinding page could read cluster Secrets](2026-10-ksail-webui-dns-rebinding.md) | Fixed in 7.203.23 ([GHSA-g57x-rm7r-hf26](https://github.com/devantler-tech/ksail/security/advisories/GHSA-g57x-rm7r-hf26)) |
| 2026-10-27 | embedded-society / altium-designer-mcp | CVSS 3.1 Medium | [Count fields from a library file sized allocations, and a failed allocation aborts the server](2026-10-altium-designer-mcp-unbounded-allocation.md) | Fixed in 1.2.1 ([GHSA-p2qw-h8m7-5x69](https://github.com/embedded-society/altium-designer-mcp/security/advisories/GHSA-p2qw-h8m7-5x69)) |
| 2026-10-27 | SylphxAI / anymd (pdf-reader-mcp) | Medium | [The documented --allow-dir restriction was not implemented in the Rust server](2026-09-anymd-allow-dir-not-implemented.md) | Fixed in @sylphx/citra 5.0.2 ([GHSA-qr3f-g5wf-225w](https://github.com/SylphxAI/anymd/security/advisories/GHSA-qr3f-g5wf-225w)) |
| 2026-10-27 | Kronova Intelligent Systems / asset-intel-orchestration-engine | CVSS 3.1 High | [An RLS policy with USING (true) let the anon role read every profile](2026-09-kronova-profiles-rls.md) | No patched version listed ([GHSA-8h52-3cfj-cmrw](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine/security/advisories/GHSA-8h52-3cfj-cmrw)) |
| 2026-10-26 | clidey / WhoDB | CVSS 3.1 Medium | [SELECT ... INTO passed the read-only and write-approval gates](2026-09-whodb-select-into.md) | Fixed in 0.134.0 ([GHSA-pvf2-6h4x-r24j](https://github.com/clidey/whodb/security/advisories/GHSA-pvf2-6h4x-r24j)) |
| 2026-04-29 | BasedHardware / Omi | CVSS 10.0 (Critical) | [Seventeen Vulnerabilities in Omi, Fourteen Days of Silence](2026-04-omi-disclosure.md) | Mostly fixed as of 2026-08-10; OMI-15 open ([update](2026-04-omi-disclosure.md#update--august-10-2026-day-117)) |

<!-- ADVISORIES:BASLA -->
## Published GitHub Security Advisories

Advisories published by the maintainers, with reporter credit accepted. Reports still in private triage are not listed here.

| Published | Repository | Severity | Summary | Advisory | CVE |
|---|---|---|---|---|---|
| 2026-10-19 | [GuyMannDude/mnemo-cortex](https://github.com/GuyMannDude/mnemo-cortex) | High | Default CORS allow_origins wildcard on unauthenticated loopback server allows cross-origin data exfiltration and state mutation | [GHSA-682p-grrx-wrwg](https://github.com/GuyMannDude/mnemo-cortex/security/advisories/GHSA-682p-grrx-wrwg) |  |
| 2026-10-17 | [devantler-tech/ksail](https://github.com/devantler-tech/ksail) | Medium | Local web UI accepts read requests from any Host, letting a DNS rebinding page read cluster Secrets | [GHSA-g57x-rm7r-hf26](https://github.com/devantler-tech/ksail/security/advisories/GHSA-g57x-rm7r-hf26) |  |
| 2026-10-09 | [embedded-society/altium-designer-mcp](https://github.com/embedded-society/altium-designer-mcp) | Medium | Unbounded allocation from attacker-controlled count fields in the Altium library parsers aborts the MCP server (DoS) | [GHSA-p2qw-h8m7-5x69](https://github.com/embedded-society/altium-designer-mcp/security/advisories/GHSA-p2qw-h8m7-5x69) |  |
| 2026-09-30 | [clidey/whodb](https://github.com/clidey/whodb) | Medium | Read-only mode and the write-approval gate are bypassed by SELECT ... INTO | [GHSA-pvf2-6h4x-r24j](https://github.com/clidey/whodb/security/advisories/GHSA-pvf2-6h4x-r24j) |  |
| 2026-09-20 | [SylphxAI/anymd](https://github.com/SylphxAI/anymd) | Medium | Documented --allow-dir / MCP_PDF_ALLOWED_DIRS filesystem allowlist is not implemented in the pure-Rust server | [GHSA-qr3f-g5wf-225w](https://github.com/SylphxAI/anymd/security/advisories/GHSA-qr3f-g5wf-225w) |  |
| 2026-09-17 | [Kronova-Intelligent-Systems/asset-intel-orchestration-engine](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine) | High | RLS policy on public.profiles grants the anon role unrestricted read, exposing user email | [GHSA-8h52-3cfj-cmrw](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine/security/advisories/GHSA-8h52-3cfj-cmrw) |  |

Last synced: 2026-10-22
<!-- ADVISORIES:BITIR -->

<!-- MERGED:BASLA -->
## Merged fixes in other projects

Pull requests merged by the maintainers of the projects below.

| Merged | Repository | Change |
|---|---|---|
| 2026-10-21 | [botbahlul/PyAutoSRT](https://github.com/botbahlul/PyAutoSRT) | [Add audioop-lts for Python 3.13 and newer](https://github.com/botbahlul/PyAutoSRT/pull/14) |
| 2026-10-14 | [jpfielding/dicos.rs](https://github.com/jpfielding/dicos.rs) | [Replace retired model id in rust-expert agent frontmatter](https://github.com/jpfielding/dicos.rs/pull/18) |
| 2026-10-13 | [kwaroran/Risuai](https://github.com/kwaroran/Risuai) | [Resolve prefillsupported from the model flag for native Claude entries](https://github.com/kwaroran/Risuai/pull/1612) |
| 2026-10-12 | [opensvc/opensvc](https://github.com/opensvc/opensvc) | [sanswitch/brocade: import telnetlib only in the telnet helper (Python 3.13)](https://github.com/opensvc/opensvc/pull/748) |
| 2026-10-06 | [sokonalysis/sokonalysis](https://github.com/sokonalysis/sokonalysis) | [Remove unused spwd import for Python 3.13](https://github.com/sokonalysis/sokonalysis/pull/15) |
| 2026-10-05 | [shengqh/ngsperl](https://github.com/shengqh/ngsperl) | [Remove unused asyncore imports so the lib scripts run on Python 3.12+](https://github.com/shengqh/ngsperl/pull/3) |
| 2026-10-03 | [GreengageDB/greengage](https://github.com/GreengageDB/greengage) | [Replace removed pipes module with shlex and subprocess for Python 3.13](https://github.com/GreengageDB/greengage/pull/615) |
| 2026-10-02 | [YoushanLiu/ANCC](https://github.com/YoushanLiu/ANCC) | [Drop unused xdrlib import so makePZfiles scripts run on Python 3.13](https://github.com/YoushanLiu/ANCC/pull/7) |
| 2026-09-26 | [edspencer/herdctl](https://github.com/edspencer/herdctl) | [docs: replace retired Claude 4 model ids in configuration examples](https://github.com/edspencer/herdctl/pull/465) |
| 2026-09-26 | [ruvnet/RuVector](https://github.com/ruvnet/RuVector) | [Replace retired Claude model ids in rvagent defaults and demo scripts](https://github.com/ruvnet/RuVector/pull/997) |
| 2026-09-26 | [akkana/scripts](https://github.com/akkana/scripts) | [hypermoon: drop the cgi import removed in Python 3.13](https://github.com/akkana/scripts/pull/32) |
| 2026-09-23 | [apache/hamilton](https://github.com/apache/hamilton) | [Update retired Claude model id in the hamilton-llm skill](https://github.com/apache/hamilton/pull/1690) |
| 2026-09-18 | [langchain-ai/memory-agent-js](https://github.com/langchain-ai/memory-agent-js) | [Update retired default model id claude-3-5-sonnet-20240620](https://github.com/langchain-ai/memory-agent-js/pull/33) |
| 2026-09-18 | [pr-pm/prpm](https://github.com/pr-pm/prpm) | [Replace retired Claude model ids in commands, skills and converter docs](https://github.com/pr-pm/prpm/pull/274) |
| 2026-09-15 | [acodercat/cave-agent](https://github.com/acodercat/cave-agent) | [docs: replace retired claude-3-sonnet-20240229 in the LiteLLM README example](https://github.com/acodercat/cave-agent/pull/28) |
| 2026-09-10 | [kayba-ai/agentic-context-engine](https://github.com/kayba-ai/agentic-context-engine) | [Replace retired Claude model ids and stop sending a rejected temperature](https://github.com/kayba-ai/agentic-context-engine/pull/141) |
| 2026-09-08 | [alinaqi/maggy](https://github.com/alinaqi/maggy) | [Replace a retired Claude model id in four skill examples](https://github.com/alinaqi/maggy/pull/54) |
| 2026-09-04 | [neuron-core/neuron-ai](https://github.com/neuron-core/neuron-ai) | [Replace retired claude-3-5-sonnet-20241022 in two skill files](https://github.com/neuron-core/neuron-ai/pull/646) |
| 2026-09-01 | [anton-abyzov/specweave](https://github.com/anton-abyzov/specweave) | [Replace retired Claude model id in the translation hooks](https://github.com/anton-abyzov/specweave/pull/1924) |
| 2026-08-31 | [kwaroran/Risuai](https://github.com/kwaroran/Risuai) | [Update retired Claude model ids in onboarding and preset defaults](https://github.com/kwaroran/Risuai/pull/1596) |
| 2026-08-24 | [FlorianBruniaux/claude-code-ultimate-guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | [Update retired model id in the issue-triage example workflow](https://github.com/FlorianBruniaux/claude-code-ultimate-guide/pull/67) |
| 2026-08-24 | [dorukozerr/kisuke.vim](https://github.com/dorukozerr/kisuke.vim) | [Replace retired Claude model ids in the Anthropic model list](https://github.com/dorukozerr/kisuke.vim/pull/15) |
| 2026-08-23 | [meyusufdemirci/iMenu-MacOS](https://github.com/meyusufdemirci/iMenu-MacOS) | [Let the menu bar settle at each edge of the synthesized drag](https://github.com/meyusufdemirci/iMenu-MacOS/pull/1) |
| 2026-08-20 | [NeoLabHQ/context-engineering-kit](https://github.com/NeoLabHQ/context-engineering-kit) | [Update the retired model id in the create-command example](https://github.com/NeoLabHQ/context-engineering-kit/pull/105) |
| 2026-08-20 | [TanStack/ai](https://github.com/TanStack/ai) | [Update retired Claude model ids in code-mode-snippets and ai-solid](https://github.com/TanStack/ai/pull/1165) |
| 2026-08-19 | [posit-dev/pointblank](https://github.com/posit-dev/pointblank) | [Update retired Anthropic model ids in the skills docs](https://github.com/posit-dev/pointblank/pull/423) |
| 2026-08-16 | [athola/claude-night-market](https://github.com/athola/claude-night-market) | [docs: update retired model id in the SDK integration example](https://github.com/athola/claude-night-market/pull/659) |
| 2026-08-10 | [enuno/unifi-mcp-server](https://github.com/enuno/unifi-mcp-server) | [chore: update retired Claude model ids in workflows and mcp-builder skill](https://github.com/enuno/unifi-mcp-server/pull/111) |
| 2026-08-09 | [vitali87/code-graph-rag](https://github.com/vitali87/code-graph-rag) | [chore: update retired Claude model ids in server.json and review workflow](https://github.com/vitali87/code-graph-rag/pull/1067) |
| 2026-08-04 | [mengxi-ream/read-frog](https://github.com/mengxi-ream/read-frog) | [docs: update retired Claude model ids in add-provider command](https://github.com/mengxi-ream/read-frog/pull/2035) |
| 2026-08-04 | [davila7/claude-code-templates](https://github.com/davila7/claude-code-templates) | [Update retired model ids in settings components and docs](https://github.com/davila7/claude-code-templates/pull/771) |
| 2026-08-02 | [muratcankoylan/Agent-Skills-for-Context-Engineering](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering) | [Update retired Sonnet 4 model id in pipeline template](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering/pull/115) |
| 2026-08-02 | [rmyndharis/antigravity-skills](https://github.com/rmyndharis/antigravity-skills) | [Replace retired Claude model id in the two AI review skills](https://github.com/rmyndharis/antigravity-skills/pull/10) |
<!-- MERGED:BITIR -->

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
