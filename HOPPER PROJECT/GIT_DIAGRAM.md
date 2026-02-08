# Case Study: Resolving Production Merge Conflicts

**Project:** Hopper Roadside Dashboard  
**Status:** Resolved ✅

## 1. The Challenge

During a routine update of the **Hopper Roadside Dashboard** on the production server, a `git pull` operation failed. The local server environment and the remote GitHub repository had "diverged."

### The Blockers:

- **Unmerged Files:** `package-lock.json` and `vite.config.js`.
- **Divergent History:** Git required a reconciliation strategy.

## 2. Architecture of the Resolution

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│                    GIT CONFLICT RESOLUTION: PRODUCTION SYNC                       │
└───────────────────────────────────────────────────────────────────────────────────┘

    GitHub Remote                Production Server               Action/Result
 (Mirza2018/Dashboard)             (srv996854)                 (Terminal/Disk)
   ┌──────────────┐              ┌──────────────┐             ┌─────────────────┐
   │              │              │              │             │                 │
   │ 1. New Commit│───git pull──>│  Divergence  │             │  FATAL ERROR!   │
   │    (1112ce8) │              │   Detected   │             │ (Unmerged Paths)│
   │              │              │              │             │                 │
   └──────────────┘              └──────┬───────┘             └────────┬────────┘
                                        │                              │
                                        ▼                              │
   ┌──────────────┐              ┌──────────────┐                      │
   │ Resolution   │              │ Manual Fixes │             1. Identify Files
   │   Strategy   │<─────────────┤   & Policy   │             2. Resolve Logic
   │              │              │              │                      │
   └──────┬───────┘              └──────┬───────┘                      │
          │                             │                              │
          │      [Policy: --theirs]     │                              │
          ├────────────────────────────>│ package-lock ───────────────>│ FIXED
          │                             │                              │
          │      [Policy: --ours]       │                              │
          ├────────────────────────────>│ vite.config  ───────────────>│ FIXED
          │                             │                              │
          │                             └──────┬───────┘               │
          │                                    │                       │
          │      [Strategy: Merge]             ▼                       │
          └────────────────────────────>┌──────────────┐               │
                                        │ pull.rebase  │        3. Define History
                                        │    false     │        4. Create Bridge
                                        │              │               │
                                        └──────┬───────┘               │
                                               │                       │
                                               ▼                       │
   ┌──────────────┐              ┌──────────────┐                      │
   │    SYNCED    │              │ npm install  │               5. Finalize &
   │  & CLEANED   │<─────────────┤ npm run build│                  Deploy
   │              │              │              │
   └──────────────┘              └──────────────┘
```
