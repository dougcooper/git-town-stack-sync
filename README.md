# git-town-stack-sync

An [OpenCode](https://opencode.ai) skill for maintaining, rebasing, and synchronizing stacked Git branches managed with [Git Town](https://www.git-town.com), with emphasis on the failure modes that turn syncs into multi-branch conflict cascades: forge squash-merges of bottom branches, externally fast-forwarded or rewritten `main`, flattened lineages with self-duplicated commits, and branch segments lost to accidental fast-forwards.

## Install

Copy the skill into your OpenCode skills directory:

```bash
mkdir -p ~/.config/opencode/skills/git-town-stack-sync
cp SKILL.md ~/.config/opencode/skills/git-town-stack-sync/SKILL.md
```

## What it covers

Seven-step workflow: pre-flight (clean tree, `rerere`, lineage inspection), forensic state diagnosis (merge-bases, net tree diffs, patch-id matching, reflog archaeology), upstream anomaly classification, squash-merge phantom-conflict handling, the step-by-step conflict resolution loop, stuck-runstate recovery, and post-sync verification (contains-parent chain, local == origin equality, project gates).

## Provenance

Battle-tested on an 11-branch production stack whose history had been restructured outside Git Town: `main` fast-forwarded manually, every branch flattened, duplicated commits inside lineages, and one branch's segment lost entirely. The forensic and push-hygiene guidance comes directly from the failures fixed there (including a `git town skip` that reset a branch and a bare `git push` refspec that hit a foreign branch under `push.default=upstream`).

## Requirements

Git Town 24.x (`sync --stack`, `git town continue`, runstate undo/recovery behavior as documented).

## License

MIT
