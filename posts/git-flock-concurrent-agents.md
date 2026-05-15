---
title: "When five agents commit at once"
description: "Two small shell scripts that prevent .git/index.lock races and silent file sweeps when multiple AI agents share a repository."
date: "2026-05-15"
tags: ["Git", "Claude Code", "Homelab", "Shell"]
draft: true
---

Last week I ran five Claude Code agents simultaneously against the same homelab repository. They were doing independent work -- updating Traefik configs, rotating secrets, bumping container images. The problem is they all used the same working tree, and git does not handle that gracefully.

The first symptom was `.git/index.lock`:

```
Another git process seems to be running in this repository, e.g.
an editor opened by 'git commit'. Please make sure all processes
are terminated then try again.
```

One agent's pre-commit hook was taking three minutes to run (a cold Docker pull for a secret scanner). Every other agent that tried to commit during that window failed immediately. The faster agents kept retrying; by the time the slow hook finished, some had already given up and the others were racing each other.

The second symptom was subtler and I didn't catch it until I read the git log. One commit with the message "fix crowdsec config" had absorbed seven unrelated files staged by a different agent. Another commit, later that day, absorbed seventeen. The messages described the work of one agent but the diffs contained the work of several.

These are two separate failure modes. I wrote two small tools to address them.

---

## The first problem: concurrent lock contention

Git serializes index writes through `.git/index.lock`. One process at a time. If two processes try to commit simultaneously, the second one sees the lockfile and fails.

The fix is to move the serialization point earlier -- before the `git add`, not at `git commit`. If the agents take turns on the whole add-commit-push sequence rather than racing to the same checkpoint, the lockfile conflict never happens.

`flock(1)` does this. It's a util-linux tool that takes an exclusive lock on any file, runs a command, and releases the lock when the command exits. One lockfile per repository, one process through at a time, everyone else waits.

```sh
# gitop: serialize git operations via flock on .git/gitop.lock
gitop 'git add foo.yml && git commit -m "add foo" -- foo.yml && git push'
```

The implementation is straightforward:

```sh
repo=$(git rev-parse --show-toplevel)
lockfile="$repo/.git/gitop.lock"
GITOP_HELD="$repo" flock -x -w 300 "$lockfile" -c "$*"
```

`flock` opens `gitop.lock`, blocks until it can take an exclusive lock (up to 300 seconds), runs the command with the lock held, and releases it when the command exits -- whether it succeeds, fails, or crashes. There's no cleanup step and no stale lock problem. The `GITOP_HELD` variable signals reentrancy: if a nested `gitop` call detects it, the inner command runs directly without trying to re-acquire the same lock.

The lock file itself is benign -- an empty file at `.git/gitop.lock`. Git ignores the `.git/` directory for tracking purposes so it doesn't need to be in `.gitignore`.

---

## The second problem: silent file sweeps

Fixing the concurrent racing doesn't fix the other problem. Even with perfect serialization, a bare `git commit -m "msg"` still commits everything currently staged -- including files staged by a different agent ten minutes ago.

Most developers know that `git add` stages files. Fewer know that `git commit` has a pathspec form that limits a commit to explicitly named paths, regardless of what else is in the index:

```sh
git commit -m "fix crowdsec config" -- apps/crowdsec/config.yml
```

The `--` separator puts git into pathspec mode. The commit is built from the listed paths only. Anything else in the staging area stays staged for the next commit. From `git-commit(1)`:

> If pathspecs are given, make a commit ... disregarding any contents
> that have been staged for other paths.

This is the correct model for a shared working tree. Each agent stages its files, each commit names exactly those files, and no one sweeps up anyone else's work.

The problem is that nothing enforces it. A `git commit -m "msg"` with no pathspec silently absorbs the whole index. The commits look fine until you read the diffs.

`gitc` is a wrapper that makes the `--` separator mandatory:

```sh
gitc -m "msg" -- file1 file2   # works
gitc -m "msg"                  # rejected: missing '--' separator
```

Everything else passes through unchanged. It's `exec git commit "$@"` with a pre-flight check.

---

## Using them together

The two tools address orthogonal failure modes and compose cleanly:

```sh
# gitop serializes; gitc enforces pathspec
gitop 'git add apps/crowdsec/config.yml && gitc -m "fix crowdsec config" -- apps/crowdsec/config.yml && git push'
```

`gitop` ensures no two of these sequences run at the same time. `gitc` ensures each commit contains only the files it says it does.

---

## The bug that surfaced while debugging this

I had both tools in place before the freeze session. So why didn't they help?

`gitop` was a function defined in `~/.zshrc`. From an interactive zsh session it worked fine. But Claude Code agents run git commands from bash subshells, and bash doesn't source `~/.zshrc`. The agents couldn't find `gitop`. The locking mechanism was present in my shell config and completely invisible to every process that needed it.

`gitc`, by contrast, was already an external script at `~/.local/bin/gitc` -- I'd moved it there earlier for exactly this reason. `gitop` had never been promoted.

The fix was one script. Same logic, same interface, now lives at `~/.local/bin/gitop` alongside `gitc`. Any shell that has `~/.local/bin` on PATH can use it.

To verify it actually worked, I spawned four agents simultaneously against the same repository. Each created a file and committed it via `gitop` from bash. All four completed with exit code 0, no `index.lock` errors, and the git log showed four clean sequential commits in the order the lock was acquired:

```
58b35c8 test: lock-agent-4
9e7f18a test: lock-agent-3
f0d295d test: lock-agent-2
285cb63 test: lock-agent-1
```

---

## The tools

Both scripts are in [git-flock](https://github.com/pike00/git-flock). POSIX sh, no dependencies beyond `flock(1)` (ships with util-linux on Linux; `brew install util-linux` on macOS) and git.

```sh
git clone https://github.com/pike00/git-flock
cd git-flock
./install.sh   # copies gitop and gitc to ~/.local/bin
```

The repository also has a test suite that exercises serialization timing, reentrancy, timeouts, and the pathspec enforcement.

---

One thing I'd note: neither tool helps if you don't use them. The habit has to be consistent -- every agent, every commit sequence, always `gitop` + `gitc`. The pre-commit hook in my repo aborts non-interactive commits with more than five staged files as a final backstop, but it fires after the damage is done. Getting the tools into the agents' PATH and into their instructions is the actual fix.
