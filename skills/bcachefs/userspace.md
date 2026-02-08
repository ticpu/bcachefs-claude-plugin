# bcachefs-tools Userspace

## Project Structure

```
src/                Rust CLI (bcachefs.rs entry point, commands/, key.rs, device_scan.rs)
c_src/              C commands (cmd_*.c, bcachefs.c dispatcher)
libbcachefs/        Kernel code compiled for userspace (static lib)
bch_bindgen/        Rust FFI bindings (bindgen-generated)
build.rs            Rust build script (links C libs)
Makefile            Build orchestration (C + Cargo)
```

## Binary

Single unified binary `bcachefs` (symlink to `target/release/bcachefs`).
Symlinks: `mkfs.bcachefs`, `fsck.bcachefs`, `mount.bcachefs`.

## Command Dispatch (src/bcachefs.rs)

Routes to Rust or C implementations:
- **Rust**: mount, list, subvolume, completions
- **C (via FFI)**: format, fsck, dump, device ops, data ops, etc.

## Rust Components

- `src/commands/mount.rs` - `libc::mount()` syscall, option parsing
- `src/commands/list.rs` - btree iteration via BtreeTrans/BtreeIter
- `src/commands/subvolume.rs` - create/delete/snapshot via ioctl
- `src/key.rs` - encryption key handling (UnlockPolicy, KeyHandle, keyutils)
- `src/device_scan.rs` - udev integration for device discovery by UUID

## C Commands (c_src/)

- `cmd_format.c` - `bcachefs format`
- `cmd_fsck.c` - `bcachefs fsck`
- `cmd_device.c` - device add/remove/online/offline/evacuate/set-state/resize
- `cmd_dump.c` / `cmd_list_journal.c` - offline metadata inspection
- `cmd_image.c` - disk image create/update
- `cmd_migrate.c` - migrate existing fs to bcachefs
- `cmd_top.c` - runtime performance stats
- `cmd_key.c` / `cmd_unlock.c` - passphrase management
- `cmd_fusemount.c` - optional FUSE mount

## FFI Bindings (bch_bindgen/)

`build.rs` uses bindgen crate. Allowlist: `cmd_*`, `bch2_*`, `bio_*`.
Rust wrappers: `Bpos` (Ord/Eq/FromStr), `btree_id` (Display/FromStr),
`printbuf` (Drop), `path_to_cstr()`.

## Build

`make` drives both C compilation (`-std=gnu11 -O2`) and `cargo build`.
Links: libbcachefs.a (whole-archive) + urcu, zstd, blkid, uuid, sodium,
zlib, lz4, udev, keyutils, aio. Optional: fuse3.

## Kernel Interaction

- Mount: `libc::mount()` syscall with bcachefs-specific options
- Ioctls: `BCH_IOCTL_SUBVOLUME_CREATE`/`DESTROY` via `libc::ioctl()`
- Device discovery: udev + superblock UUID matching
- Encryption: kernel keyring via keyutils (add_key, keyctl_search)

## Key Dependencies (Cargo.toml)

clap, anyhow, libc, uuid, udev, errno, bch_bindgen, rustix, zeroize, strum.
Package: bcachefs-tools v1.36.0, edition 2021, MSRV 1.77.0.
