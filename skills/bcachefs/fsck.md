# bcachefs Fsck & Recovery

## Self-Healing

Since v1.19 (`autofix_errors`), the default error action is `errors=fix_safe`:
when the filesystem detects a metadata inconsistency during normal operation,
it automatically schedules the relevant `PASS_ONLINE` repair pass to run in
the background. Errors marked `FSCK_AUTOFIX` are fixed without user
intervention. Errors not safe to auto-fix cause emergency read-only with a
message to run fsck and report to developers (so the error can be marked for
self-healing in future versions). This makes bcachefs self-healing for known
error patterns — no periodic fsck scheduling needed.

## Recovery Flow Overview

### Files

- `init/recovery.c` - `bch2_fs_recovery()`, journal replay, `bch2_reconstruct_alloc()`
- `init/passes.c` - pass execution engine, rewind, async/online passes
- `init/passes_format.h` - `BCH_RECOVERY_PASSES()` macro, pass flags, stable IDs
- `init/error.c` - `__bch2_fsck_err()`, `bch2_fsck_err_opt()`, inconsistency handling
- `init/fs.c` - `bch2_fs_start()` entry point, stdio redirect plumbing

### Mount-time Recovery Sequence (`__bch2_fs_recovery()`, recovery.c:641)

1. Read superblock clean section (if clean shutdown) or journal entries (if unclean)
2. `bch2_journal_read()` - scan journal buckets, build `journal_entries` radix tree
3. `bch2_journal_keys_sort()` - extract btree updates from journal into sorted array
4. `journal_replay_early()` - process btree roots, blacklist entries, clock, usage
5. Handle `reconstruct_alloc` mount option or `no_alloc_info` feature flag
6. Blacklist journal sequences after unclean shutdown (skip `JOURNAL_BUF_NR * 4` seqs)
7. `bch2_fs_journal_start()` - initialize journal for new writes
8. `read_btree_roots()` - read all btree root nodes from disk
9. Set `BCH_FS_btree_running` flag
10. `bch2_run_recovery_passes_startup()` - execute ordered recovery passes
11. Post-recovery: flush fixes, verify clean (debug mode runs fsck twice), read quotas
12. Update superblock: clear `btrees_lost_data`, `HAS_ERRORS`, `HAS_TOPOLOGY_ERRORS`

## Recovery Passes

### Pass Flags (`passes_format.h:5`)

- `PASS_SILENT` - no log output
- `PASS_FSCK` - only with `-o fsck`
- `PASS_UNCLEAN` - only after unclean shutdown
- `PASS_ALWAYS` - every mount
- `PASS_ONLINE` - can run on mounted fs
- `PASS_ALLOC` - allocation-related (skipped with `no_alloc_info`)
- `PASS_NODEFER` - cannot be deferred to background
- `PASS_FSCK_ALLOC` - shorthand for `PASS_FSCK|PASS_ALLOC`

### All Passes in Execution Order (`passes_format.h:24`)

Each entry: `name (stable_id, flags, depends_on)`.

**Early / Topology:**

- `recovery_pass_empty` (41, SILENT) - no-op placeholder so `scan_for_btree_nodes` isn't index 0
- `scan_for_btree_nodes` (37) - scan all devices for btree nodes by magic number (`btree/node_scan.c:372`). Multi-threaded per-device. Builds `found_btree_node` array. Never persisted.
- `check_topology` (4, depends: scan) - verify btree node connectivity, min/max key coverage. Repairs by reinserting scanned nodes or allocating fake roots (`btree/check.c:594`).
- `accounting_read` (39, ALWAYS, depends: topology) - read accounting btree into memory
- `alloc_read` (0, ALWAYS) - read alloc btree into memory
- `stripes_read` (1) - read EC stripe info
- `initialize_subvolumes` (2) - create default subvolume if missing
- `snapshots_read` (3, ALWAYS) - populate in-memory snapshot table

**Allocation Verification:**

- `check_allocations` (5, FSCK|ALLOC, depends: topology) - walk all btrees, verify every extent pointer has correct alloc key (`btree/check.c`). Rebuilds accounting.
- `trans_mark_dev_sbs` (6, ALWAYS|SILENT|ALLOC) - mark superblock/journal regions in alloc btree
- `fs_journal_alloc` (7, ALWAYS|SILENT|ALLOC) - ensure journal has allocated buckets

**Go Read-Write:**

- `set_may_go_rw` (8, ALWAYS|SILENT, depends: check_allocations) - set `BCH_FS_may_go_rw`, call `bch2_fs_read_write_early()` if needed. After this, btree updates go to journal instead of journal replay buffer.
- `journal_replay` (9, ALWAYS, depends: set_may_go_rw) - replay journal keys into btrees (`recovery.c:368`).

