# bcachefs Architecture

## Core Philosophy

bcachefs is a filesystem built on top of a relational-database-like btree layer.
Unlike traditional inode-centric filesystems, separate btrees store different data
types (extents, inodes, dirents, xattrs, etc.). Descended from bcache (block cache).

## Key Design Decisions

- **COW (copy-on-write)**: never overwrites data in place; writes sequentially to
  buckets, then invalidates old buckets
- **Bucket-based allocation**: disk divided into buckets (typically 512K-2MB);
  sequential writes within bucket; copy-GC handles fragmentation
- **Very large btree nodes** (128K-256K): log-structured with multiple bsets per
  node; compacted in memory; shallow trees = few seeks
- **Btree locks never held during IO**: critical for latency; transactions drop
  and retake locks aggressively
- **Journal (WAL)**: records btree updates; btree nodes written lazily; enables
  fast random-update workloads
- **Snapshots via key versioning**: not COW-btree cloning (like btrfs); millions
  of snapshots possible; snapshot ID is part of every btree key position

## 28 Btrees (bcachefs_format.h `BCH_BTREE_IDS()`)

| ID | Name | Snapshot-aware | Notes |
|----|------|---------------|-------|
| 0 | extents | yes | data extent mapping |
| 1 | inodes | yes | inode metadata |
| 2 | dirents | yes | directory entries (hash-based) |
| 3 | xattrs | yes | extended attributes (hash-based) |
| 4 | alloc | no | bucket allocation metadata |
| 5 | quotas | no | quota counters (shared limits, not snapshot-aware) |
| 6 | stripes | no | stripe metadata for erasure-coded data |
| 7 | reflink | no | indirect (shared) extents for reflinks |
| 8 | subvolumes | no | subvolume metadata |
| 9 | snapshots | no | snapshot metadata |
| 10 | lru | no | LRU eviction (write-buffered) |
| 11 | freespace | no | free space tracking |
| 12 | need_discard | no | discard queue |
| 13 | backpointers | no | reverse extent/metadata pointers (write-buffered) |
| 14 | bucket_gens | no | bucket generation tracking |
| 15 | snapshot_trees | no | snapshot tree roots |
| 16 | deleted_inodes | no | inodes pending deletion (write-buffered) |
| 17 | logged_ops | no | multi-transaction operations (sagas) |
| 18 | reconcile_work | no | reconcile work queue (write-buffered) |
| 19 | subvolume_children | no | subvolume parent-child relationships |
| 20 | accounting | no | disk accounting (write-buffered) |
| 21-25 | reconcile_* | no | reconcile hipri/pending/scan/phys btrees |
| 26 | bucket_to_stripe | no | bucket to stripe multi-mapping |
| 27 | stripe_backpointers | no | stripe backpointers for repair (write-buffered) |

## Multi-Device

- Allocator stripes across devices, biasing toward more free space
- Per-device IO latency tracking; reads directed to fastest device
- Replication: `data_replicas` / `metadata_replicas` options; any extent on any device set
- Erasure coding: Reed-Solomon on whole buckets (not extents); COW eliminates write hole
- Device labels/targets: `foreground_target`, `metadata_target`, `background_target`, `promote_target`
- Caching: `durability=0` devices act as cache; writeback or writearound modes
