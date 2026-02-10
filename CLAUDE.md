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
