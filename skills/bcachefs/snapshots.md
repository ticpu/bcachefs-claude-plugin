# bcachefs Snapshots Deep Dive

## Files

- `snapshots/snapshot.h` / `snapshot.c` - Core snapshot logic
- `snapshots/check_snapshots.c` - Fsck snapshot validation
- `snapshots/delete.c` - Dead snapshot cleanup
- `snapshots/subvolume.h` / `subvolume.c` - Subvolume lifecycle
- `snapshots/types.h` - In-memory types
- `snapshots/format.h` - On-disk format

## In-Memory Snapshot Table

### `struct snapshot_t` (types.h:13-26)

- `state` - SNAPSHOT_ID_empty/live/deleted
- `parent`, `children[2]` - Tree structure (children normalized: [0] >= [1])
- `skip[3]` - Skiplist for O(log n) ancestor queries
- `depth` - Distance from root
- `subvol`, `tree` - Associated subvolume and tree ID
- `is_ancestor[BITS_TO_LONGS(128)]` - Bitmap for near ancestors

### `struct snapshot_table` (types.h:28-36)

RCU-protected flex array indexed by `U32_MAX - id`.
Swapped atomically via `kvfree_rcu()`.

## Subvolume/Snapshot Relationship

- Every **subvolume** (btree ID 8) holds a root inode and a snapshot ID
- The snapshot ID links to a **leaf** node in the snapshots btree (ID 9)
- Only leaf snapshot nodes have the `SUBVOL` flag and a `subvol` back-reference
- Interior snapshot nodes exist purely for tree structure (no subvolume)
- `snapshot_tree` entries group related snapshots under a root and master subvolume
- **On-disk**: `struct bch_subvolume` has `snapshot`, `inode`, `creation_parent`, `fs_path_parent`
- **On-disk**: `struct bch_snapshot` has `parent`, `children[2]`, `depth`, `skip[3]`, `subvol`, `tree`

### Snapshot Creation (`bch2_subvolume_create`, subvolume.c:543-603)

- New subvolume (no source): allocates 1 snapshot node as new tree root
- Snapshot of existing: allocates 2 children of source's snapshot node
  - `new_nodes[0]` → new subvolume's snapshot
  - `new_nodes[1]` → replaces source subvolume's snapshot
  - No keys copied; both children inherit ancestor keys via snapshot tree

## Ancestor Queries

### Fast Path: `__bch2_snapshot_is_ancestor()` (snapshot.c:113-135)

Two-layer approach:

1. **Skiplist phase** (O(log n)): While target is >128 positions away, `get_ancestor_below()` (snapshot.c:89-102) jumps to the largest of 3 skip entries that doesn't overshoot. Each step covers ~3/4 of remaining distance.
2. **Bitmap phase** (O(1)): Within 128 positions, `test_ancestor_bitmap()` (snapshot.c:104-111) does a single bit test on the 128-bit `is_ancestor` bitmap.

### Skiplist Construction

- `bch2_snapshot_skiplist_get()` (check_snapshots.c:211-221): picks a random ancestor by walking `get_random_u32_below(depth)` parent hops
- Called 3 times per snapshot at creation, results sorted into `skip[3]`
- Randomized selection gives geometric spread in expectation (like classic probabilistic skip lists)
- Cost: 12 bytes (3 × `__le32`) + 16 bytes (bitmap) = 28 bytes per snapshot node

### Bitmap Population (`__bch2_mark_snapshot`, snapshot.c:305-309)

- Walks parent chain from snapshot, sets bit for each parent within 128 IDs
- Populated at mount time and snapshot creation

### Early/Simple: `__bch2_snapshot_is_ancestor_early()` (snapshot.c:74-81)

Linear parent walk: `while (id && id < ancestor) { id = parent; }`
Used during recovery when skiplist/bitmap not yet validated.

### Public API (snapshot.h:196-228)

