# FinePlayer fork of AMSMB2

Fork of [amosavian/AMSMB2](https://github.com/amosavian/AMSMB2) 4.x, with its
`libsmb2` 2.3.0 submodule **vendored in-tree** (under `Dependencies/libsmb2`) so the
package is self-contained (no submodule) and a single patch can be applied.

## Divergence from upstream

**`Dependencies/libsmb2/lib/libsmb2.c`** — `QUERY_DIRECTORY` output buffer.

Upstream libsmb2 hard-codes `req.output_buffer_length = DEFAULT_OUTPUT_BUFFER_LENGTH`
(64 KB) for directory enumeration, in both `opendir_cb` and the `query_cb`
continuation. On a high-latency link a large directory (e.g. ~48k files ≈ 7 MB of
listing) then needs ~110 round-trips ≈ 35 s.

Patched to request up to the **negotiated max transact size, capped at 1 MB**, gated
on `smb2->supports_multi_credit` (so SMB 2.0.2 servers fall back to 64 KB). 1 MB =
16 credits, safe against the 128 starting credits. This cuts the same listing to
~7 round-trips ≈ 3 s (~10×). Search `"FinePlayer fork"` in `libsmb2.c`.

No public API changed — the speedup is purely in libsmb2's directory read.

## Re-syncing with upstream

If bumping AMSMB2/libsmb2 later: re-vendor libsmb2 at the new revision, then re-apply
the two `output_buffer_length` edits (search `DEFAULT_OUTPUT_BUFFER_LENGTH` in the
directory functions).
