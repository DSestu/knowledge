# GitHub `gh stack` CLI

GitHub shipped **native stacked PRs** (private preview, April 2026) via the `gh-stack` extension. The stack map lives in the PR UI, so reviewers see context without any extension. Automates the cascade with one command.

For the underlying concept (why stacks, the rebase cascade, and the plain-GitHub manual approach), see the *PR stacking* section of [Git & CI](git_n_ci.md).

⚠️ Still **waitlist-gated** — but *not* all-or-nothing. The README says the CLI "will not work unless the feature has been enabled," which is overstated. Only the commands that touch the **remote stack object** hard-fail; the git plumbing works regardless. See [Without the feature enabled](#without-the-feature-enabled) below.

## Without the feature enabled

Reading the source (`cmd/*.go`), the feature gate is a 404 on the stacks API, not a blanket check. What that means in practice:

| Works fully (local git + ordinary PR API) | Hard-fails with exit code 9 |
|---|---|
| `init`, `add`, `modify`, `rebase`, `push`, `view`, prune | `submit` — needs to create the stack object |
| `sync` — **minus** the final link step | `link` / `unstack` — their only job *is* the stack object |
| navigation (`up`/`down`/`top`/`bottom`) | `checkout <remote-stack>` — nothing remote to pull down |

- **`gh stack sync` degrades gracefully.** It fetches, cascade-rebases, force-pushes `--force-with-lease`, and refreshes PR state via the normal PR API — then tries to link the PRs into a remote stack, gets a 404, prints `Stacked PRs are not enabled for this repository`, and reports **"Branches synced"** instead of "Stack synced". There's an explicit test for this (`TestSync_StacksUnavailable_BranchesSynced`). So the whole cascade-rebase workflow is usable un-enrolled.
- **`link` bails out cleanly.** It lists stacks as its *first* API call, so it 404s before any side effects — no half-pushed branches or half-retargeted bases.
- **Minimal local-only restack:** `gh stack rebase --no-trunk` (cascade only, no fetch/API) or plain git `git rebase origin/master --update-refs`.

**What you actually lose without enrollment** (all server-side): the stack UI, atomic bottom-up merges, auto-retargeting of the next PR's base when the bottom merges (do it by hand: `gh pr edit --base main`), branch-protection/required-checks evaluated against the stack base, `github.event.pull_request.stack` in Actions, and team sharing of the stack structure (it lives only in your local `.git` stack file).

For a **solo** workflow that just wants stacked rebases and is fine merging PRs bottom-up manually, the un-enrolled experience is fully usable.

## Install

```bash
gh extension install github/gh-stack
gh stack alias                # optional: alias `gh stack` → `gs`
```

## Full lifecycle

```bash
# Bottom layer
gh stack init avatar-migration --base main
git add -A && git commit -m "Add avatar_url column + migration"

# Stack layers on top (must be run from the TOPMOST branch; -A stages, -m commits)
gh stack add avatar-api -A -m "Add POST /users/:id/avatar endpoint"
gh stack add avatar-ui  -A -m "Add avatar upload widget"

gh stack view                 # show the stack, PR links, commits
gh stack submit --open        # push all branches + create all PRs w/ correct bases
```

## After a review forces a change to a lower branch

```bash
gh stack bottom               # checkout the bottom branch (or: gh stack down)
git commit -am "Rename column per review"
gh stack sync                 # fetch + cascading rebase + force-push + refresh PRs
# on conflict: fix, then `gh stack rebase --continue` (or `--abort`)
```

`gh stack sync` replaces the entire manual rebase cascade. Merge PR #1 in the UI, run `gh stack sync` again (next PR's base retargets to `main`), repeat bottom-up.

## Inserting a layer in the middle (not on top): `gh stack modify`

`gh stack add` **cannot** do this — it only appends on top of the stack. Restructuring is done with the interactive editor `gh stack modify`.

⚠️ **Version-gated**: `modify` shipped in **v0.0.3** (May 2026); the insert keys in **v0.0.5**. If `gh stack --help` doesn't list `modify`, upgrade first:

```bash
gh extension list                 # check installed version (v0.0.1 = April 2026, no modify)
gh extension upgrade stack
```

Preconditions: active stack checked out, **clean working tree**, no rebase in progress, no PR queued for merge, linear history. Merged branches appear locked. `?` shows the help overlay.

```bash
gh stack modify
```

Move the cursor to a branch, then:

| Key | Action |
|-----|--------|
| `i` | Insert a new **empty** branch *below* the cursor (toward trunk) |
| `I` | Insert a new **empty** branch *above* the cursor (away from trunk) |
| `Shift+↑` / `Shift+↓` | Reorder a branch up/down the stack |
| `r` | Rename branch (inline prompt) |
| `x` | Drop branch from stack (local branch + PR preserved) |
| `d` / `u` | Fold commits into branch below / above (cherry-pick) |
| `z` | Undo last staged action |
| `Ctrl+S` | Apply all staged actions |

`Ctrl+S` applies everything atomically and runs a **cascading rebase** so history stays linear (upstack branches are re-parented automatically). On conflict: fix, `git add`, then `gh stack modify --continue` (or `--abort` to restore the pre-modify state — works even if modify was interrupted).

Constraints: reordering and structural changes (drop/fold/insert/rename) **cannot coexist in one session**; branches cannot be split or moved between stacks.

After inserting: checkout the new empty branch, commit your work, then `gh stack submit` — it pushes the updated branches and recreates the stack (PR bases retargeted automatically).

**Fallback without `modify`** (old version, can't upgrade): `gh stack unstack` (removes stack tracking; branches and PRs survive) → restructure with plain git (`git checkout parent && git checkout -b new-layer`, `git rebase --onto` for the upstack branches) → `gh stack init` to re-create the stack with the new structure.

## Leaving and re-entering a stack

A stack is **not a mode you're inside** — it's persistent tracking metadata stored locally *and* on GitHub. So there's no "exit stack" command.

To go work on something unrelated, just use normal git — the stack tracking survives:

```bash
git checkout main
git checkout -b hotfix-something     # unrelated work
```

To pick the stack back up (also works on another machine — it fetches the stack from GitHub, pulls its branches, re-establishes local tracking):

```bash
gh stack checkout <stack-number>     # or a PR number, PR URL, or branch name
gh stack checkout                    # no args → picker listing ALL stacks (local + remote)
```

Don't confuse the two "checkout"-ish commands:

| Command | Scope | Use |
|---------|-------|-----|
| `gh stack checkout [id]` | **Between stacks** | Get onto / load a whole stack (stack #, PR #, PR URL, branch, or picker) |
| `gh stack switch` | **Within current stack** | Picker of branches in the stack you're already on |
| `gh stack up`/`down`/`top`/`bottom`/`trunk` | Within current stack | Step directly to a layer |

Mental model: **`checkout` = get onto a stack · `switch`/`up`/`down` = move around inside it · plain `git checkout` = step away to anything else.**

## Turning existing PRs into a stack: `gh stack link`

`gh stack link` creates or updates a stack **on GitHub** from branches/PRs that already exist — **without creating any local tracking**. It's the API-only path, designed for people who manage branches with other tools (jj, Sapling, git-town, ghstack) but still want GitHub's stack grouping. Contrast with `submit`, which drives PRs *from* local tracking state.

```bash
# Arguments are BOTTOM → TOP. Each can be a branch name, a PR number, or a PR URL.
gh stack link auth-layer api-routes ui-components     # by branch
gh stack link 41 42 43                                # by existing PR number
gh stack link 7 48 ui-polish                          # 7 is a STACK number → append 48 + ui-polish to its top
gh stack link --base develop --open feat-a feat-b     # custom base; mark PRs ready for review
```

What it does per argument, in order:

1. Lists existing stacks (this is the first API call → 404 → exit 9 if the feature is off, before any side effects).
2. Pushes any branch args to the remote.
3. Uses the existing open PR if there is one; otherwise **creates** a PR with the correct base and an auto title.
4. **Retargets base branches** so they chain: bottom PR → base branch, each higher PR → the branch below it (any PR with a wrong base is corrected via `UpdatePRBase`).
5. Creates (or updates) the stack object on GitHub.

Behavior notes:

- **Additive & idempotent.** If some PRs already belong to *one* stack, that stack is updated; existing PRs are never removed. PRs spanning *multiple* stacks are rejected (`unstack them first`).
- **Add mode.** A **stack number** as the first arg appends the rest to the top of that stack without re-listing its members. Args already in it are skipped; args in a different stack are rejected. (Stack/PR/issue numbers share one numberspace, so a number never doubles as a PR — but a branch literally named like a number stays a branch.)
- **Flags:** `--base <branch>` (bottom base, default repo default branch; ignored in add mode), `--open` (mark new *and* existing PRs ready, converting drafts), `--remote <name>`.
- **No local tracking.** After `link`, `sync`/`rebase` have nothing to work with. If you want them, run `gh stack init` + `gh stack add <branch>` afterward (purely local).

**Un-enrolled fallback** (feature off, so `link` errors): do what it does by hand — chain the bases with `gh pr edit <pr> --base <branch-below>` bottom-to-top, then optionally `gh stack init` + `gh stack add` for each branch to get local tracking so `gh stack sync`/`rebase` work.

## Command cheat sheet

| Purpose | Command |
|---------|---------|
| Install | `gh extension install github/gh-stack` |
| Start stack | `gh stack init <branch> --base main` |
| Add a layer on top | `gh stack add <branch> -A -m "msg"` (from topmost branch only) |
| Insert / reorder / rename / drop layers | `gh stack modify` (interactive TUI; needs ≥v0.0.5 for insert — `gh extension upgrade stack`) |
| See the stack | `gh stack view` |
| Create/update all PRs | `gh stack submit` |
| Stack up **existing** PRs (no local tracking) | `gh stack link <br-or-pr> <br-or-pr> …` (bottom→top) |
| Restack after edits | `gh stack sync` |
| Cascading rebase only | `gh stack rebase [--upstack\|--downstack]` |
| Local-only rebase (no fetch/API) | `gh stack rebase --no-trunk` |
| Resolve mid-rebase | `gh stack rebase --continue` / `--abort` |
| Navigate | `gh stack up` / `down` / `top` / `bottom` / `trunk` |

Docs: <https://github.github.com/gh-stack/> · CLI reference: <https://github.github.com/gh-stack/reference/cli/>
