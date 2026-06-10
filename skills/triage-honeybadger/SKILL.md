---
name: triage-honeybadger
description: Investigate and triage a Honeybadger fault or error. Accepts a fault ID, fault URL, or project name. Pulls fault details, recent occurrences, affected users, and occurrence trends, then produces a structured triage summary with severity assessment and recommended next steps.
---

# Triage Honeybadger Skill

## Steps

### 1. Resolve the Target

Accept the target from one of the following:
- **Fault ID** — e.g. `1234567`
- **Fault URL** — e.g. `https://app.honeybadger.io/projects/123/faults/456` — extract project and fault IDs from the URL
- **Project name or ID** — list recent faults for the project and let the human pick one
- **No argument** — call `mcp__honeybadger__list_projects` and ask the human which project and fault to investigate

If the Honeybadger MCP server is not configured, tell the human to set it up and stop.

### 2. Fetch Fault Details

Use `mcp__honeybadger__get_fault` with the project ID and fault ID to retrieve:
- Error class and message
- First seen / last seen timestamps
- Total occurrence count
- Resolved/ignored status
- Assignee (if any)

If the fault is not found, surface the error and stop.

### 3. Gather Occurrence Data

Run these in parallel:
- `mcp__honeybadger__list_fault_notices` — fetch recent occurrences (last 10–20) to inspect stack traces, environments, and request context
- `mcp__honeybadger__get_fault_counts` — get occurrence counts over time to assess trend (spiking, steady, declining)
- `mcp__honeybadger__list_fault_affected_users` — identify how many and which users are impacted

### 4. Check Project Health (optional context)

If no project ID is known yet or broader context is helpful:
- `mcp__honeybadger__get_project` — confirm project name, environment, and settings
- `mcp__honeybadger__get_project_occurrence_counts` — check if this fault is part of a broader error spike

### 5. Assess Severity

Based on the data collected, assign a severity level:

| Level | Criteria |
|-------|----------|
| **P1 – Critical** | Affecting many users, spiking in last hour, or blocking core functionality |
| **P2 – High** | Affecting some users, steady or growing, non-trivial impact |
| **P3 – Medium** | Low user impact, intermittent, or non-critical path |
| **P4 – Low** | Rare, no user impact, cosmetic or informational |

### 6. Produce Triage Summary

Output a structured report in this format:

```
## Honeybadger Triage: {Error Class} [{fault-id}]

**Project:** {project name}
**Environment:** {env}
**Severity:** {P1/P2/P3/P4} — {one-line justification}
**Status:** {open / resolved / ignored}
**Assignee:** {name or unassigned}

### Error
{error class}: {error message}

### Timeline
- First seen: {date}
- Last seen: {date}
- Total occurrences: {count}
- Trend: {spiking / steady / declining} ({count} in last 24h)

### Affected Users
{count} users affected — {list top users if available}

### Recent Occurrence Context
{key details from latest notice: environment, URL/action, relevant request params, top 3–5 stack frames}

### Root Cause Hypothesis
{1–3 sentences synthesizing what likely caused this based on the stack trace and context}

### Recommended Next Steps
1. {actionable step}
2. {actionable step}
3. {actionable step}
```

Keep the summary concise. Do not dump raw stack traces — extract the signal.

### 7. Offer Follow-up Actions

After the summary, ask the human if they want to:
- **Create a Linear ticket** — invoke `/create-linear-ticket` with the triage summary pre-filled (see below)
- **Comment on an existing Linear ticket** — invoke `/comment-linear` with the triage summary pre-filled
- **Post to Slack** — invoke `/post-on-slack` with the summary
- **Dig deeper** — query `mcp__honeybadger__query_insights` for custom analytics on this fault
- **Done** — no further action

#### Pre-filling `/create-linear-ticket` from triage data

When the human chooses to create a Linear ticket, pass the following context so the skill can draft a well-formed issue without asking redundant questions:

- **Title**: `{ErrorClass}: {short error message}` — keep it under 80 chars
- **Type**: Bug
- **Priority**: use the severity level assigned in Step 5, mapped to Linear priority:
  - P1 → Urgent
  - P2 → High
  - P3 → Medium
  - P4 → Low
- **Description** — include the following in the ticket description:
  - Honeybadger fault URL (use the URL returned directly by `mcp__honeybadger__get_fault`)
  - Environment: `{env}`
  - First seen / Last seen / Total occurrences
  - Affected user count
  - Root cause hypothesis from Step 6
  - Top 3–5 stack frames from the most recent occurrence
  - Recommended next steps from Step 6
