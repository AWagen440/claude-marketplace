---
name: git-prepare
description: Propose a short PR name and a matching commit message for the current git repo.
---

# git-prepare

Propose a short, on-point PR title and a matching commit message by looking at
the branch's commit history (for the PR name) and the current uncommitted
diff (for the commit message). Output text only — do not run `git commit`,
`git add`, or `git push` unless the user explicitly asks you to.

## Step 1: Gather git context

Run these from the repo root (working directory should already be inside the
repo; if not, ask for the path):

```bash
git rev-parse --abbrev-ref HEAD                      # current branch
git status --porcelain=v1                            # uncommitted changes overview
git diff HEAD                                          # full diff: staged + unstaged
git log --oneline -20                                  # recent history, for style detection
git branch -a                                          # existing branch names, for naming-convention detection
```

If `git status --porcelain` and `git diff HEAD` both come back empty, tell
the user there are no uncommitted changes and stop — there's nothing to write
a commit message for. You can still offer a PR name from pending commits if
they ask.

## Step 2: Determine the base branch for the PR name

The PR name is based on **all commits on the current branch that aren't on
its base branch** — not just the latest one.

1. Look at the current branch name from Step 1.
2. If the branch is `develop`, `main`, or `master` itself, there's no "pending
   PR" — just describe the uncommitted diff instead (skip to Step 5, and use
   the diff for both the PR name and commit message). Also do Step 3 — work
   sitting directly on the trunk branch needs its own branch before a PR can
   exist.
3. Otherwise, decide the base branch:
   - If the branch name suggests a **hotfix** (contains `hotfix`, `fix/prod`,
     `production`, or the user says so) → base is `main` if it exists,
     else `master`.
   - Otherwise → base is `develop` if it exists, else fall back to `main`/
     `master`.
   - Check which branches actually exist with `git branch -a` or
     `git show-ref --verify --quiet refs/heads/<name>`.
4. Get the branch-only commits:
   ```bash
   git log <base>..HEAD --oneline
   git diff <base>...HEAD --stat
   ```
   (Use `...` merge-base diff so it's just this branch's changes, not
   unrelated commits that landed on base since branching off.)

If there are zero commits ahead of base AND no uncommitted changes, tell the
user there's nothing to name yet.

## Step 3: Suggest a branch name (only when currently on develop/main/master)

If the current branch is not `develop`/`main`/`master`, skip this step
entirely — the branch already exists, so there's nothing to suggest.

Otherwise:

1. Detect this repo's branch-naming convention from the `git branch -a`
   output gathered in Step 1 — look for a recurring prefix pattern (e.g.
   `feature/`, `story/`, `bugfix/`, `hotfix/`, `task/`, `chore/`). When more
   than one prefix is in use, prefer whichever one's *purpose* matches this
   change (a bug fix → the prefix past bug fixes used, a new capability → the
   prefix past features/stories used) over whichever is simply most frequent.
2. If no consistent convention is found among existing branches, default to a
   plain, prefix-free kebab-case name.
3. Build the descriptive part from the substance of the uncommitted diff —
   the same read used for the commit message in Step 7 — as a short
   kebab-case phrase naming what the change does (3-6 words), not a
   restatement of the file list.
4. Do not include ticket/issue numbers, matching the same rule as Step 4's PR
   name.

Result shape: `<prefix>/<kebab-case-description>` (or just
`<kebab-case-description>` with no convention detected).

## Step 4: Fold in the branch name

If the current branch is not `develop`/`main`/`master`, factor the branch
name into the PR name — it often encodes intent (e.g. `feature/user-avatars`,
`bugfix/null-pointer-login`). Don't just echo it verbatim; use it as a signal
for what the change is about, especially useful when commit messages
on the branch are vague (e.g. "wip", "fix").

Do **not** extract or prefix ticket/issue numbers (e.g. `JIRA-123`) into the
PR name — keep the title clean, just the description.

## Step 5: Detect the commit message convention

Check, in this order, and stop at the first one found:

1. Project convention docs — look for (in repo root or `docs/`):
   - `docs/git.md`, `docs/git.MD`, `docs/GIT.md`
   - `CONTRIBUTING.md`, `CONTRIBUTING`
   - `.gitmessage`, `.github/COMMIT_CONVENTION.md`
2. Commit lint configs — `commitlint.config.js`, `.commitlintrc`,
   `.commitlintrc.json`, `.commitlintrc.yml`, or a `commitlint` key in
   `package.json`.
3. If none of the above exist, infer from `git log --oneline -20` (from
   Step 1) — look for a recurring pattern like Conventional Commits
   (`feat:`, `fix:`, `chore:`, `refactor:`, ...) or another consistent
   prefix/style used across recent commits.
4. If no convention is found anywhere, default to a plain, short imperative
   sentence (e.g. "Add retry logic to upload handler"), no prefix.

Whatever convention you find, apply it to **both** the PR name and the
commit message for consistency, unless the convention doc clearly says it's
commit-message-only (e.g. Conventional Commits is normally fine for PR titles
too).

## Step 6: Write the PR name

- Short and to the point — aim for one line, roughly 50-72 characters,
  no trailing period.
- Based on the substance of the branch's pending commits (Step 2) — what
  actually changed, not a list of commit messages glued together.
- If commits on the branch are inconsistent/scattered, identify the dominant
  theme and summarize around it rather than trying to enumerate everything.
- Apply the detected convention style (Step 5).

## Step 7: Write the commit message

- Based on the full diff of **uncommitted** changes (from `git diff HEAD` in
  Step 1) — read the actual diff content, not just filenames, so the message
  reflects what really changed (not just "update file.py").
- If the diff touches multiple unrelated things, don't split it or drop
  parts — summarize all of it into a single cohesive message that covers the
  main changes (e.g. "Fix login bug and update dependency versions" rather
  than picking one and ignoring the rest).
- Apply the detected convention style (Step 5). If the convention supports a
  body (like Conventional Commits), keep the summary line short and put
  extra detail in a short body only if the diff is large/complex enough to
  need it — don't pad a simple change with an unnecessary body.

## Step 8: Present the result

Output the branch name suggestion (only if Step 3 applied), then the PR name,
then the commit message, clearly labeled, e.g.:

```
Suggested branch name: feature/retry-logic-for-upload-handler

PR name: Add retry logic to upload handler

Commit message: feat(upload): add retry logic on transient failures
```

Briefly note which base branch you compared against (develop/main/master) and
which convention source you used (e.g. "matched docs/git.md convention"), so
the user can sanity-check it — but keep this note to one short line, not a
full explanation. If a branch name was suggested, also note which existing
branches informed the detected naming convention (or that none was found).

Do not run `git commit`, `git add`, `git checkout -b`, or any mutating git
command. If the user wants that, they'll ask separately.
