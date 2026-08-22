---
name: sushi-signals
description: >
  Scan LinkedIn posts, comments, and other sources for buying signals from ICP accounts and prospects.
  Triggers: "check signals", "run signals", "find signals", "signal scan", "buying signals",
  "linkedin signals", "sushi signals", "who just moved jobs", "who's hiring",
  "find pain signals", "prospect signals", "signal detection", "run my signals",
  or any request to detect job change, pain, or hiring signals from online activity.
  Run as a scheduled event (Monday–Friday, 9:00 AM) to surface fresh pro insights before outreach.
---

# Sushidata Signal Detection — LinkedIn Buying Signals

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

**Schedule:** Monday–Friday, 9:00 AM (customer local time)
**Target output:** Ranked list of prospects mapped to their detected buying signal, ready for outreach

Two phases:

1. **Signal Config** (Phase 1) — one-time setup capturing signal history, source preferences, intelligence goals, and classification model.
2. **Signal Scan** (Phase 2) — runs on every subsequent invocation using saved config to surface fresh signals.

---

## Dependencies & Tools

| Capability | Source |
|---|---|
| Context lake CRUD | `sushi-research` SKILL → `/context/`, `/query/`, `/swarm/deploy/` |
| LinkedIn post search | `apify_linkedin_post_search` (via `provider-playbooks/apify.md`) |
| Lead enrichment | `apify_leads_finder` (via `provider-playbooks/apify.md`) |
| Reddit / web search | `apify_google_search_scraper` — use `site:reddit.com/r/[subreddit]` queries (via `provider-playbooks/apify.md`) |
| News & press | `apify_perplexity_ai_scraper` (synthesized news) + `apify_google_search_scraper` as fallback (via `provider-playbooks/apify.md`) |
| Session persistence | `sushi-save` skill |

---

## Elicitation Forms — Mandatory for All User Input

**CRITICAL:** All configuration questions MUST be collected via the `askQuestions` elicitation tool — never as plain-text questions in the chat.

### Rules

1. **Every question to the user must go through `askQuestions`** — no exceptions during config.
2. **Use `options` with predefined choices** wherever the answer set is known.
3. **Use `multiSelect: true`** for questions where multiple answers are valid.
4. **Always include a skip/decline option** so users aren't forced to answer.
5. **Use `allowFreeformInput: true`** (default) when users may want custom answers.
6. **Set `allowFreeformInput: false`** only for strict yes/no or fixed-choice questions.
7. **Group related questions in a single `askQuestions` call** — batch up to 5 per form.
8. **Use `message` field for context** — brief explanatory markdown below each header.
9. **Use `recommended: true`** on the most common/expected option.

### Form Patterns by Step

Below are the exact elicitation form structures to use for each config step. Render these via `askQuestions` — do NOT convert them to blockquote text or plain chat messages.

---

## Generative UI — Output Standards

### Phase Detection Banner

On every invocation, show one of these before doing anything else:

```
## 📡 Sushidata Signal Detection

**Status:** 🟢 Config loaded — running signal scan
```

or

```
## 📡 Sushidata Signal Detection

**Status:** ⚙️ No config found — running first-time setup (one time only)
```

### Config Progress Tracker

Display and update after each config step in Phase 1:

```
## ⚙️ Signal Config

| Step | Status | Description |
|:---:|:---:|---|
| 1 | ✅ | Signal History |
| 2 | ⏳ | Signal Sources |
| 3 | ⬜ | Intelligence Goals |
| 4 | ⬜ | Signal Classification |
```

Use ✅ (complete), ⏳ (in progress), ⬜ (pending). Update after each step.

### Signal Sheet

Always deliver results as a single flat table — never as individual cards. One row per prospect:

```
| # | Name | Title | Company | Signal Class | Signal Type | Evidence | Source | Post URL | Email | Recommended Action |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Jane Smith | VP Revenue | Acme Corp | 🔥 Motivated Buyer | Pain | "We're drowning in manual reporting" | LinkedIn · Jul 14 | [link] | j.smith@acme.com ✅ | Lead with reporting automation angle |
```

