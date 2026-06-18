# wt

A tiny [git worktree](https://git-scm.com/docs/git-worktree) manager for the **bare repo + sibling worktrees** layout: clone a repo once, then keep many branches checked out side by side as directories you can `cd` between instantly — no stashing, no switching, no rebuilding.

Instead of one working copy that you constantly `git checkout` back and forth, `wt clone` lays a repo out like this:

```
myproject/
├── .bare/          # the bare repository (all git data lives here)
├── .git            # a file pointing at .bare
├── main/           # worktree for the main branch
├── my-feature/     # worktree for a feature branch
└── pr-1234/        # worktree for a checked-out PR
```

Each directory is a normal working copy on its own branch. `wt add my-feature` makes a new one and drops you in it; `wt cd main` jumps back. New branches fork off whatever you currently have checked out. It also APFS-clones `node_modules` and `.env*` files from a sibling worktree on creation (macOS), so a fresh worktree is ready to run without a reinstall.

## Requirements

- `zsh` (the script uses `#!/usr/bin/env zsh`). Your interactive shell can be zsh **or** bash — only the `wt` script itself needs zsh on the `PATH`.
- `git` ≥ 2.32.
- macOS for the `node_modules` / `.env` APFS fast-clone (everything else is portable; the clone step just no-ops elsewhere).

## Install

It's just one script — put it somewhere on your `PATH` and make it executable:

```sh
git clone https://github.com/ksmithbaylor/wt.git
install -m 0755 wt/wt /usr/local/bin
```

Or copy `wt` anywhere on your `PATH` by hand and `chmod +x` it. No other files are needed at runtime.

Then add the shell wrapper to your shell rc so the navigation commands can change your shell's directory (see [Why the wrapper](#why-the-wrapper)):

```sh
echo 'eval "$(wt shell-init)"' >> ~/.zshrc   # or ~/.bashrc
```

Reload your shell, and you're set.

## Quick start

```sh
wt clone git@github.com:you/project.git    # clone into the bare+worktrees layout, cd into main/
wt add my-feature                          # new branch + worktree off your current branch, cd in
# ... hack, commit, push ...
wt cd main                                 # jump back to main/
wt ls                                       # list every worktree
wt rm my-feature -d                        # remove the worktree and delete the branch
```

## Commands

### Setup

| Command | Description |
| --- | --- |
| `wt clone <url> [dir]` | Clone a repo into the bare + worktrees layout and `cd` into the default branch. |
| `wt add-fork <url>` | Rename `origin` → `upstream` and add your fork as the new `origin` (default branch pulls from upstream, pushes go to your fork). |

### Worktrees

| Command | Description |
| --- | --- |
| `wt add <branch> [base]` | New worktree + branch, `cd` in. If the branch exists on a remote it tracks it; otherwise it forks off your current branch (or `base` if given). |
| `wt save <branch>` | Move your uncommitted changes onto a brand-new branch + worktree, and `cd` in. |
| `wt rm <branch> [-d\|-D]` | Remove a worktree; `-d`/`-D` also deletes the branch. |
| `wt ls` | List all worktrees (current one marked). |
| `wt cd <branch>` | Jump to a worktree by branch name. |
| `wt path [branch]` | Print a worktree's absolute path (or the container root). For scripting: `cd "$(wt path my-branch)"`. |
| `wt pr <number>` | Fetch a GitHub PR into a new worktree and `cd` in (works for fork PRs via `upstream`). |

### Maintenance

| Command | Description |
| --- | --- |
| `wt fetch` | Fetch all remotes (works from any worktree). |
| `wt prune` | Remove stale worktree records. |
| `wt shell-init` | Print the shell wrapper (for `eval` in your rc). |

Run `wt` with no arguments for help.

## Why the wrapper

A command running as a child process can't change its parent shell's working directory — so a plain executable could never actually `cd` you into a new worktree. `wt` solves this the way tools like `zoxide` and `fzf` do: the navigation subcommands print their destination on file descriptor 3, and the one-line function from `wt shell-init` captures it and runs the `cd` in your shell.

Everything still works without the wrapper — `wt add`/`wt cd`/etc. just print where they *would* have moved you instead. And `wt path <branch>` always prints a clean absolute path on stdout, so scripts and agents can do `cd "$(wt path my-branch)"` regardless.

## License

[MIT](LICENSE)
