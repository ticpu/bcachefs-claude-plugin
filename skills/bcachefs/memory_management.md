# bcachefs Memory Management & Shrinker

## Btree Node Cache (btree/cache.c, types.h)

### Core Structure: `bch_fs_btree_cache` (types.h:177-226)

- `struct rhashtable table` - Hash table for cached nodes
- `struct list_head freeable` - Nodes ready to be freed
- `freed_pcpu` / `freed_nonpcpu` - Freed node lists by reader type
- `struct btree_cache_list live[2]` - live[0]: normal, live[1]: pinned
- `atomic_long_t nr_dirty` - Dirty node count
- `size_t nr_reserve` - Reserve pool (never go below)
- `struct task_struct *alloc_lock` - Cannibalization lock (single-threaded)
- `struct closure_waitlist alloc_wait` - Cannibalization waiters

### Three Shrinkers (seeks: lower = reclaim sooner)

| Shrinker | seeks | Target |
|----------|-------|--------|
| key_cache (key_cache.c:816) | 0 | Most aggressive |
| btree live[0] (cache.c:706) | 2 | Normal nodes |
| btree live[1] (cache.c:716) | 8 | Pinned nodes |

### Btree Cache Scan Logic (cache.c:496-604)

- Protected by `PF_MEMALLOC_NOFS`
- Keeps 3 nodes on freeable list minimum (avoid system allocator during splits)
- Accessed-bit LRU: clear BTREE_NODE_accessed on first pass, evict on second
- Triggers dirty writes when `nr_dirty + nr_to_scan >= list->nr * 3/4`
- 10 not-freed reasons tracked in `bc->not_freed[]` (types.h:141-151):
  cache_reserve, lock_intent, lock_write, dirty, read_in_flight,
  write_in_flight, noevict, write_blocked, will_make_reachable, access_bit

### Reserve Calculation: `bch2_recalc_btree_reserve()` (cache.c:40)

- Base: 16 nodes, +8 if no root, +8 * min(1, level) per btree root

### Node Allocation: `bch2_btree_node_mem_alloc()` (cache.c:815)

Fallback chain:
1. Reuse from `freed_pcpu`/`freed_nonpcpu` (matched by lock type)
2. `GFP_NOWAIT` allocation
3. Drop locks, `GFP_KERNEL`
4. Swap data buffers with freeable nodes
5. Cannibalize if lock held

### Node Data Allocation (cache.c:160-212)

Cursed hack: mm doesn't limit compaction blocking time even with vmalloc fallback.
- `bch2_mm_avoid_compaction` module param (default: true, 0644)
- If reclaim + avoid_compaction: `__vmalloc()` directly (skip kmalloc compaction)
- If GFP_NOWAIT: try kmalloc first (no compaction risk)
- Final fallback: `kvmalloc()` (mm vmalloc can fail for no sane reason on 64-bit)
- Tracks vmalloc count in `bc->nr_vmalloc`

### Aux Data (btree/bset.h:198)

Binary search trees + compiled unpack functions. Size: `DIV_ROUND_UP(btree_buf_max_u64s(b) * 5 + 7, 8)`.

### Cannibalization (cache.c:742-813)

- Single-threaded via `bc->alloc_lock` (CAS)
- `btree_node_cannibalize()` - Scan live[0]+live[1] reverse
- Two passes: first clean (`btree_node_reclaim`), then dirty (`write_and_reclaim`)
- Busy-wait with `WARN_ONCE` if all intent-locked

### Node Pinning (cache.c:243-282)

- `bch2_node_pin()` moves to live[1], `bch2_btree_cache_unpin()` moves all back
- Tracked by `pinned_nodes_start/end`, `pinned_nodes_mask[level]` per btree_id

### Reclaim Accounting Workaround (cache.c:116-124)

Manual `mm_account_reclaimed_pages()` - should be done in slub/vmalloc but
working around slub bug on kmalloc_large() path.

## Key Cache (btree/key_cache.c, key_cache_types.h)

### Structure (key_cache_types.h:7-27)

- `struct rhashtable table`
- `struct rcu_pending pending[2]` - Separate pcpu/non-pcpu deferred frees
- `atomic_long_t nr_keys, nr_dirty`
- Shrinker stats: requested_to_free, freed, skipped_dirty, skipped_accessed, skipped_lock_fail

### Why Two Pending Lists

`btree_uses_pcpu_readers()`: only BTREE_ID_subvolumes uses percpu reader locks.
Lock type set at allocation; reuse from matching list avoids six_lock re-init cost.

### Dirty Thresholds (key_cache.h)

- Soft flush: `1024 + nr_keys/2`
- Hard wait: `4096 + (nr_keys * 3) / 4`
- Resume: `2048 + (nr_keys * 5) / 8`

### Key Cache Shrinker (key_cache.c:673-825)

- `seeks=0, batch=1<<14` (most aggressive shrinker)
- Count: `nr_keys - nr_dirty - 128` (128 headroom to avoid hammering)
- Scan: iterate hash table, skip dirty, accessed-bit LRU

