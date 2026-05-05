---
name: back-to-main
description: Switch to the main branch cleanly — handles pending changes, pulls latest, and summarizes what changed.
allowed-tools: Bash, AskUserQuestion
---

# Back to Main

## Goal

Return to `main` in a clean, informed state: handle any pending changes, switch branches, pull the latest, and display a concise summary of what landed since the last pull.

## Steps

### 1. Check for pending changes

Run in parallel:
- `git status --short` — list modified/untracked files
- `git stash list` — check for existing stashes
- `git branch --show-current` — know the current branch

If already on `main` and working tree is clean, skip to step 4.

### 2. Handle pending changes (if any)

If there are uncommitted changes (staged, unstaged, or untracked files), **do not proceed silently**. Present the user with options:

> "You have pending changes on `<branch>`. How would you like to proceed?"
>
> 1. **Stash & reapply** — stash now, switch to main, pull, then reapply the stash on the original branch
> 2. **Discard** — discard all local changes (irreversible) and switch to main
> 3. **Keep on this branch** — switch to main anyway (only safe if changes are untracked or on a different branch scope)
> 4. **Abort** — stay on the current branch and do nothing

Use the `AskUserQuestion` tool for this.

Wait for the user's answer before proceeding.

#### If stash & reapply chosen:
```bash
git stash push -m "back-to-main: stash before switching"
git switch main
git pull
# then reapply:
git switch <original-branch>
git stash pop
git switch main
```
Inform the user that the stash was reapplied on `<original-branch>` and that they are now on `main`.

#### If discard chosen:
```bash
git restore .
git clean -fd
git switch main
git pull
```
Warn the user clearly that discarded changes are gone.

#### If keep on this branch chosen:
```bash
git switch main
git pull
```

#### If abort chosen:
Stop. Tell the user they are still on `<branch>` with their changes intact.

### 3. If already on main with pending changes

Same logic as above — apply the chosen resolution before pulling.

### 4. Switch to main and pull (clean state)

```bash
git switch main
git pull
```

Capture the output of `git pull` to detect whether anything was fetched.

### 5. Summarize what changed

After pulling, run:

```bash
git log --oneline --no-merges -20 ORIG_HEAD..HEAD
```

If `ORIG_HEAD` does not exist (first pull or already up to date), use:

```bash
git log --oneline --no-merges -10
```

Display a **concise bullet-point summary** of the commits that landed, grouped by type if possible (feat, fix, chore, etc.). Keep it readable — 1 line per commit, no noise.

If nothing changed (already up to date), say so clearly.

### 6. Output

Tell the user:
1. What happened to their pending changes (stashed, discarded, kept, or nothing)
2. Current branch: `main` ✓
3. Summary of new commits (or "already up to date")
4. Any stash reminder if applicable ("Your changes are stashed on `<branch>` — run `git stash pop` there when you're ready")