**Alloc Info Checks (all ONLINE|FSCK|ALLOC, depend: check_allocations):**

- `merge_btree_nodes` (45, ONLINE) - merge underfull btree nodes
- `check_alloc_info` (10) - verify alloc keys match reality: data_type, dirty/cached sectors, need_discard, freespace, bucket_gens (`alloc/check.c:574`)
- `check_lrus` (11) - validate LRU btree entries
- `check_btree_backpointers` (12) - verify backpointer btree entries point to valid extents
- `check_backpointers_to_extents` (13, ONLINE) - backpointers -> extent cross-check
- `check_extents_to_backpointers` (14) - extents -> backpointer cross-check
- `check_alloc_to_lru_refs` (15) - alloc keys reference correct LRU entries
- `fs_freespace_init` (16, ALWAYS|SILENT) - initialize freespace btree from alloc info
- `bucket_gens_init` (17) - initialize bucket_gens btree

**Snapshot Repair:**

- `reconstruct_snapshots` (38) - scan all snapshot-bearing btrees to discover snapshot IDs, rebuild missing snapshot nodes (`snapshots/check_snapshots.c:550`)
- `delete_dead_interior_snapshots` (44) - remove interior snapshot nodes with no live children
- `check_snapshot_trees` (18, ONLINE|FSCK, depends: reconstruct) - validate snapshot_tree entries
- `check_snapshots` (19, ALWAYS|ONLINE|FSCK|NODEFER, depends: reconstruct + trees) - full snapshot btree validation: parent/child links, depth, tree_id
- `check_subvols` (20, ONLINE|FSCK, depends: snapshots) - validate subvolume entries
- `check_subvol_children` (35, ONLINE|FSCK, depends: subvols) - subvolume parent-child links
- `delete_dead_snapshots` (21, ONLINE|FSCK, depends: snapshots) - delete snapshots marked for deletion

**Filesystem Structure Checks:**

- `fs_upgrade_for_subvolumes` (22) - one-time migration
- `check_inodes` (24, FSCK, depends: snapshots) - validate all inodes: flags, fields, backpointer to parent dirent (`fs/check.c:1089`)
- `check_extents` (25, FSCK, depends: inodes) - validate extent keys: inode exists, snapshot valid, no overlaps (`fs/check_extents.c:419`)
- `check_indirect_extents` (26, ONLINE|FSCK, depends: snapshots) - validate reflink indirect extents
- `check_dirents` (27, FSCK, depends: inodes) - validate directory entries: target inode exists, hash correctness, d_type matches (`fs/check.c:1698`)
- `check_xattrs` (28, FSCK, depends: inodes) - validate xattr entries belong to existing inodes (`fs/check.c:1774`)
- `check_root` (29, ONLINE|FSCK, depends: inodes) - ensure root inode/subvolume exist (`fs/check.c:1846`)
- `check_unreachable_inodes` (40, FSCK, depends: inodes + dirents) - find inodes with no directory entry; reattach to lost+found (`fs/check.c:1182`)
- `check_subvolume_structure` (36, ONLINE|FSCK, depends: subvols + inodes) - verify subvolume directory structure (`fs/check_dir_structure.c:121`)
- `check_directory_structure` (30, ONLINE|FSCK, depends: unreachable_inodes) - DFS from root, detect cycles, reattach disconnected dirs (`fs/check_dir_structure.c:279`)
- `check_nlinks` (31, FSCK, depends: inodes + dirents) - count hardlinks, fix bi_nlink mismatches (`fs/check_nlinks.c:223`)
- `check_reconcile_work` (43, ONLINE|FSCK, depends: snapshots) - verify reconcile work btree

**Late / Cleanup:**

- `resume_logged_ops` (23, ALWAYS) - resume incomplete logged operations (renames, etc.)
- `delete_dead_inodes` (32, ALWAYS) - delete inodes marked for deletion
- `kill_i_generation_keys` (47, ONLINE) - remove obsolete inode generation keys
- `fix_reflink_p` (33) - one-time reflink pointer fix
- `set_fs_needs_reconcile` (34) - one-time flag set
- `btree_bitmap_gc` (46, ONLINE) - garbage collect btree bitmap
- `lookup_root_inode` (42, ALWAYS|SILENT) - verify root inode is readable before completing recovery

### Pass Execution Engine (`passes.c:518`)

`bch2_run_recovery_passes()` iterates passes as a bitmask. On each pass:

