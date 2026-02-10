# bcachefs On-Disk Format

## Files

- `bcachefs_format.h` - canonical on-disk format reference
- `bcachefs_ioctl.h` - ioctl interface
- `data/extents_format.h` - extent pointer format
- `fs/inode_format.h`, `fs/dirent_format.h` - inode/dirent format
- `alloc/format.h` - allocation metadata format
- `sb/members_format.h` - device member format

## Superblock (struct bch_sb)

Located 4KB from device start. Redundant copies.
Key fields: `uuid` (internal), `user_uuid` (visible), `label[32]`, `block_size`,
`nr_devices`, `dev_idx`, `seq`, `version`/`version_min`.

### SB Flags (BCH_SB_* bitmasks on flags[0-6])

Btree node size, GC/root reserves, checksum types (data/metadata),
replica counts, POSIX ACL, quotas, error flags, string hash type,
compression, replication targets, erasure coding, journal settings,
write buffer size, extent BP_SHIFT.

### SB Variable Fields (bch_sb_field types)

`journal`/`journal_v2`, `members_v1`/`v2`, `crypt`, `replicas`,
`quota`, `disk_groups`, `clean` (btree roots on clean shutdown),
`counters`, `errors`, `recovery_passes`, `extent_type_u64s`.

## Key Types (37, enum bch_bkey_type)

| Type | ID | Notes |
|------|----|-------|
| deleted | 0 | |
| whiteout | 1 | hash table deletion |
| error | 2 | data loss marker |
| btree_ptr | 5 | interior node pointer |
| btree_ptr_v2 | 18 | + seq, sectors_written, min_key |
| extent | 6 | data extent |
| reservation | 7 | allocated but unwritten |
| inode/v2/v3 | 8/23/29 | inode metadata |
| dirent | 10 | directory entry |
| xattr | 11 | extended attribute |
| alloc/v2/v3/v4 | 12/20/24/27 | bucket metadata |
| quota | 13 | |
| stripe | 14 | erasure code |
| reflink_p | 15 | indirect extent pointer |
| reflink_v | 16 | indirect extent value |
| inline_data | 17 | inline file data |
| subvolume | 21 | |
| snapshot | 22 | |
| lru | 26 | |
| backpointer | 28 | reverse pointer |
| accounting | 34 | |

## Extent Pointers (data/extents_format.h)

Entry types: `ptr`(0), `crc32`(1), `crc64`(2), `crc128`(3), `stripe_ptr`(4),
`rebalance_v1`(5), `flags`(6), `reconcile`(7).

Backpointer flags: `BACKPOINTER_RECONCILE_PHYS`(0-1), `BACKPOINTER_ERASURE_CODED`(2), `BACKPOINTER_STRIPE_PTR`(3).

`bch_extent_ptr`: `dev`(8b), `offset`(44b = 8PB max), `gen`(8b),
`unwritten`(1b), `cached`(1b).

Compression types: none, lz4_old, gzip, lz4, zstd, incompressible.

## Device Members (sb/members_format.h: struct bch_member)

`uuid`, `nbuckets`, `first_bucket`, `bucket_size`, `last_mount`,
`flags` (STATE, DISCARD, DATA_ALLOWED, GROUP, DURABILITY, ROTATIONAL),
`iops[4]`, `errors[3]`, `device_name[16]`, `device_model[64]`.

States: rw(0), ro(1), evacuating(2, durability=0, triggers reconcile to write new replicas), spare(3). Max 64 devices.

## Journal Format (struct jset)

`csum`, `magic` (XOR'd UUID), `seq`, `last_seq` (oldest dirty), entries[].
Entry types: `btree_keys`, `btree_root`, `prio_ptrs` (obsolete), `blacklist`/`v2`,
`usage` (max key version only, other counters moved to accounting btree),
`data_usage` (legacy), `clock`, `dev_usage` (legacy), `log`, `overwrite`,
`write_buffer_keys` (phantom, transformed to btree_keys before disk),
`datetime`, `log_bkey`.

## Btree Node Format (struct btree_node)

`csum`, `magic`, `flags` (ID, LEVEL, SEQ), `min_key`/`max_key`,
`format` (bkey_format for packing), `keys` (first bset).
Appended entries via `btree_node_entry`.

## Metadata Versions

`Version = (major << 10) | minor`. Current: Major 1, minor 0-36.
`version` (current), `version_min` (oldest data), features/compat bits.

## Ioctls (bcachefs_ioctl.h, magic 0xbc)

### Device Management

`DISK_ADD`/`REMOVE`/`ONLINE`/`OFFLINE`/`SET_STATE`/`GET_IDX`/`RESIZE`/`RESIZE_JOURNAL`

### Query

`QUERY_UUID`, `DEV_USAGE_V2`, `QUERY_ACCOUNTING`, `QUERY_COUNTERS`, `READ_SUPER`

### Data Operations

`BCH_IOCTL_DATA` with ops: `scrub`, `rereplicate`, `migrate`,
`rewrite_old_nodes`, `drop_extra_replicas`. Returns fd for progress/cancel.

### Subvolumes

`SUBVOLUME_CREATE`/`DESTROY` (v1 and v2), `SUBVOLUME_LIST`

### Snapshot Trees

`SNAPSHOT_TREE` - queries full snapshot tree with per-node disk accounting

### Fsck

`FSCK_OFFLINE`, `FSCK_ONLINE`

### Force flags

`BCH_FORCE_IF_DATA_LOST`, `BCH_FORCE_IF_METADATA_LOST`, `BCH_FORCE_IF_DEGRADED`
