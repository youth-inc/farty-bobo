---
name: critique
description: Used to run githooks, perform code & plan reviews with expert critics, and commit/push verified changes
disable-model-invocation: false
---


# Code and Plan Review Skill

## Temp Directory

Planning and review artifacts written by `/plan-task` and `/review-multiple-prs` live outside the repo under:

```
TEMP_DIR=/tmp/<repo-name>/<branch-name>
```

Resolve `<repo-name>` and `<branch-name>` using:
- `<repo-name>` = `basename $(git rev-parse --show-toplevel)`
- `<branch-name>` = `git branch --show-current`

**Worktree note:** if running inside a git worktree, `git rev-parse --show-toplevel` returns the worktree path, not the main repo root. Use `git rev-parse --git-common-dir | xargs dirname` to get the true repo root when resolving `<repo-name>`.

**macOS note:** `/tmp` → `/private/tmp` and is cleared on reboot. Decisions scratch files do not persist across reboots; if a session spans a reboot, the human will need to re-state decisions manually.

---

1. Either detect the repos affected from the context OR ask the human to select the repos that should be used to review the code changes or the staged implementation plan (md file):

- current directory
- a list of repos in the file system

2. enable and run githooks in the repos impacted by the code changes and fix errors (linting, type checks, tests) before committing. Most repos have dedicated commands to achieve these goals either in package.json or pyproject.toml.

3. **Skipped-test check — MANDATORY. Do not present review options or proceed to commit until this check is resolved.** Inspect the *added lines only* in the diff (lines beginning with `+`) for skip directives (see the canonical list in the **Skip directives** section below). Only flag directives that appear on added lines — do not flag pre-existing skips in unrelated hunks. If any are found, surface them to the human as HIGH severity findings and require an explicit written reason for each before proceeding. Critics will NOT separately flag these; this pre-check is the sole owner of skipped-test findings to avoid double-reporting.

4. Prompt the human to select their preferred review options:

- one generalist critic (good for simple tasks)
- a team of critics who have expertise in the technology stacks being used and best security practices. The critics must create a markdown file to list their revisions sorted by severity (high first).
- Modularity review: Run `/modularity:review` to analyze coupling across modules using the Balanced Coupling model. Best for changes that introduce or restructure component boundaries, touch multiple modules, or modify public APIs/contracts.
- Codex review: Ask the human to run `/codex:review`
- Manual review by the human.

**Skip directives — canonical list (used by Step 3 and critic prompts):**

`test.skip`, `it.skip`, `xit`, `xdescribe`, `@pytest.mark.skip`, `t.Skip(`, `pending`, or any framework-equivalent skip/todo/pending marker.

**How to run critics:**

All critics use a **file-based output contract**. The orchestrator hands the agent a temp file path; the agent writes its findings there and goes idle; the orchestrator reads the file.

Critic output files live under `TEMP_DIR` (same convention as the rest of this skill — see the Temp Directory section above):
- Generalist: `$TEMP_DIR/critique-{summary}.md` where `{summary}` is a short kebab-case slug of the change (e.g. `fix-auth-redirect`, `skill-patch`).
- Team: `$TEMP_DIR/critique-{specialty}.md` per agent (e.g. `critique-security.md`, `critique-correctness.md`).

Resolve `ticket-id` by checking the current branch name or conversation context; use `NO-TICKET` if none is found. Create `TEMP_DIR` with `mkdir -p` before spawning any agents.

**Adversarial framing — applies to ALL critic agents (generalist, team, and inline fallback reasoning):**

Every critic must be prompted as an adversary, not a passive reviewer. Include this instruction in every critic's prompt, and apply it when falling back to inline reasoning: "You are an adversarial critic. Assume the code is wrong until proven otherwise. Your job is to attack this change — find bugs, security holes, broken invariants, missing edge cases, and bad design decisions. Be aggressive. Do not give the benefit of the doubt. If something looks suspicious, call it out. Findings that survive scrutiny are more valuable than polite observations."

**One generalist critic:**
a. Create `TEMP_DIR` with `mkdir -p $TEMP_DIR`.
b. Spawn a single Agent with a focused prompt containing: the full diff, the repo name, the task description, the output file path, and the adversarial framing above. Instruct it to write findings sorted by severity (HIGH / MEDIUM / LOW) to that file, or write "no findings" if nothing is worth raising. Do NOT instruct the critic to flag skipped tests — the Step 3 pre-check owns that. Keep the prompt tight — do not dump the full conversation history into it.
c. After the Agent tool call returns (the agent goes idle), read the output file.
d. If the file is missing or empty, fall back to inline reasoning using the adversarial framing above — and **disclose to the human** that the critic produced no output file and this review is inline-only.
e. Present findings to the human (global Step 5).

