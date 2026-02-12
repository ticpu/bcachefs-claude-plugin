## Diagnostic Workflow

py1hon's triage order:

1. `dmesg` — full output, **never** `dmesg | grep bcachefs` (truncates multiline messages)
2. `bcachefs fs top` — live tracepoint counters (or `perf top -e bcachefs:*` pre-mount)
3. `bcachefs fs timestats` — latency breakdown (v1.36+, else sysfs `time_stats/`)
4. `bcachefs fs usage -ha` — space accounting, per-device breakdown, degraded data
5. `bcachefs show-super -f errors` — persistent error counters
6. `cat /sys/fs/bcachefs/<UUID>/internal/journal_debug` — journal state and watermark
7. Drill into tracepoints: `perf trace -e bcachefs:<event>` or `trace-cmd stream`
8. Backtraces: `perf record -g -e bcachefs:<event>; perf script`

For stuck processes:

```sh
cat /proc/<PID>/stack
echo w > /proc/sysrq-trigger
```

## CLI Quick Reference

### Live Monitoring

```sh
bcachefs fs top <mnt> # primary live diagnostic
bcachefs fs timestats <mnt> # latency breakdown TUI (v1.36+)
watch bcachefs fs usage -h <mnt> # monitor evacuate/rebalance progress
bcachefs reconcile status <mnt> # reconcile progress
```

### `bcachefs show-super <dev>`

| Flag | Purpose |
|------|---------|
| `-f errors` | Persistent error counters |
| `-f members_v2,errors,ext,recovery_passes` | Comprehensive diagnostic |
| `-f members_v2,replicas,replicas_v0,disk_groups` | Replication/disk group config |
| `--field-only=journal_v2` | Journal layout per device |
| `-f counters` | Operation counters (updated on unmount only) |
| `-l` | Superblock physical layout |

### Journal Analysis (offline) - `bcachefs list_journal`

```sh
-a <dev> # full dump
-FD <dev> # flush timeline with timestamps
-a -t <btree>:<inode>:<offset> <dev> # filter to key
-b extents -k extents:<inum>:0-extents:<inum>:U64_MAX <dev> # all extents for inode
```

### Btree Inspection

```sh
bcachefs list -b <btree> <dev> # offline only
cat /sys/kernel/debug/bcachefs/<UUID>/btrees/<btree>/keys # online (debugfs)
cat /sys/kernel/debug/bcachefs/<UUID>/btrees/<btree>/formats # node format/efficiency
```

### `bcachefs fsck`

| Flag | Purpose |
|------|---------|
| `-n` | Dry-run, report only |
| `-k` | In-kernel online fsck |
| `-K` | In-kernel offline fsck |
| `-o recovery_passes=<pass>` | Run specific recovery pass |
| `-o reconcile_enabled=false` | Disable reconcile during fsck |

Mount-based fsck:

```sh
mount -o fsck,fix_errors,ro,nochanges <dev> <mnt> # safe dry-run (writes nothing)
mount -o fsck,fix_errors <dev> <mnt> # fsck with auto-repair
```

## sysfs / debugfs

### `/sys/fs/bcachefs/<UUID>/`

| Path | Purpose |
|------|---------|
| `internal/journal_debug` | Journal position, watermark, blocked state, write buffer |
| `internal/alloc_debug` | Allocator state |
| `dev-*/alloc_debug` | Per-device allocation state |
| `time_stats/*` | Operation latency stats (pre-v1.36) |
| `counters/*` | All internal counters |
| `sb_errors` | Superblock error list |
| `reconcile_status` | Reconcile progress and pending work |
| `snapshot_delete_status` | Snapshot deletion progress |
| `internal/moving_ctxts` | Active data move contexts |
| `dev-*/io_latency_stats_{read,write}` | Per-device IO latency |
| `dev-*/state` | Device state (also writable) |
| `options/<name>` | Runtime filesystem options |
| `btree_cache` | Btree node cache size |

### `/sys/kernel/debug/bcachefs/<UUID>/`

| Path | Purpose |
|------|---------|
| `btrees/*/keys` | Raw btree key contents |
| `btrees/*/formats` | Node format, packed key sizes, waste |
| `btree_transactions` | Active transactions and locks (`CONFIG_BCACHEFS_DEBUG`) |
| `btree_transaction_stats` | Per-transaction-type statistics |
| `btree_updates` | Pending interior node updates |

### Recovery Triggers (write `1`)

| Path | Purpose |
|------|---------|
| `internal/btree_wakeup` | Unstick hung fs (lost wakeup bug) |
| `internal/trigger_discards` | Force pending discards |
| `internal/trigger_reconcile_wakeup` | Wake stuck reconcile thread |

## Tracing

