# bcachefs Metadata Version History

All bcachefs on-disk metadata format versions, derived from git commit messages in the kernel tree.

**Source**: `BCH_METADATA_VERSIONS()` in `fs/bcachefs/bcachefs_format.h`

## Version Numbering

Versions use `BCH_VERSION(major, minor)` encoding.
- Minimum supported: 9 (internal, pre-0.10)
- Current: `bcachefs_metadata_version_max - 1`
- Required upgrade below: `bcachefs_metadata_version_btree_node_accounting` (1.31)

## 0.x Series

### 0.10 - bkey_renumber
Replaced per-btree key type numbering with a single global `BCH_BKEY_TYPES()` enum where all key types have unique numbers. Also introduced the `bcachefs_metadata_version` enum and versioning scheme.

### 0.11 - inode_btree_change
Changed inode indexing from the inode field of bpos to the offset field, eliminating special cases in the btree iterator. Unified btree node min_key handling so min_key = bkey_successor() of the previous node for all btree types.

### 0.12 - snapshot
Started treating bpos.snapshot as part of the key in btree code. Keys in snapshot-aware btrees (extents, inodes, dirents, xattrs) now have snapshot set to U32_MAX, with BTREE_ITER_ALL_SNAPSHOTS controlling whether iteration includes the snapshot field. Introduced SPOS() macro.

### 0.13 - inode_backpointers
Added bi_dir and bi_dir_offset inode fields pointing back to the inode's parent dirent. Files that have ever been hardlinked get BCH_INODE_BACKPTR_UNTRUSTED. Needed for directories to support enumerating subvolumes by reconstructing paths to subvolume roots.

### 0.14 - btree_ptr_sectors_written
Changed btree node pointer management to update parent pointers after every btree node write, tracking sectors_written. Previously pointers were only updated on split/compact; now the parent always knows how much has been written, improving crash recovery.

### 0.15 - snapshot_2
Revved the format version to trigger compatibility code that creates initial entries in the subvolumes and snapshots btrees during upgrade: root snapshot node, root subvolume, and bi_subvol on the root inode.

### 0.16 - reflink_p_fix
Fixed uninitialized second word in reflink_p pointers. The version bump triggers cleanup during upgrade that zeroes out the second word of all existing reflink_p pointers.

### 0.17 - subvol_dirent
Changed subvolume dirents to also record the subvolid of the parent subvolume, allowing subvolume dirents to be filtered when listing directories from other subvolumes. Ensures only one dirent across snapshot multiplicities points to a given subvolume.

### 0.18 - inode_v2
Introduced KEY_TYPE_inode_v2 and KEY_TYPE_alloc_v3 with a journal_seq field recording the journal sequence number at last modification. For alloc keys, tracks which journal entry must flush before bucket reuse. For inodes, enables correct fsync when evicted from VFS cache, replacing a bloom filter mechanism.

### 0.19 - freespace
Added freespace btree (extents btree of free buckets) and need_discard btree (buckets awaiting discards) for the allocator rewrite. Triggers on alloc keys keep these btrees in sync.

### 0.20 - alloc_v4
Introduced KEY_TYPE_alloc_v4 without varints, in preparation for storing backpointers in alloc keys. Varint pack/unpack was impractical for in-place mutation. bch2_alloc_to_v4() converts older types as needed.

### 0.21 - new_data_types
Merged bucket states (free, need_gc_gens, need_discard) into BCH_DATA_TYPES(). Data type 0 = BCH_DATA_free, free buckets explicitly counted. Previously buckets in transitional states were unaccounted for.

### 0.22 - backpointers
Added reverse index from device offset back to btree nodes and non-cached data extents. First 40 backpointers per bucket stored inline in alloc key, overflow in new BTREE_ID_backpointers btree. Enables copygc to find data without scanning all extents.

### 0.23 - inode_v3
Moved bi_size and bi_sectors into the non-varint portion of the inode so the write path avoids expensive unpack/pack. Added varint section offset field (allowing new non-varint fields without a new inode type). Moved bi_mode into flags for u64-aligned varint section.

### 0.24 - unwritten_extents
Added nocow mode where writes go in-place when possible. Introduced unwritten extent type, bi_nocow inode option, bucket locking for in-place write races, and a nocow write path that falls back to COW. Nocow implicitly disables checksumming and compression.

### 0.25 - bucket_gens
Added BTREE_ID_bucket_gens storing bucket generation numbers densely (256 per key) to improve mount times by reducing metadata scanned at startup. Triggers keep it in sync with the alloc btree.

### 0.26 - lru_v2
Changed LRU index from KEY_TYPE_lru (bucket in value) to KEY_TYPE_set (bucket encoded in key bits). Eliminates collision checking on insert, enabling pure btree write buffer where updates are appended and batch-inserted.

