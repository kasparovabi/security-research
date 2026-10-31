---
layout: default
title: "lean-ctx: the proxy accepted any provider key header on routes where it holds the upstream credential"
description: "Notes on GHSA-53h5-q4m8-vg3x in yvgude/lean-ctx. On a loopback bind the proxy admitted provider routes on any non-empty key header, then replaced that header with the operator's configured key. Fixed in 3.12.0."
date: 2026-10-31
---

# lean-ctx: the proxy accepted any provider key header on routes where it holds the upstream credential

**Advisory:** [GHSA-53h5-q4m8-vg3x](https://github.com/yvgude/lean-ctx/security/advisories/GHSA-53h5-q4m8-vg3x) (published 2026-10-29)
**Package:** `lean-ctx` (crates.io) and `lean-ctx-bin` (npm), affected `< 3.12.0`, patched in 3.12.0
**Weakness:** CWE-863, CVSS 3.1 `AV:L/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:N` (Medium, 4.4)

[lean-ctx](https://github.com/yvgude/lean-ctx) includes a local proxy (`lean-ctx proxy start`) that sits between agents and LLM providers. A `[[proxy.providers]]` entry with `api_key_env`, or a Bedrock entry, makes the proxy hold the upstream credential. The documentation says callers then authenticate with the lean-ctx Bearer token and never see the provider key.

## Root cause

Two pieces of logic that are each reasonable did not agree.

- On a loopback bind, `proxy_require_token` defaults to false. For that case the auth middleware has a fallback: a request to a provider route that carries a provider key header (`x-api-key`, `api-key`, `x-goog-api-key` or `Authorization`) is let through, on the assumption that the caller brings its own key for the upstream.
- On `/providers/{id}` routes with a gateway-held credential, `inject_gateway_credential` then strips the caller's header and puts in the configured key.

The fallback did not ask whether the route injects a key. So any local process that could reach 127.0.0.1 could send a junk key header, pass the middleware, and have its request forwarded with the operator's credential. The key value itself was not returned to the caller.

After the first report I found two more things. Bedrock entries behave the same way, with the request signed using the operator's AWS credentials. Routing aliases (`[proxy.routing.aliases]`) reach the same credential through `/v1/chat/completions`, because the key is injected later in the forward path, after the middleware has admitted the request on a built-in route. A change to the middleware alone would not have covered that second path. With `proxy_require_token = true`, all of these routes returned 401.

## Fix

Pull request [#2150](https://github.com/yvgude/lean-ctx/pull/2150), squashed as [`0c9134e`](https://github.com/yvgude/lean-ctx/commit/0c9134e880d299c9a9c04326015c0941c8fb5023) and released in 3.12.0:

- [`2fe4fb1`](https://github.com/yvgude/lean-ctx/pull/2150/commits/2fe4fb1cec9190b0c50089a5cd09adff49b818a8), the patch proposed in the advisory's temporary private fork, requires the lean-ctx token before the proxy injects a gateway-held key, both on `/providers/{id}` routes and on routing aliases. Requests that carry the lean-ctx token still reach the upstream with the injected key.

## What I took from it

- A "bring your own key" fallback and a "the gateway holds the key" mode cannot share a route without the auth check knowing which one applies.
- When credentials are injected in more than one place, each injection point needs its own check. Here the routing path injected the key after the middleware had already decided.
- Loopback is not a single-user boundary. Other local users, sandboxed processes and containers sharing the network namespace can all reach 127.0.0.1.

Thanks to the lean-ctx maintainer for taking both paths into the fix and publishing the advisory with 3.12.0.