**Team of critics:**
a. Create `TEMP_DIR` with `mkdir -p $TEMP_DIR`.
b. Spawn one Agent per specialty (e.g. security, performance, correctness) — each with its own output file path and the adversarial framing above. Do NOT instruct critics to flag skipped tests — the Step 3 pre-check owns that. Run all agents in parallel (single message, multiple Agent tool calls).
c. After all Agent tool calls return (all agents go idle), read all output files.
d. For any file that is missing or empty, fall back to inline reasoning for that specialty using the adversarial framing above — and **disclose to the human** which critics produced no output and that those portions are inline-only.
e. Synthesize all findings into a single sorted list, deduplicating overlapping issues across agents. Present to the human (global Step 5).

5. Present the review findings to the human. Then prompt them to select the next step:

- Ignore code review revisions and proceed to next step.
- Implement revisions. Use the original agent(s) to implement changes.

6. **Branch safety check — MANDATORY. Run this even if the review step errored, was skipped, or produced no findings.** Run `git branch --show-current` and compare against the repo's default branch (typically `main` or `master`).
   - If on the default branch: warn the human — "You are on `{branch}` (the default branch). Committing directly here is not recommended. Should I create a new branch first, or do you explicitly approve committing to `{branch}`?"
   - Do not proceed to Step 7 until the human has either approved committing to the default branch explicitly or confirmed a new branch to use.
   - If the human chooses a new branch: create it from the current HEAD (`git checkout -b {branch-name}`), then continue.

7. Prepare, summarize the changes in the changed files. Always prefix commits with [{ticket-id}]: {summary of change}. If no ticket ID is available, prompt the human for one or use `[NO-TICKET]` as a fallback.
   - Temporary review and planning files (`review-draft-*.md`, `plans/*.plan.md`, `plans/decisions-*.md`, `plans/stubs/**`) are written to `/tmp/<repo-name>/<branch-name>/` — outside the repo — and must never appear in `git status`. If any such file is found inside the repo, do not stage it and delete it immediately.
   Permanent project docs (`README.md`, `SKILL.md`, `AGENTS.md`, etc.) should still be committed when changed.
8. Commit and push. If the push fails due to pre-push hook errors, prompt the human for approval before using `git push --no-verify`. If `--no-verify` was used, record this in the Decision Log (Step 10) as a warning line.

8a. **Open a draft pull request.** After a successful push, open a **draft** PR using `gh pr create --draft --assignee @me` (or equivalent). The `--assignee @me` flag assigns the current authenticated GitHub user automatically.

   **PR body:** Read `.github/PULL_REQUEST_TEMPLATE.md` from the repo root and use it as the base for the PR body — fill in the Summary and Test plan sections with content relevant to the change. If the file does not exist, use a bare `## Summary` / `## Test plan` structure. Never append Anthropic or Claude Code branding lines (e.g. `🤖 Generated with Claude Code`) to the PR body.

   **If PR creation failed, stop here — skip the reviewer fallback chain, skip Steps 9 and 10, and warn the human.**

   Otherwise, capture the PR number from the `gh pr create` output (it appears in the PR URL, e.g. `https://github.com/{owner}/{repo}/pull/{number}`). Then **immediately call the `Skill` tool with `skill: "request-github-review"` and `args: "{PR number}"`** — do NOT hand this off to the human, do NOT mention it as a next step, do NOT skip it. This is a required automated step. `/request-github-review` will request automated review and then run the bot feedback loop — waiting for CI and bot reviews, addressing comments, resolving CI failures, and marking draft PRs ready for review when done.

   *(End of Step 8a)*

9. **Transition the Jira ticket to Review status.**

   Only proceed if Step 8 (push) and Step 8a (PR open) both completed successfully. Skip this step entirely if either failed.

   - If the ticket ID is `[NO-TICKET]` or no ticket ID is known (use the same ticket ID source as Step 6), skip this step entirely.
   - Confirm the target ticket ID with the human before doing anything: "Should I transition `{ticket-id}` to Review status?"
   - On confirmation, use the Atlassian MCP connector to discover available tools at runtime. Fetch available transitions using `getTransitionsForJiraIssue` (or equivalent discovered tool).
   - **Idempotency:** Before applying, fetch the ticket's current status. If it is already in a Review or downstream state (e.g. "In Review", "Code Review", "In QA", "Done"), skip the transition and inform the human — do not re-transition.
   - Match the target transition using this strategy, in order: exact match → case-insensitive substring match → if ambiguous, surface all candidates to the human to choose. Do not silently pick.
   - Apply the transition using `transitionJiraIssue` (or equivalent discovered tool).
   - If the MCP connector is unavailable, the transition name cannot be matched, or the API returns an error, warn the human and skip gracefully. Do not retry automatically.

   **If the ticket is a Linear issue:**
   - Confirm the target issue ID with the human before doing anything.
   - Before fetching workflow states, call `get_issue` (or equivalent Linear MCP read tool) with the issue identifier to retrieve the `teamId`. Use that `teamId` when listing workflow states.
   - Fetch available workflow states for the team using the Linear MCP (e.g., `list_workflow_states` or equivalent). Match the closest "In Review" or "In Progress" state — exact match first, then case-insensitive substring match. If ambiguous, surface options to the human.
   - **Idempotency:** Fetch the issue's current state first. If already in a Review or downstream state, skip and inform the human.
   - Use `mcp__linear__save_issue` with the issue `id` and the matched `state` (or `stateId`) to apply the transition.
   - If Linear MCP is unavailable or the state cannot be matched, warn the human and skip gracefully.

