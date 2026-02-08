# Deep Dive: The Logic of Conflict Resolution

When a project like the **Hopper Roadside Dashboard** hits a merge conflict, the developer must choose a path that ensures long-term stability.

## 1. Why `pull.rebase false`?

In this scenario, I explicitly chose the **Merge** strategy over **Rebase**.

- **The Merge Strategy:** It creates a new "Merge Commit." This acts as a bridge between the two histories. It is non-destructive and provides an **Audit Trail**. If an error occurs later, we can see exactly when the production server was synced.

- **The Rebase Strategy:** It rewrites history to make it look like a straight line. While cleaner to look at, it is risky on production servers because it changes commit hashes and can make troubleshooting difficult.

## 2. Managing the Lockfile Conflict

Manually editing a `package-lock.json` is a recipe for disaster. It is a machine-generated file.

**The Solution:** `git checkout --theirs package-lock.json`
By using the `--theirs` flag, I told Git to discard the local version of the file and accept the version from the remote repository. This ensures that the production server uses the exact same dependency versions that were tested and approved by the development team.

## 3. Dealing with Divergent Branches

When Git says `fatal: Need to specify how to reconcile divergent branches`, it is protecting the repository. By setting a default preference (Merge), we streamline future deployments while keeping a high level of historical accuracy.

---

Back to **[Case Study Report](./GIT_DEADLOCK.md)**
