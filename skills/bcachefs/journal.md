# bcachefs Journal (WAL)

## Design Overview

Write-ahead log for btree updates. All btree leaf modifications are journalled before
the btree node is written, so btree is always recoverable from journal replay after crash.
Interior btree node updates happen synchronously without the journal.

Ringbuffer of buckets (not necessarily contiguous on disk). Entries never span buckets.
Monotonically increasing 64-bit sequence numbers (`jset->seq`). Journal buckets listed
in superblock (`bch_sb_field_journal` / `bch_sb_field_journal_v2`).

Key sequence tracking:

- `cur_seq` (atomic64 `j->seq`): current open entry
- `seq_ondisk`: last entry confirmed written to disk
- `flushed_seq_ondisk`: last entry with storage flush (FUA) confirmed
- `last_seq`: oldest entry still needed (all older entries reclaimable)
- `last_seq_ondisk`: `last_seq` as confirmed written to disk

## On-Disk Format

### `struct jset` (bcachefs_format.h:1449)

```
csum            bch_csum
magic           __le64          jset_magic(c)
seq             __le64          monotonic sequence number
version         __le32          on-disk format version
flags           __le32          [0:4] csum_type, [4:5] big_endian, [5:6] no_flush
u64s            __le32          size of payload in u64 units
encrypted_start                 encryption boundary
last_seq        __le64          oldest seq still needed (0 if no_flush)
start[]         jset_entry      variable-length payload
```

### `struct jset_entry` (bcachefs_format.h:748)

```
u64s            __le16          payload size in u64s
btree_id        __u8
level           __u8
type            __u8            enum bch_jset_entry_type
pad[3]
start[]         bkey_i / __u64  payload
```

### Entry Types (`BCH_JSET_ENTRY_TYPES`, bcachefs_format.h:1311)

- `btree_keys` (0) - btree leaf key insertions (primary WAL payload)
- `btree_root` (1) - btree root pointer updates
- `prio_ptrs` (2) - obsolete
- `blacklist` (3) - single seq blacklist; prevents btree node updates with that seq from being replayed (for btree nodes written but journal entry not)
- `blacklist_v2` (4) - seq range blacklist (start, end)
- `usage` (5) - only stores max key version now (for encryption nonce); other counters moved to accounting btree
- `data_usage` (6) - **legacy**, unused; moved to accounting btree
- `clock` (7) - read/write clock
- `dev_usage` (8) - **legacy**, unused; moved to accounting btree
- `log` (9) - human-readable log messages
- `overwrite` (10) - records old value during btree update; for debugging (with `journal_transaction_names`) and `journal_rewind`
- `write_buffer_keys` (11) - **phantom entry**: transformed to `btree_keys` before disk write; never actually on disk
- `datetime` (12) - wall-clock timestamp
- `log_bkey` (13) - logged bkey for debugging

### Superblock Journal Section

- `bch_sb_field_journal` (bcachefs_format.h:682): flat array of `__le64 buckets[]`
- `bch_sb_field_journal_v2` (bcachefs_format.h:687): array of `{start, nr}` ranges (compacted)
- `bch_sb_field_journal_seq_blacklist`: array of `{start, end}` blacklist ranges
- Validated/managed in `journal/sb.c`

## Write Path

### Reservation (`journal/journal.h`, `journal/journal.c`)

1. `bch2_journal_res_get()` -> fast path `journal_res_get_fast()`: atomic CAS on
   `journal_res_state.counter` to bump `cur_entry_offset` and refcount
2. Slow path `bch2_journal_res_get_slowpath()`: close current entry, open new one,
   retry with escalating timeouts (1s, then 2x max device latency or 10s)
3. `journal_entry_open()` (journal.c:331): allocates seq, pushes pin FIFO, initializes
   `journal_buf`, sets `cur_entry_u64s`, atomically opens entry via CAS

### Pin and Populate

4. Caller writes keys via `bch2_journal_add_entry()` into reserved space
5. `bch2_journal_res_put()` fills remaining reserved space with empty entries,
   drops buf refcount via `bch2_journal_buf_put()`

### Entry Close and Write Submission

6. Entry closes when: buffer full, `journal_flush_delay` expires (`write_work`),
   or explicit flush request. `__journal_entry_close()` (journal.c:187) sets
   `JOURNAL_ENTRY_CLOSED_VAL`, records `last_seq` and `data->u64s`
7. When refcount hits 0, `bch2_journal_buf_put_final()` calls
   `bch2_journal_do_writes_locked()` which triggers `bch2_journal_write()` via closure

