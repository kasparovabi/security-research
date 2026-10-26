---
layout: default
title: "KSail: the local web UI checked Host only on writes, so a rebinding page could read cluster Secrets"
description: "Notes on GHSA-g57x-rm7r-hf26 in devantler-tech/ksail. The origin guard of ksail open web skipped GET requests, and one GET route proxies reads to the kube-apiserver with the user's kubeconfig. Fixed in 7.203.23."
date: 2026-10-27
---

# KSail: the local web UI checked Host only on writes, so a rebinding page could read cluster Secrets

**Advisory:** [GHSA-g57x-rm7r-hf26](https://github.com/devantler-tech/ksail/security/advisories/GHSA-g57x-rm7r-hf26) (published 2026-10-17)
**Package:** `github.com/devantler-tech/ksail/v7` (Go), affected `< 7.203.23`, patched in 7.203.23
**Weakness:** CWE-346, CWE-200, CVSS 3.1 `AV:N/AC:H/PR:N/UI:R/S:U/C:H/I:N/A:N` (Medium, 5.3)

[KSail](https://github.com/devantler-tech/ksail) is a CLI for creating and running local Kubernetes clusters. `ksail open web` starts a web UI on a random loopback port. The API behind it has no session or token in local mode, and the code treats the 127.0.0.1 binding as the security boundary.

## Root cause

The API had an `originGuard` middleware in `pkg/webui/api/server.go`, written to protect the unauthenticated loopback UI from DNS rebinding. It compared `Host` and `Origin` with the listener's origin, but only for mutating methods:

```go
if !isMutating(request.Method) || s.UIOrigin == "" {
    next.ServeHTTP(writer, request)
    return
}
```

Writes were covered. Reads were not, and some reads are sensitive. `GET /api/v1/clusters/{namespace}/{name}/proxy/{path...}` forwards a read to the cluster's kube-apiserver using the TLS client config and bearer token from the user's kubeconfig (`pkg/webui/api/kubeproxy.go`, `pkg/cli/clusterapi/kubeproxy.go`). The `.../watch/{path...}` stream had the same exposure, and `GET /api/v1/clusters` returned the cluster names a caller needs to build the proxy URL.

A page that rebinds its own hostname to 127.0.0.1 is same-origin from the browser's point of view, so it can read the responses. Its requests still carry the attacker's hostname in `Host`, which is why a Host check on every method is enough to stop it.

The desktop app was not affected, because it hands the handler to the Wails webview and binds no TCP listener.

## Fix

Pull request [#7850](https://github.com/devantler-tech/ksail/pull/7850) ("check Host and Origin on every local web UI request", commit [`a975a82`](https://github.com/devantler-tech/ksail/commit/a975a82c65)) makes `originGuard` apply to every request when a UI origin is configured. `Host` must address the listener origin, and `Origin`, when present, must match it. Clients that send no `Origin` still work. The desktop webview and the in-cluster operator, which have no UI origin, are unchanged.

The new tests in `pkg/webui/api/origin_guard_test.go` send a foreign `Host` to the proxy, watch, multiplexer, cluster list, settings and UI root, with and without `Origin`, and check that the request never reaches the cluster proxy. The pull request notes that these tests fail against the previous guard. The fix shipped in 7.203.23.

## What I took from it

- A rebinding defence that only covers writes leaves out any GET that returns secrets. In this API, read access meant read access to everything the user's kubeconfig can see.
- When a local server proxies to another system with the user's credentials, the proxy route is the one to test first.
- A random port slows an attacker down but does not stop one. A page left open in a tab can scan loopback ports through rebinding.

Thanks to the KSail maintainer for a quick fix with tests across all the affected routes, and for publishing the advisory.
