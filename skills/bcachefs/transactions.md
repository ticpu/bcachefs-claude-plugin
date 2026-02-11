# bcachefs Transaction System

## Files

- `btree/types.h` - Core structs: `btree_trans`, `btree_path`, `btree_iter`, `btree_insert_entry`
- `btree/iter.h` / `iter.c` - Iterator/path management, `bch2_trans_begin()`, restart macros
- `btree/update.h` / `update.c` - `bch2_trans_update()`, extent updates, key cache coordination
- `btree/commit.c` - `__bch2_trans_commit()`, trigger phases, lock acquisition
- `btree/locking.h` / `locking.c` - Lock ordering, deadlock detection
- `btree/interior.h` / `interior.c` - Node splits/merges, async interior updates
- `init/errcode.h` - All `BCH_ERR_transaction_restart_*` codes

## Core Structures

### `struct btree_trans` (types.h:508-594)

One per thread doing btree operations.

- `paths` / `sorted` / `updates` - Dynamic arrays for paths and pending updates
- `mem` / `mem_top` / `mem_bytes` - Transaction-scoped allocator (reset on restart!)
- `restart_count` / `restarted` - Restart tracking
- `journal_res` / `disk_res` - Journal and disk space reservations
- `hooks` - Transactional commit hooks

### `struct btree_path` (types.h:326-358)

Internal locking primitive. Tracks locks, node pointers, sequence numbers.

- `nodes_locked` - Bitmask of held locks (2 bits per level)
- `lock_seq[]` - Per-level sequence numbers for optimistic relocking
- `should_be_locked` - Must stay locked; relock failure triggers restart

### `struct btree_iter` (types.h:371-435)

User-facing API. References a path by index. Multiple iterators can share one path.

### `struct btree_insert_entry` (types.h:437-460)

Pending update in `trans->updates[]`.

- `old_k` / `old_v` - Key being overwritten (for triggers)
- `insert_trigger_run` / `overwrite_trigger_run` - Trigger execution tracking
- `cached` - Key cache vs btree proper

## Path vs Iterator

**Why separate?**

- **Paths** are the locking primitive: track lock state, node pointers, sequence numbers
- **Iterators** are the user API: track intent (flags, snapshot, pos)
- Multiple iterators share paths (copy-on-write via `bch2_btree_path_make_mut()`)
- Paths maintained in sorted order for lock ordering verification

## Transaction Lifecycle

### 1. Begin: `bch2_trans_begin()` (iter.c:3442-3542)

- Resets `mem_top = 0` (invalidates all `bch2_trans_kmalloc()` allocations)
- Clears updates, increments `restart_count`
- Frees unused paths
- Unlocks if holding too long (`BTREE_TRANS_MAX_LOCK_HOLD_TIME_NS = 10ms`)
- If restarted: calls `bch2_btree_path_traverse_all()` to re-traverse active paths

### 2. Iterate: `bch2_btree_iter_peek*()` (iter.c:2400+)

Handles snapshot filtering, journal overlay, key cache integration, pending updates.
Can restart on: relock failure, node change (sequence mismatch), too many paths.

### 3. Update: `bch2_trans_update()` (update.c:494-530)

Records pending update. For extents, handles splitting/merging.
For cached btrees, routes to key cache. Invariant: path must be `should_be_locked`.

### 4. Commit: `__bch2_trans_commit()` (commit.c:1096-1230)

Seven phases:

1. **Pre-validation**: Run transactional triggers, skip noop updates, check RW
2. **Lock acquisition**: Write locks on all update paths, deadlock detection
3. **Space checks**: Verify updates fit in nodes (may trigger split -> restart),
   journal reservation
4. **Triggers**: Atomic triggers (cannot fail), GC triggers if active
5. **Journal writes**: Copy updates to journal reservation
6. **Apply**: Insert keys into leaf nodes or key cache
7. **Cleanup**: Unlock, release reservations

## Six-Locks

Btree nodes use six-locks (shared/intent/exclusive). The intent state allows an
operation that modifies multiple nodes to hold intent locks for its duration,
upgrading to write only for each individual in-memory update. This avoids holding
write locks across entire split or merge operations.

## Lock Ordering

Enforced by `__btree_path_cmp()` (iter.c:49-62):

```
btree_id -> cached -> position -> level
```

Prevents deadlock: all transactions acquire locks in same global order.
`cached` separates key cache from regular btree nodes.