### Write Execution (`journal/write.c`)

8. `bch2_journal_write()` (write.c:700):
   - `bch2_journal_write_pick_flush()`: decide flush vs noflush
   - `bch2_journal_write_prep()`: compact empty entries, flush write_buffer_keys to
     btree write buffer, add btree roots + datetime + super entries
   - `journal_write_alloc()`: select devices, append extent pointers
   - `bch2_journal_write_checksum()`: encrypt + checksum
   - `journal_write_preflush()` -> `journal_write_submit()`: bio submission with
     REQ_FUA (flush writes), REQ_PREFLUSH (first device or separate flush)
   - `journal_write_done()`: update `seq_ondisk`, `flushed_seq_ondisk`,
     `last_seq_ondisk`, wake waiters, trigger reclaim

## Reclaim (`journal/reclaim.c`)

### Pin System

Each journal seq has a `journal_entry_pin_list` with refcount + per-type linked lists
(`unflushed[]`, `flushed[]`). Types: btree levels 0-3, key_cache, other.

- `bch2_journal_pin_set()` / `bch2_journal_pin_copy()` / `bch2_journal_pin_drop()`
- When refcount reaches 0, entry is reclaimable
- `bch2_journal_update_last_seq()` (reclaim.c:352): advances `j->last_seq` past
  entries with zero refcount

### Space Reclaim

`bch2_journal_reclaim_thread()` (reclaim.c:810): background kthread, runs every
`journal_reclaim_delay` ms. Calls `journal_flush_pins()` to write out dirty btree nodes
that pin old journal entries.

Flush priority: iterates pin types in order (btree3 down to other), flushing unflushed
pins for entries up to `seq_to_flush` (target: keep journal <= 50% full).

### Per-Device Bucket Tracking (`struct journal_device`, types.h:327)

```
discard_idx <= dirty_idx_ondisk <= dirty_idx <= cur_idx
```

- `bucket_seq[]`: max seq written to each bucket
- `sectors_free`: remaining space in current bucket
- Bucket transitions: dirty -> clean (when `last_seq` advances past) -> discarded

### Discards (`bch2_journal_do_discards`, reclaim.c:339)

Issues blkdev_issue_discard for journal buckets between `discard_idx` and
`dirty_idx_ondisk`. Triggers min(4, nr/2) free buckets threshold.

## Sequence Blacklisting (`journal/seq_blacklist.c`)

