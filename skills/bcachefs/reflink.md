# bcachefs Reflink Deep Dive

## Files

- `data/reflink.c` - Core reflink logic (creation, triggers, remap)
- `data/reflink.h` - Public API declarations
- `data/reflink_format.h` - On-disk structures (reflink_p, reflink_v, indirect_inline_data)
- `data/read.h` - Read path indirect extent resolution (`bch2_read_indirect_extent()`)
- `data/reconcile/trigger.c` - IO option propagation (`bch2_bkey_get_io_opts()`, `bch2_bkey_needs_reconcile()`)
- `data/reconcile/work.c` - Reconcile scan for indirect extents (`do_reconcile_scan_indirect()`)
- `data/reconcile/format.h` - `bch_extent_reconcile` structure
- `vfs/io.c` - VFS entry point (`bch2_remap_file_range()`)

## On-Disk Structures

### `bch_reflink_p` (reflink_format.h:5-18)

Pointer in extents btree replacing original extent:
- `idx_flags` (__le64): `REFLINK_P_IDX` (bits 0-56) index into reflink btree, `REFLINK_P_ERROR` (bit 56) missing indirect extent, `REFLINK_P_MAY_UPDATE_OPTIONS` (bit 57) IO option propagation permission
- `front_pad` (__le32): extends referenced range backward (for split fragments)
- `back_pad` (__le32): extends referenced range forward (for split fragments)

### `bch_reflink_v` (reflink_format.h:25-30)

Indirect extent in reflink btree (ID 7, NOT snapshot-aware):
- `refcount` (__le64): reference count
- `start[0]` (union bch_extent_entry): data pointers, CRCs, compression metadata (identical to KEY_TYPE_extent)
- May embed `bch_extent_reconcile` entry for IO option tracking

### `bch_indirect_inline_data` (reflink_format.h:32-36)

Reflinked inline data:
- `refcount` (__le64)
- `data[]` (u8): inline data bytes

## Creation Flow

### `bch2_remap_range()` (reflink.c:551)

Entry point for FICLONE/cp --reflink. For each source extent:

1. If source is `KEY_TYPE_extent`: calls `bch2_make_extent_indirect()` (reflink.c:469)
   - Allocates `reflink_v` at end of reflink btree (seeks `POS_MAX`, peeks prev)
   - Copies extent value verbatim, prepends refcount=0
   - Replaces source extent with `reflink_p`
   - Sets `REFLINK_P_MAY_UPDATE_OPTIONS` on source's reflink_p
2. If source already `KEY_TYPE_reflink_p`: no conversion needed
3. Creates new `reflink_p` in destination (trigger increments refcount)

### `bch2_remap_file_range()` (vfs/io.c:877)

VFS wrapper. Passes `may_change_src_io_path_opts = false` (XXX comment at line 933: needs mnt_idmap from pathwalk). Destination reflink_p never gets `MAY_UPDATE_OPTIONS`.

## Reflink_p Trigger

### `bch2_trigger_reflink_p()` (reflink.c:415)

On insert: resets front_pad/back_pad to 0, then delegates to `__trigger_reflink_p()`.

### `__trigger_reflink_p()` (reflink.c:380)

Walks range `[REFLINK_P_IDX - front_pad, REFLINK_P_IDX + size + back_pad)`:

### `trans_trigger_reflink_p_segment()` (reflink.c:289)

Per-fragment:
- Looks up reflink_v via `bch2_lookup_indirect_extent()`
- Insert: increments refcount, expands front_pad/back_pad if reflink_v extends beyond reflink_p range
- Overwrite: decrements refcount; detects underflow as fsck error

## Reflink_v Trigger

### `bch2_trigger_reflink_v()` (reflink.c:445)

- Calls `check_indirect_extent_deleting()`: if refcount==0, converts key to `KEY_TYPE_deleted`
- Then delegates to `bch2_trigger_extent()` for standard extent trigger (backpointers, allocation updates)

