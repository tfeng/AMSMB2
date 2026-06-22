# FinePlayer fork of AMSMB2

Fork of [amosavian/AMSMB2](https://github.com/amosavian/AMSMB2) **4.0.3**, published
under the moving tag **`4.0.3-fineplayer`** (force-recreated to point at the latest
fork commit). The Xcode project pins that tag; when it moves, clean the SwiftPM cache
and rebuild.

Two divergences from upstream 4.0.3:

## 1. libsmb2 submodule -> tfeng/libsmb2 (QUERY_DIRECTORY buffer)

`Dependencies/libsmb2` points at [tfeng/libsmb2](https://github.com/tfeng/libsmb2)
branch `fineplayer` = the exact revision 4.0.3 pins (`aff9fa6`, libsmb2-6.2-110) plus
one patch. Upstream hard-codes `req.output_buffer_length = DEFAULT_OUTPUT_BUFFER_LENGTH`
(64 KB) in both `opendir_cb` and the `query_cb` continuation, so a ~48k-file directory
needs ~110 round-trips (~35 s). Patched to request up to the negotiated max transact
size, capped at 1 MB, gated on `smb2->supports_multi_credit` (SMB 2.0.2 falls back to
64 KB) -> ~7 round-trips (~3 s, ~10x). Search `"FinePlayer fork"` in `lib/libsmb2.c`.

## 2. SMB2FileReader: persistent read handle (AMSMB2/AMSMB2.swift)

`contents(atPath:range:)` opens/reads/closes a fresh handle every call (~3 round-trips
per read). `SMB2FileReader` opens once and serves positioned reads via
`SMB2FileHandle.pread(offset:length:)` (~1 round-trip) -- for streaming media.
`SMB2Manager.openForReading(atPath:)` returns one; reuse one per connection (reads on a
single reader must not overlap). Search `"Persistent reader"` in AMSMB2.swift.

## Re-syncing with upstream

To bump: re-fork libsmb2 at the new pinned revision, re-apply the two
`output_buffer_length` edits (search `DEFAULT_OUTPUT_BUFFER_LENGTH` in the directory
functions); the reader uses only public AMSMB2 APIs.
