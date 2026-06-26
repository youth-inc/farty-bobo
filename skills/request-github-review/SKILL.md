---
name: request-github-review
description: Request an automated review on a GitHub PR. Prompts the human to pick a reviewer (Copilot, Claude, Codex on GitHub, or Codex within Claude Code). Also runs the bot feedback loop: waits for CI/bot reviews, addresses comments, resolves CI failures, then marks draft PRs ready for review.
model: haiku
disable-model-invocation: false
---

# Request GitHub Review Skill

Request an automated reviewer on a GitHub pull request. Prompts the human to select which reviewer(s) to use, then manages the bot feedback loop after review is requested — for all PRs, not just drafts.

This skill may be invoked multiple times in the PR lifecycle (e.g. after each new round of bot feedback).

## Input

A PR number. Detect it from context (e.g. recent `gh pr create` output, current branch PR) or ask the human: "Which PR number should I request a review for?"

## Steps

1. **Prompt the human to select a reviewer.**

   Present the following options and ask the human to choose one or more:

   ```
   Which reviewer(s) do you want on this PR?

   1. Copilot — GitHub Copilot code review (added as a reviewer via the GitHub API)
   2. Claude — Posts an @claude review trigger comment on the PR
   3. Codex on GitHub — Posts a @codex review trigger comment on the PR
   4. Codex within Claude Code — Runs /codex:adversarial-review locally to find critical bugs and vulnerabilities that may have escaped /critique
   ```

   Wait for the human's selection before proceeding. The human may pick more than one. If no response is given or context is ambiguous, ask again — do not guess.

2. **Execute each selected option.**

   Run only the steps corresponding to the human's selection(s):

   **Option 1 — Copilot:**
   ```
   gh pr edit {number} --add-reviewer "Copilot"
   ```
   - If this exits non-zero and the error contains any of: `"Could not resolve to a User"`, `"not found"`, `"is not a collaborator"`, `"does not have access"` — report that Copilot is not configured on this repo (permanent state, not transient) and stop this option.
   - If it fails for any **other** reason (network error, auth failure, rate limit, unexpected output) — surface the error verbatim and stop this option.

   **Option 2 — Claude:**
   ```
   gh pr comment {number} --body "@claude review"
   ```
   - If this exits non-zero, warn the human and stop this option.
   - If it exits 0: inform the human this is best-effort — the `@claude` bot must be installed and enabled on the repo for a review to actually queue. `gh` exits 0 regardless.

   **Option 3 — Codex on GitHub:**
   ```
   gh pr comment {number} --body "@codex review"
   ```
   - If this exits non-zero, warn the human and stop this option.
   - If it exits 0: inform the human this is best-effort — the `@codex` bot must be installed and enabled on the repo for a review to actually queue. `gh` exits 0 regardless.

   **Option 4 — Codex within Claude Code:**

   This runs entirely locally against the current git branch. No GitHub API call is made.

   Before invoking, ensure the PR's branch is checked out locally. If it isn't, fetch and check it out:
   ```
   gh pr checkout {number}
   ```

   Then determine the PR's base branch:
   ```
   gh pr view {number} --json baseRefName --jq '.baseRefName'
   ```

   Invoke `/codex:adversarial-review` with `--wait` (to keep the run foreground and surface output this turn) and `--base {base-branch}` (to scope the diff to what the PR actually changes):
   ```
   /codex:adversarial-review --wait --base {base-branch}
   ```

   Pass the following as the focus prompt:
   > "Focus on the most critical bugs and security vulnerabilities that escaped /critique. Look for: logic errors, edge-case failures, injection vectors, auth bypasses, data races, and improper error handling."

   Wait for the run to complete. Surface the verbatim Codex output — do not paraphrase, summarize, or reformat it.

3. **If every selected option errored before issuing its action**, stop and report what failed. (Note: Options 2 and 3 always exit 0 if the `gh` command itself ran — "no review queued" is not an error.)

4. **Bot review loop.**

   **If this skill is already running in a parent invocation (i.e. the call chain is `/request-github-review` → `/address-pr-comments` or `/resolve-ci-failures` → `/critique` → `/request-github-review`), skip Step 4 entirely** — the bot loop is already active and re-entering it would cause an infinite loop.

   After review is requested, wait for automated reviewers (bots, linters, CI) to post their feedback:

   a. **Estimate the wait time** based on PR size and recent CI history:

      i. **Measure the PR size:**
         ```
         gh pr diff {number} --patch | wc -l
         ```
         Classify: small (< 200 lines), medium (200–800 lines), large (> 800 lines).

      ii. **Sample recent CI run durations** from the last 5 completed workflow runs on the repo's default branch:
         ```
         gh run list --branch {default_branch} --status completed --limit 5 --json databaseId,updatedAt,createdAt
         ```
         For each run, compute `duration = updatedAt - createdAt` in minutes. Take the **median** as the baseline CI duration.

      iii. **Compute the wait estimate:**
         - Start with the median CI duration from (ii). If no runs are found, use 5 minutes as the fallback.
         - Add a buffer for bot reviewers (linters, code scanners): +2 minutes for small PRs, +3 for medium, +5 for large.
         - Cap the total at 15 minutes — if the estimate exceeds this, use 15 minutes. Anything longer and the human should be deciding.
         - Floor at 3 minutes — bots need at least this long to spin up.

      iv. **Announce the estimate:** "Based on recent CI runs (median: {N}min) and PR size ({size}), waiting {estimate} minutes for bot reviews on {PR_URL}. Say 'skip' to proceed immediately or give me a different number."
         - If the human responds with a number, use that instead.
         - If the human says 'skip', proceed immediately to step b.
         - Otherwise, use the computed estimate.

      v. Use `ScheduleWakeup` with the final delay (converted to seconds) to set the timer.

   b. **When the timer fires, run the feedback loop:**
      - Invoke `/address-pr-comments` — this reads all unresolved comments (including bot comments) and addresses actionable ones. If code changes are made, `/address-pr-comments` will internally invoke `/critique` to commit and push fixes.
      - Invoke `/resolve-ci-failures` — this checks CI status, investigates failures, and fixes them. If fixes are made, it will internally invoke `/critique` to commit and push.

   c. **Mark the PR as ready for review (draft PRs only):**
      - Check if the PR is still a draft: `gh pr view {number} --json isDraft --jq '.isDraft'`
      - If it is a draft, run `gh pr ready {number}` to convert it to ready-for-review status.
      - Announce to the human: "PR {PR_URL} is now marked ready for human review. Bot feedback has been addressed and CI is passing."
      - If CI is still failing after `/resolve-ci-failures` (e.g., infrastructure issues that couldn't be auto-fixed), warn the human instead: "PR {PR_URL} is marked ready for review, but CI still has failures that need manual attention: {summary of remaining failures}."
      - If the PR was already non-draft, skip the `gh pr ready` call but still announce that bot feedback has been addressed.

## Output

Report the outcome per selected option:

- **Copilot:** "Added Copilot as reviewer on PR #{number}." — or report the permanent configuration error if unavailable.
- **Claude:** "Posted `@claude review` on PR #{number} — review will be queued if the bot is installed."
- **Codex on GitHub:** "Posted `@codex review` on PR #{number} — review will be queued if the bot is installed."
- **Codex within Claude Code:** Surface the verbatim output from `/codex:adversarial-review`. Do not paraphrase or reformat.
- Surface errors verbatim for any option that failed unexpectedly.
- For all PRs: report when the bot loop completes and feedback has been addressed. For draft PRs specifically, also report when the PR is marked ready (or when CI still needs attention).