### Per-Key Memory Sizing (key_cache.c:215-228)

- Extra space to reduce transaction restart on realloc: `min(256U, (u64s * 3) / 2)` rounded to power-of-2
- +1 u64 for varint decode overread (7 bytes past buffer end)

### Allocation (key_cache.c:151-205)

1. Dequeue from matching `rcu_pending` list
2. `allocate_dropping_locks()` with GFP_KERNEL
3. Try other pending list
4. `bkey_cached_reuse()` - forcibly evict clean non-accessed entry

## Write Buffer (btree/write_buffer.c, write_buffer_types.h)

### Structure (write_buffer_types.h:51-58)

- `inc` (incoming unflushed) + `flushing` (currently flushing) double buffer
- `sorted` darray for flush ordering
- `accounting` darray with eytzinger lookup for counter accumulation

### Thresholds (write_buffer.h)

- Should flush: `inc + flushing > size / 4`
- Must wait: `inc > size * 3 / 4`

### Journal Pin Management (write_buffer.c:245-299)

Keys in write buffer don't have individual journal pins until flushed.
Buffer-wide pin prevents journal reclaim. `move_keys_from_inc_to_flushing()`
updates pins: adds pin for oldest seq in flushing, drops/updates inc pin.

### Accounting Accumulation (write_buffer.c:768-892)

During journal replay before accounting ready, accounting keys can't flush.
Updates to same counters merge via eytzinger-sorted darray to prevent overflow.

## Bio Bounce Buffer Pool (data/read.c:1681-1720)

- `BIO_BOUNCE_BUF_POOL_LEN = PAGE_SIZE << PAGE_ALLOC_COSTLY_ORDER` (32KB chunks)
- Pool size: `max(btree_node_size, encoded_extent_max) / chunk_size`
- Custom allocator: `__get_free_pages(gfp, PAGE_ALLOC_COSTLY_ORDER)`
- High-order pages reduce TLB pressure, fewer bio_vec entries
- `bio_bounce_pages_lock` mutex protects mempool access

### Bio Sets

| Field | Location | Purpose |
|-------|----------|---------|
| `bio_read` | bcachefs.h:859 | Read path |
| `bio_read_split` | bcachefs.h:860 | Split reads |
| `bio_write` | bcachefs.h:861 | Write path |
| `replica_set` | bcachefs.h:862 | Replica writes |
| `btree.bio` | btree/types.h:720 | Btree node IO |
| `writepage_bioset` | vfs/types.h:11 | VFS writeback |
| `dio_write_bioset` | vfs/types.h:12 | Direct IO write |
| `dio_read_bioset` | vfs/types.h:13 | Direct IO read |
| `nocow_flush_bioset` | vfs/types.h:14 | Nocow flush |
| `ec.block_bioset` | data/ec/types.h:49 | Erasure coding |

All initialized with `BIOSET_NEED_BVECS`.

### Usage (data/write.c)

- `bch2_bio_alloc_pages_pool()` (write.c:144) - Try direct alloc, fall back to mempool
- `bch2_bio_free_pages_pool()` (write.c:114) - Return to pool if size matches chunk

## Compression Mempools (data/compress.c:686-712)

Lazily initialized per compression type actually used (sb.features):

- `compress.bounce[READ]` - Decompression buffer (1 entry, encoded_extent_max)
- `compress.bounce[WRITE]` - Compression buffer (1 entry, encoded_extent_max)
- `compress.workspace[type]` - Per-algorithm workspace (zstd largest)

## Other Mempools

| Pool | Location | Size | Purpose |
|------|----------|------|---------|
| `btree.trans.pool` | btree/iter.c:3920 | sizeof(btree_trans) | Transaction structs |
| `btree.trans.malloc_pool` | btree/iter.c:3921 | 64KB (BTREE_TRANS_MEM_MAX) | Temp transaction allocs |
| `btree.fill_iter` | btree/init.c:54 | Calculated | Btree fill iterator |
| `btree.bounce_pool` | btree/init.c:59 | btree_sectors << 9 | Compress before write |
| `btree.interior_updates.pool` | btree/interior.c:2854 | sizeof(btree_update) | Split/merge structs |

All use min_nr=1: emergency reserves, not for reducing allocation frequency.

## Module Parameters & Static Branches

- `bch2_mm_avoid_compaction` (bool, 0644, default true) - Skip kmalloc for btree data
- `bch2_btree_shrinker_disabled` - Static branch to disable btree shrinker (cache.c:512)
- `bch2_verify_btree_ondisk` - Static branch for post-write verification

## Design Patterns

- **kvmalloc preference**: Large allocations use kvmalloc for transparent vmalloc fallback
- **Lock-type-aware caching**: Separate free lists for pcpu vs non-pcpu (subvolumes btree)
- **RCU-deferred reclaim**: Key cache uses rcu_pending for safe lockless access
- **Mempool min_nr=1**: Single-entry emergency reserves, not caching pools
- **Workarounds for MM bugs**: Explicit vmalloc for compaction, manual reclaim accounting
