---
layout: default
title: "Omi disclosure: provenance and timeline"
description: "Verifiable record of who reported the Omi backend vulnerabilities, when, and what happened next. Every claim on this page can be checked against a public artifact."
---

# Omi disclosure: provenance and timeline

This page exists so nobody has to take my word for anything. Every line below
points at an artifact you can open yourself.

## The short version

I privately reported fourteen vulnerabilities in the `BasedHardware/omi` backend
on April 15, 2026. The advisory sat in `triage`. On July 27, 103 days later, a
first-time external contributor landed a commit whose own title says it closes
the April disclosure. As of August 10, 2026 the advisory is still unpublished
and still has no CVE.

## Verifiable record

| What | Artifact | How to check it |
|---|---|---|
| Private report filed | [GHSA-4j2c-fmg6-8h42](https://github.com/BasedHardware/omi/security/advisories/GHSA-4j2c-fmg6-8h42) | Advisory page shows `kasparovabi` credited as reporter, credit state `accepted` |
| Report date | April 15, 2026 | Advisory creation date, visible to anyone with access to the advisory |
| Public writeup | [2026-04-omi-disclosure.md](2026-04-omi-disclosure.md) | First commit in this repository, `1585c37`, timestamped `2026-04-30T00:03:41+03:00` |
| The fix | Commit [`87b20078`](https://github.com/BasedHardware/omi/commit/87b2007891ab249425e96a31a81bf0e8398a0a8b) | Title: *"Harden auth, SSRF, CSRF, and BYOK boundaries from the April disclosure"*. Author: `amowpheth`. Date: 2026-07-27 |
| Gap between report and fix | 103 days | April 15 to July 27 |
| Commits in that window | 14,139 total, 4,405 touching `backend/` | `git log --since=2026-04-15 --until=2026-07-27 --oneline \| wc -l` |
| Fix author's history | One commit, ever, in this repository | `git log --author="amowpheth" --oneline \| wc -l` |
| Advisory state today | `triage`, unpublished, no CVE | GitHub advisory API |

## The single strongest piece of evidence

The commit that fixed these issues names the disclosure in its own title:

```
commit 87b2007891ab249425e96a31a81bf0e8398a0a8b
Author: amowpheth <amowpheth@gmail.com>
Date:   Sun Jul 26 15:19:35 2026 -0400

    Harden auth, SSRF, CSRF, and BYOK boundaries from the April disclosure
```

That line was written by someone else, in the vendor's own repository, in a
commit anyone can fetch. It is not my characterisation of events. It is the
project's own history recording where the work came from.

## Reproduce it yourself

```bash
git clone --filter=blob:none https://github.com/BasedHardware/omi.git
cd omi

# the fix commit, and what it says about itself
git log -1 --format='%H%n%an%n%cI%n%s' 87b20078

# how many commits went by while the advisory sat untouched
git log --since=2026-04-15 --until=2026-07-27 --oneline | wc -l
git log --since=2026-04-15 --until=2026-07-27 --oneline -- backend/ | wc -l

# how many commits the author of the fix has made to this repository
git log --author="amowpheth" --oneline | wc -l
```

## What I am not claiming

I am not claiming the fix is mine. It is not. `amowpheth` wrote it, and the SSRF
hardening in it goes beyond what I proposed in April, including a rejection of
RFC 6598 CGNAT space that I had missed.

I am not claiming the vendor is obliged to publish the advisory or assign a CVE.
That is their call.

What the record does show is the order of events: report first, silence for 103
days, then a fix that names the report. Anyone reading the repository today sees
patched code and no advisory, which is why this page exists.

## Still open

One finding from the disclosure window has not been fixed. See
[OMI-15 in the update section](2026-04-omi-disclosure.md#still-open) of the main
writeup, along with a correction to how I originally characterised its severity.
