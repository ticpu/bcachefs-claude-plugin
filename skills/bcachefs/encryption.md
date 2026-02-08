# bcachefs Encryption

## Overview

bcachefs implements authenticated encryption using ChaCha20/Poly1305 (AEAD).
All data and metadata except the superblock is encrypted when enabled.
Encryption is all-or-nothing at the filesystem level: no per-file or
per-directory control, because btree nodes are shared structures.

**Critical exception**: `nocow` data is stored UNENCRYPTED even on encrypted
filesystems. The nocow write path bypasses data path transformations including
encryption. This is because AEAD encryption is fundamentally incompatible with
mutable data.

Encryption can only be enabled at format time; post-format enablement is not
implemented (`bch2_enable_encryption()` exists but is not hooked up).

## Files

Kernel (`fs/bcachefs/`):
- `data/checksum.c` - ChaCha20/Poly1305 implementation, key request, encrypt/decrypt
- `data/checksum.h` - nonce derivation (`extent_nonce()`), nonce domains
- `data/extents_format.h` - CRC entry formats storing MAC + nonce
- `data/extents_types.h` - `bch_extent_crc_unpacked` with nonce/csum fields
- `data/read.c` - read path decryption
- `data/write.c` - write path encryption
- `bcachefs_format.h:698-743` - `bch_key`, `bch_encrypted_key`, `bch_sb_field_crypt`
- `bcachefs.h:869-870` - `chacha20_key`, `chacha20_key_set` in `struct bch_fs`

Userspace (`bcachefs-tools/`):
- `src/key.rs` - Rust key management (unlock, passphrase, keyring)
- `c_src/crypto.c` - scrypt KDF, passphrase verification
- `src/commands/mount.rs:125-153` - mount unlock flow

## Key Hierarchy

Three levels:

1. **Passphrase** → scrypt KDF → **passphrase key** (256-bit)
   - Never stored on disk
   - scrypt parameters (N, R, P) stored in `bch_sb_field_crypt.kdf_flags`
   - Default: N=2^14, R=2^3, P=2^4
   - Salt: hardcoded "bcache" (6 bytes)
   - KDF runs entirely in userspace

2. **Passphrase key** decrypts → **master key** (256-bit)
   - Master key stored encrypted in superblock (`bch_sb_field_crypt.key`)
   - Magic value `BCH_KEY_MAGIC` (0x6263682a2a6b6579, "bch**key") validates correct passphrase
   - Decryption: `chacha20(passphrase_key, sb_key_nonce, &encrypted_key)`
   - Changing passphrase re-encrypts only the master key, not data

3. **Master key** → per-extent encryption via nonces
   - Cached in `c->chacha20_key` at mount time
   - Single key for all data and metadata
   - No key rotation mechanism

## Nonce Derivation

Per-extent nonces derived from on-disk state (`checksum.h:204-220`):

```
nonce[0] = uncompressed_size << 22  (if compressed)
nonce[1] = bversion.lo[31:0]
nonce[2] = bversion.lo[63:32]
nonce[3] = bversion.hi | (compression_type << 24) ^ BCH_NONCE_EXTENT
```

Then: `nonce_add(nonce, crc.nonce << 9)` — advances by CRC nonce offset * 512 bytes.

Nonce domains (XORed into nonce[3]):
- `BCH_NONCE_EXTENT` (1<<28): data extents
- `BCH_NONCE_BTREE` (2<<28): btree nodes
- `BCH_NONCE_JOURNAL` (3<<28): journal entries
- `BCH_NONCE_POLY`: Poly1305 MAC key derivation

When extents are split (`rechecksum`, `checksum.c:336-394`), the nonce advances
by the split length in sectors. This maintains unique nonces for sub-extents.

## ChaCha20/Poly1305 Implementation

**Encryption** (`checksum.c:170-182`):
- `bch2_encrypt()` / `bch2_encrypt_bio()`: in-place ChaCha20 with master key + nonce

**Authentication** (`checksum.c:121-130`):
- Poly1305 key derived per-extent: `chacha20(master_key, nonce ^ BCH_NONCE_POLY, zero_key)`
- MAC computed over plaintext (encrypt-then-MAC: MAC computed, then data encrypted)

**MAC storage** in extent CRC entries:
- `bch_extent_crc32`: no encryption support (32-bit checksum only)
- `bch_extent_crc64`: 80-bit MAC (truncated, `chacha20_poly1305_80`)
- `bch_extent_crc128`: full 128-bit MAC (`chacha20_poly1305_128`)

Selection (`checksum.h:132-163`):
- If encryption enabled: always uses chacha20_poly1305
- `wide_macs` option → 128-bit MAC (all extents), default → 80-bit MAC (data)
- Metadata always uses 128-bit MACs regardless of `wide_macs`

