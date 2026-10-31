---
layout: default
title: "lean-ctx: an untrusted project .lean-ctx.toml could still extend write_allow_paths"
description: "Notes on GHSA-79fg-7cj5-4xmm in yvgude/lean-ctx. Workspace Trust withheld several path keys from untrusted project config but not write_allow_paths, which ctx_shell uses for redirect and download targets. Fixed in 3.12.0."
date: 2026-10-31
---

# lean-ctx: an untrusted project .lean-ctx.toml could still extend write_allow_paths

**Advisory:** [GHSA-79fg-7cj5-4xmm](https://github.com/yvgude/lean-ctx/security/advisories/GHSA-79fg-7cj5-4xmm) (published 2026-10-29)
**Package:** `lean-ctx` (crates.io) and `lean-ctx-bin` (npm), affected `< 3.12.0`, patched in 3.12.0
**Weakness:** CWE-284, CVSS 3.1 `AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:N` (Low, 3.3)

[lean-ctx](https://github.com/yvgude/lean-ctx) reads a project-local `.lean-ctx.toml`. After an external audit it added Workspace Trust: until the user trusts a workspace, security-sensitive keys from that file are withheld. `ctx_shell`, the agent's shell tool, checks the targets of redirects and downloads against `write_allow_paths`.

## Root cause

- `strip_sensitive_overrides` in `core/config/loader.rs` withheld `allow_paths`, `extra_roots` and `allow_symlink_roots`, but had no entry for `write_allow_paths`.
- `core/config/merge.rs` extended `write_allow_paths` from the project file unconditionally.

A cloned repository could therefore list any directory in its `.lean-ctx.toml` and remove the `ctx_shell` redirect and download check for it, without the workspace ever being trusted. The severity is low because the default shell allowlist can already run script files, which is a wider capability than a redirect.

In the follow-up I set the other merged keys (`excluded_commands`, `archive.raw_commands`, `read_redirect`, `profile`) from an untrusted project and checked whether secret redaction in `ctx_shell` and `ctx_read` changed. It did not. Of those keys, only `write_allow_paths` changed an outcome.

## Fix

Pull request [#2150](https://github.com/yvgude/lean-ctx/pull/2150), squashed as [`0c9134e`](https://github.com/yvgude/lean-ctx/commit/0c9134e880d299c9a9c04326015c0941c8fb5023) and released in 3.12.0:

- [`0c4a767`](https://github.com/yvgude/lean-ctx/pull/2150/commits/0c4a7672f90c3e30c670def895dba910b7f44582), the patch proposed in the advisory's temporary private fork, withholds `write_allow_paths` for untrusted workspaces and lists it among the withheld keys shown by `lean-ctx trust`.

## What I took from it

- A deny list of sensitive keys has to be kept in step with every new key that widens a boundary. A read-path key was on the list and its write-path sibling was not.
- Testing each merged key for an observable change in outcome is a quick way to sort the ones that matter from the ones that only look sensitive.

Thanks to the lean-ctx maintainer for the fix and for publishing the advisory with 3.12.0.
