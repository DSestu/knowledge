# Git & CI

## Rebase + squash onto latest master (one clean commit)

```bash
git fetch origin master
git rebase -i origin/master
# pick the first, squash the rest
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