## Kernel Keyring Integration

### Key Format

- Type: `user` (standard kernel key type)
- Description: `bcachefs:<filesystem-UUID>`
- Payload: 32 bytes (the passphrase-derived key, NOT the master key)

### Unlock Flow

1. Userspace reads superblock, checks `bch2_sb_is_encrypted()`
2. Prompts for passphrase (or reads from systemd-ask-password)
3. Derives passphrase key via scrypt (`c_src/crypto.c:79-103`)
4. Verifies by decrypting master key, checking `BCH_KEY_MAGIC`
5. Adds passphrase key to kernel keyring (`key.rs:74-100`):
   `add_key("user", "bcachefs:<UUID>", &passphrase_key, 32, KEY_SPEC_USER_KEYRING)`
6. Kernel mount calls `bch2_fs_encryption_init()` (`checksum.c:673-683`)
7. Kernel calls `request_key(&key_type_user, "bcachefs:<UUID>")` (`checksum.c:438-518`)
8. Kernel decrypts master key from superblock, caches in `c->chacha20_key`

### Keyring Search Order

Kernel `__bch2_request_key()` searches (`checksum.c:468-478`):
1. `KEY_SPEC_SESSION_KEYRING`
2. `KEY_SPEC_USER_KEYRING`
3. `KEY_SPEC_USER_SESSION_KEYRING`

Userspace `add_key()` targets (`key.rs:119-121`):
- `KEY_SPEC_USER_KEYRING` (preferred, persistent across sessions)
- `KEY_SPEC_SESSION_KEYRING` (session-local)

### Pain Points

**Session isolation**: Keys added to a session keyring are invisible from other
sessions of the same user. SSH session A's key is not visible to SSH session B,
to systemd mount units, or to cron jobs. Must use `KEY_SPEC_USER_KEYRING` for
cross-session visibility.

**Boot ordering**: Encrypted root filesystem needs the key before `pivot_root`.
Requires initramfs hooks (`initramfs/hook.in`), the `bcachefs` binary in the
initramfs, crypto kernel modules (chacha20, poly1305), and
`systemd-ask-password` integration (`key.rs:158-176`). Failed prompt = emergency
shell.

**Privilege boundaries**: `sudo mount` uses root's keyring, not the calling
user's. Systemd units run in isolated session contexts with their own keyrings.
The key must be placed in a keyring the mounting process can see.

**No re-keying**: Master key cached in `struct bch_fs` at mount time, never
re-requested from keyring. Keyring key expiry has no effect on running mounts
but breaks new mounts/fsck until key is re-added.

**systemd-ask-password** (`key.rs:158-176`):
- Used for passphrase prompting during boot
- `--id=bcachefs:<UUID>` for dedup
- `--accept-cached` to reuse previously entered passphrase
- `--keyname=<UUID>` for kernel keyring caching

## Nonce Reuse Vulnerability

### External Snapshots

AEAD requires unique (key, nonce) pairs. bcachefs derives nonces from `bversion`
which is unique within a single filesystem instance. If the underlying storage
is externally snapshotted (LVM, ZFS zvol, VM snapshot) and the snapshot mounted
read-write, both instances share the same master key and derive the same nonces
for writes to the same logical locations.

**Attack**: XOR ciphertexts from both instances → XOR of plaintexts.
If one plaintext is known (or has structure), the other is recoverable.

**Not affected**: bcachefs internal snapshots (COW gives new bversion → new nonce).

**Mitigations**:
- Never mount a snapshot of an externally-snapshotted encrypted volume read-write — keep external snapshots linear, not branching. The most recent (non-snapshot) version is safe; older snapshots must use `-o nochanges`.
- Place LUKS between snapshot layer and bcachefs (hides ciphertexts)

### Nonce Consistency in Rechecksum

Nonce management during extent splitting has been a source of bugs:
- `bch2_write_prep_encoded_data()` nonce inconsistency after partial overwrite + rechecksum
- Nonce advancement must exactly match the trimmed extent range
- `op->nonce` logged on data update inconsistency for debugging

## Limitations

- **No key rotation**: Single master key, lifetime of filesystem
- **No multi-key**: One passphrase, one key. No key-per-user or role-based access
- **No key escrow**: Lost passphrase = permanent data loss
- **No hardware key support**: No TPM, FIDO2, or smartcard integration
- **No AES-GCM**: ChaCha20 only (fast in software, but no AES-NI acceleration)
- **Enable/disable stubs**: `bch2_enable_encryption()` / `bch2_disable_encryption()`
  exist in `checksum.c:573-665` but are `#if 0` (not hooked up)

## See Also

- [format.md](format.md) - On-disk format details
- [architecture.md](architecture.md) - Overall bcachefs design