- `bch2_snapshot_is_ancestor(c, id, ancestor)` - Main entry (inline, handles id==ancestor)
- `bch2_snapshot_nth_parent(c, id, n)` (snapshot.h:108-116) - Linear walk, used only at creation
- `bch2_snapshot_root(c, id)` (snapshot.h:120-129)

## Snapshot Creation

### `bch2_snapshot_node_create()` (snapshot.c:604-618)

- parent==0: create 1 node (new tree root) via `bch2_snapshot_node_create_tree()`
- parent!=0: create 2 children via `bch2_snapshot_node_create_children()`

### `create_snapids()` (snapshot.c:508-549)

- Iterates backwards for ID allocation
- Sets parent, subvol, tree, depth
- Generates skiplist: `bch2_snapshot_skiplist_get()` + `bubble_sort()`
- Calls `__bch2_mark_snapshot()` to populate in-memory table

### Table Synchronization: `__bch2_mark_snapshot()` (snapshot.c:266-321)

Called as btree trigger on insert/update:
- Acquires `snapshots.table_lock`
- Copies all fields to in-memory table
- Initializes `is_ancestor` bitmap by walking parents
- Sets `BCH_FS_need_delete_dead_snapshots` if WILL_DELETE

## Snapshot Deletion

### Three States (format.h:48-77)

1. **WILL_DELETE** - Leaf, no subvol, pending cleanup
2. **NO_KEYS** - Interior, all keys moved to children
3. **DELETED** - Fully removed

### Main Flow: `__bch2_delete_dead_snapshots()` (delete.c:663-692)

1. **Scan**: `check_should_delete_snapshot()` (delete.c:497-556)
   - 0 live children -> `delete_leaves`
   - 1 live child -> `delete_interior` with `live_child` mapping
2. **Key deletion** (two strategies):
   - V1: `delete_dead_snapshot_keys_v1()` (delete.c:378) - iterate all btrees
   - V2: `delete_dead_snapshot_keys_v2()` (delete.c:427) - accelerated by inode ranges
3. **Node deletion**: `bch2_snapshot_node_delete()` (delete.c:156-279)

### Key-Level: `delete_dead_snapshots_process_key()` (delete.c:334-373)

- Key in `delete_leaves`: delete key
- Key in `delete_interior`: move to live_child snapshot

### Interior Cleanup: `bch2_delete_dead_interior_snapshots()` (delete.c:761-801)

- Scans for NO_KEYS with 1 live child
- Fixes depth: `bch2_fix_child_of_deleted_snapshot()` (delete.c:576-627)
- Deletes node: `bch2_snapshot_node_delete(..., true)`

## Snapshot-Aware Btree Iteration

### `__bch2_get_snapshot_overwrites()` (snapshot.c:481-506)

Finds ancestors that overwrite a key at given pos.

### `__bch2_key_has_snapshot_overwrites()` (snapshot.c:620-640)

Checks if key has overwrites in descendants.

### Visibility Rule

Keys visible in snapshot S: keys with snapshot <= S that aren't overwritten by a nearer snapshot ancestor.

## Whiteouts and Tombstones

### Key Types (bcachefs_format.h:389-427)

- **KEY_TYPE_deleted (0)** - No value, 0 bytes. Standard deletion marker.
- **KEY_TYPE_whiteout (1)** - No value, 0 bytes. Deletion marker that blocks visibility from ancestor snapshots.
- **KEY_TYPE_hash_whiteout (4)** - Used in hash btrees (dirents, xattrs) for linear probing tombstones.
- **KEY_TYPE_extent_whiteout (36)** - Extent-specific whiteout with monotonic bkey_start_pos().

### Helper Macros (btree/bkey_types.h:42-50)

```c
#define bkey_deleted(_k)         ((_k)->type == KEY_TYPE_deleted)
#define bkey_whiteout(_k)        ((_k)->type == KEY_TYPE_deleted || (_k)->type == KEY_TYPE_whiteout)
#define bkey_extent_whiteout(_k) ((_k)->type == KEY_TYPE_deleted ||
                                  (_k)->type == KEY_TYPE_whiteout ||
                                  (_k)->type == KEY_TYPE_extent_whiteout)
```

