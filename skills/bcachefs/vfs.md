# bcachefs VFS Layer

## Inodes

### Files

- `fs/inode_format.h` - on-disk `bch_inode_v3`, field macros
- `fs/inode.h` / `inode.c` - pack/unpack, CRUD operations

### On-Disk Format (inode_v3, current)

Fixed header: `bi_journal_seq`, `bi_hash_seed`, `bi_flags`, `bi_sectors`, `bi_size`, `bi_version`
Variable fields (varint-encoded, `BCH_INODE_FIELDS_v3()`):
timestamps (96-bit), uid/gid, nlink, generation, dev, storage options
(checksum, compression, replicas, targets, erasure_code), bi_dir/bi_dir_offset
(parent backpointer), bi_subvol/bi_parent_subvol, nocow, depth, casefold.

Mode stored in `bi_flags` bits 36-52 (INODEv3_MODE bitmask).

### Key Operations

- `bch2_inode_peek()` / `bch2_inode_write()` - read/update in btree
- `bch2_inode_create()` - create with snapshot logic
- `bch2_inode_rm()` - delete inode
- `bch2_inode_pack()` / `bch2_inode_unpack()` - varint encoding

### Inode Flags (`BCH_INODE_FLAGS()`)

`sync`(0), `immutable`(1), `append`(2), `nodump`(3), `noatime`(4),
`i_size_dirty`(5), `i_sectors_dirty`(6), `unlinked`(7),
`backptr_untrusted`(8), `has_child_snapshot`(9),
`has_case_insensitive`(10), `31bit_dirent_offset`(11)

### Per-Inode Options (`BCH_INODE_OPTS()`)

Stored with +1 bias (0 = use fs default): data_checksum, compression,
project, background_compression, data_replicas, promote_target,
foreground_target, background_target, erasure_code, nocow, inodes_32bit, casefold.

Note: `casefold` and `inodes_32bit` can only be toggled on empty directories.
`inodes_32bit` silently disables `shard_inode_numbers_bits` (too few bits
for sharding in 32-bit space).

## Directories

### Files

- `fs/dirent_format.h` - `struct bch_dirent`
- `fs/dirent.h` / `dirent.c` - hash lookup, CRUD

### Design

Hash-based: offset = truncated SHA1 of name. Linear probing for collisions.
Whiteouts for deletions. Provides readdir cookies.

`DT_SUBVOL` (16): special type for subvolume mount points, stores
`d_child_subvol` + `d_parent_subvol` instead of `d_inum`.

Casefold support: both regular and case-folded name stored (`d_cf_name_block`).
Unicode version for case folding is hardcoded to 12.1.0 (`CONFIG_UNICODE`).
Can only be enabled on empty directories; existing entries cannot be converted.

### Key Operations

- `bch2_dirent_lookup()` - hash-based lookup
- `bch2_dirent_create()` - create entry
- `bch2_dirent_rename()` - supports RENAME, RENAME_OVERWRITE, RENAME_EXCHANGE
- `bch2_empty_dir_snapshot()` - snapshot-aware empty check

## Extended Attributes

### Files: `fs/xattr_format.h`, `fs/xattr.h` / `xattr.c`

Hash-based like dirents. Key = hash(type || name).
Types: USER, POSIX_ACL_ACCESS, POSIX_ACL_DEFAULT, TRUSTED, SECURITY.
ACLs stored as xattrs (`fs/acl.c`).

## Snapshots & Subvolumes

See [snapshots.md](snapshots.md) for deep dive. Key concepts:
- Every btree key has snapshot dimension in bpos
- Snapshot IDs form tree with skiplist for O(log n) ancestor queries
- Subvolumes addressed as `subvol_inum` = (subvolume_id, inode_number)
- Root subvolume = `BCACHEFS_ROOT_SUBVOL` (1)

## Inode-to-Path Resolution

