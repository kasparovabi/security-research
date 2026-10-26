---
layout: default
title: "mnemo-cortex: a wildcard CORS default opened the unauthenticated loopback API to web pages"
description: "Notes on GHSA-682p-grrx-wrwg in GuyMannDude/mnemo-cortex. The agentb server allowed every origin by default and had no auth by default, so any page in the user's browser could read and change stored memory. Fixed in 4.27.0."
date: 2026-10-27
---

# mnemo-cortex: a wildcard CORS default opened the unauthenticated loopback API to web pages

**Advisory:** [GHSA-682p-grrx-wrwg](https://github.com/GuyMannDude/mnemo-cortex/security/advisories/GHSA-682p-grrx-wrwg) (published 2026-10-19)
**Package:** `mnemo-cortex` (pip), affected `>= 2.0.0, < 4.27.0`, patched in 4.27.0
**Weakness:** CWE-942, CWE-285, CVSS 3.1 `AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:N` (High, 8.1)

[mnemo-cortex](https://github.com/GuyMannDude/mnemo-cortex) is a memory layer for AI agents. Its `agentb` HTTP server stores facts, session transcripts and trajectories, and is meant to run locally next to the agent. The documented setup is a loopback server with no token, protected by a check that refuses peers that are not on loopback.

## Root cause

Two defaults met. `agentb/config.py` set `cors_origins` to `['*']`, both in the dataclass and when reading YAML without the key, and `agentb/server.py` passed that list straight to FastAPI's `CORSMiddleware`. The same config left `auth_token` empty and `scoped_tokens` empty.

The loopback peer check did not help, because a browser on the same machine is itself a loopback client. With `allow_origins=['*']`, a page on any origin could send requests to the local port and read the responses. The advisory lists what that reaches: memory reads (`/facts`, `/sessions`, transcripts, `/dream/latest`), fact writes that poison agent memory, and pausing capture.

While testing the fix I found a second path that the CORS change alone does not close. A DNS rebinding page talks to the server same-origin, and browsers leave out the `Origin` header on same-origin GETs, so an Origin check by itself still let such a page read memory. The advisory's update records this.

## Fix

The fix was proposed in the advisory's temporary private fork and merged for 4.27.0:

- [`f103285`](https://github.com/GuyMannDude/mnemo-cortex/commit/f103285275) makes the `cors_origins` default empty and adds `CORSMiddleware` only when origins are listed. Without auth, the loopback guard now also refuses a request whose `Origin` is neither loopback nor listed. A `*` in the list does not count for that check.
- [`c6d9998`](https://github.com/GuyMannDude/mnemo-cortex/commit/c6d999816a) refuses, without auth, any `Host` that is not a loopback address, `localhost`, the configured bind host or the machine's own name. That is what stops rebinding.
- [`539c327`](https://github.com/GuyMannDude/mnemo-cortex/commit/539c32758b), from the maintainer, extends the Origin guard to the opt-in mode, allows hostname origins and removes a `getfqdn` call at startup.

CLI tools, hooks and Node clients send no `Origin` header and kept working, as did the same-origin Pocket UI.

## What I took from it

- Binding to 127.0.0.1 keeps remote hosts out. It does not keep out the browser that the user is running on the same host, so a local server with no auth still needs to check where a request comes from.
- `allow_origins=['*']` is a reasonable default for a public read-only API and a poor one for a local API that holds private data.
- Origin and Host checks cover different attacks. Origin catches cross-site requests, and Host catches DNS rebinding, where the request is same-origin and may carry no Origin at all.

Thanks to the mnemo-cortex maintainer for merging the fix quickly, extending it to the opt-in mode, and publishing the advisory with a patched release.
