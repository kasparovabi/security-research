---
layout: default
title: "anymd (pdf-reader-mcp): the documented --allow-dir restriction was not implemented in the Rust server"
description: "Notes on GHSA-qr3f-g5wf-225w in SylphxAI/anymd, formerly pdf-reader-mcp. SECURITY.md told operators to scope file access with --allow-dir or MCP_PDF_ALLOWED_DIRS, and the pure-Rust server accepted both silently without enforcing them. Fixed in @sylphx/citra 5.0.2."
date: 2026-10-27
---

# anymd (pdf-reader-mcp): the documented --allow-dir restriction was not implemented in the Rust server

**Advisory:** [GHSA-qr3f-g5wf-225w](https://github.com/SylphxAI/anymd/security/advisories/GHSA-qr3f-g5wf-225w) (published 2026-09-20)
**Packages:** `@sylphx/citra` (npm), affected `>= 4.0.0, < 5.0.2`, patched in 5.0.2. `@sylphx/pdf-reader-mcp` (npm), affected `>= 4.0.0, <= 4.1.3`, patched in 8.0.0
**Severity:** Medium (the advisory assigns no CVSS vector or CWE)

[anymd](https://github.com/SylphxAI/anymd), published earlier as pdf-reader-mcp, is an MCP server that reads PDFs and other documents for AI agents. The project rewrote its server in Rust. The hardening checklist in `SECURITY.md` and `docs/api/index.md` told operators to limit which files the server can open with `--allow-dir=<path>` or the `MCP_PDF_ALLOWED_DIRS` environment variable.

## Root cause

The Rust server did not know either name. Its `main.rs` looked at the first argument only to see whether it was `doctor`, and ignored everything else. The PDF reader opened whatever path the tool call supplied, with the permissions of the process. So the flag was accepted, nothing was printed, and nothing was restricted.

There was no bug in a check here, because there was no check. The risk came from the gap between the documentation and the code. An operator who followed the checklist believed file access was scoped to one directory, while an agent driven by a prompt injection could still ask the server to read any PDF the user could read.

## Fix

Pull request [#643](https://github.com/SylphxAI/anymd/pull/643) ("enforce Citra security boundaries", commit [`984d2f9`](https://github.com/SylphxAI/anymd/commit/984d2f9596)) enforces `--allow-dir` and `MCP_PDF_ALLOWED_DIRS` natively for `read_pdf`, `search_pdf` and every `pdf_evidence` operation, with traversal and symlink escape checks, in a new `source_access.rs`. The same pull request refuses unauthenticated non-loopback HTTP binds unless an explicit override is set, and caps each request at 32 PDF sources. Integration tests over stdio and HTTP show the allowlist refusing a path on the real launcher.

The maintainers then verified the published 5.0.2 binary rather than the source. With the flag, a path outside the allowed directory gets `Access denied: path ... is outside the configured allowed directories`. Without the flag, the same call parses the file. The advisory and `docs/security/advisory-resolution-5.0.2.md` record that check.

Later, pull request [#942](https://github.com/SylphxAI/anymd/pull/942) applied the same admission check to `pdf_compare` and to media sidecars, which had reached file reads without it.

## What I took from it

- Security settings in documentation are part of the attack surface. A rewrite that drops a flag should fail loudly on it rather than accept it silently.
- An allowlist added per tool needs a test that covers every tool taking a path. New tools are where it gets missed, as the later `pdf_compare` change shows.
- Checking the artifact that users install, as the maintainers did here, is a stronger proof of a fix than reading the source.

Thanks to the maintainers for implementing the restriction with traversal and symlink checks, verifying it against the published package, and publishing the advisory.
