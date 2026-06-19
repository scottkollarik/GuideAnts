# Git Workflow — Fork Cheatsheet (private to my fork)

Lives on the **orphan `mydocs` branch** — no project code, no shared history.
Pushed to `origin` (my fork) for multi-machine sync; never PR'd upstream.

## Remotes
- `origin`   → `scottkollarik/GuideAnts` (my fork — I push here)
- `upstream` → `Elumenotion/GuideAnts`  (the original — read-only, PR target)

My local clone is the intermediary: pull changes IN from `upstream`,
push changes OUT to `origin`, propose changes to `upstream` via PR.

## Sync my fork's main with upstream
Keep `main` a clean mirror of `upstream/main` (never commit to it directly).

```
git switch main
git fetch upstream
git merge --ff-only upstream/main   # fast-forward only; refuses if diverged
git push origin main                # update my fork on GitHub
```
`git pull --ff-only upstream main` does the fetch + merge in one step.

## Make a change and propose it upstream
```
git switch main && git pull --ff-only upstream main   # start from fresh main
git switch -c feature/my-change main                   # branch off main
# ...work, commit...
git push -u origin feature/my-change                   # push branch to my fork
gh pr create --repo Elumenotion/GuideAnts \
  --base main --head scottkollarik:feature/my-change   # PR into upstream
```

## This notes branch (orphan)
```
# already created with:
#   git worktree add --orphan -b mydocs <path>
# lives at: Worktrees/GuideAnts/mydocs/
git -C <path> add . && git -C <path> commit -m "..." && git -C <path> push

# on a new machine:
git clone https://github.com/scottkollarik/GuideAnts.git
git fetch origin mydocs
git worktree add Worktrees/GuideAnts/mydocs mydocs
```

## Worktrees vs clone (mental model)
- worktree = cheap extra workbench; shares the one `.git` (history/objects).
  Only the *working files* are duplicated on disk, not the repo database.
- clone = whole second workshop; duplicates everything.
- orphan branch = empty tree, so a docs-only worktree holds ONLY these files
  (no codebase duplication). Disconnected from main's history by design.
- Full-code worktrees shine for parallel agents (each needs the whole tree);
  for a few notes, an orphan branch avoids the wasted disk.

## Worktree housekeeping
- `git worktree list`            — show all worktrees + their branches
- `git worktree remove <path>`   — clean removal (don't `rm -rf`)
- `git worktree prune`           — drop stale references

## Mental model
- `origin` = my fork (I own it, I push to it).
- `upstream` = the original (I read from it; I can't push, only PR).
- One branch = one PR. Keep `main` clean so future `--ff-only` syncs stay easy.
