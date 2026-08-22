---
name: clay-to-sushidata
description: "Extract a Clay table, understand its structure, run Sushidata research on each row, and save results to the context lake."
---

# Clay → Sushidata

Pull data from a Clay table, enrich each row with Sushidata research swarms, and save findings to the context lake for persistent access across sessions.

---

## §1 Extraction

If you need to pull the Clay table config and records, read [`references/clay-extraction.md`](../references/clay-extraction.md) for the two available paths:

- **Claude-in-Chrome MCP** — use `javascript_tool` with `credentials: 'include'` against Clay's internal API if the Chrome extension is active
- **`scripts/clay-extract.py`** — run locally with a session cookie from Chrome DevTools

If the user already provided an extract JSON or pasted Clay data, skip to §2.

**What to extract:**
- The field list (`fields[]`) — tells you what columns exist and what action type produced each one
- Example records (`exampleRecords[]`) — gives you sample data to understand the shape
- Action prompts from `typeSettings.inputsBinding[name=prompt].formulaText` — if the user wants to replicate Clay AI columns via Sushidata swarms

---

## §2 Documentation (always first — confirm before researching)

Produce this before deploying any swarms. Get user confirmation before §3.

### 2.1 — Table summary

| # | Column name | Clay action type | Output type | Notes |
|---|---|---|---|---|
| 1 | `record_id` | built-in | string | |
| … | | | | |

### 2.2 — What Sushidata can enrich

For each column, classify it:

| Column | Can Sushidata enrich? | How |
|---|---|---|
| AI research column (`use-ai` / claygent) | ✅ Yes | Deploy a swarm with the recovered prompt as the task |
| Email finding | ✅ Partial | Use Clay MCP `find-and-enrich-contacts-at-company` with Email data point |
| Company enrichment | ✅ Yes | Use Clay MCP `find-and-enrich-company` or Sushidata swarm |
| Formula / JS columns | ⛔ No | Pure computation — reconstruct output manually if needed |
| CRM writes / campaign pushes | ⛔ Out of scope | Use HubSpot or HeyReach playbooks instead |
| Cross-table lookups | ⛔ No | Export the linked table and join manually |

### 2.3 — Prompt recovery

Before writing any swarm tasks, recover the actual Clay prompts — do not approximate when the extract has the real text.

**Recovery priority (richest to weakest):**
1. **HAR file** — rendered cell values contain the exact prompt as executed
2. **clay-extract.py output** — `fields[].typeSettings.inputsBinding[name=prompt].formulaText`
3. **User description** — approximate; mark as `# APPROXIMATED`

Mark each recovered prompt with its source:
```
# RECOVERED FROM EXTRACT — field f_xxx
# APPROXIMATED — could not recover
```

### 2.4 — Assumptions log

State every unverifiable assumption. Get user confirmation before §3.

---

## §3 Sushidata Enrichment

For each row (or batch of rows), deploy a Sushidata research swarm. Size the swarm based on depth needed.

### Single company / contact research

```json
POST /swarm/deploy/
{
  "query": "Research [company_name] ([domain]). [Paste recovered Clay prompt here, translated to Sushidata context.] Return: [fields the Clay column was supposed to produce].",
  "swarmSize": 5
}
```

### Batch research (multiple companies)

When the Clay table has many rows, deploy one swarm per company or group them:

```json
POST /swarm/deploy/
{
  "query": "For each of these companies — [Company A] ([domain_a]), [Company B] ([domain_b]), [Company C] ([domain_c]) — research: [task from recovered Clay prompt]. Return findings per company.",
  "swarmSize": 8
}
```

**Swarm sizing guide:**

| Rows | Task depth | Swarm size |
|---|---|---|
| 1–5 companies | Deep per-company research | 5–8 |
| 5–20 companies | Broad signals per company | 8–15 |
| 20+ companies | Batch with summary per company | 15–20 |

### Replicating Clay AI columns (claygent-style web research)

For Clay columns that used claygent (web browsing + AI), always use two passes:

**Pass 1 — Research swarm (gather signals):**

```json
POST /swarm/deploy/
{
  "query": "Research [company_name] ([domain]). Find: (1) recent strategic initiatives from investor relations or press, (2) new product launches or pricing changes in the last 90 days, (3) GTM signals from job postings or LinkedIn. Return structured findings with source URLs.",
  "swarmSize": 6
}
```

**Pass 2 — After swarm completes, synthesize with context:**

```
Using the swarm research above, [paste the recovered Clay AI prompt with field references resolved to actual values]. Return: [Clay output schema].
```

Never combine broad research + generation in one swarm — split them.

---

## §4 Verification

Before presenting results, run evidence links through `/verify/`:

```json
POST /verify/
{
  "context": "[swarm summary text + all source URLs cited]"
}
```

Drop any links classified as `bad` or `blocked`. Note if significant sources were removed.

---

## §5 Context Lake Saving

Save after each major batch — not just at the end. This way, partial results survive session timeouts.

**Save after each company or batch:**

```json
POST /context/
{
  "serverId": "26",
  "content": "Clay → Sushidata enrichment — [Table name]. Batch: [company list]. Findings: [2–3 sentence summary of key results]. Full data: [field values or output path if saved to file]. Source URLs: [verified links].",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

**Save the table structure itself (once, at the start):**

```json
POST /context/
{
  "serverId": "26",
  "content": "Clay table structure — [Table name] ([table_id]). Columns: [list of column names and types]. Research columns being enriched via Sushidata: [list]. Extraction date: [date].",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Future sessions can retrieve all of this with a single `/query/` call.

---

## §6 Output

Present results as a clean table matching the Clay column structure. For each row:

| Company | [Column A] | [Column B] | [Column C] | Sources |
|---|---|---|---|---|
| ... | ... | ... | ... | [verified links] |

Then offer:
- **Save to HubSpot** — use the HubSpot playbook to write enriched fields to CRM contacts or companies
- **Add to HeyReach** — use the HeyReach playbook to push contacts to a LinkedIn outbound campaign
- **Research a specific row deeper** — deploy a focused swarm on a single company
- **Save this session** — write all findings to the context lake

---

## §7 What this recipe cannot replicate

Be explicit with the user when a Clay column falls outside what's possible:

| Clay capability | Status | Reason |
|---|---|---|
| Formula / JS columns | ⛔ Cannot replicate | Pure computation — reconstruct manually if needed |
| Cross-table lookups (`lookup-company-in-other-table`) | ⛔ Cannot replicate | Requires the linked table exported as data |
| Route-row / conditional branching | ⛔ Cannot replicate | Requires a workflow engine |
| CRM writes as part of the pipeline | ⚠️ Route separately | Use HubSpot playbook after enrichment completes |
| Campaign push as part of the pipeline | ⚠️ Route separately | Use HeyReach playbook after enrichment completes |
| Real-time / webhook-triggered runs | ⛔ Cannot replicate | Session-based research only |

If the user's Clay table is primarily formula/JS/routing columns with little AI research content, tell them upfront: there is not enough here to make Sushidata enrichment worthwhile for this table.

---

## §8 Critical rules

- **Recover prompts verbatim** — small prompt differences cause systematic drift in results. Always use the exact Clay prompt text, with `{{field}}` references resolved to actual values.
- **Two passes for web research** — never combine research and generation in one swarm call.
- **Save incrementally** — post to `/context/` after each batch, not only at the end.
- **Verify before presenting** — always run `/verify/` on source URLs before showing them to the user.
- **Pilot first** — for large tables, research 1–3 rows first and confirm results match expectations before running the full table.
- **Be explicit about gaps** — if a column cannot be replicated, say so clearly rather than approximating silently.
