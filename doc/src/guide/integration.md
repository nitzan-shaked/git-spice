---
title: Integration branches
icon: octicons/git-merge-16
description: >-
  Build a single branch that combines multiple stack tips
  for testing or preview deploys.
---

# Integration branches

<!-- gs:version unreleased -->

An **integration branch** is a repo-scoped singleton that is rebuilt by
sequentially merging the tips of multiple tracked branches onto trunk.
It is intended for testing or preview builds that need to combine
several otherwise-independent stacks.

Integration branches are deliberately separate from regular stack
branches:

- They are never given a PR / change request.
- They are invisible to `gs branch` commands.
- They are regenerated, not edited: every rebuild force-overwrites the
  branch.

## Configuration

Configure the singleton with $$gs integration create$$:

```freeze language="terminal"
{green}${reset} gs integration create preview --tip feat-a --tip feat-b
```

The branch name (`preview` above) must not equal trunk. Tips must be
tracked branches; an untracked branch is rejected.

Add or remove tips later with $$gs integration tip add$$ and
$$gs integration tip remove$$.

## Rebuilding

Materialize or refresh the branch with $$gs integration rebuild$$:

```freeze language="terminal"
{green}${reset} gs integration rebuild
```

The worktree must be clean. Each tip is merged with `--no-ff`; `rerere`
is enabled for these merges so any recorded conflict resolutions are
replayed automatically. Rebuild can be invoked while the integration
branch itself is checked out; when finished, the worktree remains on
the integration branch.

If a tip conflicts, the merge is left in the worktree for you to
resolve. The conflicting paths are reported. To proceed:

1. Resolve the conflicts.
2. Stage the resolved files with `git add` and commit the merge with
   `git merge --continue`.
3. Re-run `gs integration rebuild` (or `gs intrb`). The rebuild
   resumes from the next tip after the one you just merged. The
   conflict resolution is also recorded by `rerere` so future
   rebuilds skip it automatically.

To bail out of the rebuild instead, run `git merge --abort`. The
pending-rebuild state remains; clear it by re-configuring the
integration with `gs integration delete` followed by `gs integration
create`.

### Generated-file conflicts

Some files in the repository are derived from source or test runs
(CLI reference, help fixtures, mocks, recorded ShamHub fixtures).
When two branches both touch them, the conflicts mix stochastic
noise (random IDs) with real structural changes (a new flag, a new
test case, a new mock method). Picking either side blindly silently
drops the structural change from the other branch.

The `.gitattributes` file declares a `regenerate` merge driver for
these paths. The driver re-runs the appropriate generator against
the merged source so the output reflects both branches' real
changes. To activate, run once after cloning:

```freeze language="terminal"
{green}${reset} mise run setup
```

After that, `gs intrb` (and any other `git merge` in the repo) will
auto-resolve generated-file conflicts by regenerating, not by
picking a side. Each generator runs at most once per merge even when
many files in its output set conflict.

## Switching to the integration branch

Use $$gs integration checkout$$ (shorthand: `gs intco`) to switch the
worktree to the integration branch:

```freeze language="terminal"
{green}${reset} gs intco
```

This fails if the branch has not yet been materialized; run
$$gs integration rebuild$$ first.

### Auto-rebuild

After any `gs branch restack`, `gs upstack restack`, `gs stack restack`,
or `gs repo restack`, the integration branch is automatically
regenerated when at least one of its tips has moved. The auto-rebuild
is silent when no tip has drifted.

## Submitting

Push the branch to the remote with $$gs integration submit$$:

```freeze language="terminal"
{green}${reset} gs integration submit
```

This pushes with `--force-with-lease` against the hash recorded at the
last successful push. **No change request is created.**

### Auto-submit

Once the integration branch has been pushed manually at least once,
subsequent `gs stack submit`, `gs upstack submit`, and
`gs downstack submit` invocations will keep the published branch in
sync by pushing the latest rebuild.

## Auto-resolving conflicts

<!-- gs:version unreleased -->

