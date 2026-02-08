# bcachefs Reconcile (formerly Rebalance)

## Overview

Reconcile is bcachefs's background data management subsystem. It continuously ensures that all data matches configured IO path options (replicas, compression, checksum, target, erasure coding). When options change or devices are added/removed, reconcile propagates these changes to existing data by moving, rewriting, or recompressing extents.

Introduced in v1.33 (`bcachefs_metadata_version_reconcile`), reconcile replaced the older "rebalance" system with a more scalable architecture based on dedicated btrees for work tracking.

## Files

Kernel (`fs/bcachefs/data/reconcile/`):
- `format.h` - on-disk structures, btree definitions, work tracking enums
- `types.h` - runtime state (`struct bch_fs_reconcile`)
- `work.h` / `work.c` - main reconcile thread, work processing (1455 lines)
- `trigger.h` / `trigger.c` - btree triggers, option propagation (964 lines)
- `check.c` / `check.h` - fsck pass for reconcile btrees

Userspace (`c_src/`):
- `cmd_data.c` - `bcachefs reconcile` commands (wait, status)

## The 6 Reconcile Btrees

Reconcile uses 6 btrees (IDs 18, 21-25) for work tracking:

### 1. `reconcile_work` (ID 18)
- **Purpose**: Main work queue for normal-priority data extents
- **Flags**: `BTREE_IS_snapshot_field`, `BTREE_IS_write_buffer`
- **Keys**: `KEY_TYPE_set` (bitset), `KEY_TYPE_cookie` (scan markers)
- **Scope**: Logical extent positions in `BTREE_ID_extents` or `BTREE_ID_reflink`
- **Position encoding**: Via `data_to_rb_work_pos()` - reflink extents at inode 0-BCACHEFS_ROOT_INO, user extents at inode >= BCACHEFS_ROOT_INO

### 2. `reconcile_hipri` (ID 21)
- **Purpose**: High-priority work (device evacuation, degraded data)
- **Flags**: `BTREE_IS_snapshot_field`, `BTREE_IS_write_buffer`
- **Keys**: `KEY_TYPE_set`
- **Priority**: Processed before `reconcile_work`

### 3. `reconcile_pending` (ID 22)
- **Purpose**: Work that failed due to insufficient space or devices
- **Flags**: `BTREE_IS_snapshot_field`, `BTREE_IS_write_buffer`
- **Keys**: `KEY_TYPE_set`
- **Retry trigger**: Device add/resize, label change (`bch2_reconcile_pending_wakeup()`)