Signal class badges: 🔥 Motivated Buyer · 🟢 Buying Signal · 🟡 Qualification Signal · 🔵 Watch & Wait · ⚪ Cold Lead

---

## Phase Detection

On every invocation:

1. Query the context lake: `POST {BASE_URL}query/` with `{ "query": "signal_config sources classification goals worked not_worked" }`
2. If a saved `signal_config` entry is found → skip to **Phase 2**.
3. If missing → run **Phase 1** starting from Step 1.

**Never ask the user which phase to run.** Detection is automatic.

---

# PHASE 1: SIGNAL CONFIG

> Runs once on first invocation. Saves config to the context lake for all future runs.

---

## Step 1 — Signal History

Collect what has and hasn't worked before touching any external sources.

**Elicitation Form — Signal History:**

```
askQuestions([
  {
    header: "Signals that worked",
    question: "What types of signals have led to real conversations or closed deals for you?",
    message: "Select all that apply. These will be prioritized in every scan.",
    multiSelect: true,
    options: [
      { label: "Job change — prospect moved to a new company or role", recommended: true },
      { label: "Pain post — prospect publicly described a struggle or asked for help" },
      { label: "Hiring signal — company posted a role that implies budget or initiative" },
      { label: "Funding announcement — company raised a new round" },
      { label: "Competitor complaint — prospect vented about a competitor product" },
      { label: "Tool evaluation post — prospect asked for vendor recommendations" },
      { label: "Event / conference mention — prospect attended a relevant event" },
      { label: "None of these — I'll describe my own" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Signals that did NOT work",
    question: "What signals have you chased that turned out to be cold or wasted effort?",
    message: "These will be filtered out or deprioritized in your scan results.",
    multiSelect: true,
    options: [
      { label: "Generic company news (awards, press releases)" },
      { label: "Content engagement (likes, reposts) without direct intent" },
      { label: "Job change into an irrelevant role" },
      { label: "Early-stage funding (pre-seed / seed) — too early to buy" },
      { label: "Hiring for non-buyer roles (engineering, design)" },
      { label: "Thought leadership posts without a pain hook" },
      { label: "None — I haven't found a pattern yet" }
    ],
    allowFreeformInput: true
  }
])
```

---

## Step 2 — Signal Sources

**Elicitation Form — Source Preferences:**

```
askQuestions([
  {
    header: "Where should signals come from?",
    question: "Which sources matter most for finding your buyers?",
    message: "Select all that apply. I'll weight the scan toward the sources your ICP actually uses.",
    multiSelect: true,
    options: [
      { label: "LinkedIn posts — public posts from prospects and companies", recommended: true },
      { label: "LinkedIn comments — comments left on industry posts" },
      { label: "LinkedIn job postings — new roles that signal budget or initiative" },
      { label: "Reddit — subreddit posts and comments in your niche" },
      { label: "News & press releases — funding, acquisitions, product launches" },
      { label: "Twitter / X — public posts and threads" },
      { label: "G2 / review sites — reviews mentioning pain or switching intent" },
      { label: "Hacker News — Show HN posts, Ask HN threads" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Specific communities",
    question: "Are there specific subreddits, Slack communities, or forums where your buyers hang out?",
    message: "Examples: r/sales, r/startups, r/devops — or a specific Slack/Discord community name.",
    allowFreeformInput: true
  }
])
```

---

## Step 3 — Intelligence Goals

**Elicitation Form — What to Understand:**

