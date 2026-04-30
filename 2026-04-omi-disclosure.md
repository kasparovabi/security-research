---
layout: default
title: "Seventeen Vulnerabilities in Omi, Fourteen Days of Silence"
description: "CVSS 10.0 advisory against BasedHardware/Omi backend (audio-recording AI wearable, claimed 300,000 users). Filed privately on 2026-04-15, ignored for 14 days, related public security PR closed without merging. RCE, auth bypass, hardcoded encryption key, SSRF, path traversal, OAuth CSRF. Coordinated disclosure timeline included."
date: 2026-04-29
image: /assets/og-omi.png
---

# Seventeen Vulnerabilities in Omi, Fourteen Days of Silence

**Author:** Ahmet Kazankaya (`@kasparovabi` on GitHub)
**Date published:** April 29, 2026 (Day 14)
**GHSA:** [GHSA-4j2c-fmg6-8h42](https://github.com/BasedHardware/omi/security/advisories/GHSA-4j2c-fmg6-8h42) (filed April 15, 2026)
**CVSS:** 10.0 (Critical)

---

## TL;DR

On April 15, 2026, I privately reported 14 security vulnerabilities in the BasedHardware/omi backend through GitHub's coordinated disclosure process. The advisory covers remote code execution, authentication bypass, a hardcoded production encryption key in the public repository, server-side request forgery, path traversal, OAuth CSRF, and unauthenticated access to users' personal conversation data. GitHub rated the aggregate severity as CVSS 10.0 (Critical).

Fourteen days later, the state of that advisory is:

- `triage` (no maintainer ever moved it past the initial state)
- Zero comments from the maintainer team on the advisory itself
- No CVE assigned
- None of the 14 findings patched
- All of the ready-to-merge patches in my private fork untouched

During the same fourteen days, the repository received more than 300 commits on unrelated work, the founder publicly acknowledged the advisory on Twitter and then let it slide, and two external contributors opened public pull requests that referenced the same vulnerabilities by the exact file paths I had reported privately. On April 27 (Day 12) the maintainer `beastoin` closed the larger of those PRs ([#6813](https://github.com/BasedHardware/omi/pull/6813), titled "Security hardening: RCE, SSRF, IDOR, auth backdoor, firmware keys, replay, XSS, DoS") without merging and without explanation, leaving the underlying vulnerabilities intact. The smaller PR ([#6804](https://github.com/BasedHardware/omi/pull/6804), path traversal only) is still open and slowly under review.

While the original 14 findings sat in `triage`, three new vulnerabilities were *introduced* by feature commits during the same window. Two of them (BYOK fingerprint hashing, BYOK WebSocket validation gap) live in the new "Bring Your Own Keys" feature. The third, **OMI-15**, is a hardcoded admin bypass token committed to a brand-new admin panel module: `web/admin/lib/dev-auth.ts` exports `DEV_BYPASS_TOKEN = "dev-bypass-token"` and `web/admin/lib/auth.ts` accepts that exact string as a valid admin credential whenever a non-production environment variable is set. The exact anti-pattern that the original advisory flagged in the Python backend (OMI-02 / OMI-04) was therefore replicated in the TypeScript admin panel during the disclosure window.

The product collects audio recordings of private conversations, targets healthcare and sales professionals, markets itself as "SOC 2 + HIPAA compliant" with the tagline **"Open source. Own your data. Runs on your device."** Every clause of that tagline is contradicted by the code.

This writeup documents what is broken, what the vendor has said (and not said), and why the coordinated disclosure window has effectively ended.

---

## What Omi Is

Omi is a wearable AI device priced at $179 that records conversations around the user continuously. It pairs with a mobile app on iPhone and Android, a desktop app on macOS, and a web app. It transcribes speech in real time, generates summaries and action items, maintains a searchable memory of every conversation the user has had, and exposes an "App Store" of third-party integrations that can read and write to the user's memory graph.

The company markets the product for three primary personas:

- Sales professionals who want call summaries and CRM sync
- Healthcare workers who want hands-free note taking during patient interactions
- Technicians who want field-service documentation

The vendor claims 300,000 professional users. The backend is open-source Python / FastAPI, deployed on Google Cloud Run at `api.omi.me`. The codebase is [BasedHardware/omi](https://github.com/BasedHardware/omi).

## What I Found

On April 15, 2026, I conducted a static security audit of the backend directory. I reported 14 findings across four severity bands. The full technical detail is in the GHSA and in the MITRE CVE request I filed on Day 5. Summary below. Every file path and line number in the original 14 was verified again on Day 14 against `main` branch; all are still present.

In addition, I scanned the 300+ commits added during the disclosure window and identified three new findings in code that did not exist on Day 0 (OMI-15, OMI-16, OMI-17 below). Total now stands at 17.

### Critical (4)

| ID | Finding | File | Evidence |
|---|---|---|---|
| OMI-01 | Remote code execution via `eval()` on values read from Redis | `backend/database/redis_db.py` | Seven `eval()` calls at lines 117, 128, 164, 173, 224, 235, 287 on data written back as `str(...)` |
| OMI-02 | Authentication bypass via `ADMIN_KEY` prefix match | `backend/utils/other/endpoints.py` lines 29-31 | `if admin_key and token.startswith(admin_key): return token[len(admin_key):]` |
| OMI-03 | Production-grade `ENCRYPTION_SECRET` committed to the public repository | `backend/.env.template` (line 47 at filing, line 50 at publication, the same value the entire window) | `ENCRYPTION_SECRET='omi_ZwB2ZNqB2HHpMK6wStk7sTpavJiPTFg7gXUHnc4tFABPU6pZ2c2DKgehtfgi4RZv'` |
| OMI-04 | Authentication bypass via `LOCAL_DEVELOPMENT` flag returning hardcoded UID `'123'` | `backend/utils/other/endpoints.py` lines 37-39 | `if os.getenv('LOCAL_DEVELOPMENT') == 'true': return '123'` |

### High (5)

| ID | Finding | File |
|---|---|---|
| OMI-05 | Server-side request forgery in webhook URLs. No validation against private IP ranges or cloud metadata endpoints. User UID leaked to attacker-controlled URLs. | `backend/utils/app_integrations.py` |
| OMI-06 | Path traversal in file upload. Uploaded filename used unsanitized in path construction. | `backend/routers/sync.py` line 503 |
| OMI-07 | OAuth CSRF. State parameter accepted but never validated server side. | `backend/routers/oauth.py` |
| OMI-08 | Unauthenticated access to users' shared conversations, including speaker names and people profiles. | `backend/routers/conversations.py` |
| **OMI-15 (new during disclosure window)** | Hardcoded admin bypass token committed to the public repository. `DEV_BYPASS_TOKEN = "dev-bypass-token"` is accepted as a valid admin credential by `verifyAdmin()` whenever `NODE_ENV != "production"` and `DEV_BYPASS_AUTH=1` (or `NEXT_PUBLIC_DEV_BYPASS_AUTH=1`). The `NEXT_PUBLIC_` variant additionally leaks the bypass status into the client bundle. Replicates the OMI-02 / OMI-04 anti-pattern in the new TypeScript admin panel. | `web/admin/lib/dev-auth.ts`, `web/admin/lib/auth.ts` line 18 |

### Medium (6)

| ID | Finding |
|---|---|
| OMI-09 | Missing CORS middleware. Any browser origin can issue authenticated requests via ambient user sessions. |
| OMI-10 | Rate limiting explicitly disabled on the WebSocket transcription endpoint. |
| OMI-11 | Container runs as root. No `USER` directive in Dockerfile. |
| OMI-12 | Agent proxy communicates with user virtual machines over unencrypted HTTP. |
| **OMI-16 (new during disclosure window)** | BYOK feature stores user API key fingerprints with unsalted SHA-256 (`hashlib.sha256(raw_key.encode()).hexdigest()` at `backend/utils/byok.py` line 189). Known prefixes (`sk-`, `sk-ant-`, `AIza`) make rainbow tables practical if Firestore leaks. HMAC with a per-user salt or a memory-hard KDF would be appropriate. |
| **OMI-17 (new during disclosure window)** | The BYOK middleware uses Starlette's `BaseHTTPMiddleware`, which does not fire for WebSocket connections. The current code documents this and requires every WS handler to manually call `extract_byok_from_websocket()` and `set_byok_keys()`. A future WS endpoint that forgets the manual call silently bypasses BYOK validation; defense in depth requires an ASGI-level middleware instead of per-handler discipline. |

### Low (2)

| ID | Finding |
|---|---|
| OMI-13 | API keys hashed with unsalted SHA-256 instead of a memory-hard KDF. |
| OMI-14 | Internal Python exception strings echoed to clients via `HTTPException(detail=str(e))`. |

I submitted patches for every critical and high severity finding in the same advisory. The patches are in branch `advisory-fix-1` of the private disclosure fork. They touch the same files listed above, pass the advisory's own CI once rebased, and were tagged as mergeable by GitHub's own check on April 15. They have not been merged.

## Why This Is Notable

Security researchers file advisories all the time. Maintainers sometimes take a while to respond. Neither is novel. What is notable here is the combination of four things:

**1. The vulnerability classes are entry-level.**

These are not subtle corner cases. `eval()` on values read from a cache is a pattern that gets flagged by the default configuration of every Python linter. Returning the suffix of a token as the authenticated user's identity is what a developer writes when they first learn about prefix matching and forget that it has no cryptographic meaning. A hardcoded production encryption key in `.env.template` is caught by `gitleaks`, `trufflehog`, and GitHub's own secret scanning. The advisory surfaced mistakes that any routine scan would catch.

OMI-15 makes this point with exceptional clarity: the same anti-pattern that the original advisory called out in the Python backend (a hardcoded credential that takes any caller to admin) was added to the new TypeScript admin panel during the disclosure window. The team flagged that exact class of bug in their own Python code (the BYOK security-fix commit on Day 5 specifically refactored `endpoints.py`), then shipped the same class of bug in TypeScript twelve days later in `web/admin/lib/dev-auth.ts`. The codebase is not just failing to absorb the advisory; it is reproducing the advisory's findings in newly written modules.

**2. The vendor has been actively editing the vulnerable files while the advisory sat.**

`backend/utils/other/endpoints.py` (where the two authentication bypasses live) was modified on April 20 in commit `5f5e315` ("Fix BYOK security flaws: ContextVar, cache, Gemini, WebSocket, validation"). That commit added 60 lines to the same function I flagged. Someone with commit access opened the file, added BYOK validation to the wrapper around `verify_token`, and did not touch the bypass that occupies lines 29-43 of the same file. This is not "we did not see the advisory." This is "we saw the file and chose not to change these specific lines."

**3. Two public PRs exposed the advisory content, then the larger one was closed without merging.**

On April 18 (Day 3 of the private disclosure), two external contributors opened pull requests that collectively touch every file in the advisory:

- [PR #6813](https://github.com/BasedHardware/omi/pull/6813) by `@anthonyonazure`, titled *"Security hardening: RCE, SSRF, IDOR, auth backdoor, firmware keys, replay, XSS, DoS."* The PR body explicitly references "RCE via `eval()` on Redis data," "hardcoded plugin secrets," and "auth backdoor." The new unit tests in that PR are named `test_admin_key_timing_safe.py`, `test_auth_admin_key.py`, and `test_ssrf_guard.py`. These names match OMI-02 and OMI-05 of the advisory almost one-to-one.
- [PR #6804](https://github.com/BasedHardware/omi/pull/6804) by `@JasonOA888`, titled *"fix(security): prevent path traversal in file upload endpoints."* This addresses the same class as OMI-06.

Whatever embargo the coordinated disclosure process was meant to protect ended on April 18, not by my choice; the vulnerable file paths and attack classes were already enumerated in the public PR list.

On April 27 (Day 12) the maintainer `beastoin` *closed* PR #6813 without merging and without comment. The closing event in the GitHub timeline is just `closed by beastoin`, followed seconds later by an auto-generated bot message thanking the contributor. Nine of the fourteen advisory findings were addressed by that PR; the closure leaves them all intact in `main`. PR #6804 (the smaller one, path traversal only) remains open and is being slowly reviewed for tests; that flow demonstrates the team is capable of engaging with security PRs when they choose to. Worth noting that PR #6804 modifies `backend/routers/chat.py` while OMI-06 in this advisory points at the same vulnerability class in a sibling file `backend/routers/sync.py` lines 503 and 1280; even if #6804 lands as currently scoped, the unsanitized `filename = file.filename` pattern in `sync.py` will remain.

**4. The founder acknowledged the advisory publicly, then took no action.**

On April 19, I posted a short Twitter thread asking for a triage update. The founder replied:

> "will check but team is small, we need maintainers"
>
> "thank you!"
>
> "Can you send me your github here pls"

I sent my GitHub profile. Ten days later, the advisory is still in `triage`. The team has shipped more than 300 commits since that exchange, including a new Bring-Your-Own-Keys feature whose own security flaws were fixed within 15 hours of introduction. The team's ability to ship security fixes quickly is demonstrated daily. The decision not to ship the advisory's fixes is a prioritization choice, not a capacity problem.

## The Marketing and the Code

The Omi homepage carries the tagline **"Open source. Own your data. Runs on your device."** The store page for the $179 device advertises **"Enterprise security: SOC 2 + HIPAA"** and **"Encryption: TLS in transit + AES-256 at rest."**

Check each claim against the code.

### "Runs on your device"

The "Quick Start" that the vendor actively promotes on their homepage and on Twitter is:

```bash
git clone https://github.com/BasedHardware/omi.git && cd omi/desktop && ./run.sh --yolo
```

That command sets `OMI_SKIP_BACKEND=1`, `OMI_SKIP_AUTH=1`, and `OMI_SKIP_TUNNEL=1`. All backend traffic is routed to `api.omi.me`. The README itself, one line below the command, says: **"Builds the macOS app, connects to the cloud backend, and launches. No env files, no credentials, no local backend."**

The tagline is factually inverted. Only the UI runs locally. The recording of the user's conversations, the transcription, the AI processing, the storage, and the memory graph all run in the vendor's cloud.

### "Own your data"

Two ways this claim fails.

First, the audio goes through at least five third-party systems: Deepgram for speech-to-text, OpenAI for language processing, Google Cloud Storage for the audio files, Pinecone for vector embeddings, and Upstash Redis for short-lived caches. The privacy policy lists all of these and they are necessary for the product to function. The user does not own their data because their data is distributed across a supply chain of processors.

Second, the master AES-256 encryption key that protects "conversations, memories, and chat messages at rest" is committed in plaintext to the public repository. Every deployment that retained the default value, which passes the 32-byte validation check in `backend/utils/encryption.py`, has effectively no encryption against anyone with access to the public GitHub repository. That includes the vendor's own default deployment if `backend/.env.template` was ever the starting point for the operational `.env`.

### "SOC 2 + HIPAA compliant"

SOC 2 Type II requires evidence of regular vulnerability scanning, documented change management, and enforced access controls. A `bandit` run against this repository flags the `eval()` calls on the first pass. A `gitleaks` run flags the committed encryption key. A line-by-line review of `verify_token()` flags the prefix-match bypass. None of these are advanced techniques. If the SOC 2 audit was performed as the vendor claims and it did not surface these findings, that is its own story.

HIPAA goes further. The Security Rule's technical safeguards require access controls, audit controls, integrity verification, and transmission security for Protected Health Information. A hardcoded master key in a public repository does not satisfy the Access Control standard. An authentication bypass that accepts the suffix of an arbitrary token as the authenticated user does not satisfy the Person or Entity Authentication standard. Transmitting PHI to OpenAI and Deepgram without the user being able to opt out per-request does not satisfy the Transmission Security standard without a very carefully written Business Associate Agreement.

I do not know whether BasedHardware has BAAs with every processor. I do know that if a HIPAA-covered entity used Omi for patient conversations and those conversations are now in a system with unpatched CVSS 10 vulnerabilities, that entity has an independent breach notification obligation. The 60-day HIPAA notification clock starts when the entity discovers the breach. Reading this post counts as discovery.

### "Enterprise security"

OMI-11 ships a production container that runs as `root` because there is no `USER` directive in the Dockerfile. This is Docker 101. An enterprise-security review would have caught it on the first container scan. The presence of OMI-11 and OMI-13 (unsalted SHA-256 for API key hashing, deprecated fifteen years ago) in the same codebase as the "Enterprise security" claim is its own signal.

## The 1-Command Self-Hosting Pivot

On April 20, the founder posted a public statement of intent:

> "I chose the direction of a local, transparent and developer-friendly AI."
>
> "I want everyone to be able to change omi for themselves in just 1 prompt."
>
> "Local backend install in 1 command coming soon."
>
> "Viva La Open Source."

The current state of the backend makes this direction dangerous. If the "1 command" installer ships before the advisory patches, every new self-hosted deployment will be compromised by default:

- The installer will copy `backend/.env.template` to `.env` as the standard Python workflow dictates. The `ENCRYPTION_SECRET` will be the leaked default. The new user's conversations will be decryptable by anyone who reads the public repository.
- If the installer runs with `LOCAL_DEVELOPMENT=true` (the name strongly suggests this is the default for local installs), the backend will authenticate any request as UID `'123'`. The new user's backend is unauthenticated by default.
- If the installer stands up Redis with any network exposure, the `eval()` RCE becomes accessible.

A vendor cannot encourage a migration to self-hosted deployments on Monday and leave a CVSS 10 advisory in `triage` on Tuesday. These two actions are in direct tension. The resolution is to merge the advisory's patches before the installer ships, or to delay the installer until the patches are merged.

## Timeline

| Date | Event |
|---|---|
| 2026-04-15 | Private GHSA-4j2c-fmg6-8h42 filed with 14 findings and ready-to-merge patches in branch `advisory-fix-1`. CVSS 10.0 assigned by GitHub. |
| 2026-04-15 to 2026-04-29 | Zero maintainer comments on the advisory. State remains `triage`. |
| 2026-04-18 | External contributors open PR #6804 (path traversal) and PR #6813 (RCE, SSRF, auth backdoor, hardcoded secrets, path traversal, XSS, DoS). |
| 2026-04-19 | Founder `@kodjima33` acknowledges the advisory on Twitter, requests my GitHub handle, takes no further action. |
| 2026-04-20 | Commit `5f5e315` touches the same `endpoints.py` file as OMI-02 and OMI-04, adds 60 lines of unrelated code, leaves the bypasses in place. |
| 2026-04-20 | Founder publicly announces direction toward self-hosted "1-command" installs. |
| 2026-04-20 | I submit CVE request to MITRE under the CNA-of-Last-Resort process because GitHub (the CNA covering GHSAs) has not assigned a CVE. |
| 2026-04-21 | Second Twitter thread from me: advisory status summary, no response. |
| 2026-04-22 | Omi receives 100 additional commits in 24 hours. Zero security commits. |
| 2026-04-27 | Maintainer `beastoin` closes PR #6813 without merging and without explanation. PR #6804 begins active review with maintainer requests for tests. |
| 2026-04-29 | Final repo scan during the disclosure window identifies three new findings introduced by feature commits in this same window: OMI-15 (hardcoded admin bypass token in `web/admin/lib/dev-auth.ts`), OMI-16 (BYOK unsalted SHA-256 fingerprinting), OMI-17 (BYOK WebSocket validation gap). |
| 2026-04-29 | This post. GHSA still `triage`, no CVE, no patches in `main`, no MITRE response yet. Total findings now 17. |

## Why Fourteen Days

The industry default for coordinated disclosure is between 30 and 90 days. I am publishing on Day 14 for four reasons.

1. The vulnerable file paths are already public via PR #6813 and PR #6804. Any attacker who reads the open pull request list of the repository has a roadmap to the vulnerable code. The embargo protects nothing that is not already out.
2. The vendor is actively promoting a direction that distributes the vulnerable code to more users (the self-hosted installer push). The longer the window, the more new users are compromised by default when that installer ships.
3. The vendor has had sufficient opportunity to act. Fourteen days with zero engagement, while 300 commits land on unrelated work, is sufficient. The Google Project Zero 90-day window is designed around vendors who are actually working on the fix. The window does not apply when the vendor is actively choosing not to.
4. The healthcare use case surfaces a regulatory clock for some users. If any HIPAA-covered entity used Omi with PHI, their own 60-day clock is already running. They need the information to make a notification decision.

I informed the vendor on Day 5 that Day 14 would be the public disclosure. I reiterated the timeline on Day 6 and Day 7. The vendor has had the date on the record for nine days.

## For Omi Users

If you use Omi today:

1. Assume that any conversation you have recorded through the product is stored in a system that currently contains a CVSS 10 RCE, an authentication bypass, and a hardcoded encryption key. Act accordingly.
2. If your conversations include any Protected Health Information, consult your compliance officer about the notification obligations under HIPAA §164.400.
3. If your conversations include any information covered by a Non-Disclosure Agreement with a third party, consult legal counsel about breach notification obligations under that NDA.
4. Delete conversations you do not want preserved, via the app's delete feature or by emailing `help@omi.me`. Note that this relies on the vendor's honoring the delete request through the processors (Google, Pinecone, Deepgram, OpenAI).
5. If you self-host the backend, do not rely on `backend/.env.template` as a starting point for `ENCRYPTION_SECRET`. Rotate the value to a 32-byte random secret. Set `LOCAL_DEVELOPMENT=false`. Do not set `ADMIN_KEY` unless you have a plan for rotating it when the advisory's bypass pattern is fixed upstream.
6. If you self-host the admin panel (`web/admin`), set `NODE_ENV=production` explicitly and do not set `DEV_BYPASS_AUTH=1` or `NEXT_PUBLIC_DEV_BYPASS_AUTH=1` in any internet-reachable environment. The hardcoded `dev-bypass-token` in `web/admin/lib/dev-auth.ts` becomes a complete admin backdoor as soon as those flags are enabled.

If you are a healthcare provider using Omi for patient interactions: stop using it for PHI until the vendor issues a public statement about the advisory and the notification requirements.

## For the Vendor

The advisory patches are still in `advisory-fix-1`. They are still mergeable. I am still available.

The items that would resolve the public-disclosure posture on the vendor side:

1. Merge `advisory-fix-1`, or its equivalent from PR #6813, and ship a release that contains the fixes.
2. Assign the CVE through the GHSA interface (the button is there; no approval from me is required).
3. Publish a security statement on `omi.me` acknowledging the issues, listing the mitigations for existing users, and describing the notification steps for HIPAA-covered users who had PHI in the system during the affected window.
4. Review the BYOK feature's interaction with the `ADMIN_KEY` bypass. If the `ADMIN_KEY` bypass is used internally by the team (I do not know; it appears to have internal utility), the BYOK headers and user keys are currently exposed to anyone with the key.

If instead the current posture continues, the next steps are:

- This post propagates.
- Reports at `r/netsec` and Hacker News.
- A direct brief to Brian Krebs with full documentation, including this post.
- A follow-up writeup on Day 30 with the full technical reproduction.
- A continued clock on any HIPAA breach notification obligations for covered entities.

## References

- GHSA-4j2c-fmg6-8h42 (currently private): https://github.com/BasedHardware/omi/security/advisories/GHSA-4j2c-fmg6-8h42
- PR #6813 (public, unmerged): https://github.com/BasedHardware/omi/pull/6813
- PR #6804 (public, unmerged): https://github.com/BasedHardware/omi/pull/6804
- Vulnerable file: `backend/database/redis_db.py`
- Vulnerable file: `backend/utils/other/endpoints.py`
- Leaked key: `backend/.env.template` line 47
- Twitter thread from Day 6: [link when posted]
- MITRE CVE request: pending assignment

## Disclosure

I have no commercial relationship with BasedHardware. I have never purchased an Omi device. This research was self-funded. Patches were offered free of charge. This post contains no proof-of-concept exploit code, consistent with the coordinated disclosure principle of proportional risk: the code paths are already public via PR #6813; the attack classes are well documented in the security literature; the remaining technical detail (a runnable exploit) remains private until Day 30.

---

*This is a coordinated-disclosure writeup, published on Day 14 after a private report to the vendor. It is not a call to exploit the issues. Readers running Omi infrastructure should treat this as a notification to assess their own exposure and, if applicable, their breach notification obligations.*
