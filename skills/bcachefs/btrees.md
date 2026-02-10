# bcachefs Btrees Catalog

## Overview

bcachefs uses 28 btrees (BTREE_ID_NR = 28, bcachefs_format.h:532-640). Each btree has specific properties and stores different types of data. Btrees are identified by `enum btree_id`.

## Btree Properties (bcachefs_format.h:524-530)

- `BTREE_IS_extents` - Keys have size field (extent-like)
- `BTREE_IS_snapshots` - Snapshot-aware (uses snapshot field in bpos)
- `BTREE_IS_snapshot_field` - Uses snapshot field differently (not ancestor-aware)
- `BTREE_IS_data` - Tracks user/metadata data
- `BTREE_IS_write_buffer` - Updates batched via write buffer

## Write-Buffered Btrees

Write-buffered btrees batch updates for performance. Updates accumulate in memory and flush periodically, making these updates **unordered and eventually consistent**. See btree/write_buffer.h.

Write-buffered: lru, backpointers, deleted_inodes, reconcile_work, accounting, reconcile_hipri, reconcile_pending, reconcile_work_phys, reconcile_hipri_phys, stripe_backpointers.

## Key Format (struct bpos)

bpos has three fields (bcachefs_format.h:136-160):
- `inode` (u64) - meaning varies per btree
- `offset` (u64) - meaning varies per btree
- `snapshot` (u32) - snapshot ID (0 if not snapshot-aware)

## Btree Catalog

### 0. BTREE_ID_extents

**Purpose**: File data extent mappings. Maps (inode, offset) to data location on disk.

**Flags**: BTREE_IS_extents | BTREE_IS_snapshots | BTREE_IS_data

**Key format**:
- inode: file inode number
- offset: end of extent in sectors (512-byte units)
- snapshot: snapshot ID

**Value types**:
- KEY_TYPE_extent (bch_extent, data/extents_format.h:267) - data pointers, CRCs, compression
- KEY_TYPE_reservation (bch_reservation, data/extents_format.h:295) - preallocated space
- KEY_TYPE_reflink_p (bch_reflink_p, data/reflink_format.h:5) - pointer to reflink btree
- KEY_TYPE_inline_data (bch_inline_data, data/extents_format.h:303) - small files inlined
- KEY_TYPE_whiteout - deletion marker
- KEY_TYPE_extent_whiteout - extent deletion marker
- KEY_TYPE_error (bch_error, bcachefs_format.h:447) - unreadable data
- KEY_TYPE_cookie (bch_cookie, bcachefs_format.h:465) - used by reconcile for option change tracking

**Notable details**: Extent values contain variable-length union bch_extent_entry list: pointers (bch_extent_ptr), CRCs (crc32/crc64/crc128), stripe pointers, reconcile entries. Size field in bkey indicates extent size.

**Related btrees**: reflink (for shared extents), backpointers (reverse mapping), reconcile_work/hipri/pending (background processing).

---

### 1. BTREE_ID_inodes

**Purpose**: File/directory inode metadata.

**Flags**: BTREE_IS_snapshots

**Key format**:
- inode: inode number
- offset: 0
- snapshot: snapshot ID

**Value types**:
- KEY_TYPE_inode (bch_inode, fs/inode_format.h:8) - original version
- KEY_TYPE_inode_v2 (bch_inode_v2, fs/inode_format.h:17) - added journal_seq
- KEY_TYPE_inode_v3 (bch_inode_v3, fs/inode_format.h:27) - inline bi_sectors/bi_size/bi_version
- KEY_TYPE_inode_generation (bch_inode_generation, fs/inode_format.h:42) - generation for reused inodes
- KEY_TYPE_whiteout

**Notable details**: Inode values use variable-length fields (BCH_INODE_FIELDS_v2/v3). Contains mode, uid/gid, timestamps, size, nlink, device, checksum/compression options, targets, subvol info. Flags include sync, immutable, append, unlinked, backptr_untrusted, has_child_snapshot, etc (fs/inode_format.h:136-148).

