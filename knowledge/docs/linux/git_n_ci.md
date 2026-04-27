# Git & CI

## Change the base branch of a PR to the latest commit of master

```bash
git fetch origin master
git reset --soft origin/master
git commit -m "your message"  
git push --force-with-lease  
```

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
