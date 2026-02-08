---
name: bcachefs
description: >
  bcachefs filesystem expert reference — ALWAYS load this skill before answering any
  technical question about bcachefs. Covers: architecture, on-disk format, all 27 btrees,
  bpos/bkey layout, six-locks, COW design, transactions & restarts, journal/WAL,
  allocator & buckets, snapshots & subvolumes, encryption (ChaCha20/Poly1305), fsck &
  recovery passes, VFS layer (inodes/dirents/xattrs), error codes, memory management,
  reconcile/rebalance, metadata versions, and userspace tools (bcachefs-tools, Rust+C).
  Use for: writing/reviewing bcachefs code, debugging kernel issues, understanding
  internals, navigating fs/bcachefs/ source, or any bcachefs-related question.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Task
---

# bcachefs Development Assistant

You are helping with bcachefs filesystem development. bcachefs is a COW filesystem
built on a relational-database-like btree layer, descended from bcache.

## Source Locations

- **Kernel source**: Check CLAUDE.md or project settings for the user's kernel tree path.
  If not configured, ask the user. The `KERNEL_VERSION` file at plugin root records the
  commit these docs were written against.
- **Userspace tools**: https://evilpiepirate.org/git/bcachefs-tools.git
- **Principles of Operation**: `doc/bcachefs-principles-of-operation.tex` in bcachefs-tools.
  This is the **primary reference document** for bcachefs — read it first when working on
  any bcachefs subsystem. It covers design decisions, on-disk structures, algorithms, and
  trade-offs in depth. Located in the bcachefs-tools repo; if available in the user's
  working directory, always consult it before the skill reference docs below.

## Source Layout (fs/bcachefs/)

```
btree/          B-tree core (iter, update, cache, locking, write_buffer)
alloc/          Allocator (foreground, background, buckets, disk_groups)
data/           IO paths (read, write, move, copygc, compress, checksum, ec/, reconcile/)
journal/        WAL (journal, write, reclaim, seq_blacklist)
fs/             VFS layer (inode, dirent, xattr, check, quota, acl, namei)
snapshots/      Snapshots and subvolumes
sb/             Superblock (io, members, counters, errors)
init/           Recovery (recovery, passes, error, errcode, fs)
debug/          Sysfs, tests
util/           Utilities (printbuf, six, darray, closure, clock, eytzinger)
```

## Detailed Reference

Read the reference docs bundled with this skill (same directory as this file).
Pick the relevant ones based on $ARGUMENTS before answering questions.

- [architecture.md](architecture.md) - Overall design, btree-centric architecture
- [btree.md](btree.md) - B-tree implementation, node format, iteration
- [transactions.md](transactions.md) - Transaction lifecycle, restarts, locking, triggers
- [allocator.md](allocator.md) - Space allocation, journal, recovery
- [memory_management.md](memory_management.md) - Btree cache, shrinkers, key cache, bio pools
- [snapshots.md](snapshots.md) - Snapshot table, ancestor queries, deletion, subvolumes, whiteouts
- [vfs.md](vfs.md) - Inodes, directories, xattrs, quotas, encryption
- [format.md](format.md) - On-disk format, ioctls, key types
- [infra.md](infra.md) - Error handling, debugging, utilities
- [userspace.md](userspace.md) - bcachefs-tools structure
- [fsck.md](fsck.md) - Recovery passes, fsck checks, journal replay, topology repair
- [journal.md](journal.md) - WAL journal: format, write path, reclaim, replay, blacklisting
- [btrees.md](btrees.md) - All 27 btrees: key types, value formats, properties, relationships
- [reconcile.md](reconcile.md) - Reconcile (rebalance): background data movement, 7 btrees, work lifecycle
- [versions.md](versions.md) - All 47 metadata versions (0.10-1.36): format changes, feature introductions
- [encryption.md](encryption.md) - ChaCha20/Poly1305 AEAD, key hierarchy, keyring pain, nonce vulnerability

## Key Architecture Points

- **27 separate btrees** for different data types (extents, inodes, dirents, xattrs, alloc, snapshots, etc.)
- **bpos** = (inode, offset, snapshot) - three-dimensional sort key
- **Large COW btree nodes** (128K-256K) with log-structured bsets
- **Six-locks** (read/intent/write) - btree locks never held during IO
- **Journal (WAL)** for crash recovery; btree nodes written lazily
- **Snapshots via key versioning** - snapshot ID part of every bpos, not COW-btree cloning
- **Bucket-based allocation** with copy-GC for fragmentation
- **400+ error codes** hierarchical on Linux errnos (`bch2_err_matches()`)
- **~48 recovery passes** with dependency tracking

## Userspace Tools

Single binary `bcachefs` (Rust + C). Symlinks: mkfs/fsck/mount.bcachefs.

- **Rust**: mount, list, subvolume, completions (`src/commands/`)
- **C**: format, fsck, dump, device ops, data ops (`c_src/cmd_*.c`)
- **FFI**: `bch_bindgen/` (bindgen-generated)

Build: `make` (C + Cargo)
Test: `cargo check` -> `cargo clippy --fix --allow-dirty` -> `cargo test`

## Guidelines

- Read code comments carefully - bcachefs has many edge cases
- Use `git blame`/`git log` to understand reasoning behind changes
- When searching, check the user's kernel checkout (path from CLAUDE.md or ask)
- Reference specific files and line numbers in answers
- bcachefs uses printbuf everywhere for text formatting (`prt_printf()` etc.)
- Error handling: `fsck_err()` for fixable, `bch2_fs_inconsistent()` for corruption
- Transactions restart on lock conflicts - check for `transaction_restart`
