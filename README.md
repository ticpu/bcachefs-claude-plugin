# bcachefs-dev Claude Code Plugin

Development context plugin for [bcachefs](https://bcachefs.org/) filesystem work.
Provides architecture reference, subsystem documentation, and code navigation
for AI agents working on bcachefs kernel code and userspace tools.

## Install

```
claude plugin marketplace add ticpu/bcachefs-claude-plugin
claude plugin install bcachefs-dev@bcachefs-claude-plugin
```

Restart any running Claude Code instance to pick up the new plugin.

Verify the installation:

```
❯ claude plugin list bcachefs-claude-plugin
Installed plugins:

  ❯ bcachefs-dev@bcachefs-claude-plugin
    Version: 1.1.0
    Scope: user
    Status: ✔ enabled
```

Or load directly for the current session:

```
claude --plugin-dir /path/to/bcachefs-claude-plugin
```

> **Migrating from `bcachefs-dev`?** If you previously installed the old
> marketplace name, remove it first:
> ```
> claude plugin uninstall bcachefs-dev@bcachefs-dev
> claude plugin marketplace remove bcachefs-dev
> ```

## Update

Refresh the marketplace cache then update:

```
claude plugin marketplace update bcachefs-claude-plugin
claude plugin update bcachefs-dev@bcachefs-claude-plugin
```

## Kernel Source

Skill documents reference kernel source files with paths like `btree/types.h:508`
relative to `fs/bcachefs/`. The `KERNEL_VERSION` file records the commit hash
these references were written against.

To make kernel source available for code navigation, point Claude at your
kernel checkout via `--add-dir` or your project's CLAUDE.md:

```
claude --add-dir /path/to/linux/fs/bcachefs
```

Or in your project's `.claude/CLAUDE.md`:

```markdown
- Kernel tree: /path/to/linux
```

## Topics

Use `/bcachefs <topic>` to load context on a subsystem:

- `architecture` - Overall design, 28 btrees, multi-device
- `btree` - Node format, bpos/bkey, iteration, bsets
- `btrees` - All 28 btrees: key types, value formats, properties
- `transactions` - Transaction lifecycle, restarts, locking, triggers
- `allocator` - Buckets, journal, recovery, data IO
- `memory_management` - Btree cache, shrinkers, key cache
- `snapshots` - Snapshot table, ancestor queries, deletion
- `vfs` - Inodes, directories, xattrs, quotas, encryption
- `encryption` - ChaCha20/Poly1305, key hierarchy, nonce handling
- `format` - On-disk format, ioctls, key types
- `infra` - Error codes, fsck, printbuf, recovery passes
- `fsck` - Recovery passes, journal replay, topology repair
- `journal` - WAL format, write path, reclaim, replay
- `reconcile` - Background data movement, work lifecycle
- `versions` - All 47 metadata versions (0.10-1.36)
- `userspace` - bcachefs-tools structure

## License

GPL-2.0-only (skill documentation derived from GPL-2.0 kernel source)
