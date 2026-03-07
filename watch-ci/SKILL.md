---
name: watch-ci
description: |
  Monitor GitHub CI checks on a PR or branch, automatically fix failures, and loop until green. Handles Cursor Bugbot review comments, failed workflow logs, lint/format/test errors. Asks the user only when the fix is ambiguous.

  Use when the user says "watch ci", "watch the ci", "monitor ci", "fix ci", "watch PR", links a GitHub PR and asks to monitor it, or asks to keep fixing CI until it passes.
---

# Watch CI

Continuously monitor GitHub CI checks, fix failures, and loop until all checks pass. Works with any GitHub repository.

## Input Resolution

1. **PR URL provided** (e.g. `https://github.com/owner/repo/pull/123`): Extract owner, repo, and PR number. Use PR mode.
2. **No PR URL**: Detect repo from cwd via `gh repo view --json nameWithOwner -q .nameWithOwner`. Use the current branch. If a PR exists for the branch, use PR mode. Otherwise use branch mode.

## The Loop

Repeat until all checks pass or a bail condition is hit:

### 1. Poll Check Status

**PR mode:**
```bash
gh pr checks <number> --repo <owner/repo>
```
After fetching, exclude any check whose name case-insensitively matches `release`, `deploy`, or `publish`.

**Branch mode:**
```bash
gh run list --branch <branch> --limit 5 --json status,conclusion,databaseId,name,headSha
```
After fetching, apply two filters in order:
1. Discard runs whose `name` case-insensitively matches `release`, `deploy`, or `publish`.
2. Discard runs whose `headSha` does not match the local `git rev-parse HEAD`.

Only the remaining runs are evaluated.

### 2. Evaluate Results

- **No matching runs found** (zero results after filtering, or GitHub hasn't created the run yet) -- treat as pending. Sleep and re-poll.
- **All passing** -- every matched run/check has a success conclusion. Report success and stop.
- **Any pending/in_progress** -- sleep and re-poll. Use exponential backoff: start at 15 seconds, double each iteration, cap at 120 seconds. Reset the backoff timer after any push.
- **Any failed** -- proceed to step 3.

### 3. Gather Failure Context

Collect ALL of the following before attempting any fix:

a. **Failed check logs:**
```bash
gh run view <run-id> --log-failed --repo <owner/repo>
```

b. **Bugbot review comments** (PR mode only):
```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments --jq '.[] | select(.user.login | test("cursor")) | {path: .path, body: .body, line: .line}'
```

c. **PR review comments** (PR mode, catches review-level feedback):
```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq '.[] | select(.user.login | test("cursor")) | {state: .state, body: .body}'
```

### 4. Analyze and Fix

For each failure, read the relevant source code before deciding on a fix. Do NOT guess.

**Unambiguous fixes** -- apply without asking:
- Formatting errors (`cargo fmt`, `stylua`, `prettier`, etc.)
- Lint warnings with a single clear resolution
- Snapshot test updates (`cargo insta accept`, jest `--updateSnapshot`)
- Missing imports, unused variables, trivial compile errors
- Dependency lock file drift (`cargo update`, `npm install`)

**Ambiguous fixes** -- ask the user via AskQuestion:
- Test failures where the test might be correct and the implementation wrong (or vice versa)
- Multiple valid approaches to fix a compile/type error
- Architectural decisions (e.g. "should this be a breaking change?")
- Any failure where the root cause is unclear after reading the code

**Investigation protocol for test failures:**
1. Read the failing test to understand what behavior it asserts
2. Read the implementation code the test exercises
3. Determine root cause: is the test wrong, or is the implementation wrong?
4. If the implementation regressed, fix the implementation
5. If the test fixture is stale from an intentional change, update the fixture
6. Never delete or gut a test just to make CI pass

### 5. Commit and Push

- Stage only files related to the fix
- Write a descriptive commit message: what failed, why, and what was changed
- Never force-push or rewrite history
- Never skip pre-commit hooks (no `--no-verify`)

```bash
git add <specific files>
git commit -m "$(cat <<'EOF'
fix: <what was fixed>

<CI check name> failed because <root cause>.
<description of the fix>.
EOF
)"
git push
```

### 6. Re-poll

After pushing, reset the backoff timer to 15 seconds and return to step 1.

## Bail Conditions

Stop the loop and ask the user if:

- The **same check fails 3 times** in a row after attempted fixes -- something deeper is wrong.
- A failure requires **credentials, secrets, or infrastructure changes** the agent cannot make.
- The fix would require a **force push, history rewrite, or branch reset**.
- CI has been **pending for more than 15 minutes** with no status change.

## Completion

When all checks pass, report:
- Total number of loop iterations
- Fixes applied (list of commits)
- Any checks that were flaky (passed after retry without code changes)