1. Clear from `current_passes`, run `p->fn(c)`, flush journal
2. On success: mark `passes_complete`, update `pass_done`
3. On rewind (`BCH_ERR_restart_recovery`): restore passes from `rewound_to` onwards
4. Failed passes tracked in `passes_failing` - excluded from retry in same iteration

`bch2_run_recovery_passes_startup()` computes initial pass set:
`PASS_ALWAYS | (unclean ? PASS_UNCLEAN : 0) | (fsck ? PASS_FSCK : 0) | sb.recovery_passes_required | opts.recovery_passes`. Passes that can run online are deferred unless `PASS_NODEFER` or have offline dependents.

### Rewind Mechanism

When a later pass discovers damage requiring an earlier pass, `__bch2_run_explicit_recovery_pass()` (`passes.c:368`):

- Sets the pass bit in `sb.recovery_passes_required` (persistent) or `scheduled_passes_ephemeral`
- If in recovery and pass is before current: sets `rewound_to`, returns `BCH_ERR_restart_recovery`
- Cannot rewind past `set_may_go_rw` once RW
- Passes before `set_may_go_rw` modify journal replay keys buffer directly

## Journal Replay (`recovery.c:368`)

Two-phase replay:

1. **Accounting keys first** - replayed with `BCH_TRANS_COMMIT_skip_accounting_apply` to avoid double-counting. Accumulates deltas onto existing btree values using `bch2_accounting_accumulate()`. Sets `BCH_FS_accounting_replay_done` when complete.

2. **All other keys** - first attempt in sorted (btree) order for locality. Keys that fail (journal deadlock risk) are collected and retried in journal sequence order, unpinning journal entries as replay progresses via `replay_now_at()`.

Key properties of `struct journal_key`:
- `allocated` - synthesized by repair code, not from journal
- `overwritten` - already superseded, skip
- `journal_seq_offset` - offset from `journal_entries_base_seq`

## Btree Topology Repair

### Node Scan (`btree/node_scan.c:372`)

`bch2_scan_for_btree_nodes()`: parallel per-device scan reading every btree-sized block. Identifies nodes by magic number and checksum. Deduplicates by cookie (unique node ID). Produces sorted `found_btree_node` array.

### Topology Check (`btree/check.c:594`)

`bch2_check_topology()`: for each btree root:

1. `bch2_topology_check_root()` - verify root is readable; if not, try scanned nodes or allocate fake root
2. `btree_check_root_boundaries()` - root min_key == POS_MIN, max_key == SPOS_MAX
3. `bch2_btree_repair_topology_recurse()` - recursive DFS checking child coverage (no gaps, no overlaps). Repairs by reinserting scanned nodes, adjusting boundaries, or dropping broken nodes.

If entire btree is empty after repair, allocates level-0 fake root.

## Alloc Info Reconstruction (`recovery.c:143`)

`bch2_reconstruct_alloc()` (mount option `-o reconstruct_alloc`):

1. Sets recovery passes: `check_allocations`, `check_alloc_info`, `check_lrus`, `check_extents_to_backpointers`, `check_alloc_to_lru_refs`
2. Silences ~20 expected fsck error types in `errors_silent`
3. Clears `BCH_COMPAT_alloc_info` compat bit
4. Kills all alloc-related btrees (alloc, backpointers, need_discard, freespace, bucket_gens, lru, accounting)
5. Recovery passes rebuild everything by walking extent/btree pointer btrees

## Snapshot Reconstruction (`snapshots/check_snapshots.c:550`)

`bch2_reconstruct_snapshots()`:

1. Iterates all snapshot-bearing btrees (inodes, extents, dirents, xattrs)
2. Groups snapshot IDs that appear on the same inode/extent into trees
3. Merges overlapping groups
4. For each discovered snapshot ID missing from snapshot btree: creates new snapshot node via `check_snapshot_exists()`
5. Limited: cannot reconstruct multi-node snapshot trees

## Error Handling

### `__bch2_fsck_err()` (`init/error.c:447`)

Central fsck error reporting. Per error ID (`enum bch_sb_error_id`):

1. Check `errors_silent` bitmap - auto-fix silently if set
2. Ratelimit after `FSCK_ERR_RATELIMIT_NR` (10) instances per error type
3. Memoize response on transaction restart (same message = same answer)
4. Decision matrix based on `fix_errors` option:
   - `FSCK_FIX_yes` -> auto-fix (return `fsck_fix`)
   - `FSCK_FIX_no` -> ignore if `FSCK_CAN_IGNORE`, else `fsck_errors_not_fixed`
   - `FSCK_FIX_ask` -> prompt user via stdio redirect
   - `FSCK_FIX_exit` -> `fsck_errors_not_fixed`