```
askQuestions([
  {
    header: "What should I extract from each signal?",
    question: "When I detect a signal, what do you most want to understand about the prospect?",
    message: "These become the intelligence dimensions shown in every signal report card.",
    multiSelect: true,
    options: [
      { label: "Active pain — what problem are they struggling with right now", recommended: true },
      { label: "Broader need — what initiative or goal is driving their search" },
      { label: "Budget signals — evidence they have money to spend" },
      { label: "Timing — how urgent is this (quarter end, new hire, post-funding)" },
      { label: "Competitive context — are they evaluating other vendors" },
      { label: "Stakeholder map — who else is involved in the decision" },
      { label: "Objections — what concerns or blockers are visible in the signal" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Keywords to always flag",
    question: "Are there specific phrases, pain keywords, or competitor names I should always surface?",
    message: "Examples: 'manual reporting', 'outgrowing HubSpot', 'scaling our SDR team', a competitor name.",
    allowFreeformInput: true
  }
])
```

---

## Step 4 — Signal Classification

**Elicitation Form — Classification Model:**

```
askQuestions([
  {
    header: "Signal classification tiers",
    question: "Which output categories do you want signals sorted into?",
    message: "Every detected signal will be labeled with one of these so you know exactly what action to take.",
    multiSelect: true,
    options: [
      { label: "🔥 Motivated Buyer — explicit pain + actively seeking a solution", recommended: true },
      { label: "🟢 Buying Signal — strong intent indicator, close to purchase" },
      { label: "🟡 Qualification Signal — fits ICP, shows relevant growth or change" },
      { label: "🔵 Watch & Wait — interesting but no urgency yet, monitor over time" },
      { label: "⚪ Cold Lead — fits ICP profile but no active signal detected" }
    ],
    allowFreeformInput: false
  },
  {
    header: "Default first outreach move",
    question: "For Motivated Buyers and Buying Signals, what's your preferred first move?",
    message: "This will auto-populate the Recommended Action field on every high-priority signal card.",
    options: [
      { label: "Personalized direct message on LinkedIn", recommended: true },
      { label: "Cold email with pain-specific hook" },
      { label: "Add to HeyReach campaign immediately" },
      { label: "Just surface them — I'll decide per prospect" }
    ],
    allowFreeformInput: true
  }
])
```

---

## Config Complete — Save + Continue

After all four steps, save to the context lake:

