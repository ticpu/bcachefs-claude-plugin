# bcachefs-claude-plugin

Claude Code plugin providing bcachefs development context.

## Releasing

1. Bump `version` in `.claude-plugin/plugin.json` (semver: patch for fixes, minor for new/updated skills, major for breaking changes)
2. Commit: `release: v<version>`
3. Annotated signed tag: `git tag -as v<version>` with changelog body
4. Push: `git push && git push --tags`

## Testing locally after push

```
claude plugin marketplace update bcachefs-dev
claude plugin update bcachefs-dev@bcachefs-dev
```

The marketplace update fetches the latest remote; without it, `plugin update` compares against a stale local clone.

## Updating skill docs for a new kernel version

`KERNEL_VERSION` tracks the full kernel commit hash (line 1) and a per-file index of which kernel commit each skill doc was last reviewed against (`<short-hash> <date> <file>`).

When updating for a new kernel version:

1. Update line 1 with the new full commit hash
2. After updating a skill doc, update its entry with the new short hash and date
3. Files with an older hash than line 1 are stale and need review