5. `FSCK_AUTOFIX` errors self-heal with `errors=continue` or `fix_safe`

### Flags per error (`enum bch_fsck_flags`)

- `FSCK_CAN_FIX` - repair code exists
- `FSCK_CAN_IGNORE` - safe to continue without fixing
- `FSCK_AUTOFIX` - fix automatically even outside fsck (online self-healing)
- `FSCK_ERR_SILENT` - suppress output

### Inconsistency Detection (`init/error.c:29`)

`__bch2_inconsistent_error()`: sets `BCH_FS_error`, then based on `opts.errors`:
- `continue` - return, keep running
- `ro` / `fix_safe` - emergency read-only
- `panic` - kernel panic

`__bch2_topology_error()`: sets `BCH_FS_topology_error`, schedules `check_topology` pass.

### `bch2_btree_lost_data()` (`recovery.c:46`)

Called when a btree root is unreadable. Per btree type, schedules appropriate recovery:
- `alloc` -> `check_alloc_info`
- `backpointers` -> `check_btree_backpointers` + `check_extents_to_backpointers`
- `snapshots` -> `reconstruct_snapshots` + `scan_for_btree_nodes`
- Default -> `check_topology` + `scan_for_btree_nodes`

Always schedules `check_topology` and `check_allocations`.

### Fsck Exit Codes (`fs/check.c:1890`)

`bch2_fs_fsck_errcode()`: returns bitmask:
- `1` - errors fixed
- `4` - errors remain / fatal error

## Online vs Offline Fsck

### Offline Fsck (`fs/check.c:1959`)

`bch2_ioctl_fsck_offline()` -> `BCH_IOCTL_FSCK_OFFLINE`:

1. Opens filesystem with `read_only=1`, `nostart=true`
2. Spawns `bch2_fsck_offline_thread_fn()` via `thread_with_stdio`
3. Thread calls `bch2_fs_start()` which runs full recovery including fsck passes
4. Forces `errors != panic` -> `ro` for safety
5. Calls `bch2_fs_exit()` when done

### Online Fsck (`fs/check.c:2086`)

`bch2_ioctl_fsck_online()` -> `BCH_IOCTL_FSCK_ONLINE`:

1. Takes `ro_ref` on mounted filesystem
2. Spawns `bch2_fsck_online_thread_fn()` via `thread_with_stdio`
3. Acquires `recovery.run_lock` (fails if recovery already running)
4. Runs `PASS_FSCK & PASS_ONLINE` passes via `bch2_run_recovery_passes()`
5. Sets `c->stdio` and `c->stdio_filter` for interactive prompt routing
6. Restores `fix_errors` option when done

### Pass Online Capability

Only passes with `PASS_ONLINE` flag can run on a mounted filesystem. Key passes that are offline-only: `check_inodes`, `check_extents`, `check_dirents`, `check_xattrs`, `check_nlinks`, `check_unreachable_inodes`. These require exclusive access to avoid racing with VFS operations.

## Thread-with-File Mechanism

### Files

- `util/thread_with_file.c` / `thread_with_file.h` - generic kernel thread with fd interface
- `fs/check.c:1912` - `struct fsck_thread`

### Design

`struct thread_with_stdio` wraps a kernel thread with a `stdio_redirect`:
- Creates an anonymous fd returned to userspace
- Userspace reads/writes fd for fsck interaction (error messages, y/n prompts)
- `bch2_stdio_redirect_write()` sends output to userspace
- `bch2_stdio_redirect_readline_timeout()` reads user input with timeout
- `c->stdio` pointer routes `bch2_print()` and `bch2_fsck_ask_yn()` through the redirect
- `c->stdio_filter` ensures only the fsck thread's prints go through stdio (not other threads)

### Userspace Side

In userspace builds (`#ifndef __KERNEL__`), `bch2_fsck_ask_yn()` reads directly from stdin. The `bcachefs fsck` command in bcachefs-tools uses the ioctl path when running as root with a mounted `/dev/bcachefs-ctl` chardev, or falls back to direct `bch2_fs_open()` + `bch2_fs_start()`.

## Async Recovery Passes (`passes.c:585`)

`bch2_run_async_recovery_passes()`: queues work on `system_long_wq` to run passes marked `PASS_ONLINE` from `sb.recovery_passes_required | scheduled_passes_ephemeral`. Excludes ratelimited passes. Used for background self-healing when errors are detected at runtime. Triggered after `check_snapshots` completes (wakes copygc + reconcile).