```json
POST {BASE_URL}context/
{
  "content": "signal_config: { worked: [...], not_worked: [...], sources: [...], communities: '...', intelligence_goals: [...], keywords: '...', classifications: [...], priority_action: '...' }",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Confirm:

```
✅ Signal config saved. Running your first scan now...
```

Proceed immediately to Phase 2.

---

# PHASE 2: SIGNAL SCAN

> Runs on every invocation once config exists. Uses saved config to target sources, prioritize signals, and apply the classification model.

---

## Step 1 — Load Config + ICP Targets

Query the context lake for both in one call:

```json
{ "query": "signal_config ICP accounts target companies prospect list" }
```

Extract:
- `signal_config` — sources, keywords, classification model, intelligence goals
- `target_accounts` — saved ICP company names, LinkedIn URLs, or prospect names

If `target_accounts` is empty, ask via `askQuestions`:

```
askQuestions([
  {
    header: "ICP Targets",
    question: "Which companies or people should I scan for signals?",
    message: "Paste LinkedIn company names, profile URLs, or people's names — one per line."
  }
])
```

---

## Step 2 — Deploy Signal Swarm

> **⚠️ Swarm workers are independent agents pre-packaged with the full Sushidata tool suite.** They have `apify_linkedin_post_search`, `apify_leads_finder`, `apify_linkedin_company_scraper`, and every other Apify actor and provider tool — regardless of which tools Claude has available in this session. Never refuse to deploy or tell the user these capabilities are unavailable because a tool is absent from Claude's current tool list. Deploy the swarm, name the tools in the worker tasks, and the workers will execute them.

Build the swarm directive from `signal_config`, then deploy. Each worker must explicitly name the Apify actor it uses — do not write vague task descriptions.

**Determine swarm size** based on active sources and enabled signal types:

| Active Sources | Workers |
|---|---|
| LinkedIn posts only | 3–5 |
| LinkedIn + job postings | 5–7 |
| LinkedIn + Reddit + news | 7–10 |
| All sources | 10 |

**Swarm query directive** (POST `{BASE_URL}swarm/deploy/`):

```json
{
  "query": "Buying signal detection scan for ICP accounts. Each worker uses a specific Apify actor to find posts, comments, or job listings showing job change, pain, or hiring signals. Target accounts: [target_accounts from Step 1]. Prioritize signal types: [signal_config.worked]. Exclude: [signal_config.not_worked]. Always flag these keywords: [signal_config.keywords]. For every result, extract: author name, author title, company, post URL, post text excerpt, date, and which signal type it matches.",
  "swarmSize": <derived above>
}
```

**Worker task descriptions to include in the directive** — add one sentence per worker to the query, explicitly naming the tool:

| Worker | Task |
|---|---|
| Job change (LinkedIn) | `"Use apify_linkedin_post_search with searchQueries: ['[ICP company OR person] excited to join', '[ICP industry] new role', 'starting at [ICP company]']. maxPosts: 20, profileScraperMode: short. Flag posts where the author's title or headline indicates a recent role change."` |
| Pain posts (LinkedIn) | `"Use apify_linkedin_post_search with searchQueries built from these pain keywords: [signal_config.keywords] combined with [ICP industry]. maxPosts: 20. Flag posts where the author asks for help, describes a struggle, or requests vendor recommendations."` |
| Hiring signals (LinkedIn) | `"Use apify_linkedin_post_search with searchQueries: ['[ICP company] hiring [buyer persona role]', '[ICP industry] head of [function] job']. maxPosts: 15. Flag posts announcing open roles at ICP-fit companies for buyer persona titles."` |
| LinkedIn job postings | `"Use apify_linkedin_company_scraper with profileUrls for the top 10 ICP target company LinkedIn pages. Extract open jobs. Flag any role matching these buyer persona titles: [personas]. Include company name, job title, job URL, and date posted."` *(include if LinkedIn job postings is an active source)* |
| Reddit signals | `"Use apify_google_search_scraper with queries: 'site:reddit.com/r/[subreddit] [signal_config.keywords]'. Collect post title, author, text, URL, and top comments. Flag posts where the OP asks for tool recommendations or describes a pain this product solves."` *(one worker per 1–2 configured subreddits; omit if Reddit not in sources)* |
| News / press | `"Use apify_perplexity_ai_scraper with queries: '[company] funding OR acquisition OR product launch'. Then use apify_google_search_scraper with queries: '[company] funding OR acquisition OR product launch site:techcrunch.com OR crunchbase.com OR businesswire.com' as a fallback. Flag results showing funding rounds, acquisitions, or strategic hires at ICP-fit companies."` *(include if News is an active source)* |

> **Do not apply any self-imposed time limit.** `/swarm/deploy/` is a heavy operation. The only hard limit is **5 minutes total** across deploy + polling.

After deploying, show the orchestrator's plan and each worker's label + task to the user before polling.

---

## Step 3 — Poll Swarm + Collect Results

Poll `POST {BASE_URL}swarm/status/` every ~30 seconds with the `workers` array from the deploy response:

```json
{ "workers": ["<doId>", "<doId>", ...] }
```

Show progress after each poll:

```
⏳ Signal swarm running — 3 / 7 workers done...
```

**Polling rules:**
- Only stop when `allDone: true` — or after 5 minutes of wall time
- Never stop early due to slow progress — workers commonly take 2–5 minutes
- `errored` and `unknown` workers still count as pending until `allDone` is `true`
- If 5 minutes elapse with zero completed workers, discard and redeploy with a refined directive
- Do not mix old and new worker IDs in the same status call

Once `allDone` is `true`, collect all `workers[].output` fields into a raw signal pool. Deduplicate entries by post URL.

---

## Step 4 — Classify Each Signal

Apply the saved `signal_config.classifications` model:

1. **Match against worked signals** — flag items matching patterns marked as working.
2. **Filter against not-worked signals** — discard or deprioritize excluded patterns.
3. **Keyword scan** — flag any content containing terms from `signal_config.keywords`.
4. **Assign class:**

| Class | Criteria |
|---|---|
| 🔥 Motivated Buyer | Explicit pain statement + request for solution or vendor |
| 🟢 Buying Signal | Job change into buyer role, tool eval post, competitor complaint |
| 🟡 Qualification Signal | Hiring for relevant role, funding round, strategic initiative post |
| 🔵 Watch & Wait | General pain content but no active search, or early signal with no urgency |
| ⚪ Cold Lead | ICP account match only — no active signal detected |

Extract intelligence dimensions per `signal_config.intelligence_goals` (pain, timing, budget, competitors) and attach as notes to each classified item.

Discard items below 🔵 Watch & Wait unless the user opted into Cold Leads.

---

## Step 5 — Enrich Prospects

For each flagged item, extract the poster's name and company. Fill missing contact data:

```json
{
  "searchQueries": ["<first name> <last name> <company>"],
  "maxResults": 1
}
```

Pull: full name, title, company, LinkedIn URL, email (if available).

---

## Step 6 — Deliver the Report

Show the Phase Detection Banner, then the summary line:

```
Found X signals across Y prospects — [N] 🔥 Motivated Buyers · [N] 🟢 Buying Signals · [N] 🟡 Qualification · [N] 🔵 Watch & Wait
```

**Always deliver results as a sheet — no exceptions.** Render a single flat table with one row per prospect, sorted by signal class priority (🔥 → 🟢 → 🟡 → 🔵 → ⚪):

| # | Name | Title | Company | Signal Class | Signal Type | Evidence | Source | Post URL | Email | Recommended Action |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Jane Smith | VP Revenue | Acme Corp | 🔥 Motivated Buyer | Pain | "We're drowning in manual reporting" | LinkedIn post · Jul 14 | [link] | j.smith@acme.com ✅ | Lead with reporting automation angle |
| 2 | Tom Lee | — | BetaCo | 🟡 Qualification | Hiring | Posted Head of RevOps role | LinkedIn Jobs · Jul 13 | [link] | — | Reach out before new hire starts |

Column notes:
- **Signal Class**: use the emoji tier badge (🔥 🟢 🟡 🔵 ⚪)
- **Signal Type**: Pain · Job Change · Hiring · Tool Eval · Funding · Competitor Complaint
- **Evidence**: a direct quote or 1-sentence description — be specific, not generic
- **Email**: show address if found + ✅ verified / ⚠️ unverified; show `—` if not found
- **Recommended Action**: pull from `signal_config.priority_action` + tailor to the specific signal

After the sheet, call out the **Top 3 rows** with a suggested first outreach line for each.

Then ask:

```
askQuestions([
  {
    header: "Next step",
    question: "What would you like to do with these signals?",
    multiSelect: true,
    options: [
      { label: "Save signals to Sushidata", recommended: true },
      { label: "Add top prospects to HeyReach campaign" },
      { label: "Draft outreach messages for top 3" },
      { label: "Export to CSV" },
      { label: "Nothing — just reviewing" }
    ],
    allowFreeformInput: false
  }
])
```

---

## Step 7 — Save to Context Lake (if requested)

```json
POST {BASE_URL}context/
{
  "content": "signal_scan: { date: '<today>', signals: [{ prospect, company, class, signal_type, evidence, post_url, intelligence: { pain, timing, budget, competitors }, recommended_action }] }",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Scheduled Run Behavior

When invoked automatically (Monday–Friday 9:00 AM):

1. Skip Phase 1 — config must already exist.
2. Load config and ICP targets from context lake silently.
3. Run Steps 2–5 silently.
4. Deliver the signal report (Step 6 cards + summary).
5. Auto-save scan results (Step 7) without prompting.
6. If no signals found: `No new signals detected today. Will check again next scheduled run.`
