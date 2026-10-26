---
layout: default
title: "altium-designer-mcp: count fields from a library file sized allocations, and a failed allocation aborts the server"
description: "Notes on GHSA-p2qw-h8m7-5x69 in embedded-society/altium-designer-mcp. File-derived counts in the SchLib and PcbLib readers reached Vec::with_capacity and a growth loop with no upper bound. An allocation failure is an abort, which catch_unwind cannot stop. Fixed in 1.2.1."
date: 2026-10-27
---

# altium-designer-mcp: count fields from a library file sized allocations, and a failed allocation aborts the server

**Advisory:** [GHSA-p2qw-h8m7-5x69](https://github.com/embedded-society/altium-designer-mcp/security/advisories/GHSA-p2qw-h8m7-5x69) (published 2026-10-09)
**Package:** `altium-designer-mcp`, affected `<= 1.2.0`, patched in 1.2.1
**Weakness:** CWE-789, CWE-400, CWE-834, CVSS 3.1 `AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:N/A:H` (Medium, 5.5)

[altium-designer-mcp](https://github.com/embedded-society/altium-designer-mcp) is an MCP server, written in Rust, that reads and writes Altium Designer schematic (`.SchLib`) and footprint (`.PcbLib`) libraries. The crate forbids `unsafe`, so the issue is availability only.

## Root cause

Three readers took a count from the file and used it before checking it against anything:

- `src/altium/schlib/reader/parsers.rs`, `parse_polyline` and `parse_polygon`. `LocationCount` was checked only for a lower bound, then passed to `Vec::with_capacity`.
- `src/altium/schlib/read_io.rs`. The `CompCount` header property went to `Vec::with_capacity` on every `SchLib::open`.
- `src/altium/pcblib/read_io.rs`. The numeric suffix of `LibRef{idx}` and `CompDescr{idx}` keys drove a `while len <= idx { push(...) }` loop.

The project has a safety net for failing tools. The dispatcher wraps each tool call in `std::panic::catch_unwind` (`guard_panics` in `src/mcp/server.rs`), and the release profile keeps unwinding on purpose so that net works. An allocation failure is not a panic, though. A count above `isize::MAX` bytes gives a "capacity overflow" panic, which the net catches. A count below that but larger than available memory makes the allocator call `handle_alloc_error`, which aborts the process. Over stdio that ends the session. Over the `--http` transport it ends every session on the process.

## Fix

Pull request [#566](https://github.com/embedded-society/altium-designer-mcp/pull/566), released as 1.2.1:

- [`b77cedb`](https://github.com/embedded-society/altium-designer-mcp/commit/b77cedb84e), the patch from my report, refuses a `CompCount` larger than the header's fields and a `LibRef`/`CompDescr` index that reaches the header's field count, and capped `LocationCount` at a fixed limit.
- [`a4e7ccf`](https://github.com/embedded-society/altium-designer-mcp/commit/a4e7ccfc79), from the maintainer, replaces that fixed cap with a bound tied to the record. The pull request explains why the cap was not enough: a 36-byte record under the cap could still claim 16 MiB, so a file of a thousand such records still aborted. A vertex needs at least one key unless it sits at the origin, so a record can describe at most one vertex per key plus a few. With that bound, the memory a file can claim grows with its size.
- [`a811cfb`](https://github.com/embedded-society/altium-designer-mcp/commit/a811cfbfa4), also from the maintainer, stops the `SectionKeys` walk at the stream's fields. A huge `KeyCount` there did not allocate but kept the read spinning, which was not in my report.

The pull request also lists the other count-driven loops in the readers and why each is already bounded by the bytes it reads. A follow-up commit documents the count bounds in the threat model.

## What I took from it

- In Rust, `catch_unwind` is not a general crash guard. Allocation failure aborts, so any size that comes from input has to be checked before it reaches `with_capacity`.
- A fixed cap per record is weaker than a bound tied to the input. If many records can each claim the cap, the total is still unbounded. Bounding by what the remaining bytes could describe keeps memory proportional to file size.
- A count that drives a loop is the same bug without the allocation. It shows up as a hang instead of an abort.

Thanks to the maintainer for keeping my patch as the first commit, tightening it with a better bound, finding the related `KeyCount` hang, and shipping 1.2.1 with the advisory.
