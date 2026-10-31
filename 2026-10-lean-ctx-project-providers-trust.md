---
layout: default
title: "lean-ctx: project-local provider configs skipped Workspace Trust and could send environment secrets to a repository-chosen URL"
description: "Notes on GHSA-rrww-7924-w72f in yvgude/lean-ctx. Provider definitions under a project's .lean-ctx directory were loaded for untrusted workspaces, could replace built-in providers and could name any environment variable as a credential. Fixed in 3.12.0."
date: 2026-10-31
---

# lean-ctx: project-local provider configs skipped Workspace Trust and could send environment secrets to a repository-chosen URL

**Advisory:** [GHSA-rrww-7924-w72f](https://github.com/yvgude/lean-ctx/security/advisories/GHSA-rrww-7924-w72f) (published 2026-10-29)
**Package:** `lean-ctx` (crates.io) and `lean-ctx-bin` (npm), affected `< 3.12.0`, patched in 3.12.0
**Weakness:** CWE-522, CWE-829, CVSS 3.1 `AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:L/A:N` (High, 7.1)

[lean-ctx](https://github.com/yvgude/lean-ctx) is a context gateway for AI coding agents. It runs as an MCP server, a CLI and a local proxy, and decides what an agent reads, which shell commands it may run and which outside data sources it can query. Those data sources are "config providers", REST endpoints described in TOML or JSON files.

## Root cause

lean-ctx already had a Workspace Trust model for the project-local `.lean-ctx.toml`: until the user trusts a workspace, its security-sensitive keys are withheld. Provider definitions are a second project-local configuration channel, and that model did not reach them.

- `config_provider/discovery.rs` added `<project>/.lean-ctx/providers.toml` and `<project>/.lean-ctx/providers/` to the search paths with no trust check.
- Discovery deduplicated by id with project-local entries winning, and the registry overwrote any entry that was not pinned, including the built-in `github` provider.
- The HTTP provider read whichever environment variable the file named as its credential, and `base_url` was not restricted by scheme or host.

Taken together, a repository could define a provider that reused a built-in id, pointed at a host of its choice and named a variable such as `GITHUB_TOKEN`. When the agent later called `ctx_preload` or `ctx_provider`, the server sent that variable from its own environment to the repository's URL and returned the response to the agent as provider content.

After the first report I checked the remaining auth types (`api_key` as header or query parameter, `basic`, `header`). They read environment variables through the same helper and sent them the same way. The automatic preload at the start of a session did not call providers, so an explicit `ctx_preload` or `ctx_provider` call was needed.

## Fix

Pull request [#2150](https://github.com/yvgude/lean-ctx/pull/2150), squashed as [`0c9134e`](https://github.com/yvgude/lean-ctx/commit/0c9134e880d299c9a9c04326015c0941c8fb5023) and released in 3.12.0:

- [`fd7a47b`](https://github.com/yvgude/lean-ctx/pull/2150/commits/fd7a47b5e8e530eee92dee1d0dee8cd518d85fbd), the patch proposed in the advisory's temporary private fork, loads project-local provider files only for a trusted workspace. An untrusted project registers none of its providers, and the built-in `github` provider stays in place.

Untrusted workspaces that relied on project providers now need `lean-ctx trust` once.

## What I took from it

- A trust boundary has to cover every file in the project that can change behaviour, not only the main config file. Here the second channel was a directory of provider definitions.
- Letting project data replace a built-in by id is a quiet way to change what the agent believes it is talking to.
- A config field that names an environment variable is a read primitive over the process environment. It belongs in configuration the user wrote, not in a file that arrives with a repository.

Thanks to the lean-ctx maintainer for applying the fixes from all four reports in one pull request, crediting each commit and publishing the advisories with 3.12.0.