Prevents reuse of journal sequence numbers that correspond to btree bset updates
that were flushed to disk but never journalled. Without this, recovery could see
partial ordering violations (bset B flushed after bset A, but A's journal entry lost).

- Bsets record max journal seq of updates they contain
- On recovery, if a bset's seq > newest journal seq, the bset is ignored
- The skipped seq is blacklisted in superblock (`bch_sb_field_journal_seq_blacklist`)
- Lookup via eytzinger0 sorted table (`journal_seq_blacklist_table`)
- `bch2_journal_seq_is_blacklisted()`: O(log n) lookup
- `bch2_blacklist_entries_gc()`: removes entries no longer needed

## Journal Replay (`journal/read.c`, `init/recovery.c`)

### Read Phase (`bch2_journal_read`, read.c:1345)

1. Read all journal buckets from all devices in parallel (closure per device)
2. `journal_read_bucket()`: sequential scan, validate magic/version/checksum
3. `journal_entry_add()`: deduplicate across replicas, keep entry with good checksum,
   maintain `genradix` indexed by seq
4. Find most recent flush entry; ignore newer noflush entries (will be blacklisted)
5. Determine replay range: `last_seq` from newest flush entry to that entry's seq

### Replay Phase (`bch2_journal_replay`, recovery.c:368)

1. Replay accounting keys first (write buffer can't flush accounting until done)
2. Attempt sorted replay (better btree locality), fall back to journal-order for
   entries that would cause deadlock
3. `replay_now_at()`: advances `j->replay_journal_seq`, unpins completed entries
4. After replay: flush any repair keys, set `JOURNAL_replay_done`

## Watermarks and Thresholds (`reclaim.c:69`)

`bch2_journal_set_watermark()` sets `j->watermark`:

- `BCH_WATERMARK_stripe` (normal): clean space > 25% of total
- `BCH_WATERMARK_reclaim` (restricted): clean space <= 25%, or pin FIFO < 25% free,
  or btree write buffer full

When watermark is `reclaim`, only reclaim-priority reservations succeed; normal btree
updates block until space is freed.

`JOURNAL_may_skip_flush` set when clean_ondisk space is healthy relative to clean space.

## Flush Mechanism

### Noflush Writes

Feature `BCH_FEATURE_journal_no_flush`: journal entries can be written without
storage-level flush (no REQ_FUA/REQ_PREFLUSH). `JSET_NO_FLUSH` flag in jset.
`last_seq` is zeroed in noflush entries. Controlled by `journal_flush_delay` option
and `JOURNAL_may_skip_flush` flag.

### Separate Flush

When `nr_rw_members > 1`, flush is issued as separate bio to all RW devices before
the journal data write (`journal_write_preflush`, write.c:459). This parallelizes
the flush across devices.

## Multi-Device Journal

Journal entries are replicated across `metadata_replicas` devices. `journal_write_alloc()`
(write.c:107) selects devices sorted by stripe balance, appends `bch_extent_ptr` for each.
Device selection respects `metadata_target` / `foreground_target`. Falls back to all
devices if target devices insufficient.

Each device maintains independent `journal_device` state with its own ringbuffer indices.
Per-device bios submitted in parallel via closure.

## Key Data Structures

### `struct journal` (types.h:181) - embedded in `struct bch_fs`

- `reservations`: atomic `journal_res_state` (cur_entry_offset + idx + 4x buf refcounts)
- `buf[JOURNAL_BUF_NR]` (16 buffers): journal entries in flight
- `pin`: FIFO of `journal_entry_pin_list`, one per active seq
- `space[journal_space_nr]`: discarded, clean_ondisk, clean, total
- `watermark`, `cur_entry_u64s`, `cur_entry_sectors`, `cur_entry_error`

### `union journal_res_state` (types.h:102)

Packed atomic64: `cur_entry_offset:22, idx:2, buf0_count:10, buf1_count:10, buf2_count:10, buf3_count:10`

Sentinel values: `JOURNAL_ENTRY_CLOSED_VAL`, `JOURNAL_ENTRY_BLOCKED_VAL`, `JOURNAL_ENTRY_ERROR_VAL`

### `struct journal_buf` (types.h:32)

Per-entry buffer: `data` (jset*), `sectors`, `buf_size`, `last_seq`, flags
(`noflush`, `must_flush`, `write_started`, `write_allocated`, `write_done`, `empty`)

### `struct journal_device` (types.h:327) - embedded in `struct bch_dev`

Per-device: `bucket_seq[]`, `buckets[]`, `sectors_free`, ring indices
(`discard_idx`, `dirty_idx_ondisk`, `dirty_idx`, `cur_idx`), `bio[JOURNAL_BUF_NR]`

### `struct journal_entry_pin` (types.h:89)

Linked-list node with `flush` callback and `seq`. Flush callbacks:
`bch2_btree_node_flush0/1`, `bch2_btree_key_cache_journal_flush`

### Size Limits

- `JOURNAL_ENTRY_SIZE_MIN`: 64 KiB
- `JOURNAL_ENTRY_SIZE_MAX`: 4 MiB
- `JOURNAL_SEQ_MAX`: 2^56 - 1 (upper 8 bits reserved for write buffer)
- `BCH_JOURNAL_BUCKETS_MIN`: 8

## Journal Rewind (`journal_rewind` mount option)

Disaster recovery: rolls the filesystem back to a specified journal sequence
number by discarding all leaf-level btree updates to non-allocator btrees at
or above that sequence. Automatically triggers full fsck to rebuild allocation
metadata.

- Set via `journal_rewind=<seq>` mount option (opts.h:391, `u64`)
- `journal/read.c:148-149`: truncates `last_seq` to rewind point
- `journal/read.c:185`: retains log entries and overwrites during replay
- `journal/read.c:1447-1449`: emits "rewinding from [seq]" message
- Only leaf node updates to non-alloc btrees are rewound (interior btree
  updates are structural and not desirable to revert)
- Data in buckets whose generation was incremented and overwritten after the
  target sequence is unrecoverable
- `list_journal -b alloc` can identify when specific buckets were recycled
- Rewind window bounded by journal size: `last_seq` to `seq`
- Requires `journal_transaction_names` (default on) for meaningful diagnosis

## Journal Debug (sysfs)

`/sys/fs/bcachefs/<uuid>/internal/journal_debug` reports runtime journal state:
dirty entries / capacity, sequence numbers (`seq`, `seq_ondisk`, `last_seq`,
`last_seq_ondisk`, `flushed_seq_ondisk`), watermark level, flush/noflush write
counts, average write size, reclaim status, current entry state.