**Related btrees**: extents (file data), dirents (directory entries), deleted_inodes (cleanup tracking).

---

### 2. BTREE_ID_dirents

**Purpose**: Directory entries. Maps filename hash to inode.

**Flags**: BTREE_IS_snapshots

**Key format**:
- inode: directory inode number
- offset: hash of filename (64-bit, truncated sha1)
- snapshot: snapshot ID

**Value types**:
- KEY_TYPE_dirent (bch_dirent, fs/dirent_format.h:16) - directory entry
- KEY_TYPE_whiteout - for hash collision handling
- KEY_TYPE_hash_whiteout - deletion marker in hash table

**Notable details**: Uses linear probing for hash collisions. Value contains target inode (d_inum) or subvolume IDs (d_child_subvol, d_parent_subvol), file type (d_type), casefold flag, name. DT_SUBVOL (16) for subvolume entries. Max name length BCH_NAME_MAX (512).

**Related btrees**: inodes (target inodes), subvolumes (for DT_SUBVOL entries).

---

### 3. BTREE_ID_xattrs

**Purpose**: Extended attributes.

**Flags**: BTREE_IS_snapshots

**Key format**:
- inode: inode number
- offset: hash of xattr name
- snapshot: snapshot ID

**Value types**:
- KEY_TYPE_xattr (bch_xattr, fs/xattr_format.h:11)
- KEY_TYPE_whiteout
- KEY_TYPE_hash_whiteout
- KEY_TYPE_cookie

**Notable details**: Similar hash table structure to dirents. Value contains type (user/system/trusted/security/posix_acl_access/posix_acl_default), name length, value length, name+value data (x_name_and_value).

**Related btrees**: inodes (owner).

---

### 4. BTREE_ID_alloc

**Purpose**: Per-bucket allocation metadata. Tracks bucket state, generation, sector counts.

**Flags**: 0 (not snapshot-aware, not extent-like)

**Key format**:
- inode: device index
- offset: bucket number
- snapshot: 0

**Value types**:
- KEY_TYPE_alloc_v4 (bch_alloc_v4, alloc/format.h:59) - current version
- KEY_TYPE_alloc_v3 (bch_alloc_v3, alloc/format.h:45)
- KEY_TYPE_alloc_v2 (bch_alloc_v2, alloc/format.h:28)
- KEY_TYPE_alloc (bch_alloc, alloc/format.h:5)

**Notable details**: alloc_v4 fields: gen, oldest_gen, data_type (free/sb/journal/btree/user/cached/parity/stripe), dirty_sectors, cached_sectors, stripe_sectors, io_time[2], journal_seq_nonempty, journal_seq_empty, stripe_refcount, nr_external_backpointers. Flags: NEED_DISCARD, NEED_INC_GEN, BACKPOINTERS_START, NR_BACKPOINTERS.

**Related btrees**: backpointers (reverse pointers), bucket_gens (generation cache), need_discard (discard tracking), freespace (free space index).

---

### 5. BTREE_ID_quotas

**Purpose**: User/group/project quota limits and counters. NOT snapshot-aware (quotas are shared filesystem-wide limits).

**Flags**: 0

**Key format**:
- inode: quota type (QTYP_USR=0, QTYP_GRP=1, QTYP_PRJ=2)
- offset: uid/gid/project id
- snapshot: 0

**Value types**:
- KEY_TYPE_quota (bch_quota, fs/quota_format.h:25)

**Notable details**: Value contains two counters (Q_SPC=space, Q_INO=inodes), each with hardlimit/softlimit.

**Related btrees**: inodes (bi_uid, bi_gid, bi_project), accounting (actual usage tracking).

---

### 6. BTREE_ID_stripes

**Purpose**: Erasure coding stripe metadata.

**Flags**: BTREE_IS_data

