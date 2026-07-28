# Git & CI

## ZSHrc snippet to pull every repo in a folder in parallel

Put this in the `.zshrc`.

This supposes that every repo is in `$HOME/github/`, change that accordingly.

```zsh
alias pullall='for d in $HOME/github/*/; do (cd "$d" && if [ -d .git ]; then echo "Pulling in $(pwd)" && git pull; fi) & done; wait'
```

## Rebase + squash onto latest master (one clean commit)

```bash
# Main or master ?
set DEFAULT_BRANCH (git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')

git fetch origin $DEFAULT_BRANCH
git rebase origin/$DEFAULT_BRANCH
# pick the first, squash the rest (p, or s)
# git rebase -i origin/$DEFAULT_BRANCH
# You can use autosquash with sed here
GIT_SEQUENCE_EDITOR="sed -i '2,\$s/^pick/squash/'" git rebase -i origin/$DEFAULT_BRANCH
# Force with lease avoid push if someone else pushed in the meantime
git push --force-with-lease
```

⚠️ Do **not** use `git reset --soft origin/master && git commit` unless you have already rebased. On a branch that is behind master, that pattern silently reverts other people's merged work.

## Cancel a rebase in progress

```bash
git rebase --abort
```

Restores the branch to its state before the rebase started. Safe to run any time mid-rebase.

# Undo last commit without losing changes

```bash
git reset HEAD~1 --soft
```

# Running precommit hooks on files changed in the PR

```
alias pc="git diff --name-only --diff-filter ACMR origin/master...HEAD | xargs pre-commit run --files"
```

# GIT

## AIO discard, checkout master, prune etc

```
git reset --hard HEAD && git checkout master && git pull && git gc --prune=now && git repack -Ad && git prune
```

## Discard changes or stash

```
git reset --hard HEAD
```

or

```
git stash --include-untracked
...later...
git pop
```

## Permanently ignore changes on a file

In some cases we don't want to gitignore a file. This can happen if you work with multiple people, and an IDE extension always creates a file. The file creation is only happenning to you, and you don't want to add too much pollution in the gitignore.

In this case, you can permanently ignore the change of a file with the following snippet

```bash
git update-index --assume-unchanged FILE
```

OR, edit `.git/info/exclude` and add the path of the excluded file.

## Rebase a branch onto latest master and squash into one commit

When you want a PR with **one clean commit** on top of latest master.

### Safe pattern (always works)

```bash
git fetch origin master
git rebase -i origin/master
# in the editor: keep first commit as 'pick', mark all others as 'squash' (or 'fixup')
git push --force-with-lease
```

Interactive rebase replays each commit in order, so conflicts are scoped per-commit and the resulting squashed commit only contains *your* changes.

### Shortcut pattern (only if branch is up to date with master)

```bash
git fetch origin master
git rebase origin/master            # MUST do this first
git reset --soft origin/master
git commit -m "your message"
git push --force-with-lease
```

⚠️ **Never run `git reset --soft origin/master && git commit` on a branch that is behind master.** The resulting commit will silently include negative diffs that revert other people's merged work — because the diff is computed as "current working tree vs master", not "your commits". Always rebase first so HEAD's tree only differs from master by *your* changes.

### Sanity check before squashing

```bash
git log --oneline origin/master..HEAD   # should show only your own commits
```

If you see commits you didn't author, the branch is behind master — rebase first.

## Recovering a branch you accidentally squashed wrong

If `git reset --soft` + commit produced a bogus single commit and you've already pushed/reverted, your original work is in the reflog:

```bash
git reflog --date=iso          # find the SHA from before the reset
git reset --hard <old-sha>     # restore branch tip
git rebase origin/master       # bring up to date
git push --force-with-lease
```

## Pushing to both Github and Huggingface

```bash
git remote add origin git@(...).git
git remote set-url --push origin git@(...).git
git remote set-url --add --push origin https://{user}:{token}@huggingface.co/spaces/{user}/{space}
git remote -v
git push --set-upstream origin main
```

## Purge and clean local .git folder

```bash
git gc --prune=now
git repack -Ad
git prune
```

```bash
git gc --prune=now && git repack -Ad && git prune
```

## Check sanity of local repo

```bash
git gc --prune=now
```

## Removing untracked files

for dry run use `-n` parameter

### Remove EVERY untracked file (including gitignored)

```bash
git clean -fdx
```

### Remove untracked file (but keep gitignored)

```bash
git clean -fd
```

### Remove untracked files (only gitignored ones)

```bash
git clean -fdX
```

# PR stacking (stacked pull requests)

## The idea

Instead of one big branch → one big PR, build a **chain of branches**, each branched off the previous one, and open a **separate PR per link**. Each PR's *base* is its parent branch, so its diff shows only the changes that link adds — not everything below it.

```
        main
          │
          ▼
   ┌─────────────┐
   │  branch A   │──▶ PR #1  (base: main)
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │  branch B   │──▶ PR #2  (base: branch A)
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │  branch C   │──▶ PR #3  (base: branch B)
   └─────────────┘
```

Each PR stays small and reviewable, and you keep working forward without waiting for reviews. Merge **bottom-up**: as each lands, GitHub auto-retargets the next PR's base to `main`.

**Use it when** the work has dependent layers (e.g. migration → service → API → UI) that can't be split into independent PRs off `main`. **Skip it** for a single atomic change, or when layers are truly independent (just open parallel PRs instead).

## Cost: the rebase cascade

If a reviewer makes you change a *lower* branch, every branch above it must be rebased onto the new version. By hand:

```bash
git checkout branch-A         # amend / add commits per review
git checkout branch-B
git rebase branch-A           # replay B on the new A
git checkout branch-C
git rebase branch-B           # replay C on the new B
git push --force-with-lease branch-A branch-B branch-C
```

⚠️ Always `--force-with-lease`, never `--force` — it refuses to clobber commits you haven't seen. Tooling exists precisely to automate this cascade.

## Manual stacking on plain GitHub

GitHub has always supported this natively — a PR's base can be any branch, and it auto-retargets to `main` when its parent merges.

```bash
git checkout main
git checkout -b feature-A     # ... commit ...   → PR #1 base=main
git checkout -b feature-B     # ... commit ...   → PR #2 base=feature-A
git checkout -b feature-C     # ... commit ...   → PR #3 base=feature-B
# set each PR's base with: gh pr create --base <parent-branch>
```

## Native `gh stack` CLI

GitHub ships native stacked PRs via the `gh-stack` extension, which automates the whole rebase cascade. It has its own page: **[GitHub `gh stack` CLI](gh_stack.md)**.