```sh
perf trace -e bcachefs:<event> # live event stream
perf trace -e bcachefs:error_throw -- <cmd> # trace errors during a command
trace-cmd stream -e bcachefs:<event> # preferred (perf mangles enum symbols)
perf record -g -e bcachefs:<event>; perf script # backtraces
perf record -ag; perf report --no-children # flat CPU profile
```

`perf top -e bcachefs:*` — fallback for `fs top` when fs not mounted (leaks memory if left open).

### Key Tracepoints

| Tracepoint | Symptom |
|------------|---------|
| `error_throw` | Any internal error — first tracepoint for mysterious failures |
| `bucket_alloc_fail` | Allocation failures, ENOSPC-like behavior |
| `reconcile_data` / `rebalance_extent` | Reconcile not making progress |
| `copygc_wait` / `copygc` | Copy-GC stalls |
| `write_buffer_flush` / `write_buffer_maybe_flush` | Write buffer backlog |
| `transaction_commit` | Transaction behavior investigation |
| `trans_restart_upgrade` / `trans_restart_split_race` | Transaction restart storms |
| `io_move_start_fail` | Data move failures |
| `data_update_done_no_rw_devs` | Insufficient writable devices for moves |
| `journal_entry_close` | Journal behavior during mount |
| `discard_buckets` | Discard not running |

## Performance Problems

### High sync/fsync Latency Under Write Load

`sync()` → `bch2_journal_flush()` waits for current journal seq to be written
with FUA. Heavy writes generate journal entries faster than reclaim frees them.

```sh
bcachefs fs timestats <mnt> # journal_flush_seq, data_write
cat /sys/fs/bcachefs/<UUID>/internal/journal_debug # watermark at "reclaim" = blocked
```

Key timestats (`BCH_TIME_STATS()` in `bcachefs.h`):

| Stat | Meaning |
|------|---------|
| `journal_flush_seq` | Direct sync() latency |
| `journal_flush_write` | Journal entry write time |
| `blocked_journal_low_on_space` | Journal buckets full |
| `blocked_journal_low_on_pin` | Pin FIFO full |
| `blocked_journal_max_in_flight` | All 16 journal buffers in flight |
| `blocked_allocate` | Foreground allocation stalls |
| `blocked_write_buffer_full` | Write buffer backlog |

Common causes: journal too small (`show-super --field-only=journal_v2`, resize with
`bcachefs device resize-journal`), watermark at `reclaim`, reconcile competing for IO.

### Reconcile/Rebalance Stuck

```sh
bcachefs reconcile status <mnt>
bcachefs fs top <mnt> # reconcile_data events
perf trace -e bcachefs:reconcile_data
perf trace -e bcachefs:io_move_start_fail
echo 1 > /sys/fs/bcachefs/<UUID>/internal/trigger_reconcile_wakeup # if zero progress
cat /sys/kernel/debug/bcachefs/<UUID>/btrees/reconcile_work/keys # stuck extents
cat /sys/kernel/debug/bcachefs/<UUID>/btrees/reconcile_scan/keys # scan cookie
```

Advanced: stuck extents may have `bi_nlink=0` and mismatched `bi_background_target`,
or reflink pointers missing `MAY_UPDATE_OPTIONS` bit. Cross-reference extents and
reflink btrees via debugfs to identify specific blocked entries.

### Journal Livelock During Device Add

`bch2_journal_flush_all_pins` blocks indefinitely when reconcile and copygc
continuously generate new pins. Stack shows `bch2_journal_flush_all_pins` in
mount/device-add path.

```sh
cat /proc/<PID>/stack
bcachefs show-super -f journal_v2 <dev>
cat /sys/kernel/debug/bcachefs/<UUID>/btree_updates
```

## Hung / Unresponsive Filesystem

```sh
echo w > /proc/sysrq-trigger # D-state tasks
cat /proc/<PID>/stack  # specific process
cat /sys/kernel/debug/bcachefs/<UUID>/btree_transactions # CONFIG_BCACHEFS_DEBUG
echo 1 > /sys/fs/bcachefs/<UUID>/internal/btree_wakeup # try lost-wakeup fix
cat /sys/fs/bcachefs/<UUID>/internal/journal_debug
```

## Memory Problems

```sh
cat /sys/fs/bcachefs/<UUID>/btree_cache # cache size
sort -g /proc/allocinfo | grep bcachefs # CONFIG_MEM_ALLOC_PROFILING
grep bch_inode_info /proc/slabinfo # inode cache
echo 3 > /proc/sys/vm/drop_caches # test reclaimability
# btree_cache should drop to ~16MB; if not, shrinker bug
```

Mitigations: exclude snapshot dirs from `updatedb`/`mlocate`, prefer `zram` over
`zswap` (better bcachefs reclaim interaction), ensure swap for heavy snapshot workloads.