Even with rerere and the `regenerate` merge driver handling the
mechanical conflicts, rebuilds can still surface hand-resolution
conflicts when two tips touch the same file. For local-only integration
branches, you can configure a script that resolves these conflicts
automatically.

### Configuration

Two git-config keys control the feature:

```freeze language="terminal"
{green}${reset} git config spice.integration.resolver /path/to/resolver.sh
{green}${reset} git config spice.integration.autoResolve true
```

Per-invocation overrides are available on $$gs integration rebuild$$:

- `--auto-resolve` enables auto-resolve for this invocation.
- `--no-auto-resolve` disables it, even when the config has it on.

### The resolver protocol

The script is run from the repository root with the conflict-in-progress
worktree. It must produce a single JSON document on stdout matching:

```json
{
  "assumptions":      ["string", ...],
  "questions":        ["string", ...],
  "unresolved_files": ["string", ...]
}
```

All fields are optional. `{}` means "everything resolved cleanly."

| Response                                            | Behavior                                                                              |
|-----------------------------------------------------|---------------------------------------------------------------------------------------|
| Empty (or `assumptions` only)                       | Assumptions logged at info level. Files staged, merge committed.                      |
| `questions` populated                               | Each question is asked interactively. Answers append to the resolution file. Resolver is re-invoked. |
| `unresolved_files` populated, no `questions`        | Auto-resolve fails with an error; conflict surfaces normally for manual resolution.   |
| Non-zero exit code or invalid JSON                  | Auto-resolve fails; conflict surfaces normally. Output is logged at error level.      |

Anything in `unresolved_files` is left unstaged; everything else is
staged and committed.

### The resolution file

A file named `.integration_resolution.json` is maintained at the
repository root. **Add it to `.gitignore`** — it contains session
state, not source.

```json
{
  "current_merge": {
    "ours":   "preview",
    "theirs": "feat-a"
  },
  "resolutions": [
    {
      "merging_branches": {"ours": "preview", "theirs": "feat-a"},
      "resolution_instructions": [
        {"question": "...", "answer": "..."}
      ]
    }
  ]
}
```

`current_merge` is rewritten before each resolver invocation so the
script knows which merge is in progress. `resolutions` accumulates
Q&A pairs *across rebuilds*, so once you've answered a question for a
given (ours, theirs) pair, future rebuilds will not re-ask it.

Entries are pruned automatically when their branches are
untracked (`gs branch untrack`), deleted (`gs branch delete`), or
removed by `gs repo sync` after the underlying CR merges.

You may also edit the file by hand between iterations — for example,
to add standing guidance.

### Example: Claude-Code resolver

```sh
#!/bin/sh
# resolver.sh
claude --print --max-turns 30 <<'EOF'
You are resolving merge conflicts on a throwaway integration branch.

Read .integration_resolution.json to find the current merge and any
previously-recorded instructions. Use 'git ls-files --unmerged' to find
conflicted paths and 'git log -p ours theirs' to understand intent on
both sides.

Edit files in place to resolve conflicts. Then emit a single JSON
document on stdout:

  {"assumptions": [...], "questions": [...], "unresolved_files": [...]}

All fields are optional. {} means everything resolved cleanly.
If you can't resolve confidently, ask a question rather than guessing.
EOF
```

For Claude Code, you'll typically pre-approve the tools it needs in
`~/.claude/settings.json`:

```json
{
  "tools": {
    "Read": "allow",
    "Edit": "allow",
    "Bash(git log:*)": "allow",
    "Bash(git show:*)": "allow",
    "Bash(git diff:*)": "allow",
    "Bash(git ls-files:*)": "allow"
  }
}
```

### Iteration loop

When the resolver returns questions, git-spice will prompt you for
each in order. Your answers are appended to the resolution file
before the resolver is re-invoked.

If a resolution turns out to be wrong, the integration branch is
throwaway: investigate the diff, update your prompt or hand-edit
`resolution_instructions`, then run $$gs integration rebuild$$ again.

If rerere has recorded an incorrect resolution from an earlier run,
`git rerere clear` wipes the cache.

## Removing the configuration

Use $$gs integration delete$$ to remove the configuration. The
underlying Git branch is not deleted.
