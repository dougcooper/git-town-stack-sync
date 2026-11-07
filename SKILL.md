---
name: git-town-stack-sync
description: Guide and automate syncing, rebasing, and resolving conflicts across stacked Git branches managed with Git Town. Use when the user asks to sync a Git Town stack, rebase stacked branches after a parent branch is merged or squashed into main, resolve cascading rebase conflicts, or repair parent-child lineage in Git Town.
---

# Git Town Stack Sync

Maintain, rebase, and synchronize stacked Git branches using Git Town, specifically handling upstream squash-merges, parent lineage updates, and cascading conflict resolution.

## When to Use

* A branch at the bottom of a Git Town stack has been merged or squash-merged into `main` via the forge Web UI (GitHub/GitLab).
* The user wants to sync a multi-branch feature stack (`git town sync --stack` or `git town sync --all`) and handle conflicts cleanly.
* A squash-merge left orphaned commits in child branches, causing phantom conflicts during a standard sync.
* The user needs to verify or repair the parent-child branch lineage before or after syncing.

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

### 2. Detect How the Bottom Branch Landed in Main

How you sync depends on whether the remote branch was deleted by the forge upon merge/squash, or if it still exists on `origin`.

* **Case A: Remote branch was auto-deleted on merge/squash.**
  Run `git town sync --stack` (or `git town sync --all`). Git Town detects the deleted remote tracking branch, removes the shipped local branch, updates child branch parent pointers to `main`, and rebases/merges the stack.

* **Case B: Remote branch still exists after a manual Web UI squash-merge.**
  If the remote branch was not deleted on the forge, `git town sync` will attempt to pull from it rather than recognizing it as shipped. Delete the remote tracking branch first (or delete the local branch and re-parent):
  ```bash
  git push origin --delete <shipped-branch-name>
  git town sync --stack
  ```

### 3. Handle Squash-Merge Phantom Conflicts

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
   # Force-push with lease and continue syncing the rest of the stack
   git push --force-with-lease
   git town sync --stack
   ```

### 4. Step-by-Step Conflict Resolution Loop

When real code conflicts arise during `git town sync --stack`:

1. Inspect conflicted files:
   ```bash
   git status
   ```
2. Resolve the conflict markers in the listed files.
3. Stage the resolved files:
   ```bash
   git add <resolved-files>
   ```
4. Resume the Git Town orchestration (never run raw `git rebase --continue` when Git Town is driving, so Git Town can complete its queue of subsequent branches):
   ```bash
   git town continue
   ```
5. If a resolution goes wrong at any point, revert the entire sync transaction back to the pre-sync state:
   ```bash
   git town undo
   ```

### 5. Post-Sync Verification

After `git town sync --stack` completes:

1. Verify the updated branch hierarchy:
   ```bash
   git town branch
   ```
2. Confirm the first active feature branch in the stack does not contain stray duplicate commits from `main`:
   ```bash
   git log main..HEAD --oneline
   ```
3. If PR/MR target branches on GitHub/GitLab did not automatically update when the bottom branch was deleted, run `git town propose` or update the base branch via CLI (`gh pr edit --base main` / `glab mr update --target-branch main`).

## Gotchas

* **Never use `git rebase --continue` directly during a Git Town operation.** Always use `git town continue` so Git Town resumes its internal runstate and continues syncing child branches up the stack.
* **`--ours` vs `--theirs` reversal in rebase:** When Git Town uses the `rebase` sync-feature-strategy, Git swaps the meaning of `--ours` (becomes `main`/parent) and `--theirs` (becomes your feature branch commits). Verify which side you want before running `git checkout --ours/--theirs`.
* **Stale remote branches block cleanup:** Git Town relies on remote tracking branch deletion (`git fetch --prune`) to know a branch was shipped in the Web UI. Enable "Automatically delete head branches" in GitHub/GitLab repo settings to make `git town sync --stack` seamless.

---

# Blank `SKILL.md` Template & Format Specification

Use the following structure when creating any new `SKILL.md` file for an LLM agent.

```markdown
---
name: your-skill-name
description: Action-oriented summary of what the skill does. Use when the user mentions [specific keywords, tools, error messages, or workflows].
---

# Title Matching Skill Name

One to two sentences summarizing the core purpose and outcome of this skill.

## When to Use

* Trigger scenario 1 (e.g., specific user phrasing or intent)
* Trigger scenario 2 (e.g., specific error state or environment condition)
* Trigger scenario 3 (e.g., recurring multi-step workflow)

## Steps

### 1. Pre-Flight / Inspection

1. First verifiable step or command:
   ```bash
   <command-to-inspect-state>
   ```
2. Expected output or condition required to proceed.

### 2. Core Execution Workflow

* **Case A: [Standard Happy Path]**
  1. Run the primary action:
     ```bash
     <primary-command>
     ```
* **Case B: [Edge Case / Branching Condition]**
  1. Handle the exception cleanly:
     ```bash
     <fallback-command>
     ```

### 3. Verification

1. Verify that the workflow succeeded using a concrete check:
   ```bash
   <verification-command>
   ```

## Gotchas

* **[Pitfall 1]:** Explain *why* this mistake happens and the exact command/pattern to use instead.
* **[Pitfall 2]:** Note counter-intuitive flags, state reversals, or environment quirks.
```

## Format & Validation Checklist

* **`name` (Required in YAML Frontmatter):** 1 to 5 words, max 64 characters, lowercase `a-z`, `0-9`, and hyphens (`-`) only. Must not start or end with a hyphen.
* **`description` (Required in YAML Frontmatter):** Under 1024 characters. This is the **only** field read by the router before deciding whether to load the skill—include both *what* it does and *when* to trigger it.
* **Body Length:** Keep under 500 lines to preserve context window efficiency.
* **Tone & Style:** Use imperative verbs ("Inspect conflicted files", not "Conflicted files should be inspected") and explain the *why* behind non-obvious steps.
* **Formatting Rules:** Ensure there is a blank line before every list, and avoid em-dashes or en-dashes in strict parsers (use hyphens or periods instead).