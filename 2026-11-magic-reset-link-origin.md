---
layout: default
title: "Magic: the emailed sign in link took its domain from the Origin header, so a root session token could go to an attacker's site"
description: "Notes on GHSA-5rxf-pwv4-5fm9 in polterguy/magic. The unauthenticated send-reset-password-link endpoint built the link from the request's Origin header and attached a full session token. Fixed in v24.4.23."
date: 2026-11-16
---

# Magic: the emailed sign in link took its domain from the Origin header, so a root session token could go to an attacker's site

**Advisory:** [GHSA-5rxf-pwv4-5fm9](https://github.com/polterguy/magic/security/advisories/GHSA-5rxf-pwv4-5fm9) (published 2026-11-15)
**Affected:** polterguy/magic `<= v24.4.20`, patched in v24.4.23
**Weakness:** CWE-640, CWE-20, CVSS 3.1 `AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:H` (High)

[Magic](https://github.com/polterguy/magic) is a low-code backend and dashboard. In July 2026 it gained a "Forgot password" flow (commit [`2ed95a1`](https://github.com/polterguy/magic/commit/2ed95a1dbcb8eed06170fc510dd3406ee61d30e0)). Anyone can ask for a link by username, and the server emails the account owner a link that signs them in.

## Root cause

The endpoint is `POST /magic/system/auth/send-reset-password-link`, implemented in `backend/files/system/auth/send-reset-password-link.post.hl`. It is unauthenticated by design. For the requested user it issued a ticket carrying all of the user's roles, valid for 20 minutes, and built the link as

```
<Origin>/authentication/magic-link?backend=<backend>&token=<JWT>
```

`<Origin>` came straight from the request's `Origin` header. The backend URL was only a fallback for requests that sent no `Origin`. A comment in the file explained why. A dashboard running on another port during development should still get a working link.

A browser sets `Origin` honestly, but any other client can send whatever it likes. So the person asking for the link picked the domain in it, and the real server then emailed a genuine message to the real account owner. For `root`, the default administrator, one click handed a root session to that domain for 20 minutes. The token was a plain JWT, so it kept working after first use even though the email said the link works once. The `backend` parameter still pointed at the real server, so the receiving page could forward the user on to the real dashboard.

The attack needs SMTP to be configured, an email address on the target account, and a click within 20 minutes.

## Fix

The maintainer answered the report on 18 Oct, pushed a first fix the same day, and tightened it over the next three weeks.

- [`e755324`](https://github.com/polterguy/magic/commit/e755324b6eadf47bc01b4ddd626661e12dc7fe2d) (18 Oct) honored `Origin` only when it matched one of the origins in `magic:frontend:urls`, the setting CORS already uses.
- [`27f6711`](https://github.com/polterguy/magic/commit/27f6711236783178d9e6775aac1fe4eaec3cb5e4) and [`76eafd0`](https://github.com/polterguy/magic/commit/76eafd0d79a3c81a5c30b8a9faf70b67c5f20f1f) (23 and 31 Oct) stopped taking the `backend` part of the link from the `Host` header and require `magic:backend:url` instead. No link is sent if it is unset.
- [`fba24ca`](https://github.com/polterguy/magic/commit/fba24ca9dfa5f641d675dcd17b00b47b7f37d499) (2 Nov) ignores `Origin` altogether. The dashboard part is the first entry in `magic:frontend:urls`, and no link is sent if that is unset.
- [`560fbb4`](https://github.com/polterguy/magic/commit/560fbb48f3b08d87f542886e394b7dda3f256663) (7 Nov) sends links by username only, never by email address.

v24.4.23 contains all of these.

## What I took from it

- An emailed link inherits the trust of the email. If any part of its URL comes from the request, the requester can aim the server's own mail at their site.
- `Origin` and `Host` are client input on any endpoint a script can call. An allow list works, and a value taken from configuration is simpler still.
- A "reset password" link that carries a full role session is a login token. Limiting it to the password change, and making it single use on the server side, shrinks what one click can lose.

Thanks to the Magic maintainer for the fast response, the fix, and for publishing the advisory.