## Snapshot Issues

### Deletion Extremely Slow

Most common: `snapshot_deletion_v2` not enabled. Dmesg shows:

```
requested incompat feature 1.26: snapshot_deletion_v2 currently not enabled
```

Fix:

```sh
echo incompat > /sys/fs/bcachefs/<UUID>/options/version_upgrade
umount <mnt> && mount <dev> <mnt>
```

Monitor: `cat /sys/fs/bcachefs/<UUID>/snapshot_delete_status`

### "key in missing snapshot" / "key in deleted snapshot"

- **"key in deleted snapshot"**: Normal during deletion. Auto-heals (>= 1.19 `FSCK_AUTOFIX`).
- **"key in missing snapshot"**: Snapshot ID absent from btree. Should not happen with v2.
  Fix: `mount -o fsck,fix_errors`

### "error finding live child of snapshot" → Emergency Read-Only

Fixed in >= 1.33.4. Older kernels: `bcachefs fsck -k <dev>`

### Snapshot Corruption After Upgrading to 1.33.3

Regression: `delete_dead_interior_snapshots` ran before RW, corrupting in-memory
snapshot table. Fix: upgrade to >= 1.33.4. For data corruption, journal rewind:

```sh
bcachefs list_journal -FD <dev>
mount -o nochanges,fix_errors,journal_rewind=<SEQ>,journal_rewind_no_extents <dev> <mnt>
```

### Subvolume Appears Empty After Mount

`check_unreachable_inodes` may have moved data to `lost+found`. **Stop all RW mounts.**

```sh
bcachefs dump -o dump.qcow2 <dev>                                        # metadata dump first
bcachefs list_journal -a --flush-only --datetime <dev>                    # find target seq
mount -o nochanges,fix_errors,journal_rewind=<SEQ>,journal_rewind_no_extents <dev> <mnt>
```

### `erofs_trans_commit` During Unmount After Snapshot Delete

Harmless race. Deletion resumes on next mount.

### Snapshot Diagnostic Commands

```sh
bcachefs list -b snapshots <dev>                                          # offline
cat /sys/kernel/debug/bcachefs/<UUID>/btrees/snapshots/keys               # online
bcachefs list_journal -a -t snapshots:0:<SNAP_ID> <dev>                   # journal history
cat /sys/fs/bcachefs/<UUID>/snapshot_delete_status                        # deletion progress
cat /sys/kernel/debug/bcachefs/<UUID>/btree_transactions                  # what worker is doing
bcachefs subvolume list <mnt>
bcachefs subvolume list-snapshots <mnt>
```

### Snapshot Error Quick Reference

| Error | Severity | Action |
|-------|----------|--------|
| `key in deleted snapshot ... deleting` | Low | Normal self-healing |
| `key in missing snapshot` | Medium | Run check_snapshots; enable v2 |
| `error finding live child of snapshot` | High | Upgrade kernel; fsck |
| `snapshot tree points to missing subvolume` | Low | fsck auto-repairs |
| `snapshot with incorrect depth/skiplist field` | Low | fsck auto-repairs |
| `Found N empty interior snapshot nodes` | Info | Normal after upgrade |
| `snapshot deletion did not run correctly` | High | fsck; upgrade kernel |
| `delete_dead_snapshots(): error erofs_trans_commit` | Low | Harmless unmount race |
| `bch2_evict_inode(): ... ENOTEMPTY` | None | Informational |
| `snapshot_deletion_v2 not enabled` | Medium | `version_upgrade=incompat` |
| `bch2_subvolume_wait_for_pagecache_and_delete` stuck | High | Close open files; upgrade |

## Recovery Mount Options

| Option | Purpose |
|--------|---------|
| `fsck,fix_errors` | In-kernel fsck with auto-repair |
| `fsck,fix_errors,ro,nochanges` | Safe dry-run fsck (writes nothing) |
| `nochanges` | Mount without writing anything |
| `degraded` / `degraded=very` | Mount with missing devices |
| `verbose` | Verbose mount logging |
| `reconstruct_alloc` | Rebuild allocation metadata from scratch |
| `journal_rewind=<seq>` | Roll back to journal sequence number |
| `journal_rewind_no_extents` | Rewind without touching extents btree |
| `norecovery,ro` | Skip all recovery, read-only access |

## Kernel Config for Debugging

| Config | Enables |
|--------|---------|
| `CONFIG_BCACHEFS_DEBUG` | `btree_transactions` debugfs, extra validation |
| `CONFIG_MEM_ALLOC_PROFILING` | `/proc/allocinfo` per-callsite memory tracking |
| `CONFIG_FTRACE` | Tracepoints |
| `CONFIG_STACKTRACE` | `/proc/<PID>/stack` |