Every inode (v2/v3) stores a **single parent backpointer**: `bi_dir` (parent inode
number) and `bi_dir_offset` (position in parent's dirents btree). Together they
point directly to the dirent that names this inode.

### Backpointer Lifecycle (fs/namei.c)

- **Create** (namei.c:149-150): set to parent dir + dirent offset
- **Unlink** (namei.c:279-283): cleared to 0/0 only if the backpointer matches
  the dirent being removed (hardlinks: only the matching link clears it)
- **Rename** (namei.c:402-414): updated to new parent + new offset
- **Check** (fs/inode.h:260-263): `bch2_inode_has_backpointer()` tests non-zero

### Path Resolution (fs/check_dir_structure.c:539-644)

`bch2_inum_to_path()` walks up the tree:
1. Read inode, get `bi_dir` + `bi_dir_offset`
2. Look up dirent at `SPOS(bi_dir, bi_dir_offset, snapshot)` → filename
3. Move to parent inode (`inum = bi_dir`), repeat until root

Returns `ENOENT_inode_no_backpointer` if any inode in the chain has no
backpointer (orphan, or hardlinked file whose tracked link was removed).

### NFS Export (vfs/fs.c:1635-1752)

Full NFS export support via `bch_export_ops`:
- `bch2_get_parent()` (fs.c:1635-1647): returns parent via `bi_dir`
- `bch2_get_name()` (fs.c:1649-1744): fast path uses `bi_dir_offset` for
  O(1) dirent lookup; falls back to **linear scan** of parent's dirents for
  hardlinked files where the backpointer points elsewhere
- `bch2_encode_fh()` / `bch2_fh_to_dentry()` / `bch2_fh_to_parent()`

### Hardlink Limitation

`bi_dir`/`bi_dir_offset` stores only **one** backpointer. Files with multiple
hardlinks have the backpointer set to whichever link was created or renamed
last. There is no reverse index from child inode to all parent dirents.
`bch2_get_name()` handles this by falling back to iterating all dirents in the
target parent directory.

### Fsck Validation (fs/check.c)

`bch2_check_dirents` verifies bidirectional consistency: for each dirent, the
target inode's `bi_dir`/`bi_dir_offset` should point back to that dirent
(checked via `inode_points_to_dirent()` in namei.h:65-70). Mismatches are
repaired.

### No Userspace Interface

Neither an ioctl nor a CLI command exposes inode-to-path resolution.
`bch2_inum_to_path()` is only called kernel-internally: fsck checks, error
reporting (`init/error.c`), NFS exports, snapshot/subvolume listing, and debug
messages. The scrub command logs affected file paths only to the kernel log.

Missing: a `BCH_IOCTL_INUM_TO_PATH` ioctl and `bcachefs inum-to-path` CLI
command. The kernel implementation is complete.

## Namei Operations (fs/namei.h)

- `bch2_create_trans()` - create file/dir/subvol/snapshot
- `bch2_link_trans()` / `bch2_unlink_trans()` / `bch2_rename_trans()`
- `bch2_inum_to_path()` - resolve path from inum (see above)
- `bch2_reinherit_attrs()` - propagate parent attrs to child

## Quotas (fs/quota.h, quota_format.h)

Types: USR, GRP, PRJ. Counters: SPC (space), INO (inodes).
`bch2_quota_acct()`, `bch2_quota_transfer()`. Conditional: `CONFIG_BCACHEFS_QUOTA`.
Project ID inherited on create/rename; `-EXDEV` on cross-project rename.

## Encryption (data/checksum.h)

ChaCha20/Poly1305 AEAD. Nonce domains: `BCH_NONCE_EXTENT`(1<<28),
`BCH_NONCE_BTREE`(2<<28), `BCH_NONCE_JOURNAL`(3<<28).
Checksum types: none, crc32c, crc64, xxhash, chacha20_poly1305_{80,128}.
Key from superblock `bch_sb_field_crypt`, scrypt KDF.