### 4. `reconcile_scan` (ID 23)
- **Purpose**: Scan cookies and btree node backpointers
- **Flags**: None (not write-buffered)
- **Keys**: `KEY_TYPE_cookie` (inode 0), `KEY_TYPE_backpointer` (inode 1+)
- **Inode 0**: Scan cookies indicate option changes need propagation
- **Inode 1+**: Backpointers to btree nodes needing reconcile (since btree nodes can't be tracked in work btrees)

#### Scan Cookie Types (encoded in offset field):
- `RECONCILE_SCAN_COOKIE_fs` (0): Entire filesystem scan
- `RECONCILE_SCAN_COOKIE_metadata` (1): Metadata-only scan
- `RECONCILE_SCAN_COOKIE_pending` (2): Retry pending work
- `RECONCILE_SCAN_COOKIE_device` (32+): Device-specific scan (device index in offset - 32)
- `>= BCACHEFS_ROOT_INO`: Per-inode scan (offset = inode number)

### 5. `reconcile_work_phys` (ID 24)
- **Purpose**: Physical address tracking for rotational devices
- **Flags**: `BTREE_IS_write_buffer`
- **Keys**: `KEY_TYPE_set`
- **Position**: Device backpointer positions (device, bucket, offset)
- **Rationale**: For spinning disks, process extents in LBA order for efficiency

### 6. `reconcile_hipri_phys` (ID 25)
- **Purpose**: High-priority physical work
- **Flags**: `BTREE_IS_write_buffer`
- **Keys**: `KEY_TYPE_set`

### Mapping Tables

`reconcile_work_btree[]` maps work IDs to logical btrees:
- `RECONCILE_WORK_hipri` -> `BTREE_ID_reconcile_hipri`
- `RECONCILE_WORK_normal` -> `BTREE_ID_reconcile_work`
- `RECONCILE_WORK_pending` -> `BTREE_ID_reconcile_pending`

`reconcile_work_phys_btree[]` maps to physical btrees:
- `RECONCILE_WORK_hipri` -> `BTREE_ID_reconcile_hipri_phys`
- `RECONCILE_WORK_normal` -> `BTREE_ID_reconcile_work_phys`

## On-Disk Structures

### `struct bch_extent_reconcile` (format.h:90-134)

Embedded in extent keys to track IO options and reconcile state:

```c
__u64   type:8,                    // BCH_EXTENT_ENTRY_reconcile
        unused:2,
        ptrs_moving:5,              // Bitmap of pointers being evacuated
        hipri:1,                    // High-priority (tracked in reconcile_hipri)
        pending:1,                  // Unable to complete (tracked in reconcile_pending)
        need_rb:5,                  // Bitmap of BCH_RECONCILE_* fields needing work

        data_replicas_from_inode:1, // These 6 fields indicate if option
        data_checksum_from_inode:1, // comes from inode vs extent
        erasure_code_from_inode:1,
        background_compression_from_inode:1,
        background_target_from_inode:1,
        promote_target_from_inode:1,

        data_replicas:3,            // Desired replicas (1-7)
        data_checksum:4,            // enum bch_csum_type
        erasure_code:1,             // EC enabled
        background_compression:8,   // enum bch_compression_opt
        background_target:10,       // Target device/label (0-1023)
        promote_target:10;          // Read promotion target
```

**Fields:**
- `need_rb`: Bitmap of 6 reconcile options (data_replicas, data_checksum, erasure_code, background_compression, background_target, promote_target)
- `hipri`: Set when extent needs evacuation (device failing/removed) or is degraded
- `pending`: Set when move failed due to ENOSPC or insufficient devices
- `ptrs_moving`: Bitmap indicating which pointers are being evacuated

### `struct bch_extent_reconcile_bp` (format.h:136-144)

For btree nodes: index into `reconcile_scan` btree (inode 1+) to avoid bloating work btrees with btree nodes.

### Legacy: `struct bch_extent_rebalance_v1` (format.h:52-88)

Older format, converted during upgrade.

## Work Item Lifecycle

### 1. Creation: Triggers

Work enters the system via btree triggers (`trigger.c:368`, `__bch2_trigger_extent_reconcile()`):

**Option change scan** (`bch2_set_reconcile_needs_scan_trans()`):
- User changes filesystem/inode option (replicas, compression, target, etc.)
- Creates scan cookie in `reconcile_scan` btree (work.c:132-149)
- Cookie value increments on each option change
- Reconcile thread scans affected extents and updates their `bch_extent_reconcile` entries

**Extent insert/update** (`trigger.c:375-402`):
- Level 0 (data): Updates `reconcile_work`/`hipri`/`pending` bitset via `reconcile_work_mod()`
- Level 1+ (btree nodes): Manages backpointer in `reconcile_scan` inode 1+ via `reconcile_bp_add()`/`reconcile_bp_del()`

**Physical tracking** (for rotational devices):
- Backpointers track device+LBA for efficient sequential access
- Updated via `BACKPOINTER_RECONCILE_PHYS` flag in backpointer triggers

**Disk accounting** (`trigger.c:405-434`):
- Maintains counters per reconcile type (replicas, checksum, EC, compression, target, hipri, pending)
- Used for status reporting

### 2. Scan Phase

When scan cookie exists (work.c:902-941, `do_reconcile_scan()`):

**Filesystem scan** (`RECONCILE_SCAN_fs` / `_metadata`):
- Iterates `BTREE_ID_extents`, `BTREE_ID_reflink`
- For each extent: `bch2_update_reconcile_opts()` (trigger.c:810-852) checks if IO options match
- Creates `bch_extent_reconcile` entry with `need_rb` bitmap if mismatch
- Inserts work into appropriate btree (`reconcile_work` / `reconcile_hipri` / `reconcile_pending`)

**Device scan** (`RECONCILE_SCAN_device`):
- Iterates backpointers for device (`do_reconcile_scan_bps()`, work.c:769-793)
- Checks if data needs evacuation or re-targeting

**Inode scan** (`RECONCILE_SCAN_inum`):
- Scans single inode's extents
- Triggered by per-inode option change

**Indirect extent handling** (`do_reconcile_scan_indirect()`, work.c:795-857):
- Reflink extents may be referenced by inodes with different options
- Stored options (`*_from_inode` fields) avoid repeated lookups

After scan completes, cookie is deleted (`bch2_clear_reconcile_needs_scan()`, work.c:169-193).

### 3. Processing Phases

The reconcile thread (`bch2_reconcile_thread()`, work.c:1268-1293) processes work in phases (work.c:971-1012, `reconcile_phases[]`):

```
Phase 0: SCAN cookies             (reconcile_scan inode 0)
Phase 1: HIPRI btree nodes        (reconcile_scan inode RECONCILE_WORK_hipri+)
Phase 2: HIPRI phys               (reconcile_hipri_phys)
Phase 3: HIPRI logical            (reconcile_hipri)
Phase 4: NORMAL btree nodes       (reconcile_scan inode RECONCILE_WORK_normal+)
Phase 5: NORMAL phys              (reconcile_work_phys)
Phase 6: NORMAL logical           (reconcile_work)
Phase 7: PENDING btree nodes      (reconcile_scan inode RECONCILE_WORK_pending+)
Phase 8: PENDING logical          (reconcile_pending)
```

**Phase execution** (work.c:1125-1266, `do_reconcile()`):
- Buffers up to 1024 work entries (`RECONCILE_WORK_BUF_NR`) to avoid write buffer contention
- For each phase, iterates btree and processes entries
- Physical phases spawn per-device threads for rotational devices (`do_reconcile_phys()`, work.c:1076-1095)

### 4. Extent Processing

**Data extents** (`do_reconcile_extent()`, work.c:604-632):
1. Look up extent in `BTREE_ID_extents` or `BTREE_ID_reflink`
2. `reconcile_set_data_opts()` (work.c:283-456): Analyze `bch_extent_reconcile.need_rb` bitmap
3. Build `data_update_opts`:
   - `ptrs_kill`: Pointers to delete (wrong target, offline, excess replicas, wrong checksum/compression)
   - `ptrs_kill_ec`: EC stripes to remove
   - `extra_replicas`: Add replicas if under-replicated
   - `target`: Desired target device/label
4. `bch2_move_extent()` (data/move.c): Perform actual data move
5. On success: Work entry deleted via trigger
6. On failure (ENOSPC, insufficient devices): Set `pending` flag, move to `reconcile_pending`

**Btree nodes** (`do_reconcile_btree()`, work.c:686-715):
- Fetch node via backpointer in `reconcile_scan`
- Process similarly to data, but must evacuate entire node (can't partial-write btree nodes)

**Physical processing** (`do_reconcile_extent_phys()`, work.c:634-683):
- For rotational devices: Process in device LBA order
- Look up extent via backpointer
- Set `data_opts.read_dev` to prefer reading from specific device

### 5. Completion

Work entries auto-delete via btree triggers when `bch_extent_reconcile` is removed from extent (or `need_rb` becomes 0).

## Reconcile Options (6 tracked fields)

From `BCH_RECONCILE_OPTS()` (format.h:147-153):

1. **`data_replicas`**: Number of replicas (1-7)
   - Triggers on under/over-replication or device evacuation
   - Sets `hipri` if degraded
2. **`data_checksum`**: Checksum type (none, crc32c, crc64, xxhash, etc.)
   - Recomputes checksums if type changes
3. **`erasure_code`**: Erasure coding enabled (bool)
   - Converts between EC and non-EC
   - Waits for existing EC stripe completion before conversion
4. **`background_compression`**: Compression type (none, lz4, gzip, zstd, etc.)
   - Recompresses data (unless marked incompressible)
5. **`background_target`**: Target device/label (10 bits, 0-1023)
   - Moves data to match target
   - Sets `ptrs_moving` bitmap for pointers on wrong devices
6. **`promote_target`**: Read promotion target (for caching tiers)

Each option has a `*_from_inode` flag indicating if the value should be read from the inode (for user data) or stored in the extent (for indirect extents).

## Interaction with Other Subsystems

### Move Path (data/move.c)

Reconcile uses the generic `bch2_move_extent()` API:
- `moving_context`: Tracks stats, rate limiting, writepoint
- `data_update_opts`: Specifies ptrs_kill, target, extra_replicas
- `bch2_data_update_in_flight()`: Avoids duplicate work
- Returns `BCH_ERR_data_update_fail_need_copygc` if copygc needed first

### Copygc (data/copygc.c)

Reconcile and copygc coordinate:
- Reconcile waits for copygc if move fails with `_need_copygc` (work.c:1223-1232)
- Copygc has priority for allocator resources
- Both use same writepoint (`c->allocator.reconcile_write_point`)

### Allocator

Reconcile respects allocator watermarks:
- Uses `BCH_WATERMARK_stripe` normally
- Rate limited by IO clock (work.c:943-962, `reconcile_wait()`)
- Disk reservations acquired before commits (`bch2_trans_commit_lazy()`)

### Snapshots

Reconcile is snapshot-aware:
- `BTREE_IS_snapshot_field` on work btrees
- `per_snapshot_io_opts` (trigger.h:144-171): Caches inode options per snapshot
- Different snapshots of same inode may have different options

### Write Buffer

Most reconcile btrees use `BTREE_IS_write_buffer`:
- Work updates buffered in memory
- Flushed periodically to reduce btree churn
- `bch2_btree_write_buffer_flush_sync()` forced between phases (work.c:1149, 1192)

## User Interface

### Filesystem Options (opts.h:501-511)

- **`reconcile_enabled`** (bool, default 1): Enable/disable reconcile
- **`reconcile_on_ac_only`** (bool, default 0): Only run on AC power

### Device Option (opts.h:554)

- **`rotational`** (bool, per-device): Use physical btrees for this device

### Commands (c_src/cmd_data.c:384+)

**`bcachefs reconcile wait [--types=TYPES] <mountpoint>`**:
- Waits for reconcile to finish processing specified types
- Types: replicas, checksum, erasure_code, compression, target, high_priority, pending
- Default: all except pending
- Reads disk accounting (`BIT(BCH_DISK_ACCOUNTING_reconcile_work)`)
- Wakes reconcile thread via sysfs `trigger_reconcile_wakeup`

**`bcachefs reconcile status [--types=TYPES] <mountpoint>`**:
- Shows sectors pending per reconcile type
- Reads sysfs `reconcile_status` for thread state
- Displays phase, position, backtrace

### Sysfs (debug/sysfs.c:224+, 343+)

Read-only:
- **`reconcile_status`**: Detailed status (phase, position, wait state, thread backtrace)
- **`reconcile_scan_pending`**: Boolean indicating if scan cookies exist

Write-only triggers:
- **`trigger_reconcile_wakeup`**: Wake thread immediately
- **`trigger_reconcile_pending_wakeup`**: Retry pending work (sets `RECONCILE_SCAN_pending` cookie)

### Status Output (work.c:1295-1366, `bch2_reconcile_status_to_text()`)

When waiting:
- IO wait duration (iotime-based rate limiting)
- IO wait remaining
- Wall-clock duration waited

When running:
- Current scan type (fs, metadata, pending, device, inum) with progress
- Current phase (scan, btree, phys, normal) and priority (hipri, normal, pending)
- Work position

## Rate Limiting and Efficiency

### IO Clock-Based Waiting (work.c:943-962)

After completing all work, reconcile waits:
- Duration: `min_member_capacity >> 6` (approx 1.5% of smallest device capacity)
- Uses IO clock (`atomic64_read(&c->io_clock[WRITE].now)`) for fairness
- Ensures foreground IO not starved

### Buffer Strategy

- Work entries buffered in darray (1024 entries) to avoid write buffer thrashing
- Reverse iteration order for LIFO processing

### Physical Btree Optimization

For rotational devices:
- Extents tracked by device+LBA in `reconcile_work_phys`
- Multi-threaded processing: one thread per rotational device (work.c:1025-1074)
- Sequential LBA access minimizes seeks

### Pending List Optimization (work.c:559-577)

If extent on `reconcile_pending` list is on rotational device:
- Removed from pending list
- Will be picked up later by `reconcile_work_phys` in LBA order
- Avoids random seeks retrying failed moves

## Power Management (work.c:1423-1453)

If `CONFIG_POWER_SUPPLY`:
- Registers power supply notifier (`bch2_reconcile_power_notifier()`)
- Tracks battery state in `c->reconcile.on_battery`
- Pauses if on battery and `reconcile_on_ac_only` set

## Error Handling

### Pending State

Moves fail with:
- `BCH_ERR_data_update_fail_no_rw_devs`: No writable devices
- `BCH_ERR_insufficient_devices`: Can't satisfy replication
- `ENOSPC`: No space

On failure:
- `bch2_extent_reconcile_pending_mod()` (work.c:467-503): Sets `pending` flag
- Extent moved to `reconcile_pending` btree
- Event trace logged (`reconcile_set_pending`, work.c:517-526)

### Fsck Pass (check.c)

`check_reconcile_work` recovery pass:
- Verifies work btrees consistent with extent state
- Rebuilds work entries if missing
- Deletes stale entries

### Validation (trigger.c:21-42)

`bch2_extent_reconcile_validate()`:
- `pending` requires `need_rb`
- `hipri` requires `data_replicas` in `need_rb`
- `data_replicas` must be non-zero

## Event Tracing

Reconcile uses event tracing for debugging (work.c):
- `reconcile_clear_scan`: Cookie deletion (186)
- `reconcile_set_pending`: Work marked pending (517)
- `reconcile_data`: Data extent processed (625)
- `reconcile_phys`: Physical extent processed (673)
- `reconcile_btree`: Btree node processed (707)
- `reconcile_scan_*`: Per-scan-type events (727)

## Backpointer Flags

In `struct bch_backpointer` (bcachefs_format.h:495):
- `BACKPOINTER_RECONCILE_PHYS` (bits 0-1): Physical reconcile priority
  - Maps work ID to physical btree for rotational device tracking

## Version History

- **v1.33** (`BCH_VERSION(1, 33)`, bcachefs_format.h:847): Introduced reconcile subsystem
- Replaced older rebalance architecture (single btree, no scan cookies, no physical tracking)
- Commands updated: `bcachefs data rereplicate` deprecated (cmd_data.c:47-49)

## Source References

Kernel:
- `fs/bcachefs/data/reconcile/format.h`: Format structures, btree IDs
- `fs/bcachefs/data/reconcile/work.c:76-112`: Scan cookie encode/decode
- `fs/bcachefs/data/reconcile/work.c:198-261`: Work entry buffering
- `fs/bcachefs/data/reconcile/work.c:283-456`: Set data opts (decide what to kill/add)
- `fs/bcachefs/data/reconcile/work.c:530-602`: Move extent with error handling
- `fs/bcachefs/data/reconcile/work.c:971-1012`: Phase definitions
- `fs/bcachefs/data/reconcile/work.c:1125-1266`: Main reconcile loop
- `fs/bcachefs/data/reconcile/trigger.c:368-435`: Btree trigger
- `fs/bcachefs/data/reconcile/trigger.c:448-574`: Needs reconcile check
- `fs/bcachefs/data/reconcile/trigger.c:743-808`: Set needs reconcile
- `fs/bcachefs/opts.h:501-511`: Reconcile options

Userspace:
- `c_src/cmd_data.c:386-617`: Reconcile wait/status commands