**Key format**:
- inode: 0
- offset: stripe index
- snapshot: 0

**Value types**:
- KEY_TYPE_stripe (bch_stripe, data/ec/format.h:7)

**Notable details**: Value contains sectors, algorithm, nr_blocks, nr_redundant, csum_granularity_bits, csum_type, disk_label, followed by variable-length sections: pointers (bch_extent_ptr array), checksums (2D array), block sector counts (u16 array).

**Related btrees**: extents (stripe_ptr entries reference stripes), lru (stripe LRU tracking), bucket_to_stripe (reverse mapping).

---

### 7. BTREE_ID_reflink

**Purpose**: Indirect (shared) extents for reflinks. NOT snapshot-aware (the indirect extents themselves aren't; pointers to them in the extents btree are).

**Flags**: BTREE_IS_extents | BTREE_IS_data

**Key format**:
- inode: 0
- offset: reflink index (end of extent)
- snapshot: 0

**Value types**:
- KEY_TYPE_reflink_v (bch_reflink_v, data/reflink_format.h:25) - shared extent with refcount
- KEY_TYPE_indirect_inline_data (bch_indirect_inline_data, data/reflink_format.h:32) - shared inline data
- KEY_TYPE_error

**Notable details**: reflink_v contains refcount and extent entries (like extent btree). Referenced by KEY_TYPE_reflink_p in extents btree via REFLINK_P_IDX field.

**Related btrees**: extents (reflink_p pointers), backpointers.

---

### 8. BTREE_ID_subvolumes

**Purpose**: Subvolume metadata. Subvolumes are independent filesystem namespaces.

**Flags**: 0

**Key format**:
- inode: 0
- offset: subvolume ID (SUBVOL_POS_MIN=POS(0,1) to SUBVOL_POS_MAX=POS(0,S32_MAX))
- snapshot: 0

**Value types**:
- KEY_TYPE_subvolume (bch_subvolume, snapshots/format.h:9)

**Notable details**: Value contains flags (RO, SNAP, UNLINKED), snapshot, inode (root inode), creation_parent, fs_path_parent, otime. BCACHEFS_ROOT_SUBVOL=1. Subvolumes form tree via creation_parent.

**Related btrees**: snapshots (associated snapshot), subvolume_children (parent-child mapping), dirents (DT_SUBVOL entries).

---

### 9. BTREE_ID_snapshots

**Purpose**: Snapshot tree structure. Each snapshot is a node in the tree.

**Flags**: 0

**Key format**:
- inode: 0
- offset: snapshot ID
- snapshot: 0

**Value types**:
- KEY_TYPE_snapshot (bch_snapshot, snapshots/format.h:35)

**Notable details**: Value contains flags (WILL_DELETE, SUBVOL, DELETED, NO_KEYS), parent, children[2], subvol, tree (snapshot_tree ID), depth, skip[3] (skiplist for ancestor queries), btime. Flags control deletion and lifecycle.

**Related btrees**: snapshot_trees (tree metadata), subvolumes, deleted_inodes (cleanup).

---

### 10. BTREE_ID_lru

**Purpose**: LRU index for read caching and copygc. Tracks bucket access time/fragmentation.

**Flags**: BTREE_IS_write_buffer

**Key format**:
- inode: LRU type (BCH_LRU_read=0, BCH_LRU_fragmentation=1, BCH_LRU_stripes=2)
- offset: time (48 bits, LRU_TIME_MAX) or fragmentation constant
- snapshot: 0

**Value types**:
- KEY_TYPE_set - simple bitset, idx field points to device+bucket or stripe

**Notable details**: Value (bch_lru, alloc/lru_format.h:5) contains idx (bucket or stripe identifier). BCH_LRU_BUCKET_FRAGMENTATION=(1<<16)-1, BCH_LRU_STRIPE_FRAGMENTATION=(1<<16)-2 are special fragmentation values.

**Related btrees**: alloc (buckets being tracked), stripes (for LRU_stripes type).

---

### 11. BTREE_ID_freespace

**Purpose**: Free space index. Extent-based index of free regions for fast allocation.

**Flags**: BTREE_IS_extents

**Key format**:
- inode: device index
- offset: end of free region
- snapshot: 0

**Value types**:
- KEY_TYPE_set - marks free extent

**Notable details**: Allows fast allocation by scanning for free extents without iterating all buckets.

**Related btrees**: alloc (underlying bucket state).

---

### 12. BTREE_ID_need_discard

**Purpose**: Tracks buckets needing discard/TRIM.

**Flags**: 0

**Key format**:
- inode: device index
- offset: bucket number
- snapshot: 0

**Value types**:
- KEY_TYPE_set

**Notable details**: Simple index for discard operations. Related to BCH_ALLOC_V4_NEED_DISCARD flag.

**Related btrees**: alloc (bucket state).

---

### 13. BTREE_ID_backpointers

**Purpose**: Reverse mapping from bucket to btree position. Enables verification and repair.

**Flags**: BTREE_IS_write_buffer

**Key format**:
- inode: device index
- offset: bucket number * bucket_size + bucket_offset
- snapshot: 0

**Value types**:
- KEY_TYPE_backpointer (bch_backpointer, bcachefs_format.h:484)

**Notable details**: Value contains btree_id, level, data_type, bucket_gen, flags (BACKPOINTER_RECONCILE_PHYS), bucket_len, pos (target bpos). Used for fsck and copygc.

**Related btrees**: All data btrees (referenced), alloc (nr_external_backpointers tracking).

---

### 14. BTREE_ID_bucket_gens

**Purpose**: Cached bucket generation numbers. Avoids reading alloc btree for staleness checks.

**Flags**: 0

**Key format**:
- inode: device index
- offset: bucket_base (aligned to KEY_TYPE_BUCKET_GENS_NR=256)
- snapshot: 0

**Value types**:
- KEY_TYPE_bucket_gens (bch_bucket_gens, alloc/format.h:90)

**Notable details**: Each key contains 256 u8 generation numbers (`KEY_TYPE_BUCKET_GENS_NR = 1 << 8 = 256`). Packed representation for fast stale pointer detection.

**Related btrees**: alloc (source of generation info).

---

### 15. BTREE_ID_snapshot_trees

**Purpose**: Snapshot tree metadata. Each tree has a root snapshot and master subvolume.

**Flags**: 0

**Key format**:
- inode: 0
- offset: snapshot tree ID
- snapshot: 0

**Value types**:
- KEY_TYPE_snapshot_tree (bch_snapshot_tree, snapshots/format.h:85)

**Notable details**: Value contains master_subvol, root_snapshot. Provides persistent identifier for each snapshot tree.

**Related btrees**: snapshots (tree field references this), subvolumes (master_subvol).

---

### 16. BTREE_ID_deleted_inodes

**Purpose**: Tracks inodes pending deletion. Used for cleanup after crash.

**Flags**: BTREE_IS_snapshot_field | BTREE_IS_write_buffer

**Key format**:
- inode: inode number
- offset: 0
- snapshot: snapshot ID (but BTREE_IS_snapshot_field means not ancestor-aware)

**Value types**:
- KEY_TYPE_set

**Notable details**: Written when inode.bi_nlink reaches 0 but inode not yet deleted. Recovery pass delete_dead_inodes removes them.

**Related btrees**: inodes (inodes being tracked).

---

### 17. BTREE_ID_logged_ops

**Purpose**: Logged filesystem operations for crash consistency (truncate, finsert, inode allocation).

**Flags**: 0

**Key format**:
- inode: operation type (LOGGED_OPS_INUM_logged_ops=0, LOGGED_OPS_INUM_inode_cursors=1)
- offset: operation-specific ID
- snapshot: 0

**Value types**:
- KEY_TYPE_logged_op_truncate (bch_logged_op_truncate, fs/logged_ops_format.h:10) - truncate in progress
- KEY_TYPE_logged_op_finsert (bch_logged_op_finsert, fs/logged_ops_format.h:24) - fallocate insert range
- KEY_TYPE_inode_alloc_cursor (bch_inode_alloc_cursor, fs/inode_format.h:178) - next inode hint

**Notable details**: logged_op_truncate: subvol, inum, new_i_size. logged_op_finsert: state, subvol, inum, dst_offset, src_offset, pos. inode_alloc_cursor: bits, gen, idx.

**Related btrees**: inodes (operations target inodes), extents (data modifications).

---

### 18. BTREE_ID_reconcile_work

**Purpose**: Pending normal-priority rebalance/copygc work. Bitset of extents needing processing.

**Flags**: BTREE_IS_snapshot_field | BTREE_IS_write_buffer

**Key format**:
- inode: inode number or 0 for reflink
- offset: extent offset
- snapshot: snapshot ID (not ancestor-aware)

**Value types**:
- KEY_TYPE_set - marks extent needing work
- KEY_TYPE_cookie - placeholder

**Notable details**: Works with bch_extent_reconcile entries in extent values. See data/reconcile/format.h for rebalance infrastructure.

**Related btrees**: extents, reflink (extents being tracked), reconcile_hipri (high-priority queue), reconcile_pending (pending work), reconcile_work_phys (physical tracking).

---

### 19. BTREE_ID_subvolume_children

**Purpose**: Parent-to-child subvolume mapping. Reverse index of creation_parent.

**Flags**: 0

**Key format**:
- inode: parent subvolume ID
- offset: child subvolume ID
- snapshot: 0

**Value types**:
- KEY_TYPE_set

**Notable details**: Enables efficient enumeration of child subvolumes.

**Related btrees**: subvolumes (parent-child relationships).

---

### 20. BTREE_ID_accounting

**Purpose**: Filesystem accounting counters (inodes, replicas, compression, snapshots, btrees, etc).

**Flags**: BTREE_IS_snapshot_field | BTREE_IS_write_buffer

**Key format**: Uses struct disk_accounting_pos (alloc/accounting_format.h:230), a union overlaid on bpos. First byte is type, rest varies:
- type: BCH_DISK_ACCOUNTING_* (nr_inodes, persistent_reserved, replicas, dev_data_type, compression, snapshot, btree, rebalance_work, inum, reconcile_work, dev_leaving)
- Fields vary per type (device, data_type, compression type, snapshot ID, btree ID, inode, etc)

**Value types**:
- KEY_TYPE_accounting (bch_accounting, alloc/accounting_format.h:48) - array of up to 3 s64/u64 counters

**Notable details**: Updates are DELTAS not absolute values. Version field tracks journal sequence for replay. Write buffer handles accumulation. Counter meanings vary: dev_data_type=[nr_buckets, live_sectors, fragmentation]; compression=[nr_extents, uncompressed_size, compressed_size]; btree=[sectors, nr_nodes, nr_interior]; etc.

**Related btrees**: All btrees contribute to accounting.

---

### 21. BTREE_ID_reconcile_hipri

**Purpose**: High-priority rebalance work (e.g., evacuating failed devices).

**Flags**: BTREE_IS_snapshot_field | BTREE_IS_write_buffer

**Key format**:
- inode: inode number or 0 for reflink
- offset: extent offset
- snapshot: snapshot ID (not ancestor-aware)

**Value types**:
- KEY_TYPE_set

**Notable details**: Processed before reconcile_work. Set when bch_extent_reconcile.hipri flag is set.

**Related btrees**: extents, reflink, reconcile_work, reconcile_hipri_phys.

---

### 22. BTREE_ID_reconcile_pending

**Purpose**: Rebalance work waiting for space/resources (target full).

**Flags**: BTREE_IS_snapshot_field | BTREE_IS_write_buffer

**Key format**:
- inode: inode number or 0 for reflink
- offset: extent offset
- snapshot: snapshot ID (not ancestor-aware)

**Value types**:
- KEY_TYPE_set

**Notable details**: Set when bch_extent_reconcile.pending flag is set. Retried on device resize/add/label change.

**Related btrees**: extents, reflink, reconcile_work.

---

### 23. BTREE_ID_reconcile_scan

**Purpose**: Reconcile scan cookies and btree node backpointers for background processing.

**Flags**: 0

**Key format**:
- inode: 0 for scan cookies, 1 for btree node backpointers
- offset: varies (cookie type or btree node position)
- snapshot: 0

**Value types**:
- KEY_TYPE_cookie - scan cookie (option change tracking)
- KEY_TYPE_backpointer - btree node needing processing

**Notable details**: Scan cookies (inum=0): RECONCILE_SCAN_COOKIE_fs=0, _metadata=1, _pending=2, _device=32+. KEY_TYPE_cookie entries contain incrementing version numbers to detect option changes. Btree nodes (inum=1) processed before regular data.

**Related btrees**: All btrees (scan targets).

---

### 24. BTREE_ID_reconcile_work_phys

**Purpose**: Physical (device-level) tracking of normal-priority rebalance work.

**Flags**: BTREE_IS_write_buffer

**Key format**:
- inode: device index
- offset: physical offset
- snapshot: 0

**Value types**:
- KEY_TYPE_set

**Notable details**: Tracks work by physical location for device evacuation.

**Related btrees**: reconcile_work, backpointers.

---

### 25. BTREE_ID_reconcile_hipri_phys

**Purpose**: Physical tracking of high-priority rebalance work.

**Flags**: BTREE_IS_write_buffer

**Key format**:
- inode: device index
- offset: physical offset
- snapshot: 0

**Value types**:
- KEY_TYPE_set

**Notable details**: Tracks high-priority work by physical location.

**Related btrees**: reconcile_hipri, backpointers.

---

### 26. BTREE_ID_bucket_to_stripe

**Purpose**: Reverse mapping from bucket to stripe.

**Flags**: 0

**Key format**:
- inode: device index
- offset: bucket number
- snapshot: 0

**Value types**:
- KEY_TYPE_set - idx field is stripe index

**Notable details**: Enables fast lookup of which stripe a bucket belongs to.

**Related btrees**: stripes (referenced), alloc (buckets).

---

### 27. BTREE_ID_stripe_backpointers

**Purpose**: Backpointers indexed by stripe pointers, for pointers to `BCH_SB_MEMBER_INVALID` that point to a stripe. Enables stripe repair for data on invalid/removed devices.

**Flags**: BTREE_IS_write_buffer

**Key format**: Same as backpointers (device, offset, discriminator).

**Value types**:
- KEY_TYPE_backpointer (with `BACKPOINTER_ERASURE_CODED` and `BACKPOINTER_STRIPE_PTR` flags)

**Related btrees**: stripes, backpointers, alloc.

## Btree ID Enum and Ordering

Btrees ordered 0-27 via BCH_BTREE_IDS() macro (bcachefs_format.h:534-646). Each entry: `x(name, nr, flags, valid_key_types)`. Enum btree_id generated at line 642-647.

## Reconstructible Btrees

Some btrees can be reconstructed from others (bcachefs_format.h:1489-1535):
- All alloc-related (btree_id_is_alloc): alloc, backpointers, stripe_backpointers, need_discard, freespace, bucket_gens, lru, accounting, reconcile_work/hipri/pending/scan
- snapshot_trees, deleted_inodes, subvolume_children

Recovery can rebuild these via scan or from other data.

## Maximum Btree Depth

BTREE_MAX_DEPTH = 4 (bcachefs_format.h:1537). Btree nodes up to 4 levels deep.
