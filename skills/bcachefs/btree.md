# bcachefs B-tree

## Files

- `btree/types.h` - core types: `btree`, `btree_path`, `btree_iter`, `btree_trans`
- `btree/iter.h` - iterator interface, traversal
- `btree/update.h` - `bch2_trans_update()`, `bch2_trans_commit()`
- `btree/interior.h` - `btree_update` struct for node splits/merges
- `btree/locking.h` - lock ordering, six-lock integration
- `btree/cache.h` - node cache (rhashtable), LRU eviction
- `btree/key_cache.h` - leaf-level key cache, dirty tracking
- `btree/write_buffer.h` - deferred writes for accounting/LRU/backpointers
- `btree/bset.h` - auxiliary search trees, key packing, eytzinger layout
- `bcachefs_format.h` - on-disk key/node format

## Key Position (bpos)

```c
struct bpos { u64 inode; u64 offset; u32 snapshot; };
```
Three-dimensional sort: inode -> offset -> snapshot.
For extents, offset points to END of extent.

## Key Format (bkey)

```c
struct bkey {
    u8  u64s;           // key+value size in u64s
    u8  format:7;       // 0=packed (per-node format), 1=unpacked
    u8  needs_whiteout:1;
    u8  type;           // KEY_TYPE_* enum
    struct bversion bversion; // 96-bit (hi:32, lo:64)
    u32 size;           // extent size in sectors
    struct bpos p;
};
```

Keys packed per-node using `bkey_format` (variable bit-widths per field, 30-50% savings).

## Node Structure

- Large COW nodes (128K-256K), `BCH_SB_BTREE_NODE_SIZE` in superblock
- Up to 3 bsets per node (`MAX_BSETS`): log-structured appends, compacted in memory
- Auxiliary search tree per bset: eytzinger layout, cache-line aligned, compressed keys
- `struct btree`: `data` (on-disk), `set[3]` (bset_tree), `c.level` (0=leaf), `c.btree_id`, `c.lock` (six_lock)

## Iteration

- `struct btree_iter`: tied to `btree_trans`, references a `btree_path`
- `btree_path`: per-level state (`l[BTREE_MAX_DEPTH]`) with node pointers and lock sequences
- Flags: `BTREE_ITER_slots`, `BTREE_ITER_intent`, `BTREE_ITER_is_extents`, `BTREE_ITER_all_snapshots`, `BTREE_ITER_cached`, `BTREE_ITER_prefetch`

## Transactions (btree_trans)

- `bch2_trans_begin()` / `bch2_trans_commit()`
- Collects updates in memory, commits atomically
- Journal reservation + disk reservation + write locks (ordered)
- Optimistic concurrency: restarts on lock conflicts (`transaction_restart_nested`)
- Triggers: transactional (can generate more updates), atomic (journal lock held)

## Locking

- six-lock: read / intent / write (intent blocks other intents)
- Lock ordering: btree_id -> cached -> position -> level (inverted for deadlock avoidance)
- Lock sequence numbers for optimistic relocking after drop

## Node Splits (btree/interior.h: `struct btree_update`)

1. Allocate new leaf nodes
2. Redistribute keys
3. Update parent pointers in-memory
4. Wait for new leaves to write to disk (write_blocked_list)
5. Write parent (makes new nodes reachable)
6. Free old nodes after parent durable

Merge triggered when node < 1/3 full.

## Write Buffer (btree/write_buffer.h)

Defers writes to certain btrees (accounting, LRU, backpointers).
Two-level: `inc` (incoming) + `flushing`. Flush threshold: size/4, must-wait: size*3/4.

## Key Cache (btree/key_cache.h)

Caches leaf keys separately. RCU-protected hash table. Dirty thresholds:
soft (1024 + nr_keys/2), hard (4096 + nr_keys*3/4).
