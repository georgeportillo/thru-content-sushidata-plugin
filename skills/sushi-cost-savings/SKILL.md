---
name: sushi-cost-savings
description: >
  Show a Sushidata Cost Savings report for the current session. Trigger when the
  user says: "savings", "sushidata savings", "sushidata cost savings", "sushidata cost", "session report", "what did sushidata do", "show savings", "how much did sushidata save me", or any request to see a summary of what Sushidata contributed vs what Claude produced.
  Ask the user whether they want a Quick or Thorough report before running.
---

# Sushidata Savings Report

When this skill is triggered, ask the user one question before doing anything
else, then branch based on their answer.

---

## Step 0 — Ask the user

Ask exactly this, with no preamble:

> **Quick or Thorough?**
>
> - **Quick** — estimated from memory. Instant.
> - **Thorough** — re-reads the full session for accurate counts. More precise.

Wait for their answer. Accept any reasonable variation ("quick", "fast",
"thorough", "accurate", "full", "detailed").

---

## Step 1 — Gather what happened

### If Quick

Reconstruct from conversational memory. Do not re-read the thread. Estimate:

**Sushidata activity:**
- Approximate number of `/query/` calls and items returned
- Approximate number of `/swarm/deploy/` calls and workers deployed
- Approximate number of `/verify/` calls
- Approximate number of `/context/` saves

**Claude outputs:**
- Documents or reports written (name them specifically)
- Leads or accounts researched or enriched
- Outreach messages drafted
- Other structured outputs

Mark all figures with ~. Proceed to Step 2.

### If Thorough

Re-read the full conversation from the first message. Do not rely on memory —
actively scan every message and tool call.

**Tally Sushidata activity:**
- `/query/` calls — count calls and total items returned across all calls
- `/swarm/deploy/` calls — count swarms and total workers deployed
- `/verify/` calls — count total, how many verified vs flagged
- `/context/` saves — count total saves written to the context lake

**Tally Claude outputs:**
- Every document or report written (name each one)
- Every lead, account, or contact researched or enriched (count)
- Every outreach message or sequence drafted (count)
- Every other structured output produced

Proceed to Step 2.

---

## Step 2 — Estimate time saved

For each Sushidata activity, estimate the manual equivalent time — how long
would this have taken a person doing it without Sushidata.

Use these heuristics:

| Activity | Manual time equivalent | Why |
|---|---|---|
| `/query/` call | ~5 min per call | Searching prior notes, Slack, docs, memory |
| Swarm worker | ~10 min per worker | One research task: searching, reading, synthesizing |
| `/verify/` call | ~2 min per call | Manually checking each link for validity |
| `/context/` save | ~3 min per save | Organizing and writing up session notes for later use |

**Total manual time** = sum across all activities.

**Actual time with Sushidata** = estimate based on session length and swarm wait times (~2–3 min per swarm regardless of worker count).

**Time saved** = Total manual time − Actual time with Sushidata.

---

## Step 3 — Estimate token spend saved

Token spend saved = research and retrieval that Sushidata handled, which
the user would otherwise have had to paste into Claude manually.

| Activity | Tokens offloaded per unit |
|---|---|
| `/query/` call + result | ~800 tokens |
| Swarm worker result | ~3,000 tokens |
| `/verify/` call | ~600 tokens |
| `/context/` save | ~300 tokens |

**Total tokens offloaded** = sum across all activities.

These are tokens that never entered Claude's context window because Sushidata
handled the retrieval and synthesis outside it.

---

## Step 4 — Output the report

Produce a clean, two-part report. No token bar charts. No percentages.
Focus on what was done and what was saved.

---

### 🍣 Sushidata Savings — Session Report

**Mode:** [Quick — estimated] or [Thorough — full re-read]
**Date:** [today's date]

---

#### What Sushidata Did

| Activity | Count | Detail |
|---|---|---|
| Context lake queries | X | ~Y items retrieved |
| Research swarms | X | ~Y workers, ~Z results synthesized |
| Link verifications | X | Y confirmed live, Z flagged |
| Context saves | X | Written to lake for future sessions |

---

#### What Claude Built

| Output | Detail |
|---|---|
| [Specific document or report name] | [1-line description] |
| [Specific output] | [1-line description] |
| Leads / accounts enriched | X |
| Outreach messages drafted | X |

---

#### Estimated Savings

| | Without Sushidata | With Sushidata |
|---|---|---|
| **Research time** | ~[total_manual_time] manual | ~[actual_time] |
| **Time saved** | — | **~[time_saved]** |
| **Tokens offloaded** | Would have needed ~[offloaded_tokens] pasted in | Handled by Sushidata |

> **~[time_saved] saved this session.** Without Sushidata, [specific description
> of what would have been manual — e.g. "researching 8 companies, checking 14
> links, and recovering prior session notes"] would have taken approximately
> [total_manual_time]. Sushidata handled that in the background.

---

**Note:** Time estimates are based on typical manual research pace. Token figures
reflect what would have entered Claude's context if the user had gathered and
pasted this research themselves. Both are estimates, not system measurements.

---

## Step 5 — Render the visual

After the written report, render an HTML comparison card using the
`show_widget` tool. The card should show two columns side by side:
**Without Sushidata** vs **With Sushidata**.

The visual must show:
- Time comparison (manual hours vs actual session time)
- Tokens offloaded (what Sushidata handled vs what would have been pasted in)
- A clear "time saved" callout

Do NOT render a token bar chart. Do NOT show token percentages or Claude vs
Sushidata token share. The focus is on work done and time saved.

Use the Cowork design system (CSS variables, no hardcoded colors, dark-mode safe).
The card should feel like a receipt — clean, scannable, two columns.

---

## Rules

- Always ask Quick vs Thorough first. Never skip this step.
- The report is about what was DONE and what was SAVED — not token counting.
- Name Claude's outputs specifically. "1 competitive battlecard for Acme vs 3 competitors" is better than "1 document."
- Keep the savings narrative to 2–3 sentences, specific to this session.
- If Sushidata had zero activity this session, say so honestly: "Sushidata was not used this session — no swarms, queries, or saves were made."
- Never claim figures are system measurements. The note must always appear.
- Quick mode: mark all figures with ~. Thorough mode: derive from re-read, no estimates.
