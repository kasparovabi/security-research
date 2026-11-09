---
layout: default
title: "Rufus: two heap writes from line handling, in md5sum.txt from an ISO and in the downloaded sbat_level.txt"
description: "Notes on GHSA-877g-r8h5-fq8p and GHSA-hcpc-vq8c-923h in pbatard/rufus. UpdateMD5Sum() wrote 32 hex characters at the start of a line without checking the line was that long, and GetSbatEntries() counted LF before converting CR, so a CR-only file overflowed its array. Both fixed for 4.16."
date: 2026-11-09
---

# Rufus: two heap writes from line handling, in md5sum.txt from an ISO and in the downloaded sbat_level.txt

**Advisories:** [GHSA-877g-r8h5-fq8p](https://github.com/pbatard/rufus/security/advisories/GHSA-877g-r8h5-fq8p) and [GHSA-hcpc-vq8c-923h](https://github.com/pbatard/rufus/security/advisories/GHSA-hcpc-vq8c-923h) (both published 2026-11-05)
**Product:** `rufus`, affected `>= 4.5, <= 4.16_BETA` and `>= 4.11, <= 4.16_BETA`, patched in 4.16
**Weakness:** CWE-787 (and CWE-131 for the second), CVSS 3.1 `AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:H` and `AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:L/A:H` (both Medium)

[Rufus](https://github.com/pbatard/rufus) writes ISO images to USB drives on Windows. It runs elevated, and people feed it ISOs downloaded from all over the internet. Both bugs are in small text parsers that trusted the shape of a line.

## Root cause

**md5sum.txt.** Many Linux ISOs ship an `md5sum.txt`. When Rufus rewrites a config file during extraction (for example the boot label, which it changes whenever the ISO label does not fit FAT), it also updates that file's MD5 in `md5sum.txt` so the media check still passes. `UpdateMD5Sum()` in `src/hash.c` found the file name with `strstr()`, walked back to the start of the line and wrote 32 hex characters there. The only check was an `assert()` on the first character. If the matching line was the last one in the file and shorter than 32 bytes, the write ran up to 31 bytes past the end of the heap buffer. The ISO supplies both the config file and `md5sum.txt`, so the person who built the ISO controls the trigger. The bytes written are the hex digits of an MD5, which limits what they can be.

The same report covered two smaller issues on the ISO path. `uprintf()` in `src/stdio.c` assumed `safe_vsnprintf()` returns a negative value on truncation, which holds for `_vsnprintf_s` but not for the `vsnprintf` that MinGW builds used, so a long enough log line (for example a Rock Ridge symlink target of about 4 KB) moved the write pointer past the 4096-byte buffer. `SetupWinPE()` in `src/wue.c` looped to `size - 32` with an unsigned `size`, so a `setupldr.bin` under 32 bytes wrapped the bound.

**sbat_level.txt.** At startup Rufus downloads `sbat_level.txt` from rufus.ie to know which bootloaders are revoked. `GetSbatEntries()` in `src/parser.c` sized its array by counting `\n`, and converted `\r` to `\n` in the same loop, after the count. A file with CR-only line endings was counted as one line, got a two-entry array, and then had one entry per line written past it. Reaching this needs control of the response, which means control of the site or a TLS-intercepting proxy the machine already trusts. The maintainer's commit message points out that an attacker in that position could also change the download links, so the realistic risk is low. A broken line ending in the real file would still have crashed every client that checks for updates.

## Fix

Both fixes went in on 2026-11-03 as commits from the private fork pull requests:

- [`de4cf676c`](https://github.com/pbatard/rufus/commit/de4cf676c2aedf73b0ddfc234e420d19c8969c2e) checks that the line starts with 32 hex characters inside the buffer before writing, and skips the entry with "Invalid MD5 sum entry - Ignored" otherwise. It also rejects a `setupldr.bin` under 64 bytes and allocates it with a terminator. For `uprintf()` the maintainer took a different route from my patch. Instead of clamping the pointer, the commit makes `safe_vsnprintf()` map to `_vsnprintf_s()` on every toolchain, so the existing truncation check is correct everywhere.
- [`842873017`](https://github.com/pbatard/rufus/commit/842873017f289713a5d5009320c11f31cfee36ed) converts `\r` before counting, so CR, LF and CRLF files all get the right number of entries.

## What I took from it

- An `assert()` on the first byte is not a length check. When code walks back to the start of a line and writes a fixed number of bytes, it has to know the line holds that many.
- When a loop both counts and rewrites a delimiter, the order matters. Normalize first, then count.
- A wrapper that hides the difference between `vsnprintf` and `_vsnprintf_s` is only safe if every caller agrees on what it returns. Making the wrapper behave the same on every toolchain fixed the caller and any others like it.

Thanks to the maintainer for merging the fixes within two days of the reports, keeping them as separate commits that name the advisories, and publishing both advisories with credit.
