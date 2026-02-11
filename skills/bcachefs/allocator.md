# bcachefs Allocator, Journal, Recovery

## Allocator

### Files

- `alloc/types.h` - watermarks, open_bucket, write_point
- `alloc/foreground.c` - `bch2_bucket_alloc()`, `bch2_bucket_alloc_set()`
- `alloc/background.c` - persistent alloc btree, bucket state machine
- `alloc/buckets.h` / `buckets.c` - usage tracking, `dev_ptr_stale()`, disk reservations
- `alloc/format.h` - on-disk `bch_alloc_v4`

### Watermark Levels (allocation priority, highest to lowest)

`stripe` -> `normal` -> `copygc` -> `btree` -> `reclaim`

### Open Buckets (`OPEN_BUCKETS_COUNT = 1024`)

`struct open_bucket`: bucket being actively written. Tracks `sectors_free`, `gen`.
Hash table for O(1) lookup by dev+bucket. Prevents double-allocation.

### Write Points (`WRITE_POINT_MAX = 32`)

`struct write_point`: coalesces writes to multiple devices for replication.
State machine: stopped -> waiting_io -> waiting_work -> runnable -> running.

### Bucket State Machine

Free -> Allocated (open bucket) -> Dirty (data written) -> Clean (btree flushed) -> Discarded -> Free

### bch_alloc_v4 (alloc/format.h)

- `gen` / `oldest_gen`: generation numbers (prevent stale pointers; also used for
  bootstrapping allocation info during journal replay and repairing certain
  corruption that would otherwise be unrecoverable)
- `data_type`: free, sb, journal, btree, user, cached, parity, stripe_head
- `dirty_sectors` / `cached_sectors` / `stripe_sectors`
- `io_time[2]`: read/write timestamps
- `journal_seq_nonempty` / `journal_seq_empty`
- Flags: `NEED_DISCARD`, `NEED_INC_GEN`, `BACKPOINTERS_START`, `NR_BACKPOINTERS`

### Btree-Based Allocation Infrastructure

The allocator originally used in-memory bucket arrays and scanning threads for
free lists, discard lists, and eviction heaps. These were replaced by btree-based
structures: a freespace btree (ID 11), a discard btree (ID 12), and an LRU btree
(ID 10). The scanning threads are gone entirely; the replacement code is
transactional and trigger-based, easier to debug and reason about. The old threads
were prone to excessive CPU usage when nearly full; the btree approach eliminated
those corner cases.

A fragmentation LRU btree tracks bucket fill levels so copygc can find the most
fragmented buckets directly instead of scanning the entire alloc btree.

### SMR/Zoned Device Support

Buckets map naturally to zones: allocating a bucket means writing to it once in
append-only fashion, then never writing to it again until discarding and reusing
the whole thing — exactly the semantics a zoned device requires.

### Disk Reservations

`bch2_disk_reservation_get()`: pre-allocate sectors. Flags: NOFAIL, PARTIAL.
Reserve factor of 6 for safety margin.

## Journal

### Files

- `journal/types.h` - `struct journal`, journal buffers, pin system
- `journal/journal.h` / `journal.c` - core journal logic
- `journal/write.c` - journal entry writing, replication
- `journal/reclaim.c` - space reclaim, btree flush triggers

### Design

Ringbuffer of buckets. Entries don't span buckets. Monotonic sequence numbers.
Sequence flow: `cur_seq` (active) -> `seq_ondisk` (confirmed) -> `last_seq` (oldest needed).

### Journal Buffers (`JOURNAL_BUF_NR = 16`)

Double-buffered. Flags: `noflush`, `must_flush`, `write_started`, `write_done`.

### Pin System

`journal_entry_pin_list`: tracks objects pinning sequences.
Separate lists by type: btree levels 0-3, key cache, other.
Refcount to 0 = entry reclaimable.

### Space Tracking

Four perspectives: `journal_space_discarded`, `_clean_ondisk`, `_clean`, `_total`.
Device tracking: `bucket_seq[]`, `cur_idx`, `dirty_idx_ondisk`, `discard_idx`.

### Reclaim

Escalation: normal (`BCH_WATERMARK_stripe`) -> low space (`BCH_WATERMARK_reclaim`).
Triggers btree node flushing to free journal entries.

## Recovery

### Files

- `init/recovery.c` - main recovery logic
- `init/passes.h` / `passes_format.h` - recovery pass definitions
- `init/passes_types.h` - `struct bch_fs_recovery`

### Recovery Passes (~48, defined in `BCH_RECOVERY_PASSES()`)

Ordered passes with dependency tracking (bitmask). Key flags:

- `PASS_ALWAYS` - runs every mount
- `PASS_FSCK` - only with fsck
- `PASS_ONLINE` - can run on mounted fs
- `PASS_ALLOC` - allocation-related
- `PASS_SILENT` - no log output
- `PASS_NODEFER` - cannot be deferred

Key passes in order:
`scan_for_btree_nodes` -> `check_topology` -> `accounting_read` -> `alloc_read` ->
`snapshots_read` -> `check_allocations` -> `set_may_go_rw` -> `journal_replay` ->
`check_alloc_info` -> `check_snapshots` -> `check_subvols` -> `check_inodes` ->
`check_extents` -> `check_dirents` -> `check_xattrs` -> `check_root` ->
`check_directory_structure` -> `check_nlinks` -> `delete_dead_inodes`

### Superblock tracks: `recovery_passes_required`, `btrees_lost_data`, `errors_silent`

## Data IO

### Read Path (data/read.c: `struct bch_read_bio`)

1. Extent lookup in btree
2. Pointer selection: device congestion (EWMA latency), availability
3. Bio submission to selected device
4. Retry on other replicas if failure; self-healing writes on correctable errors
5. Cache promotion if `promote_target` set

### Write Path (data/write.c: `struct bch_write_op`)

1. Allocate via open buckets/write points
2. Optional compression, checksum, EC encoding
3. Parallel writes to replicas
4. Journal entry for extent update
5. Btree index update

`bch2_write_op_init()`: sets csum_type, compression_opt, watermark, targets.

### Data Movement (data/move.c: `struct moving_context`)

`move_pred_fn` callback selects extents to move.
`bch2_move_extent()` -> read -> write to new location -> btree update.
Rate-limited via `bch_ratelimit`.

### Copy GC (data/copygc.c)

Evacuates fragmented buckets. Selects by LRU fragmentation index.
Skips open/in-use buckets.
