---
layout: default
title: "Avalonia: a CF_DIB from the clipboard or a drop could make GDI read outside the buffer on Windows"
description: "Notes on GHSA-x3fw-83fm-ggpp in AvaloniaUI/Avalonia. ReadDataFromHGlobal passed the DIB header to StretchDIBits without checking it against the size of the buffer, so a crafted bitmap returned bytes of the reading process's own heap as pixels. Fixed in Avalonia.Win32 11.3.23 and 12.1.6."
date: 2026-11-23
---

# Avalonia: a CF_DIB from the clipboard or a drop could make GDI read outside the buffer on Windows

**Advisory:** [GHSA-x3fw-83fm-ggpp](https://github.com/AvaloniaUI/Avalonia/security/advisories/GHSA-x3fw-83fm-ggpp) (published 2026-11-10)
**Product:** NuGet `Avalonia.Win32`, affected `>= 11.3.9, <= 12.1.5`, patched in 11.3.23 and 12.1.6
**Weakness:** CWE-125 and CWE-190, CVSS 3.1 `AV:L/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:L` (Low)

[Avalonia](https://github.com/AvaloniaUI/Avalonia) is a cross-platform .NET UI framework. Since 11.3.9 an app can read an image from the clipboard or from a drag and drop with `TryGetBitmapAsync`. On Windows, when the data object offers `CF_DIB`, Avalonia converts the packed bitmap to a 32 bpp `Bitmap` with GDI.

## Root cause

`OleDataObjectHelper.ReadDataFromHGlobal` in `src/Windows/Avalonia.Win32/OleDataObjectHelper.cs` copied the HGLOBAL into a managed array of exactly `GlobalSize` bytes and read the `BITMAPINFOHEADER` from its start. It then called `StretchDIBits` with the width and height from that header and a bits pointer of `ptr + biSize + extra`, where `extra` is the colour table size. Nothing compared any of those numbers with the length of the array.

GDI trusts the header it is given, so there were two ways to send it outside the array:

- **Short buffer.** A header that describes more scan lines than the buffer holds. GDI reads the missing rows from whatever follows the array.
- **Colour table offset.** For `BI_RGB` at 8 bpp or less the colour table size was computed as `(int)biClrUsed * 4`. A large `biClrUsed` wraps to a negative number, so the bits pointer lands before the array, at an offset the sender picks. GDI caps its own palette at 256 entries, so the call still succeeds. This path came in with 11.3.12.

The array lives in the reading process's GC heap. The bytes GDI read came back as pixels in the `Bitmap` the app received, so what leaked was the pasting process's own memory. A second, smaller problem sat in the same function. `biHeight = int.MinValue` made `Math.Abs` throw `OverflowException`, which crashes an app that awaits `TryGetBitmapAsync` in an `async void` handler without a `try`.

Who gains from this depends on where the clipboard comes from. Another process in the same session can already read the target's memory directly. The interesting cases are boundaries such as RDP clipboard redirection, VM clipboard sharing, or an app that copies a DIB from a document verbatim, and in each of them the attacker only sees the bytes if the resulting image is saved, shared or copied back to them. That is why the advisory is rated Low.

## Fix

The fix is part of [#22757](https://github.com/AvaloniaUI/Avalonia/pull/22757), merged on 2026-11-09 as [`ae5ae6124e`](https://github.com/AvaloniaUI/Avalonia/commit/ae5ae6124e62ac22ce310b823211cf6d6a4386c5). A new `TryGetDibPixelDataOffset` runs before any pointer is taken and accepts the DIB only when:

- the buffer holds a full header and `biSize` lies between the header size and the buffer length,
- width and height are positive and `biPlanes` is 1,
- the format is `BI_RGB` at 1, 4, 8, 16, 24 or 32 bpp, or `BI_BITFIELDS` at 16 or 32 bpp,
- at 8 bpp or less, `biClrUsed` is no larger than `2^bpp`,
- the 32 bpp copy fits in an `int`,
- header, colour table and DWORD-aligned stride times height, computed in `long`, fit in the buffer.

`ReadDataFromHGlobal` then takes the bits offset from that check. Requiring a positive height also covers the `int.MinValue` case. The same pull request adds `OleDataObjectHelperTests` to the Win32 integration tests.

The timing is worth stating plainly. The pull request was merged at 09:23 UTC on 2026-11-09 and 11.3.23 and 12.1.6 were released that afternoon, while my report reached the maintainers at 14:07 UTC the same day. They had found and fixed the problem themselves as part of a wider pass over native memory handling. They still opened the advisory from the report, listed the affected and patched versions, and published it the next day.

## What I took from it

- When a buffer crosses a process boundary, every length and offset in its header is input. Passing them to an API that trusts them, here GDI, moves the bounds check to code that does not do one.
- A cast from `uint` to `int` before a multiplication turns "too large" into "negative". Bounding the count first and doing the size arithmetic in `long` against the buffer length closes the offset case and the short buffer case.
- A read past the end that comes back as image data is easy to miss in testing. The image looks like noise, and the process does not crash.

Thanks to the Avalonia maintainers for shipping the fix on both the 11.3 and 12.1 lines and for publishing an advisory, with credit, for a bug they had already closed.
