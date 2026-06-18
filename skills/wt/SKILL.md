---
name: wt
description: Create, navigate, list, and remove git worktrees with the `wt` CLI in a "bare repo + sibling worktrees" layout (recognizable by a `.bare/` directory at the repo root and branch directories beside it). Use when the user asks to make a new worktree, work on a branch in parallel without disturbing the current one, check out a PR into its own directory, move uncommitted changes onto a fresh branch, or move between worktrees — and whenever you're operating inside such a repo and need a branch's path. Covers the agent-critical rule that `cd` does not persist between shell commands, so worktree paths must be resolved with `wt path`.
---

# wt — git worktree manager

`wt` manages a repo laid out as one bare clone with many branch worktrees living side by side:

```
myproject/
├── .bare/          # the bare repository (git data)
├── .git            # file: "gitdir: ./.bare"
├── main/           # worktree on branch `main`
├── my-feature/     # worktree on branch `my-feature`
└── pr-1234/        # worktree for PR #1234
```

You're in such a repo if there is a `.bare/` directory at the container root. Each sibling directory is a full working copy on its own branch, so branches don't fight over a single checkout.

## The one rule that matters for agents

**`cd` does not carry over between your shell commands.** Each command you run is a fresh shell, so you cannot `wt cd foo` in one step and expect to be there in the next. `wt cd`/`wt add`/`wt save`/`wt pr` are built to move an *interactive* human's shell (via a wrapper function); that machinery is a no-op for you.

So, to act on a worktree, **resolve its absolute path and use it explicitly**:

```sh
# Get a worktree's path (clean stdout, always works — no wrapper needed):
wt path my-feature            # -> /abs/.../myproject/my-feature
wt path                       # -> /abs/.../myproject   (the container root)

# Do work in a worktree by chaining within ONE command:
cd "$(wt path my-feature)" && npm test

# Or run git against it without cd-ing at all:
git -C "$(wt path my-feature)" status
```

Never assume the current directory persists — re-resolve with `wt path` (or pass `-C <path>`) in each command.

## Creating worktrees

```sh
wt add <branch> [base]
```

- Creates a new worktree + branch and (for humans) cd's in. **New branches fork off the branch currently checked out where the command runs**, unless you pass an explicit `base`. So run `wt add` from the worktree you want to branch from, or pass `base` to be unambiguous.
- If `<branch>` already exists locally, it just adds a worktree for it.
- If it exists on a remote (`origin`/`upstream`), the new worktree tracks that remote branch.
- The worktree directory is the branch name with `/` replaced by `-` (e.g. `feat/login` → `feat-login/`). After creating, its path is `$(wt path <branch>)`.

```sh
wt save <branch>
```

Moves your **uncommitted** changes (tracked + untracked) onto a new branch + worktree, leaving the original worktree clean. The new branch forks from the current tip, so the stash applies cleanly. Use this when you started hacking in the wrong place and want to peel the work off onto its own branch.

```sh
wt pr <number>
```

Fetches a GitHub PR into a new `pr-<number>/` worktree (works for fork PRs — it fetches from `upstream` when present, else `origin`).

## Navigating & inspecting

```sh
wt ls            # list every worktree, branch, short SHA, and path (current one marked *)
wt path <branch> # absolute path to one worktree (use this to cd or to pass to git -C)
wt path          # absolute path to the container root
```

To present or move between worktrees for the user, prefer `wt ls` for an overview and `wt path` for a precise location.

## Removing

```sh
wt rm <branch>        # remove the worktree, keep the branch
wt rm <branch> -d     # also delete the branch (safe: refuses if unmerged)
wt rm <branch> -D     # also force-delete the branch
```

`wt rm` refuses to remove the worktree you are currently inside. If you need to remove the current one, `cd "$(wt path main)"` (or any other worktree) first, then remove. Use `wt prune` to clear stale records after manually deleting a directory.

## Setup commands (rarely needed mid-task)

- `wt clone <url> [dir]` — create the bare+worktrees layout from scratch.
- `wt add-fork <url>` — rename `origin` → `upstream` and add a fork as the new `origin`.

## Gotchas

- Run `wt` from anywhere inside the repo tree; it locates the container root via `.bare` automatically. If you get `not a wt-managed repository (no .bare found)`, you're outside such a repo — fall back to ordinary `git worktree` commands.
- Decorative progress goes to **stderr**; data (`wt path`, `wt ls`) goes to **stdout**. Capturing `$(wt path …)` yields only the path.
- A new worktree starts from the *current* branch — double-check you're branching from the intended base (or pass `base` explicitly).
