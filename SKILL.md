---
name: git-town-stack-sync
description: Guide and automate syncing, rebasing, and resolving conflicts across stacked Git branches managed with Git Town. Use when the user asks to sync a Git Town stack, rebase stacked branches after a parent branch is merged or squashed into main, resolve cascading rebase conflicts, repair parent-child lineage, diagnose an anomalous or externally rewritten stack state (fast-forwarded main, flattened lineages, duplicate commits, lost segments), or recover a branch segment that was fast-forwarded away.
---

# Git Town Stack Sync

Maintain, rebase, and synchronize stacked Git branches using Git Town, specifically handling upstream squash-merges, parent lineage updates, and cascading conflict resolution.

## When to Use

* A branch at the bottom of a Git Town stack has been merged or squash-merged into `main` via the forge Web UI (GitHub/GitLab).
* The user wants to sync a multi-branch feature stack (`git town sync --stack` or `git town sync --all`) and handle conflicts cleanly.
* A squash-merge left orphaned commits in child branches, causing phantom conflicts during a standard sync.
* The user needs to verify or repair the parent-child branch lineage before or after syncing.
* The stack state looks anomalous: `main` fast-forwarded outside Git Town, branch tips equal to their parent, duplicated commits inside one lineage, or a branch whose work seems missing.
* A branch segment was lost or the person says a branch "should be ahead" of its parent despite matching tips.

## Steps

### 1. Pre-Flight Inspection and Preparation

Before running any sync operation, ensure the repository is clean and conflict-caching is enabled so repeated conflicts across stacked branches resolve automatically.

1. Verify working tree status:
   ```bash
   git status --short
   ```
   If dirty, stash or commit changes before proceeding.

2. Ensure Git `rerere` (reuse recorded resolution) is active to prevent re-solving identical conflicts as they cascade up the stack:
   ```bash
   git config --global rerere.enabled true
   git config --global rerere.autoupdate true
   ```

3. Inspect the current stack lineage and remote tracking branches:
   ```bash
   git fetch --all --prune
   git town branch
   ```

### 2. Forensic State Diagnosis (Before Any Sync)

Skip nothing here when the stack looks off. Real stacks get reworked outside Git Town: `main` fast-forwarded by hand, branch lineages flattened or re-based by parallel sessions, commits duplicated inside a single lineage, segments lost. Diagnose first so the conflict cascade is predictable instead of discovered mid-rebase.

1. Map each stack branch's actual fork point from `main`:
   ```bash
   for b in <branch-N>; do
     printf '%s: %s\n' "$b" "$(git log -1 --oneline $(git merge-base main "$b"))"
   done
   ```
   A fork point equal to `main`'s tip means the branch is fully based on current `main` (its parent sync is a no-op). Identical fork points across siblings mean the stack was flattened: each branch carries its cumulative segment in its own lineage, and Town's recorded parentage may lag reality.

2. Measure net tree diffs, not commit counts. Commit-subject and patch-id matching cannot see squash-shaped or fast-forward absorption; tree diffs can:
   ```bash
   git diff --shortstat main <branch>
   ```
   A shrinking or empty diff walking down the chain flags a branch with no unique content; a growing one shows real unshipped work.

3. Predict what the rebase will drop with patch-ids. `git rebase` silently drops a replayed commit whose patch-id already exists in the upstream range:
   ```bash
   git show <commit> | git patch-id --stable | awk '{print $1}'
   ```
   Compare patch-id sets for the branch-only commits against the parent range. Matches tell you which commits auto-drop (cheap and safe); same-subject non-matches tell you exactly where conflicts will land. Duplicate pairs inside one lineage replay as empty commits: expect a halt, then `git rebase --skip`.

4. Run reflog archaeology on any branch whose tip equals its parent's tip:
   ```bash
   git reflog show <branch>
   ```
   `branch: Created from HEAD` followed by a `rebase (finish)` landing at the parent's tip means the branch was fast-forwarded and its segment vanished, not that it never had work.