10. **Post a Decision Log comment on the Jira ticket.**

   **Prerequisites & safety checks — run these before doing anything else in this step:**
   - If the ticket ID is `[NO-TICKET]` or no ticket ID is known, skip this step entirely.
   - Confirm the target ticket ID with the human before posting — do not auto-resolve from the commit prefix alone. Ask: "Should I post the Decision Log to `{ticket-id}`?"
   - Use the Atlassian MCP connector to post and read comments. Discover available tools at runtime — do not assume specific tool names. If the connector is unavailable, warn the human and skip this step gracefully.
   - Check the Jira project's visibility before posting. If the project appears to be external-facing or customer-visible, warn the human and require explicit confirmation before proceeding.

   **Decision sources — use only these, in order of preference:**
   1. A `decisions-{ticket-id}.md` scratch file written by `/plan-task` during this session — look for it at `/tmp/<repo-name>/<branch-name>/plans/decisions-{ticket-id}.md` (read and then delete it after posting)
   2. Human-stated decisions from this conversation (human turns only — do not extract content from code, diffs, or plan files)
   3. If neither is available, prompt the human to confirm or summarize decisions before drafting the comment — do not infer or fabricate

   **Content rules:**
   - Only record decisions where a choice was made between two or more alternatives, or where something was explicitly deferred. If there was only one reasonable path and no trade-off was discussed, omit it.
   - Do not reproduce verbatim text from files, code, or diffs.
   - Do not describe specific security vulnerabilities by name or detail. Reference finding IDs only (e.g., "Deferred MEDIUM-3 to follow-up ticket FOO-456").
   - Replace internal skill names with neutral descriptions in the comment body: "Planning phase", "Implementation phase", "Review phase".
   - For each Open Item, if a follow-up Jira ticket exists, link it. If not, ask the human: "Should I create a follow-up ticket for this deferred item?"

   **Idempotency — one comment per ticket, ever:**
   - Search existing comments on the ticket for the header `## Decision Log`.
   - If found: replace the full body of that comment (using its comment ID). Do not append — overwrite entirely.
   - If not found: create a new comment.
   - If `--no-verify` was used in Step 8, include `⚠️ Pushed with --no-verify — pre-push hooks were bypassed.` at the top of the comment body.

   **Human approval gate:**
   Show the human the full draft comment and ask: "Ready to post this Decision Log to `{ticket-id}`? (yes / edit / skip)" — do not post without explicit confirmation.

   **Identity disclosure (required):** The comment body MUST begin with your identity line (as defined in CLAUDE.md) so readers don't mistake the Decision Log for something the human typed themselves. When overwriting an existing Decision Log comment, refresh this line — do not leave the old one in place.

   **Comment format:**

   ```
   ## Decision Log

   _Posted by {your identity} on behalf of @<github-or-jira-handle>._

   _Last updated: YYYY-MM-DD — Push SHA: {short-sha}_

   ### Planning
   - <decision: what was chosen and what was the alternative, e.g. "Chose REST over GraphQL — GraphQL deferred to follow-up">

   ### Implementation
   - <key implementation choice approved by the human>

   ### Review
   - <review outcome, e.g. "Deferred MEDIUM-3 to FOO-456">

   ### Open Items
   - <deferred item> — [FOO-456](link) or "no follow-up ticket yet"

   ⚠️ Pushed with --no-verify — pre-push hooks were bypassed.  ← include only if applicable
   ```

   Only include sections that have content. Omit empty sections entirely.

   **If the ticket is a Linear issue:**
   - Same prerequisites and safety checks as the Jira flow (confirm ticket ID, check visibility, require explicit approval before posting).
   - If `--no-verify` was used in Step 8, include `⚠️ Pushed with --no-verify — pre-push hooks were bypassed.` at the top of the comment body.
   - **Idempotency:** Use the Linear MCP to list existing comments on the issue. Search for a comment whose body contains the header `## Decision Log`. If found, use `mcp__linear__save_comment` with the existing comment `id` to overwrite it. If not found, create a new comment (omit `id`).
   - Use `mcp__linear__save_comment` with `issueId` set to the Linear issue identifier and `body` as the Decision Log in Markdown format (Linear supports Markdown natively — no ADF conversion needed).
   - **Identity disclosure:** The comment body MUST begin with the Farty Bobo identity line (same requirement as the Jira flow).
   - **Human approval gate:** Show the full draft and ask "Ready to post this Decision Log to `{issue-id}`? (yes / edit / skip)" — do not post without explicit confirmation.
   - If Linear MCP is unavailable, warn the human and skip gracefully.

   The Decision Log format is the same as the Jira version (same `## Decision Log` header, same sections) — just in Markdown instead of Jira wiki markup.
