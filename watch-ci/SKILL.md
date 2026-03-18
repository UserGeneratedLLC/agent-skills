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

### 2. Check Bugbot Comments (PR mode only)

This step runs **every iteration**, regardless of whether CI is passing, pending, or failed. Bugbot often posts review comments while CI is still running; waiting for a CI failure delays fixes unnecessarily.

a. **Fetch inline review comments:**
```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments --jq '.[] | select(.user.login | test("cursor")) | {id: .id, path: .path, body: .body, line: .line}'
```

b. **Fetch review-level comments:**
```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq '.[] | select(.user.login | test("cursor")) | {id: .id, state: .state, body: .body}'
```

c. **Deduplicate:** Compare each fetched comment `id` against the **handled set** (see Bugbot Deduplication below). Discard any comment whose `id` is already in the set.

d. **If new (unhandled) comments exist:** For each new comment, read the relevant source code, analyze, and fix using the same rules as step 5 (Analyze and Fix). After committing and pushing the fix, add the comment's `id` to the handled set. Then reset the backoff timer and return to step 1.

e. **If no new comments:** Continue to step 3.

### 3. Evaluate Results

- **No matching runs found** (zero results after filtering, or GitHub hasn't created the run yet) -- treat as pending. Sleep and re-poll.
- **All passing** -- every matched run/check has a success conclusion. Before reporting success, perform one final bugbot comment check (same as step 2a-2c). If new unhandled comments are found, process them per step 2d (fix, commit, push, reset backoff, return to step 1). Only if no new comments exist, report success and stop.
- **Any pending/in_progress** -- sleep and re-poll. Use exponential backoff: start at 15 seconds, double each iteration, cap at 120 seconds. Reset the backoff timer after any push.
- **Any failed** -- proceed to step 4.

### 4. Gather Failure Context

Collect ALL of the following before attempting any fix:

a. **Failed check logs:**
```bash
gh run view <run-id> --log-failed --repo <owner/repo>
```

b. **Unhandled bugbot comments** (PR mode only): Any bugbot comments NOT already in the handled set from step 2. Normally step 2 catches these first, but if a comment appeared between step 2 and step 4 in the same iteration, re-fetch and filter here.

### 5. Analyze and Fix

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

### 6. Commit and Push

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

### 7. Re-poll

After pushing, reset the backoff timer to 15 seconds and return to step 1.

## Bugbot Deduplication

Maintain a **handled set** of bugbot comment IDs across loop iterations (not persisted across separate `watch-ci` invocations).

- **Key:** The GitHub comment `id` field (unique per comment, numeric).
- **Add to set:** After a fix for a bugbot comment is committed and pushed, add that comment's `id` to the handled set.
- **Filter:** On each iteration, step 2 fetches all bugbot comments and discards any whose `id` is already in the handled set. Only remaining comments are processed.
- **Re-posted issues:** If bugbot posts a **new comment** (new `id`) about the same file or issue -- for example after a new commit triggers a fresh review -- it is treated as a new finding and processed normally, because it has a different `id`.
- **Resolved comments:** A comment that was previously handled but has since been marked "resolved" or "outdated" on GitHub is still in the handled set and will not be re-processed. Only a new comment (new `id`) triggers re-processing.

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