5. Hunt the lost work on parallel or foreign branches (`origin/<user>/refactor/<name>` patterns, stale sibling stacks). When you find it, stop and confirm the source with the person before rebuilding: whose commits belong on which branch is their call, not yours.

6. Surface every anomaly as a question rather than folding it into a sync verdict.

### 3. Detect How the Bottom Branch Landed in Main

How you sync depends on whether the remote branch was deleted by the forge upon merge/squash, if it still exists on `origin`, or whether `main` moved by external fast-forward or rewrite.

* **Case A: Remote branch was auto-deleted on merge/squash.**
  Run `git town sync --stack` (or `git town sync --all`). Git Town detects the deleted remote tracking branch, removes the shipped local branch, updates child branch parent pointers to `main`, and rebases/merges the stack.

* **Case B: Remote branch still exists after a manual Web UI squash-merge.**
  If the remote branch was not deleted on the forge, `git town sync` will attempt to pull from it rather than recognizing it as shipped. Delete the remote tracking branch first (or delete the local branch and re-parent):
  ```bash
  git push origin --delete <shipped-branch-name>
  git town sync --stack
  ```

* **Case C: `main` moved by external fast-forward or lineage rewrite (no forge merge involved).**
  If step 2's forensics show `main` carrying the stack's commits with matching SHAs, or flattened lineages with self-duplicates, there is nothing to un-ship. Run `git town sync --stack` directly: rebase drops patch-identical commits automatically, and the conflict loop handles drifted duplicates and empty replays.

### 4. Handle Squash-Merge Phantom Conflicts

When a parent branch (`branch-A`) is squash-merged into `main`, `main` receives a single new commit with a different SHA than the original commits on `branch-A`. When `branch-B` (child of `branch-A`) is rebased onto `main`, Git may try to replay `branch-A`'s original granular commits on top of the squash commit, causing phantom conflicts.

If `git town sync` hits conflicts caused by replaying already-squashed parent commits:

1. **Option 1: Fast Resolution by Skipping Redundant Commits (During Rebase Strategy)**
   Note: In a `git rebase`, `--ours` refers to the upstream target (`main`, which has the squash commit) and `--theirs` refers to the branch being rebased. If the conflict is strictly because the parent commit is already squashed into `main`, skip the redundant parent commit:
   ```bash
   git rebase --skip
   git town continue
   ```

2. **Option 2: Clean Surgical Rebase (`git rebase --onto`)**
   If `branch-B` contains several commits from `branch-A` that were squashed into `main`, abort the messy sync and surgically transplant only `branch-B`'s unique commits onto `main`:
   ```bash
   git town undo
   # Identify the last commit of the old parent branch (branch-A)
   git rebase --onto main <last-commit-of-branch-A> <branch-B>
   # Update Git Town parent metadata so branch-B knows its parent is now main
   git town set-parent main
   # Force-push with lease and continue syncing the rest of the stack.
   # Always use a full src:DST refspec (see the push-hygiene gotcha)
   git push --force-with-lease=branch-B:<expected-sha> origin branch-B:refs/heads/branch-B
   git town sync --stack
   ```

### 5. Step-by-Step Conflict Resolution Loop

When real code conflicts arise during `git town sync --stack`:

1. Inspect conflicted files:
   ```bash
   git status
   ```
2. Resolve the conflict markers in the listed files. For formatting or rename conflicts where the upstream side is the newer design, take the upstream side (`git checkout --ours <file>` is safe under rebase; modify/delete conflicts have no upstream version for deleted paths, so `git rm` them when the tips confirm the deletion).
3. Stage the resolved files:
   ```bash
   git add <resolved-files>
   ```
4. Resume the Git Town orchestration with a non-interactive editor, so Git Town keeps its runstate and continues syncing child branches up the stack:
   ```bash
   GIT_EDITOR=true git town continue
   ```
   Plain `git town continue` can hang waiting for an interactive rebase editor. Never run raw `git rebase --continue`: it is safe at intermediate halts, but if it completes the branch's final pause, Town's runstate points at an already-finished step and later `git town continue` fails with `fatal: no rebase in progress`.