### `check_indirect_extent_deleting()` (reflink.c:433)

When `BTREE_TRIGGER_insert` and refcount==0:
- Sets type to `KEY_TYPE_deleted`, size=0, val_u64s=0
- Clears `BTREE_TRIGGER_insert` flag (so extent trigger only processes overwrite side)

## One-Way Transformation

Reflink is currently one-way: reflink_v never converts back to KEY_TYPE_extent at refcount=1.

De-indirection challenges:
- reflink_v trigger doesn't know which reflink_p holds last reference
- fcollapse/finsert cause transient 1→0→1 refcount fluctuations in single transaction
- reflink_p keys not merged (merged pointer could span unbounded reflink_v fragments)

## Read Path

### `bch2_read_indirect_extent()` (read.h:93)

If extent is `KEY_TYPE_reflink_p`:
- Sets `*data_btree = BTREE_ID_reflink`
- Calls `bch2_lookup_indirect_extent()` (reflink.c:257)
- Resolves to reflink_v with actual device pointers

### `bch2_lookup_indirect_extent()` (reflink.c:257)

- Computes `reflink_offset = REFLINK_P_IDX + offset_into_extent`
- Peeks in reflink btree at `POS(0, reflink_offset)`
- Missing extent in live range: calls `bch2_indirect_extent_missing_error()` which sets `REFLINK_P_ERROR`
- Missing extent in pad-only range: adjusts pad instead

## IO Option Propagation

### Problem

reflink_v has no backpointer to owning inode → cannot look up per-inode IO options.

### Solution: `bch_extent_reconcile` entry (reconcile/format.h:90-134)

Embedded in reflink_v's extent entries. Stores:
- Desired IO options (replicas, checksum, compression, target, promote_target, erasure_code)
- `*_from_inode` flags: which options came from inode vs filesystem defaults
- `need_rb` bitmask: which options don't match current data

### `bch2_bkey_get_io_opts()` (reconcile/trigger.c:854)

Three modes:
- `IO_OPTS_user` (extents btree, reflink_p): options from referencing inode
- `IO_OPTS_reflink` (reflink btree, reflink_v): filesystem defaults + embedded reconcile entry overrides
- `IO_OPTS_metadata` (btree ptrs): filesystem defaults with metadata=true

### Reconcile scan (reconcile/work.c:863)

When scanning extents btree:
- For each `reflink_p` with `REFLINK_P_MAY_UPDATE_OPTIONS` set
- Calls `do_reconcile_scan_indirect()` (work.c:795)
- Walks reflink btree for the pointer's range
- Updates reflink_v's embedded reconcile entry with referencing inode's current options
- Without the flag: inode's options are NOT propagated to indirect extent

### Read path uses

- reflink_v's CRC/compression entries for actual IO (how data was written)
- Referencing inode's options only for promote decisions

## Snapshot Interaction

- Reflink btree (ID 7) is NOT snapshot-aware (no `BTREE_IS_snapshots` flag)
- reflink_v keys shared across all snapshots
- reflink_p keys in extents btree ARE snapshot-aware
- Snapshot creation: both subvolumes see same reflink_p through normal visibility
- Write to one snapshot: creates new KEY_TYPE_extent, trigger decrements shared refcount

## GC

### `bch2_gc_reflink_start()` (reflink.c:757)

Builds table of all reflink_v entries (offset, size, refcount=0).

### During extent GC

Updates reference counts in table via `gc_trigger_reflink_p_segment()` (reflink.c:346).

### `bch2_gc_reflink_done()` (reflink.c:738)

Corrects any refcount mismatches found during GC.

## Key Type Mapping

- `KEY_TYPE_extent` → `KEY_TYPE_reflink_v` (via `bkey_type_to_indirect()`)
- `KEY_TYPE_inline_data` → `KEY_TYPE_indirect_inline_data`
- Feature flag: `BCH_FEATURE_reflink` (always set), `BCH_FEATURE_reflink_inline_data` (for inline)