### 0.27 - fragmentation_lru
Added LRU index of buckets ordered by fragmentation so copygc no longer scans every bucket. A fragmentation_lru field in bch_alloc_v4 records each bucket's LRU position.

### 0.28 - no_bps_in_alloc_keys
Removed inline backpointers from alloc keys. The btree write buffer made the separate backpointers btree efficient enough. Version bump triggers fsck to clean up remaining inline backpointers.

### 0.29 - snapshot_trees
Added BTREE_ID_snapshot_trees with KEY_TYPE_snapshot_tree providing persistent per-snapshot-tree identifiers. Each bch_snapshot points to its snapshot_tree. Designates one subvolume per tree as "main" for quota accounting.

## 1.x Series

### 1.0 - major_minor
Introduced major.minor version numbering, replacing the flat integer scheme. Prior versions placed under major 0. Added BCH_SB_VERSION_UPGRADE_COMPLETE and changed upgrade handling from boolean to enum (none/compatible/incompatible).

### 1.1 - snapshot_skiplists
Extended KEY_TYPE_snapshot with depth (distance from root) and skip[3] (skiplist entries for ancestor walking). Makes bch2_snapshot_is_ancestor() O(ln(n)) instead of O(n) in snapshot tree depth.

### 1.2 - deleted_inodes
Added deleted_inodes bitset btree to track inodes pending deletion, eliminating full inodes btree scan after unclean shutdown. check_inodes became fsck-only.

### 1.3 - rebalance_work
Added rebalance_work bitset btree to eliminate scanning for extents needing background rebalance (background_target, background_compression). Added bch_extent_rebalance extent field indicating pending work and which options to use, allowing per-inode options to propagate to indirect extents. Maintained by the extent trigger.

### 1.4 - member_seq
Added bch_member->seq tracking last superblock write sequence number and bch_sb->write_time for last write timestamp. Enables split-brain detection when members diverge.

### 1.5 - subvolume_fs_parent
Added fs_path_parent field to bch_subvolume recording the parent subvolume in the directory tree, enabling efficient subvolume path resolution.

### 1.6 - btree_subvolume_children
Added subvolume_children bitset btree recording parent-to-child subvolume relationships. Enables efficient subvolume listing and recursive deletion.

### 1.7 - mi_btree_bitmap
Added 64-bit per-device btree_allocated_bitmap to bch_member tracking which ranges have btree nodes. Accelerates btree node scan during recovery by skipping empty ranges.

### 1.8 - bucket_stripe_sectors
Added stripe_sectors and BCH_DATA_unstriped to alloc keys for accounting unstriped data in erasure-coded stripe buckets. Upgrade/downgrade requires regenerating alloc info only if erasure coding is in use.

### 1.9 - disk_accounting_v2
Version marker for the new btree-based disk space accounting system, replacing replicas-table-based accounting. Requires check_alloc_info recovery pass on upgrade.

### 1.10 - disk_accounting_v3
Fixed padding bytes erroneously included in disk_accounting_key from v2. All unused bytes must be zeroed, so the padding was a correctness bug.

### 1.11 - disk_accounting_inum
Added per-inode disk accounting counter tracking number of extents and total size (keyspace sectors and actual disk usage, across any snapshot ID). Enables reporting extra space from snapshot versioning and cheaply finding fragmented files by average extent size.

### 1.12 - rebalance_work_acct_fix
Fixed rebalance_work accounting which incorrectly keyed off the presence of rebalance_opts in extents. Those options are kept around after rebalance for indirect extents since the inode's options are not directly available.

### 1.13 - inode_has_child_snapshots
Added BCH_INODE_has_child_snapshot flag fixing a race where snapshotting while an unlinked file is open, then reattaching in the child, made the file appear deletable in the parent. New deletion rule: unlinked, non-open files without child snapshot references are deleted.

### 1.14 - backpointer_bucket_gen
Backpointers now include bucket generation number; obsolete bucket_offset field removed. Expensive forced incompatible upgrade requiring extents_to_backpointers pass to regenerate all backpointers, but enables cheaper check_extents_to_backpointers by comparing per-bucket backpointer sums against sector counts.

### 1.15 - disk_accounting_big_endian
Changed disk accounting key sort order so the type tag is the most significant byte, causing same-type keys to sort together. Fixes a mount time regression by allowing startup accounting to skip non-mirrored counter types instead of interleaving them.

### 1.16 - reflink_p_may_update_opts
Added flag to bch_reflink_p indicating whether a reflink pointer may propagate IO path option changes (nr_replicas, compression) back to the indirect extent. Previously option changes could not apply to reflinked data. Initially only set for source extents.

### 1.17 - inode_depth
Added bi_depth to directory inodes recording depth from root. Makes check_directory_structure fsck pass efficient: checking parent has strictly smaller depth guarantees chains terminate at root, instead of following backpointers per directory.