5. When a replayed commit is strictly redundant with work already upstream (or becomes empty), skip it and resume:
   ```bash
   git rebase --skip
   git town continue
   ```
6. If a resolution goes wrong at any point, revert the entire sync transaction back to the pre-sync state:
   ```bash
   git town undo
   ```

### 6. Recovering a Stuck or Abandoned Runstate

1. `git town skip` is not a safe "clear this error" tool: it aborts the branch's current operation and `git reset --hard`s the branch to its *pre-operation* state, discarding conflicts you already resolved and rebase work you completed manually. Never reach for it to silence a stale runstate.
2. If `git town continue` fails with `fatal: no rebase in progress` after the branch's rebase actually finished:
   - Complete the remaining steps manually with explicit full refspecs (see the push-hygiene gotcha), then run a fresh all-no-op `git town sync --stack` to clear the runstate.
3. A `--non-interactive` refusal ("interactivity disabled via CLI") means Git Town has a pending dialog, not that flags are wrong. Find what it wants with `git town status` or `git town sync --stack --verbose` instead of forcing non-interactive mode.

### 7. Post-Sync Verification

After `git town sync --stack` completes:

1. Verify the updated branch hierarchy:
   ```bash
   git town branch
   ```
2. Walk the contains-parent chain. This stays reliable on flattened or rewritten stacks where `git log main..HEAD` degrades to listing the whole replayed segment:
   ```bash
   prev=main; ok=1
   for b in <child branches in order>; do
     git merge-base --is-ancestor "$prev" "$b" || ok=0; prev=$b
   done
   [ "$ok" = 1 ] && echo CHAIN-INTACT
   ```
3. Confirm every stack branch's local tip equals its remote tip (a mismatch means a push was skipped or landed on the wrong remote branch):
   ```bash
   git branch -vv
   ```
4. If PR/MR target branches on GitHub/GitLab did not automatically update when the bottom branch was deleted, run `git town propose` or update the base branch via CLI (`gh pr edit --base main` / `glab mr update --target-branch main`).
5. Re-run the project's gates (fmt/doc/test) on pushed tips before refreshing MRs. A sync that re-homes commits verifies history, not content.

## Gotchas

* **Never use `git rebase --continue` directly during a Git Town operation.** It works at intermediate halts but breaks if it completes the branch's final pause: Town's runstate then points at a finished step and `git town continue` repeatedly fails. Resume with `GIT_EDITOR=true git town continue` throughout.
* **`--ours` vs `--theirs` reversal in rebase:** When Git Town uses the `rebase` sync-feature-strategy, Git swaps the meaning of `--ours` (becomes `main`/parent) and `--theirs` (becomes your feature branch commits). Verify which side you want before running `git checkout --ours/--theirs`.
* **`git town skip` abandons the branch's operation, not the current error.** It aborts the branch's rebase and `git reset --hard`s the branch to its pre-operation state, silently discarding resolved conflicts and manually completed rebase work. Use it only when you consciously want to abandon that branch's sync.
* **Push with full `src:DST` refspecs only.** `git checkout -B <local> origin/<foreign>` silently makes the foreign branch `<local>`'s upstream, and with a local `push.default=upstream` (common in multi-repo worktrees) a bare `git push origin <local>` force-updates the foreign branch instead. After checking out from a foreign remote, re-set tracking (`git branch --set-upstream-to=origin/<local> <local>`) and always push `local:refs/heads/local`. Prefer a named lease (`--force-with-lease=<ref>:<expected-sha>`) over a blanket lease so an unexpected remote state refuses instead of clobbering.
* **Stale remote branches block cleanup:** Git Town relies on remote tracking branch deletion (`git fetch --prune`) to know a branch was shipped in the Web UI. Enable "Automatically delete head branches" in GitHub/GitLab repo settings to make `git town sync --stack` seamless.
* **Branch equality with its parent is a question, not a verdict.** A branch whose tip equals its parent's may have lost work to a fast-forward. Check its reflog and the parallel/foreign branches before reporting "no own commits", and let the person confirm where lost work should live.
