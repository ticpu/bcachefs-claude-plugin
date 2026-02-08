# bcachefs Metadata Version History

This document details all bcachefs on-disk metadata format versions. Each version represents a format change that requires an on-disk upgrade.

**Source**: `fs/bcachefs/bcachefs_format.h` in kernel tree

## Version Numbering

Versions use major.minor format: `BCH_VERSION(major, minor)`
- Major version changes indicate significant format overhauls (0.x -> 1.x)
- Minor version changes indicate incremental format additions
- Current minimum supported version: 9 (internal, pre-0.10)
- Current version: determined by `bcachefs_metadata_version_max - 1`

## 0.x Series (Initial Development)

### 0.10 - bkey_renumber
Renumbered key type values for better organization. Required compatibility code to translate old key type numbers to new ones during read.

### 0.11 - inode_btree_change
Changed inode btree key format: swapped inode and offset fields in bpos for better locality. Inodes now use offset field for inode number instead of inode field.

### 0.12 - snapshot
**Major feature**: Introduced snapshot support. Added snapshot field to bpos (struct bpos gained snapshot field). Added KEY_TYPE_snapshot, KEY_TYPE_subvolume. Enabled copy-on-write snapshots.

### 0.13 - inode_backpointers
Added backpointers from data extents to inodes they belong to, enabling more efficient fsck and data recovery.

### 0.14 - btree_ptr_sectors_written
Added sectors_written field to btree_ptr_v2 to track how much of a btree node has been written (important for partial writes and recovery).

### 0.15 - snapshot_2
Snapshot improvements: fixed subvolume root inode tracking and snapshot/subvolume relationship handling. Required full fsck.

### 0.16 - reflink_p_fix
Fixed reflink_p (reflink pointer) key handling to correctly update reflink reference counts.

### 0.17 - subvol_dirent
Properly linked subvolumes to their directory entries, enabling correct subvolume parent tracking.

### 0.18 - inode_v2
Introduced KEY_TYPE_inode_v2 with improved field layout and additional metadata fields (better flags, timestamps, etc.).

### 0.19 - freespace
Added freespace btree (BTREE_ID_freespace) for tracking free space, improving allocation performance.

### 0.20 - alloc_v4
**Major allocator change**: Introduced KEY_TYPE_alloc_v4 with comprehensive allocation metadata (dirty_sectors, cached_sectors, stripe pointer, etc.). Replaced older alloc key versions.

### 0.21 - new_data_types
Refined data type classifications in the allocator (data vs metadata vs cached).

### 0.22 - backpointers
**Major feature**: Introduced backpointers btree (BTREE_ID_backpointers) mapping physical locations back to logical extents. Enables efficient space accounting and evacuate/rebalance operations.

### 0.23 - inode_v3
Introduced KEY_TYPE_inode_v3 with even more fields (bi_hash_seed moved to value, additional flags for features like encryption per-inode).

### 0.24 - unwritten_extents
Added support for unwritten/preallocated extents (allocated but not yet written, reading returns zeros).

### 0.25 - bucket_gens
Added bucket_gens btree (BTREE_ID_bucket_gens) tracking generation numbers for buckets, improving stale pointer detection. Required bucket_gens_init recovery pass.

### 0.26 - lru_v2
Improved LRU (Least Recently Used) tracking for cache eviction with better metadata.

### 0.27 - fragmentation_lru
Added LRU tracking for fragmented extents to prioritize defragmentation of heavily fragmented data.

### 0.28 - no_bps_in_alloc_keys
Removed backpointers from alloc keys (moved fully to backpointers btree), simplifying allocator keys.

### 0.29 - snapshot_trees
Added snapshot_trees btree (BTREE_ID_snapshot_trees) for hierarchical snapshot tree management.

## 1.x Series (Stable Format)

### 1.0 - major_minor
**Major version bump**: Transitioned to 1.x version series, indicating format stabilization. This version marks the introduction of the major.minor versioning scheme.

### 1.1 - snapshot_skiplists
Added skiplist structure to snapshots for O(log n) ancestor queries instead of O(n). Greatly improves snapshot deletion performance.

### 1.2 - deleted_inodes
Added deleted_inodes btree (BTREE_ID_deleted_inodes) tracking unlinked but not yet deleted inodes (for crash recovery of unlink operations).

### 1.3 - rebalance_work
Added infrastructure for background rebalance work tracking (moving data between devices to meet replication/compression targets).

### 1.4 - member_seq
Added sequence numbers to member devices for detecting stale device superblocks in multi-device filesystems.

### 1.5 - subvolume_fs_parent
Added fs_parent field to subvolumes, tracking parent directory for proper path resolution.

### 1.6 - btree_subvolume_children
Added subvolume_children btree (BTREE_ID_subvolume_children) for efficient parent->child subvolume lookups.

### 1.7 - mi_btree_bitmap
Added bitmap to member_info tracking which btrees have data on each device (for device removal).

### 1.8 - bucket_stripe_sectors
Added stripe_sectors tracking to alloc keys for erasure coding support (tracking sectors used by stripes per bucket).