### The `needs_whiteout` Flag (bcachefs_format.h:203-209)

Bitfield in `struct bkey_packed` and `struct bkey`:
- 1 bit flag stored alongside `format` field
- **Purpose**: Marks keys in written bsets that require whiteout preservation during btree node rewrites
- **Lifecycle**: Set when keys from written bsets are overwritten, cleared when compacted

### When Whiteouts Are Created

#### Snapshot Deletion (btree/update.c:92-118)

`need_whiteout_for_snapshot()` determines if deletion requires whiteout:
1. Check if pos has parent snapshot
2. Iterate ancestors at same position (all_snapshots)
3. If ancestor has non-whiteout key: whiteout needed
4. Otherwise: regular KEY_TYPE_deleted sufficient

Called before:
- `bch2_btree_delete_at()` - converts to KEY_TYPE_whiteout if needed (update.c:508-513)
- Extent overwrite - creates whiteout for partial overlap (update.c:215-220)

#### Hash Collision Tombstones (fs/str_hash.h:219-242)

`bch2_hash_needs_whiteout()` checks linear probe chain:
- Scans forward from deletion point
- If any entry with `hash <= start->pos.offset`: whiteout needed (maintains probe chain)
- Used in dirents/xattrs to prevent lookup failures after deletion

#### Btree Node Commit (btree/commit.c:207-213)

When overwriting key in written bset:
```c
if (k->needs_whiteout) {
    if (bkey_deleted(&insert->k))
        push_whiteout(b, insert->k.p);  // Deleted key -> unwritten whiteout
    else
        insert->k.needs_whiteout = true;  // New key inherits flag
    k->needs_whiteout = false;
}
```

### Extent Whiteout Type Selection (btree/update.h:113-128)

`extent_whiteout_type()` chooses KEY_TYPE_extent_whiteout when:
- Btree is extent+snapshots (extents/reflink_p)
- Key is deleted
- Snapshot is leaf
- Feature flag `bcachefs_metadata_version_extent_snapshot_whiteouts` enabled

**Benefit**: bkey_start_pos() monotonic (excluding KEY_TYPE_whiteout), simplifies extent iteration.

### Whiteout Visibility and Filtering

#### BTREE_ITER_nofilter_whiteouts Flag

Used when internal code needs to see whiteouts:
- `bch2_trans_update_extent()` - extent overwrite logic (update.c:247)
- `bch2_btree_key_cache_fill()` - key cache fill from btree (key_cache.c:316)

#### Filtering During Iteration (btree/iter.c:2538-2555, 2965-2968, 3004-3007)

Normal iteration (`!BTREE_ITER_nofilter_whiteouts`):
- `bkey_extent_whiteout()` keys converted to KEY_TYPE_deleted
- Makes whiteouts invisible to userspace
- Extent whiteout comment: "no extents overlap with this whiteout - bkey_start_pos() is monotonically increasing"

### Unwritten Whiteouts Area (btree/interior.h:233-241)

Btree nodes have separate whiteout storage:
- Located at `(u64 *) btree_data_end(b) - b->whiteout_u64s`
- Grown backwards from end of node buffer
- Holds whiteouts created during transaction commit via `push_whiteout()` (interior.h:319-337)

#### `push_whiteout(b, pos)` (btree/interior.h:319-337)