## Deadlock Detection (locking.c:297-417)

`bch2_check_for_deadlock()`:
1. Build lock graph of waiting transactions (max depth 8)
2. Walk held locks, check waiters for cycles
3. Choose victim via `btree_trans_abort_preference()`:
   - Prefer non-`lock_may_not_fail`
   - Prefer non-write-lock waiters
   - Prefer transactions not in `traverse_all`
4. Abort with `BCH_ERR_transaction_restart_would_deadlock`

**Lock sharing optimization** (locking.c:277-293): If another path in same
transaction already holds the lock, just increment refcount.

## Optimistic Concurrency

- **Sequence numbers** (locking.h:89-93): After drop+reacquire, if
  `lock_seq != six_lock_seq(&b->c.lock)` -> node modified -> restart
- **`should_be_locked`**: Paths must stay locked; relock failure -> restart
- **SRCU**: Read transactions hold SRCU read lock protecting nodes from being freed

## Transaction Restarts (errcode.h:207-228)

### Locking Conflicts

- `relock` / `relock_path` / `relock_path_intent` - Failed reacquire after drop
- `would_deadlock` / `would_deadlock_write` - Deadlock detected
- `deadlock_recursion_limit` - Lock graph overflow
- `upgrade` - Lock upgrade failed

### Resource Limits

- `too_many_iters` - Path array exhausted (> `BTREE_ITER_NORMAL_LIMIT = 256`)
- `mem_realloced` - Transaction memory reallocated (invalidates all pointers!)

### Concurrency Races

- `lock_node_reused` - Node freed while we held pointer
- `key_cache_raced` - Key cache entry modified by another transaction
- `split_race` - Node split during traversal
- `lock_root_race` - Root pointer changed

### Nested Helpers

- `nested` - Used by `nested_lockrestart_do()` to signal "succeeded after restart"
- `commit` - Used by `bch2_trans_commit_lazy()` to signal "need actual commit"

## Restart Macros (iter.h:862-902)

### `lockrestart_do(_trans, _do)`

Basic retry loop: `bch2_trans_begin()` -> `_do` -> restart if transaction_restart error.

### `nested_lockrestart_do(_trans, _do)`

For operations within larger transactions. Only calls `bch2_trans_begin()` after
first restart. Returns `transaction_restart_nested` if succeeded after restart.

### `commit_do(_trans, _disk_res, _journal_seq, _flags, _do)`

Combines retry loop with commit: `_do ?: bch2_trans_commit(...)`.

## Triggers

### Transactional (`BTREE_TRIGGER_transactional`) (commit.c:554-600)

- Run before commit, can generate more updates
- Sorted by `btree_trigger_order()`: alloc=U8_MAX, stripes=U8_MAX-1, others=btree_id
- Can fail -> restart transaction
- Example: extent accounting

### Atomic (`BTREE_TRIGGER_atomic`)

- Run after journal reservation (commit point)
- Cannot fail (failure -> emergency read-only)
- Run while holding write locks
- Example: quota, usage counters

**Optimization** (commit.c:505-508): If old and new keys have same trigger,
call once with `BTREE_TRIGGER_insert|BTREE_TRIGGER_overwrite`.

## Write Buffer vs Direct vs Key Cache

| Method | Used for | Check |
|--------|----------|-------|
| Write buffer | High-frequency (accounting, LRU, backpointers) | `btree_type_uses_write_buffer()` |
| Direct btree | Most btrees | Default |
| Key cache | alloc, inodes, logged_ops | `btree_id_cached()` (iter.h:507) |

Key cache keys also exist in btree; flushing involves complex coordination
(update.c:432-456).

## Interior Node Splits

If node full during commit, `bch2_btree_split_leaf()` called (commit.c:944-958).
If split + pending interior updates -> restart with
`transaction_restart_split_with_interior_updates` to avoid nested complexity.

Interior updates applied asynchronously after children written to disk.

## Memory Lifetime

`trans->mem` allocations (via `bch2_trans_kmalloc()`) only valid until next
`bch2_trans_begin()`. On restart, `mem_top = 0` invalidates everything.
Code must save `restart_count` and check `trans_was_restarted()` before using
allocated memory.

## Notable Comments

types.h:110-114 - XXX: proposed optimization to add delete sequence numbers,
avoiding full re-traversal when node modified but position still valid.

commit.c:619-630 - Disabled dynamic fault injection (`#if 0`), noted as TODO.
