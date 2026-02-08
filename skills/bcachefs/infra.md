# bcachefs Infrastructure

## Error Codes (init/errcode.h)

400+ bcachefs-specific error codes built on Linux errnos.
Hierarchical: `BCH_ERR_*` with parent classes (e.g., `ENOSPC_*`, `ENOMEM_*`).
`bch2_err_matches(err, class)` for class-based matching.
`bch_err_throw(c, name)` to create errors with filesystem context.
`bch2_err_class(err)` strips bcachefs-specific codes back to POSIX errno.

## Error Handling (init/error.h, error.c)

Three categories:
1. **Inconsistency errors**: `bch2_fs_inconsistent()` - metadata corruption detected
2. **Fsck errors**: `fsck_err()` macro - fixable during fsck, configurable action
3. **Fatal errors**: `bch2_fatal_error()` - unrecoverable
4. **IO errors**: per-device tracking, triggers device state changes

`bch2_run_explicit_recovery_pass()`: schedules a recovery pass at runtime.

## Fsck (fs/check.h, check.c)

Main check functions (each a recovery pass):
- `bch2_check_inodes()` - validate inode metadata
- `bch2_check_extents()` - validate extent pointers
- `bch2_check_indirect_extents()` - validate reflink extents
- `bch2_check_dirents()` - validate directory entries
- `bch2_check_xattrs()` - validate extended attributes
- `bch2_check_root()` - validate root inode
- `bch2_check_subvolume_structure()` - validate subvolume tree
- `bch2_check_unreachable_inodes()` - find orphaned inodes
- `bch2_check_directory_structure()` - validate dir tree
- `bch2_check_nlinks()` - validate link counts

Uses `inode_walker` for snapshot-aware inode traversal.
`fsck_err()` / `fsck_err_on()` macros for conditional error reporting/fixing.
Same implementation runs in userspace (`bcachefs fsck`) and kernel (`mount -o fsck`).

## Printbuf (util/printbuf.h)

Dynamic string buffer for all text formatting throughout bcachefs.
- `prt_printf()`, `prt_str()`, `prt_char()`, `prt_newline()`
- Indentation: `prt_indent_add()` / `prt_indent_sub()`
- Tabstops: `prt_tab()`, `prt_tab_rjust()`
- `prt_human_readable_s64()` / `_u64()` - human-readable sizes
- `printbuf_reset()`, `CLASS(printbuf, name)()` for RAII cleanup
- `PRINTBUF` macro for stack allocation

## Sysfs (debug/sysfs.c)

Macro-based: `SHOW(fn)` / `STORE(fn)` generate sysfs ops using printbuf.
At `/sys/fs/bcachefs/<uuid>/`:
- `options/` - all runtime-changeable options
- Time stats for: blocked_allocate, journal ops, btree ops, data IO
- Internals: btree cache, dirty nodes, transactions, journal pins, open buckets

## Recovery Passes (init/passes_format.h)

~48 passes defined in `BCH_RECOVERY_PASSES()` macro.
Each entry: `x(name, stable_id, flags, dependencies)`.
Two enums: `bch_recovery_pass` (execution order), `bch_recovery_pass_stable` (persistent ID).
Flags: `PASS_SILENT`, `PASS_FSCK`, `PASS_UNCLEAN`, `PASS_ALWAYS`, `PASS_ONLINE`, `PASS_ALLOC`, `PASS_NODEFER`.
Dependencies as bitmask: pass won't run until dependencies complete.
Runtime state in `struct bch_fs_recovery`: current_pass, passes_complete, passes_failing.

## Six Locks (util/six.h)

Three-state lock: read / intent / write.
- Read: shared, multiple readers
- Intent: blocks other intents (serializes writers), allows readers
- Write: exclusive, blocks everything
Sequence numbers for optimistic relocking. Waiter lists per lock.

## Darray (util/darray.h)

Type-safe dynamic arrays: `DARRAY(type)`, `DARRAY_PREALLOCATED(type, n)`.
`darray_push()`, `darray_pop()`, `darray_top()`.
`darray_for_each()`, `darray_for_each_reverse()`.
`darray_sort()`, `darray_find()`.
`CLASS(darray, name)()` for RAII.

## Closure (vendor/closure.h)

Async completion mechanism: refcounting + wait lists + continuation.
`closure_init()`, `closure_get()`, `closure_put()`.
`CLOSURE_CALLBACK(fn)` macro. `closure_wake_up()` for waiters.
Used for in-flight IO tracking.

## Thread with File (util/thread_with_file.h)

Background thread with stdio redirected to file descriptors.
Used for long-running operations (fsck, data jobs) that need user interaction.
Returns fd to userspace; read for output, close to cancel.
