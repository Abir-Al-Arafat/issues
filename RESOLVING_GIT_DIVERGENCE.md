# Server Maintenance Log: Resolving Git Divergence

**Date:** 2026-02-25  
**Project:** Hopper Roadside Dashboard  
**Server:** srv996854

## Issue

While attempting a `git pull` on the production/staging server, Git returned a fatal error: `Need to specify how to reconcile divergent branches`. This occurred because the local server branch and the remote `main` branch had different commit histories.

## Resolution Steps

### 1. Set Merge Strategy

To resolve the divergence, we opted for a **Standard Merge**. This preserves the history of both branches and combines them.

```bash
git config pull.rebase false
```

### 2. Execute Pull

Initiated the pull request to fetch and integrate the remote changes:

```bash
git pull
```

### 3. Finalize Merge Message

Git opened the nano editor to confirm the merge commit message. The default message was used:

**Action:** Ctrl + O (Save) → Enter → Ctrl + X (Exit)

## Outcome

The local repository is now synchronized with the remote main branch.

## Best Practices for Future Pulls

To avoid the manual editor popup in the future when merging on this server, use the `--no-edit` flag:

```bash
git pull --no-edit
```

Alternatively, if the server should always exactly match the repository without keeping local changes:

```bash
git fetch origin
git reset --hard origin/main
```
