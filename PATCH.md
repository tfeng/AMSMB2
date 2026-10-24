# FinePlayer fork of AMSMB2

Fork of [amosavian/AMSMB2](https://github.com/amosavian/AMSMB2) **4.0.3**, tracked on
the `FXPlayer` branch. The Xcode project (`project.yml`) pins a specific commit on
that branch via `revision:`, not the branch tip itself — bump the revision there when
this branch moves.

Two divergences from upstream 4.0.3:

## 1. libsmb2 submodule -> tfeng/libsmb2 (QUERY_DIRECTORY buffer)

`Dependencies/libsmb2` points at [tfeng/libsmb2](https://github.com/tfeng/libsmb2)
branch `FXPlayer` = upstream `master` as of 2026-10-24 (`c51b6a6`) plus one patch.
Rebased onto current upstream from the prior base (`aff9fa6`, Aug 2025) to pick up
~400 commits of accumulated fixes this fork had missed for over a year — notably the
PR #431 security fixes (`5e75eeb`/`883e787`: OOB write in `smb2_add_iovector` via
chained PDUs, CVE-2025-57632; `70754b0`: NULL deref in `dcerpc_bind_async`; `d0801c5`:
off-by-one in compat `strdup`), the heap OOB read in `smb2_decode_file_all_info`
(`588a779`), and `b4a409e` ("Free everything when a call is cancelled or the context
is destroyed") — the closest upstream fix to a device crash this fork was carrying in
`smb2_read_data`/`smb2_service_fd`.

Upstream hard-codes `req.output_buffer_length = DEFAULT_OUTPUT_BUFFER_LENGTH`
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

To bump: rebase `tfeng/libsmb2`'s `FXPlayer` branch onto the new upstream commit,
re-apply the `output_buffer_length` patch (search `DEFAULT_OUTPUT_BUFFER_LENGTH` in the
directory functions — two call sites, in `opendir_cb` and `query_cb`), push, then bump
the submodule pointer here and the `revision:` in FXPlayer's `project.yml` to match.
