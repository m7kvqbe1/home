+++
title = 'Parallel Agents with Ticket-Scoped Git Worktrees'
date = 2026-09-16T09:00:00+01:00
draft = false
+++

My day-to-day workspace is twenty-odd sibling repositories under one directory, and a typical ticket touches two to four of them. That was fine when I was the one typing. It stopped being fine when a coding agent started doing the typing.

[Claude Code](https://www.anthropic.com/claude-code) will happily spend ten minutes on a change across a couple of repos, and for most of that time I'm waiting. The obvious move is to start the next ticket, except the agent is working in a checkout, and `git stash && git checkout` underneath it is a good way to end up with half of ticket A on ticket B's branch. Every habit I had assumed one pair of hands per working tree.

So the bottleneck moved. Typing speed doesn't matter much any more. What matters is how many isolated streams of work I can keep open, and how quickly I can tell which one needs me. These are the small tools I built for that.

## Overview

- **Ticket-scoped worktrees**: one command creates matching `git worktree`s across every repo a ticket touches, on the same branch name.
- **Dependencies cloned, not symlinked**: APFS copy-on-write, because Vite refuses a symlink that points outside the worktree.
- **Safe teardown**: removal refuses if anything is uncommitted or unpushed.
- **A slash command**: the agent decides *which* repos and *which* ticket; a script does the rest.
- **Hooks that survive worktrees**: commit conventions checked against the repo a command actually targets.
- **A fast workspace**: dormant repos hidden from search, because an agent greps far more than I do.
- **A dashboard**: every branch, PR, CI run and unresolved review thread, ranked by what's blocking.

## The unit of isolation is the ticket

Claude Code can already run a subagent in a throwaway worktree. That suits a *task*. A *ticket* spans several repos, outlives any one session, and needs a branch name that matches the ticket number so CI and the PR template line up. So the helper isolates per ticket:

```text
~/Projects/
  portal/                     ← main checkout
  e2e-tests/
  .worktrees/
    PROJ-1755/
      portal/                 ← PROJ-1755/selected-items-total-size
      e2e-tests/              ← PROJ-1755/selected-items-total-size
    PROJ-1762/
      broker/                 ← PROJ-1762/retry-backoff
```

```bash
ticket-worktree.sh 1755 portal e2e-tests --desc selected-items-total-size
```

Each named repo gets a worktree off the remote default branch, reusing the branch if it already exists:

```bash
for r in "${repos[@]}"; do
  main="$ROOT/$(resolve "$r")"
  wt="$target/$(resolve "$r")"
  git -C "$main" fetch origin "$default" --quiet
  if git -C "$main" show-ref --verify --quiet "refs/heads/$branch"; then
    git -C "$main" worktree add "$wt" "$branch"
  else
    git -C "$main" worktree add -b "$branch" "$wt" "origin/$default"
  fi
done
```

### Dependencies without the install

A bare worktree needs a dependency install before anything runs, multiplied by every repo in the ticket. My first version symlinked the main checkout's dependency directory into the worktree. That's instant, but it doesn't work. Vite resolves the symlink to its real path, notices that path is outside the worktree root, and refuses to serve it. The first test run fails on the first setup file.

The fix is an APFS copy-on-write clone:

```bash
if cp -Rc "$main/node_modules" "$wt/node_modules" 2>/dev/null; then
  echo "   cloned node_modules"
else
  ln -s "$main/node_modules" "$wt/node_modules"   # non-APFS fallback
fi
```

`cp -c` uses `clonefile(2)`, so the copy shares blocks with the original until something writes to it. For a dependency tree of several hundred megabytes it takes about fifteen seconds and costs almost nothing on disk, and every tool sees an ordinary directory inside the worktree. Two agents on two tickets no longer share mutable state either. If the branch changes the lockfile you still need a real install, but that fails loudly, and the clone wins the rest of the time.

## Teardown that refuses to lose work

"Delete this directory" is the sort of instruction I'll eventually give an agent late on a Friday, so removal checks first:

```bash
dirty=$(git -C "$wt" status --porcelain \
  | grep -Ev '^\?\? \.env' | wc -l)
unpushed=$(git -C "$wt" log --oneline '@{u}'..HEAD | wc -l)

if [ "$dirty" != "0" ] || [ "$unpushed" != "0" ]; then
  echo "REFUSING to remove $repo — $dirty uncommitted, $unpushed unpushed."
  continue
fi
```

The dirty check ignores the files the helper itself dropped in, and the unpushed check counts only *this worktree's* branch against its own upstream. The naive version kept refusing because some unrelated local branch was ahead. Only once the worktree is gone does it delete the branch. There is no `--force`.

## Handing it to the agent

The script is deliberately dumb. The judgement (which repos does this ticket touch?) is what I want the model for:

```markdown
---
description: Create matched git worktrees across sibling repos for one ticket
allowed-tools: Bash(~/scripts/ticket-worktree.sh*), Bash(git -C *), Bash(cd *)
---

Run the helper with: $ARGUMENTS

If the issue number wasn't given, take it from the current branch name before
asking me for it. If no repos were named, work out which ones the ticket
touches and propose a list rather than guessing silently.
```

`allowed-tools` pins the command to the helper and read-only git, so the agent can't improvise a `git worktree remove --force` when the script says no. Asked to set up a ticket I've only described in prose, it reads the ticket, works out it needs the portal and the e2e suite but not the broker, and proposes that before running anything. Same split as [my last post](/posts/claude-code-presentation-deck-plugin/): judgement in the model, mechanics in a script.

## Running them side by side

From there, parallelism is just tmux: one window per ticket, each shell started in that ticket's directory, each running its own agent.

What makes it tolerable is a notification hook. A `UserPromptSubmit` hook stamps the start of each turn; a `Stop` hook raises a macOS notification only if the turn ran over a minute, labelled with directory and branch:

```bash
[ "$elapsed" -ge 60 ] || exit 0
label="$(basename "$cwd") ($(git -C "$cwd" branch --show-current))"
osascript -e "display notification \"Finished after ${elapsed}s\" \
  with title \"Claude Code\" subtitle \"${label}\""
```

Quick exchanges stay silent. Long runs tell me which ticket wants me back.

## Guardrails that survive the move

Parallel agents multiply every small inconsistency. Three commit conventions coexist in this workspace, and a model fresh from one repo will carry its convention into the next. A `PreToolUse` hook validates the message before the commit runs. The subtlety is working out which repo a command targets, since agents love `cd repo && git commit`:

```python
def repo_name(cwd, tokens):
    if "-C" in tokens:
        cwd = os.path.join(cwd, tokens[tokens.index("-C") + 1])
    top = subprocess.run(["git", "-C", cwd, "rev-parse", "--show-toplevel"],
                         capture_output=True, text=True)
    return os.path.basename(top.stdout.strip())
```

`rev-parse --show-toplevel` returns the worktree's own root, whose basename is still the repo name, so `.worktrees/PROJ-1755/portal` and `portal` resolve to the same convention with no special-casing.

## Keep the workspace fast

This one is unglamorous and it was the biggest single win. The projects directory had accumulated about 130 dormant repos. I never noticed the cost because I rarely searched everything. An agent does, constantly. I moved them into `_archive/` and added a `.ignore` at the root:

```text
_archive/
```

ripgrep honours `.ignore`, and so does the agent's search tool. Searchable files went from roughly a million to about five thousand, and stale copies of functions stopped turning up as false leads. If you do one thing from this post, do this.

## Knowing where everything is

`/in-flight` gathers local git state for every repo and, in parallel, asks GitHub for the open PR, its checks, review decision and the count of unresolved review threads (that last one needs GraphQL, the REST API doesn't expose it). Clean repos on `main` are dropped:

```text
REPO           BRANCH                                 LOCAL   PR     CI / REVIEW
broker         PROJ-1762/retry-backoff                3∆      —
  > e2e-tests  PROJ-1755/selected-items-total-size    clean   #203   1 failing
portal         PROJ-1740/rename-batch-actions         clean   #598   checks green  ✓approved
  > portal     PROJ-1755/selected-items-total-size    ↑2      #612   checks green  review needed

4 in flight, 14 idle on main
```

Indented rows are linked worktrees, found via `git worktree list --porcelain` and listed after their main checkout, or on their own, as with `e2e-tests` above, when the main checkout is idle on `main`. The first version only scanned main checkouts, so a ticket that lived entirely in worktrees was invisible to the one tool whose job is to show me everything. I only noticed when I went looking for one.

The slash command asks the model to rank that (failing CI, then unresolved comments, then approved-and-green, then forgotten local work) and end with one suggested next action. A stand-up for one person and their agents, in about four seconds.

## What I'd still change

The dependency clone should compare lockfile hashes and warn when the branch has diverged, rather than leave me to find out from a confusing test failure. And the removal path could offer to push before refusing, since "unpushed" is usually the only thing standing between me and a clean workspace.

But the lesson holds. Git already had the primitive for isolating parallel work. The rest is a few hundred lines of shell and being disciplined about what the model decides versus what a script does.
