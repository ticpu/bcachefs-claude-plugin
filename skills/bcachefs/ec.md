# bcachefs Erasure Coding Deep Dive

## Files

- `data/ec/ec.c` - Core EC helpers
- `data/ec/ec.h` - EC types and declarations
- `data/ec/create.c` - Stripe creation, allocation, reuse
- `data/ec/io.c` - Parity generation/recovery (raid5/raid6)
- `data/ec/format.h` - On-disk stripe structure
- `data/ec/trigger.c` - Stripe btree triggers, LRU fragmentation tracking
- `data/ec/trigger.h` - `stripe_lru_pos()`, inline helpers
- `alloc/foreground.c` - `bucket_alloc_from_stripe()` EC bucket allocation
- `alloc/foreground.h` - `alloc_request_get()` EC gating (`ec_replicas < 2`)

## On-Disk Structure

### `bch_stripe` (format.h:7-51)

```
sectors          (__le16)  - stripe size in sectors (= bucket size)
algorithm        (4 bits)  - parity algorithm
needs_reconcile  (1 bit)   - stripe needs reconcile (degraded/repair)
unused           (3 bits)
nr_blocks        (__u8)    - total blocks (data + parity)
nr_redundant     (__u8)    - parity block count (1 or 2)
csum_granularity_bits (__u8)
csum_type        (__u8)
disk_label       (__u8)
ptrs[]           - variable-length: bch_extent_ptr array, then checksums, then block sector counts
```

Stored in stripes btree (ID 6), keyed by `POS(0, stripe_index)`.

## Stripe Geometry

### Redundancy

- `redundancy = data_replicas - 1` (create.c:1288)
- `data_replicas=1`: EC disabled (`ec_replicas < 2` check in foreground.h:255)
- `data_replicas=2`: 1 parity block (RAID-5 / XOR)
- `data_replicas=3`: 2 parity blocks (RAID-6 / Reed-Solomon P+Q)
- Hard limit: `BUG_ON(np > 2)` in `raid_gen()` (io.c:49)

### Width

- Dynamic: `min(nr_active_devs, BCH_BKEY_PTRS_MAX=16)` total blocks
- `nr_data = total - redundancy`, `nr_parity = redundancy`
- No configuration to limit width to subset of available devices
- Each block = one bucket on one device

### Device Eligibility

`disk_label_ec_devs()` (create.c:665):
1. `target_rw_devs()` - read-write devices matching disk label target
2. Filter out `durability == 0` (cache devices)
3. `pick_blocksize()` (create.c:633) - most common bucket size among candidates
4. Filter out devices with different bucket size

### Minimum Devices

`h->insufficient_devs = h->nr_active_devs < h->redundancy + 2` (create.c:1184)
- RAID-5: minimum 3 devices
- RAID-6: minimum 4 devices
- With only `redundancy + 1` devices, replication is more efficient than EC

## Stripe Head

### `ec_stripe_head` (ec.h)

Per-configuration stripe accumulator, keyed by `(disk_label, algorithm, redundancy, watermark)`.

### `bch2_ec_stripe_head_get()` (create.c:1247)

1. Finds or creates stripe head for the configuration
2. `ec_stripe_head_devs_update()` (create.c:1173) recalculates device set
3. If insufficient devices, returns NULL (EC disabled for this request)
4. Calls `ec_new_stripe_alloc()` if no active stripe
5. Returns open bucket from stripe for the write

### `ec_new_stripe_alloc()` (create.c:1307)

Allocates `ec_stripe_new` with all buckets pre-allocated:
- Data buckets: `nr_data` slots
- Parity buckets: `nr_parity` slots
- Uses `BCH_WATERMARK_stripe` for allocation
- Falls back to `stripe_reuse()` if allocation fails

## Write Lifecycle

### Step 1: Bucket allocation

`bucket_alloc_from_stripe()` (foreground.c:764) hands out one data bucket from the active stripe to each write. Sets `ob->ec` linking to `ec_stripe_new`.

### Step 2: Data buffering

`bch2_writepoint_ec_buf()` (create.c:610) copies write data into in-memory stripe buffer for later parity computation.

### Step 3: Stripe completion

When all data blocks allocated: `ec_stripe_new_set_pending()` (create.c:1214).

### Step 4: Parity and commit

`ec_stripe_create()` (create.c:512):
1. `zero_out_rest_of_ec_bucket()` - pad partially-filled data buckets
2. If reusing old stripe: read old block data
3. `bch2_ec_generate_ec()` → `raid_gen()` - generate parity
4. `bch2_ec_generate_checksums()` - checksum parity blocks
5. Write parity blocks to disk
6. Insert stripe key into stripes btree
7. `stripe_update_extents()` - walk backpointers for each data bucket, add `bch_extent_stripe_ptr` to data extents

