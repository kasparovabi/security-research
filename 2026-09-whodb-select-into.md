---
layout: default
title: "WhoDB: SELECT ... INTO passed the read-only and write-approval gates"
description: "Notes on GHSA-pvf2-6h4x-r24j in clidey/whodb. A keyword denylist decided whether SQL could write, and INTO, OUTFILE and DUMPFILE were not on it. Fixed in 0.134.0."
date: 2026-10-26
---

# WhoDB: SELECT ... INTO passed the read-only and write-approval gates

**Advisory:** [GHSA-pvf2-6h4x-r24j](https://github.com/clidey/whodb/security/advisories/GHSA-pvf2-6h4x-r24j) (published 2026-09-30)
**Package:** `github.com/clidey/whodb` (Go), affected `>= 0.88.0`, patched in 0.134.0
**Weakness:** CWE-863, CVSS 3.1 `AV:L/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N` (Medium)

[WhoDB](https://github.com/clidey/whodb) is a database client with a CLI and an MCP server. Two of its features decide whether a SQL statement may write. The CLI has a read-only mode, and the MCP server has `--confirm-writes`, which asks the user before running anything that changes state. Both relied on one answer from `core/src/sqlguard/classify.go`.

## Root cause

The classifier looked at the first keyword. A statement starting with `SELECT` counted as a read unless one of the words in `mutatingKeywords` appeared somewhere in it (`INSERT`, `UPDATE`, `DELETE`, `CREATE` and so on). `INTO`, `OUTFILE` and `DUMPFILE` were not in that list, so these all came back as `Mutating=false`:

```sql
SELECT * INTO evil_copy FROM users                -- PostgreSQL creates and fills a table
SELECT * FROM users INTO OUTFILE '/tmp/dump.txt'  -- MySQL writes a file on the server
```

The same verdict fed three paths named in the advisory: the CLI read-only check in `cli/internal/database/manager.go`, the confirmation prompt in `cli/pkg/mcp/tools.go`, and MCP read-only validation in `cli/pkg/mcp/validation.go`.

The maintainers' own testing, written up in the advisory, adds a detail I had not covered. On PostgreSQL the database-enforced read-only transaction added in 0.130.0 still stopped the table creation with SQLSTATE 25006. On MySQL 8.4 it did not, because `INTO OUTFILE` is a server-side file operation that the transaction mode WhoDB uses does not block. My report only contained a deterministic classifier reproduction, and I said so. The end-to-end writes were confirmed by the maintainers.

## Fix

Commit [`9de47fb`](https://github.com/clidey/whodb/commit/9de47fb365) ("update for protected reads") changes the approach instead of adding three more words to the denylist. The package comment now says that only a bounded read grammar is treated as automatic, and anything outside it, including syntax whose effects cannot be established, is reported as needing approval. The same commit adds read-guard integration tests that go through the MCP execution path, and touches the ClickHouse plugin and the shared connection pool. The fix shipped in 0.134.0.

## What I took from it

- A denylist of write verbs is the wrong shape for "is this SQL a read". SQL has several ways to write from inside a `SELECT`, and they differ per dialect. An allowlist of the read grammar that fails closed is easier to keep correct.
- A read-only transaction is a useful second boundary, but it is not the same boundary on every backend. MySQL `INTO OUTFILE` is the example here.
- For anything that lets a model run SQL, the database account is the boundary that does not depend on parsing. The advisory's interim advice is to connect with an account that has no `CREATE` in the target schema and no MySQL `FILE` privilege, and to set `secure_file_priv`.

Thanks to the WhoDB maintainers for reproducing the issue on both databases, widening the affected range to the real one, and documenting the result in the advisory.
