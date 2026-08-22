# Scheduled Tasks

Use Cowork's built-in scheduler to automate recurring GTM and research workflows — competitive
digests, prospect list refreshes, email verification runs, or any repeating intelligence task.

## When to use

- "Send me a competitor update every Monday morning"
- "Refresh my prospect list every week"
- "Run this research automatically every day at 9am"
- "Set up a recurring alert when [signal] changes"
- "Remind me to follow up on this in 3 days"

---

## How Cowork scheduling works

Cowork's scheduler runs a task prompt on a cron schedule or at a specific future time. Each
scheduled task calls back into this same session context, so it has access to all connected
tools (Sushidata, HeyReach, HubSpot, Apify).

**Two modes:**
- **Recurring** — runs on a cron schedule (e.g., every Monday at 9am, daily at 6am)
- **One-time** — fires at a specific future time (e.g., "in 3 days", "next Tuesday at 2pm")

---

## Recipe 1: Weekly competitive intelligence digest

Set up a Monday morning swarm that checks for new signals across your tracked competitors and
delivers a digest.

### Step 1: Confirm the setup with the user

Before creating the schedule, confirm:
- Which competitors to track (default: use the battlecard from the quickstart if it exists)
- What day and time to deliver (default: Monday 9am)
- Where to send it (chat only, or also save to Sushidata context lake)

### Step 2: Create the scheduled task

Use `create_scheduled_task` with a weekly cron:

```
Tool: create_scheduled_task
cronExpression: "0 9 * * 1"   ← Monday 9am
prompt: "Run a Sushidata competitive intelligence digest for [COMPANY].
  Check for new signals in the last 7 days for these competitors: [A], [B], [C].
  For each: new hires in GTM roles, product/pricing changes, customer wins, press.
  Deploy a Sushidata swarm (swarmSize: 6), verify sources, and present a structured digest.
  Save results to the Sushidata context lake."
```

Confirm the task ID returned and share it with the user so they can reference or cancel it.

### Step 3: Tell the user what to expect

> "Your competitive digest is scheduled for every Monday at 9am. I'll research [A], [B], and [C]
> automatically, verify the sources via Sushidata, and deliver the results here. You can cancel
> or update it anytime — just say 'show my scheduled tasks' or 'cancel the competitor digest'."

---

## Recipe 2: Daily prospect list refresh

Automatically refresh a TAM or prospect segment and push net-new verified contacts to HeyReach.

### Step 1: Confirm parameters

- ICP definition (industry, headcount, geography, role)
- How many net-new contacts per run
- Target HeyReach campaign (must already exist in the UI)

### Step 2: Create the scheduled task

```
Tool: create_scheduled_task
cronExpression: "0 7 * * 1-5"   ← weekdays at 7am
prompt: "Run a daily prospect refresh for [COMPANY].
  ICP: [definition]. Goal: find [N] net-new verified contacts not already in HeyReach campaign [ID].
  Steps: (1) Use fullenrich_search_people for ICP-matching domains found via WebSearch or Sushidata query.
  (2) Check FullEnrich confidence scores — drop low-confidence results before outbound.
  (3) Add verified contacts to HeyReach campaign [ID] via heyreach_add_to_campaign.
  (4) Save a summary of contacts added to the Sushidata context lake.
  Report how many contacts were added and any issues."
```

---

## Recipe 3: One-time future task (reminder or delayed action)

Use `fireAt` instead of `cronExpression` for a task that fires once at a specific time.

```
Tool: create_scheduled_task
fireAt: "2026-05-26T09:00:00Z"   ← specific ISO 8601 datetime
prompt: "Follow up: check whether [COMPANY] has responded to our outreach.
  Query the Sushidata context lake for any saved notes on [COMPANY].
  If no response logged, draft a follow-up message and present it."
```

To express relative times: "in 3 days" → add 3 days to today's date in ISO 8601.

---

## Recipe 4: Weekly Sushidata context summary

Get a digest of everything saved to the context lake in the past week — research findings,
decisions, and notes — so nothing gets lost.

```
Tool: create_scheduled_task
cronExpression: "0 8 * * 5"   ← Fridays at 8am
prompt: "Query the Sushidata context lake for all research and findings from the past 7 days.
  POST /query/ with: 'Summary of all research, decisions, and findings from this week'.
  Present the key themes, open questions, and suggested next steps."
```

---

## Managing scheduled tasks

### List all scheduled tasks
Say: *"Show me my scheduled tasks"* → use `list_scheduled_tasks`

### Update a task's schedule or prompt
Say: *"Change the competitor digest to Wednesdays"* → use `update_scheduled_task` with the task ID
and updated `cronExpression`

### Cancel a task
Say: *"Cancel the weekly competitor digest"* → use `update_scheduled_task` to disable it, or
delete via the Cowork scheduled tasks interface

---

## Cron reference

| Schedule               | Expression       |
| ---------------------- | ---------------- |
| Every Monday at 9am    | `0 9 * * 1`      |
| Daily at 7am           | `0 7 * * *`      |
| Weekdays at 7am        | `0 7 * * 1-5`    |
| Every Friday at 8am    | `0 8 * * 5`      |
| First of month at 9am  | `0 9 1 * *`      |
| Every hour             | `0 * * * *`      |

All times are UTC unless otherwise specified. Adjust for your timezone as needed.

---

## Best practices

- **Keep prompts self-contained** — scheduled tasks run without conversation context. Include all
  ICP definitions, company names, campaign IDs, and parameters directly in the prompt.
- **Save results to Sushidata** — always end the prompt with an instruction to save findings to
  the context lake so each run builds on the last.
- **Start with a one-time test** — before committing to a weekly schedule, use `fireAt` to run
  the task once in the next few minutes and verify the output.
- **Be specific about scope** — vague prompts produce vague outputs. "Check competitors" is
  worse than "Check [A], [B], [C] for hires, product changes, and press from the last 7 days."

---

## Save to Context Lake

Scheduled task definitions should be saved so future sessions know what's already running and can avoid duplicates:

```json
POST /context/
{
  "serverId": "26",
  "content": "Scheduled task created. Task: {{task description}}. Schedule: {{cron expression or fireAt time}}. Purpose: {{what it does and why}}. Task ID: {{ID returned by create_scheduled_task}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