Creates whiteout in unwritten area:
1. Pack key at pos (or use unpacked if won't fit)
2. Set `k.needs_whiteout = true`
3. Increment `b->whiteout_u64s`
4. Copy to `unwritten_whiteouts_start(b)`

#### Merging on Write (btree/write.c:392-406)

When flushing btree node:
- Unwritten whiteouts merged with bset via `bch2_sort_keys_keep_unwritten_whiteouts()`
- Whiteouts in written bsets have `needs_whiteout` set via `bch2_set_bset_needs_whiteout(i, true)` (write.c:573)
- After write: `b->whiteout_u64s = 0` (write.c:402)

### Whiteout Cleanup

#### Compaction (btree/sort.c:362-429)

`bch2_drop_whiteouts()` / `bch2_compact_whiteouts()`:
- Drops deleted keys (both written and unwritten)
- Compacts bsets to reclaim space
- Triggered by mode: COMPACT_LAZY vs COMPACT_ALL

#### Sorting (btree/sort.c:308-343)

`bch2_sort_whiteouts()`:
- Sorts unwritten whiteout area by position
- Called before node write to ensure merge ordering

#### Interior Node Filtering (btree/iter.c:621-627)

Interior nodes skip whiteouts during traversal:
```c
if (level)
    bch2_btree_node_iter_peek(&l->iter, l->b);  // Advances past whiteouts
```

Comment: "Iterators to interior nodes should always be pointed at the first non whiteout"

#### Snapshot Deletion Tombstone Handling (snapshots/delete.c:334-373)

`delete_dead_snapshots_process_key()`:
- Deletes keys in dead leaf snapshots
- Moves keys from interior snapshots to live_child
- Both operations use `bch2_btree_delete_at(..., BTREE_UPDATE_internal_snapshot_node)`
- Relies on `need_whiteout_for_snapshot()` to determine if whiteout needed

### KEY_TYPE_deleted vs KEY_TYPE_whiteout

| Aspect | KEY_TYPE_deleted | KEY_TYPE_whiteout |
|--------|------------------|-------------------|
| **Size** | 0 bytes (header only) | 0 bytes (header only) |
| **Snapshot visibility** | Visible in all snapshots | Blocks ancestor snapshots |
| **Use case** | No parent snapshot | Has parent with same key |
| **Compaction** | Removed during sort | Preserved if needs_whiteout set |
| **Iterator filtering** | Returned as deleted | Converted to deleted (unless nofilter) |
| **Hash tables** | Breaks probe chain | Maintains probe chain |
| **Extent variant** | Same behavior | Has KEY_TYPE_extent_whiteout for monotonic start_pos |

## Subvolume Lifecycle

### `struct bch_subvolume` (format.h:9-25)

- `snapshot` - Current snapshot ID
- `inode` - Root inode
- `creation_parent` / `fs_path_parent` - Hierarchy
- Flags: RO, SNAP, UNLINKED

### `bch2_subvolume_create()` (subvolume.c:543-603)

- `src_subvolid == 0`: New subvolume (no parent snapshot)
- `src_subvolid != 0`: Snapshot of existing
  - Source gets `new_nodes[1]`, new subvol gets `new_nodes[0]`
  - Sets SNAP and RO flags

### `bch2_subvolume_unlink()` (subvolume.c:521-541)

Commit hook pattern: registers `subvolume_unlink_hook` (subvolume.c:498-501).
Sets UNLINKED, triggers `bch2_subvolume_wait_for_pagecache_and_delete()`.

### `__bch2_subvolume_delete()` (subvolume.c:409-452)

- Clears master_subvol if needed
- Deletes subvolume key
- Calls `bch2_snapshot_node_set_deleted()` on snapshot

## Initialization

### `bch2_initialize_subvolumes()` (subvolume.c:605-633)

- Root snapshot tree ID 1
- Root snapshot ID U32_MAX
- Root subvolume BCACHEFS_ROOT_SUBVOL (1)

### `bch2_snapshots_read()` (snapshot.c:642-672)

Reverse iteration of BTREE_ID_snapshots (ancestors before descendants).
Calls `__bch2_mark_snapshot()` on each to populate table.

## Fsck (check_snapshots.c)

### `bch2_check_snapshot_trees()` (check_snapshots.c:183-191)

Validates root_snapshot, master_subvol consistency.

### `bch2_check_snapshots()` (check_snapshots.c:415-437)

Reverse iteration. Validates parent/child pointers, depth, skiplist, SUBVOL flag.

### `bch2_reconstruct_snapshots()` (check_snapshots.c:550-594)

Scans all snapshot-bearing btrees, recreates missing snapshot entries.