### 1.9 - disk_accounting_v2
**Major accounting change**: Moved space accounting from in-memory counters to on-disk accounting btree, enabling persistent and accurate accounting.

### 1.10 - disk_accounting_v3
Refined disk_accounting_v2 with better key validation and endianness handling.

### 1.11 - disk_accounting_inum
Added per-inode accounting tracking (space used per inode) in the accounting system.

### 1.12 - rebalance_work_acct_fix
Fixed accounting bugs in rebalance work tracking.

### 1.13 - inode_has_child_snapshots
Added bi_has_child_snapshots flag to inodes indicating whether an inode has data in child snapshots (optimization for snapshot deletion).

### 1.14 - backpointer_bucket_gen
Added bucket generation number to backpointers, enabling detection of stale backpointers without reading alloc keys.

### 1.15 - disk_accounting_big_endian
Fixed disk accounting to work correctly on big-endian systems.

### 1.16 - reflink_p_may_update_opts
Enabled reflink_p keys to carry extent options/flags that can be updated independently of the underlying reflink_v extent.

### 1.17 - inode_depth
Added directory depth tracking to inodes for path resolution optimization.

### 1.18 - persistent_inode_cursors
Added persistent inode allocation cursors (KEY_TYPE_inode_alloc_cursor in logged_ops btree) for better inode number allocation.

### 1.19 - autofix_errors
**Behavior change**: Changed default error action to fix_safe, automatically fixing safe-to-fix errors during normal operation.

### 1.20 - directory_size
Added directory size tracking to inode metadata for efficient directory listing.

### 1.21 - cached_backpointers
Enabled caching of backpointers in memory for faster backpointer lookups during IO operations.

### 1.22 - stripe_backpointers
Added backpointers for erasure-coded stripes, enabling proper stripe accounting and recovery.

### 1.23 - stripe_lru
Added LRU tracking for stripes to manage stripe cache eviction.

### 1.24 - casefolding
**Major feature**: Added casefold support (case-insensitive filenames) with per-directory flags. Added BCH_SB_CASEFOLD superblock flag.

### 1.25 - extent_flags
Added flags field to extent keys for storing extent-level metadata (e.g., nocow, compression hints).

### 1.26 - snapshot_deletion_v2
Improved snapshot deletion algorithm with better handling of snapshot trees and interior node deletion.

### 1.27 - fast_device_removal
Enabled fast device removal when no_stale_ptrs compat flag is set (no need to scan for stale pointers).

### 1.28 - inode_has_case_insensitive
Added bi_has_case_insensitive flag to inodes, marking whether directory uses casefold. Required for proper casefold directory handling.

### 1.29 - extent_snapshot_whiteouts
Introduced KEY_TYPE_extent_whiteout for efficient handling of deleted extents in snapshot leaf nodes (instead of regular whiteouts).

### 1.30 - 31bit_dirent_offset
Extended directory entry offset field to 31 bits (from previous limit), supporting much larger directories.

### 1.31 - btree_node_accounting
**Major feature**: Added per-btree-node space accounting in the accounting system, enabling accurate btree space tracking.

### 1.32 - sb_field_extent_type_u64s
Added superblock field tracking u64 sizes for extent types, supporting variable-size extent formats.

### 1.33 - reconcile
**Major feature**: Introduced reconcile system for managing IO options inheritance and updates. Added multiple reconcile btrees (BTREE_ID_reconcile_work, BTREE_ID_reconcile_hipri, BTREE_ID_reconcile_pending, BTREE_ID_reconcile_scan, BTREE_ID_reconcile_work_phys, BTREE_ID_reconcile_hipri_phys). Tracks and propagates extent option changes through reflink chains.

### 1.34 - extented_key_type_error
Extended KEY_TYPE_error format to zero-byte value (previously had 8-byte value), saving space when marking error keys.

### 1.35 - bucket_stripe_index
Added stripe index tracking to alloc keys, enabling efficient stripe->bucket lookups and proper stripe reference counting.

### 1.36 - no_sb_user_data_replicas
Removed redundant user_data replicas tracking from superblock (now tracked via disk accounting).

## Upgrade Requirements

**Required upgrade below version 1.31 (btree_node_accounting)**: Filesystems below this version must be upgraded before mounting with current bcachefs-tools.

## Downgrade Support

The downgrade table in `sb/downgrade.c` specifies recovery passes and errors to silence when downgrading. Not all versions support downgrade - downgrading typically requires running fsck to regenerate metadata in older format.

## Version Checks in Code

Version checks throughout the codebase use patterns like:
```c
if (version >= bcachefs_metadata_version_foo)
    // Use new format
else
    // Use old format or compatibility code
```

Key files with version checks:
- `btree/read.c`, `btree/read.h` - Btree node compatibility
- `btree/bkey_methods.c` - Key format compatibility
- `sb/downgrade.c` - Upgrade/downgrade tables
- `init/recovery.c` - Recovery pass requirements
- Individual subsystem files check their specific version requirements

## See Also

- `architecture.md` - Overall bcachefs design
- `format.md` - Current on-disk format details
- `sb/downgrade_format.h` - Downgrade table structure
- `init/passes_format.h` - Recovery passes triggered by upgrades
