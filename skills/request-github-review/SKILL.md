---
name: request-github-review
description: Request an automated review on a GitHub PR. Tries Copilot first, falls back to @codex review comment if Copilot is not configured.
model: haiku
disable-model-invocation: false
---

# Request GitHub Review Skill

Request an automated reviewer on a GitHub pull request using the Copilot → Codex fallback chain.

## Input

A PR number. Detect it from context (e.g. recent `gh pr create` output, current branch PR) or ask the human: "Which PR number should I request a review for?"

## Steps

1. **Try Copilot first.**

   ```
   gh pr edit {number} --add-reviewer "Copilot"
   ```

2. **If that command exits non-zero**, inspect the error output:

   - If it contains any of: `"Could not resolve to a User"`, `"not found"`, `"is not a collaborator"`, `"does not have access"` — these signal that Copilot is permanently not configured on this repo. Fall through to Step 3.
   - If it fails for any **other** reason (network error, auth failure, rate limit, unexpected output) — **do not fall through**. Surface the error to the human and stop.

3. **Fall back to @codex review.**

   Post a trigger comment:

   ```
   gh pr comment {number} --body "@codex review"
   ```

   This is best-effort. The Codex bot must be installed and enabled on the repo for the comment to queue a review — `gh` exits 0 regardless of whether a review was actually queued. Inform the human of this uncertainty.

4. **If Step 3 fails** (i.e. `gh pr comment` exits non-zero), warn the human that no automated reviewer could be added and stop.

## Output

Report the outcome clearly:

- "Added Copilot as reviewer on PR #{number}." — if Step 1 succeeded.
- "Copilot is not available as a reviewer on this repo (permanent configuration state, not a transient failure). Posted `@codex review` on PR #{number} — review will be queued if the Codex bot is installed." — if Step 3 was used.
- Surface errors verbatim if either step failed unexpectedly.
