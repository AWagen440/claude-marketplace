---
name: git-commit
description: Run git-prepare, then actually execute it — branch, stage, commit, and (after confirming) push.
---

# git-commit

`git-prepare` only proposes a branch name / PR name / commit message as text.
This skill runs that proposal and then executes it: create the branch if
needed, stage the relevant files, commit, and push — pausing once, right
before the push, for explicit confirmation.

## Step 1: Run git-prepare

Invoke the `git-prepare` skill to get its three outputs:

- a suggested branch name (only produced when currently on `develop`/`main`/`master`)
- a PR name (informational only — this skill never opens a PR)
- a commit message

If git-prepare reports there are no uncommitted changes, stop here and relay
that to the user — there is nothing for this skill to do.

Display all three outputs to the user before moving on to Step 2 — including
the PR name. This skill never opens a PR itself, but the user still benefits
from seeing the suggested name now, so they have it in hand when they run
`gh pr create` (or their usual flow) later.

## Step 2: Create/switch branch if needed

If Step 1 produced a branch name (meaning you were on `develop`/`main`/`master`),
create and switch to it:

```bash
git checkout -b <branch-name>
```

If you were already on a non-trunk branch, git-prepare won't have suggested a
name — skip this step and stay on the current branch.

## Step 3: Stage the relevant changes

Run `git status` to see the full set of changed/untracked files. The default
is to stage **everything currently pending** — the commit message from Step 1
was written from the full diff of all uncommitted changes (git-prepare reads
`git diff HEAD` and is explicit about not dropping parts of it), so what gets
staged should match what that message describes. Stage every pending file
explicitly **by name** (never a blanket `git add -A` or `git add .`, so
nothing is swept in silently) — naming them individually is a safety
mechanism, not a cue to cherry-pick a subset.

Exclude a file only when it's clearly unrelated to this change or shouldn't
be committed at all (leftover scratch output, something still in progress) —
that's the exception, not the default. If excluding something would make the
Step 1 commit message inaccurate (describing changes that are no longer all
staged), regenerate the message to match what's actually being committed
rather than committing a mismatch.

Before staging, scan for anything that could carry secrets (`.env`,
credentials, keys, tokens) even under an innocuous filename. If you find
something suspicious, warn the user and exclude it rather than staging it
silently.

## Step 4: Commit

Commit the staged files with the exact commit message produced in Step 1,
passed via a heredoc so multi-line bodies format correctly:

```bash
git commit -m "$(cat <<'EOF'
<commit message from Step 1>
EOF
)"
```

If a pre-commit hook fails, fix the underlying issue, re-stage, and create a
**new** commit — never `--no-verify`, and never amend to route around a hook
failure (the failed attempt didn't produce a commit, so amending would rewrite
whatever commit came before it).

## Step 5: Confirm, then push

Show the user the commit that was just created (hash + subject, `git show
--stat HEAD` or similar) and the branch it's on, and ask for explicit
confirmation before pushing. Do not push until they say yes — this is the one
step in the chain that touches shared/remote state, so it doesn't run on
autopilot like Steps 2-4.

Once confirmed:

```bash
git push -u origin <branch-name>   # first push of a new branch
git push                            # branch already tracks a remote
```

If the push is rejected as a non-fast-forward, stop and tell the user —
never force-push without them explicitly asking for it.

## Step 6: Report

Report the branch name, the commit (hash + subject), the PR name from Step 1,
and where it was pushed (or that it wasn't, if the user declined). Do not
create a PR — that's outside this skill's scope; point the user at
`gh pr create` (or their usual flow) if they want one next.