### 1.18 - persistent_inode_cursors
Added persistent on-disk cursors for inode number allocation, avoiding full inodes btree scan from the beginning on each startup to find the next free inode number.

### 1.19 - autofix_errors
Changed default error action for pre-existing filesystems to fix_safe, matching the default for newly created filesystems. Makes self-healing the default behavior upon upgrade.

### 1.20 - directory_size
Added directory size accounting. Parent directory size is updated when subdirectory entries are created or deleted. Fsck automatically recalculates on upgrade.

### 1.21 - cached_backpointers
Cached pointers now have backpointers. Enables killing cached pointers in the bucket_invalidate path when invalidating or reusing buckets, instead of leaving stale pointers for gc_gens garbage collection (which requires a full metadata scan).

### 1.22 - stripe_backpointers
Stripes now have backpointers. Needed for scrub (stripe checksums verified separately from extents within the stripe) and for efficient evacuate and repair paths.

### 1.23 - stripe_lru
Added persistent LRU for stripes ordered by number of empty blocks (reuse priority). Replaces the in-memory stripes heap, eliminating the need to read all stripes at startup.

### 1.24 - casefolding
Implemented case-insensitive filename lookups using the same UTF-8 lowering/normalization as ext4 and f2fs. Uses a version number to gate feature usage plus an old-style incompat feature bit so kernels without casefolding support can detect it.

### 1.25 - extent_flags
Added extent field for bitflags applying to entire extents. Incompatible (unknown extent fields cannot be parsed). Also adds BCH_EXTENT_poisoned flag marking data as known-bad (e.g., after checksum error required new checksum) so reads return errors.

### 1.26 - snapshot_deletion_v2
Speeds up snapshot deletion by only processing extents/dirents/xattrs btrees if an inode for the snapshot ID was present. Adds "deleted" flag to snapshot IDs instead of fully deleting them, so "key in missing snapshot" errors can trigger automatic repair.

### 1.27 - fast_device_removal
Uses backpointers to find pointers to removed devices instead of full metadata scan. Requires BCH_SB_MEMBER_DELETED_UUID (incompatible). Device indexes not reused until fsck verifies no remaining pointers, since backpointers are not fully trusted.

### 1.28 - inode_has_case_insensitive
Added BCH_INODE_has_case_insensitive flag tracking whether a directory has case-insensitive descendants, so overlayfs can disallow mounting. Cheap upgrade ensures the flag is set on existing inodes. Create, rename, and fssetxattr maintain the flag.

### 1.29 - extent_snapshot_whiteouts
Added KEY_TYPE_extent_whiteout (incompat feature). The existing KEY_TYPE_whiteout breaks the monotonically increasing start-position invariant for extents needed for lookup termination; the new type preserves it.

### 1.30 - 31bit_dirent_offset
32-bit programs need 31-bit readdir offsets. When inodes_32bit is enabled, dirent hashes are masked to 31 bits.

### 1.31 - btree_node_accounting
Added btree node number accounting: total btree nodes (ignoring replication) and non-leaf node count, for recovery progress reporting. Mandatory upgrade/downgrade.

### 1.32 - sb_field_extent_type_u64s
Added superblock section recording sizes of extent entry types. Allows adding new extent entry types without breaking backwards compatibility, since older code can skip unknown types using recorded sizes.

### 1.33 - reconcile
Reworked rebalance into "reconcile." bch_extent_rebalance.needs_rb flags now track all reasons an extent needs rebalance processing. Split single rebalance_work counter into separate counters for different io path options (compression, checksum, replicas, erasure_code, etc.). Centralized logic in bch2_bkey_set_needs_rebalance().

### 1.34 - extented_key_type_error
Extended KEY_TYPE_error with a bch_error structure containing a typed error code (err field) and specific reasons (unknown, device_removed, double_allocation). Previously error keys had an empty value; now they carry structured information.

### 1.35 - bucket_stripe_index
Replaced direct alloc key stripe pointer with a separate btree for bucket-to-stripe references, allowing buckets to be members of multiple stripes.

### 1.36 - no_sb_user_data_replicas
Stopped storing replicas entries for user data in the superblock. On a 50-drive filesystem with 3x replication, the replicas section would contain 50^3 entries. Checking is deferred to accounting_read time.

## Downgrade Support

The downgrade table in `sb/downgrade.c` specifies recovery passes and errors to silence when downgrading. Not all versions support downgrade; downgrading typically requires fsck to regenerate metadata in older format.

## Version Checks in Code

```c
if (version >= bcachefs_metadata_version_foo)
    // new format
else
    // old format / compatibility
```

Key files: `btree/read.c`, `btree/bkey_methods.c`, `sb/downgrade.c`, `init/recovery.c`.