## Stripe Fragmentation and Reuse

### Fragmentation Tracking

`stripe_lru_pos()` (trigger.h:93):
- All data blocks empty → `STRIPE_LRU_POS_EMPTY` (1) → auto-delete
- No empty blocks → 0 (not in LRU)
- Partially empty → `LRU_TIME_MAX - blocks_empty` (lower = reuse first)

Tracked in LRU btree under `BCH_LRU_STRIPE_FRAGMENTATION`.

### Auto-Deletion of Empty Stripes

Two paths:
- Trigger inline: `bch2_trigger_stripe()` (trigger.c:355) converts to `KEY_TYPE_deleted` if empty and not open
- Worker: `bch2_ec_stripe_delete_work()` (create.c:91) scans LRU for `STRIPE_LRU_POS_EMPTY`

### Stripe Reuse

`stripe_reuse()` (create.c:958):
1. Scans fragmentation LRU (starting at position 2, skipping empty range)
2. `may_reuse_stripe()` (create.c:861) checks: same disk_label, algorithm, nr_redundant
3. `init_new_stripe_from_old()` (create.c:922): copies non-empty blocks, allocates fresh buckets for empty slots

## Re-striping via Reconcile

`erasure_code` is a reconcile option (`BCH_RECONCILE_OPTS()` in reconcile/format.h:150).

Detection: `bch2_bkey_needs_reconcile()` (reconcile/trigger.c:548):
```c
if (!unwritten && r.erasure_code != ec)
    r.need_rb |= BIT(BCH_RECONCILE_erasure_code);
```

Action (reconcile/work.c:393-413):
- EC wanted: sets `extra_replicas = 1`, `BCH_WRITE_must_ec` → write goes through EC path
- EC not wanted: `ptrs_kill_ec` → removes stripe pointer from extents

## Stripe Repair

### `bch2_stripe_repair()` (create.c:1402)

Repairs degraded stripes with data on invalid/removed devices:
1. Validates stripe and identifies live data blocks
2. Calculates how many blocks need evacuation from bad devices
3. If evacuation needed: evacuates data blocks and returns `-BCH_ERR_stripe_needs_block_evacuate`
4. If no evacuation: allocates a new stripe with recovered blocks, reconstructs lost parity data
5. Manages the stripe state machine through the EC creation process

Called from `do_reconcile_stripe()` (reconcile/work.c:538) when a stripe has `needs_reconcile` set.

### `do_reconcile_stripe()` (reconcile/work.c:538)

Reconcile now handles stripes directly:
1. Checks if key is a stripe with `needs_reconcile` flag set
2. Calls `bch2_stripe_repair()` to perform repair
3. Handles retry logic for block evacuation errors
4. Maintains a stripe retry queue for stripes requiring block evacuation

### Backpointer Flags for EC

Two new flags in `struct bch_backpointer` (bcachefs_format.h:496-497):
- `BACKPOINTER_ERASURE_CODED` (bit 2): Marks EC-coded extents. Reconcile backpointer scans skip these, deferring handling to stripe-level reconciliation.
- `BACKPOINTER_STRIPE_PTR` (bit 3): Marks stripe pointers. Used by `stripe_backpointers` btree (ID 27) for stripe repair on invalid/removed devices.

### `stripe_backpointers` Btree (ID 27)

Write-buffered btree storing `KEY_TYPE_backpointer` entries indexed by stripe pointers. Enables stripe repair for data on `BCH_SB_MEMBER_INVALID` devices. See btrees.md entry 27.

## Fault Isolation

- EC and non-EC writes segregated at open-bucket level
- Non-EC extents never carry `stripe_ptr` entries, never read through EC reconstruction
- Btree nodes explicitly checked: `stripe_update_extent()` (create.c:173) calls `bch2_fs_inconsistent()` if backpointer has `level > 0`
- Crash during stripe creation: extra replicas from staging phase remain valid; reconcile detects and retries

## Related Btrees

- `stripes` (ID 6): stripe metadata, parity checksums
- `bucket_to_stripe` (ID 26): reverse mapping bucket → stripe
- `stripe_backpointers` (ID 27): backpointers for stripe repair on invalid/removed devices
- `alloc` (ID 4): per-bucket `stripe_refcount`, `stripe_sectors`
- `lru` (ID 10): `BCH_LRU_STRIPE_FRAGMENTATION` entries
- `backpointers` (ID 13): stripe blocks point back to stripes btree

## Parity Algorithms

- RAID-5 (1 parity): `raid5_recov()` from Linux `lib/raid6/recov.c`
- RAID-6 (2 parity): `raid6_call.gen_syndrome()` from Linux `lib/raid6/`
- Recovery: `raid_rec()` (io.c:52) handles 1-2 failed blocks
