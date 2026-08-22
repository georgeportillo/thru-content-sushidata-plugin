---
name: sushi-onboard
description: >
  Onboard a new user to the Sushidata GTM system end-to-end. Trigger when the user says:
  "/sushi-onboard", "onboard", "get me started", "set up sushidata", "help me get started",
  "walk me through sushidata", "first time setup", "onboard me", "start from scratch",
  or any request to begin a new GTM workflow or get introduced to the system.
---

# Sushidata Onboarding

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

When triggered, display this banner before anything else:

```
## 🍣 Sushidata Onboarding

Let's get your GTM system running in a few steps.
```

---

## Phase 1 — ICP Check

### Step 1A: Query the context lake

Run:

```json
POST {BASE_URL}query/
{ "query": "ICP ideal customer profile target companies industries personas criteria filters" }
```

**If a saved ICP is found:** Surface it clearly:

```
### ✅ ICP Found

Here's your saved ICP:

[Render the ICP in a clean table — company size, industry, titles, geography, 
tech stack signals, and any other criteria found]

Does this still look right, or do you want to update it?
```

Ask via `askQuestions`:

```
askQuestions([
  {
    header: "ICP Confirmation",
    question: "Does your ICP look accurate?",
    options: [
      { label: "Yes, this is correct — let's move on", recommended: true },
      { label: "I want to update it" }
    ],
    allowFreeformInput: false
  }
])
```

If the user wants to update, run the elicitation form in **Step 1B** and save the updated ICP to the context lake when complete.

**If no ICP is found:** Check if there is any ICP-like context in the current conversation (company type, target persona, verticals mentioned). If so, present it as an inferred ICP and ask the user to confirm or refine before using it.

If no context is available at all, run the full ICP elicitation form in **Step 1B**.

---

### Step 1B: ICP Elicitation Form

Collect ICP definition via `askQuestions` — two forms batched together:

**Form 1 — Company profile:**

```
askQuestions([
  {
    header: "Target company size",
    question: "What size companies do you sell to?",
    multiSelect: true,
    options: [
      { label: "Startup (1–50 employees)" },
      { label: "SMB (51–200 employees)", recommended: true },
      { label: "Mid-market (201–1,000 employees)" },
      { label: "Enterprise (1,000+ employees)" }
    ],
    allowFreeformInput: true
  },
  {
    header: "Target industries",
    question: "Which industries or verticals are your best customers in?",
    message: "List the top 1–3. Examples: SaaS, FinTech, Healthcare, eCommerce, Professional Services.",
    allowFreeformInput: true
  },
  {
    header: "Geography",
    question: "What geographies do you focus on?",
    multiSelect: true,
    options: [
      { label: "United States", recommended: true },
      { label: "Canada" },
      { label: "United Kingdom" },
      { label: "Europe (EMEA)" },
      { label: "APAC" },
      { label: "Global — no restriction" }
    ],
    allowFreeformInput: true
  }
])
```

**Form 2 — Buyer persona:**

```
askQuestions([
  {
    header: "Target job titles",
    question: "What titles or functions do your buyers usually hold?",
    message: "Examples: VP of Sales, Head of Marketing, CTO, RevOps Lead.",
    allowFreeformInput: true
  },
  {
    header: "Seniority",
    question: "What seniority levels are your economic buyers and champions?",
    multiSelect: true,
    options: [
      { label: "C-suite / Owner" },
      { label: "VP / Director", recommended: true },
      { label: "Manager / Lead" },
      { label: "Individual contributor" }
    ],
    allowFreeformInput: false
  }
])
```

After collecting, synthesize the ICP into a clean summary table and show it to the user. Then save it to the context lake:

```json
POST {BASE_URL}context/
{
  "content": "ICP defined during onboarding: [synthesized ICP summary with all fields collected]",
  "messageId": "msg-<timestamp>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Confirm: `✅ ICP saved to your context lake — it will be available in every future session.`

---

## Phase 2 — Choose Your Use Case

Once the ICP is confirmed, ask:

```
askQuestions([
  {
    header: "What would you like to do first?",
    question: "Choose what you want Sushidata to run for you:",
    message: "You can always do the other options later — this just sets what runs first.",
    options: [
      {
        label: "🎯 Find prospects — discover and enrich 10 leads matching my ICP",
        description: "Runs a swarm to find real leads with LinkedIn URLs and emails",
        recommended: true
      },
      {
        label: "📡 Find buying signals — scan LinkedIn, Reddit, and news for live signals",
        description: "Detects job changes, pain posts, hiring signals, and funding events"
      },
      {
        label: "⚡ Both — run prospects and signals in parallel",
        description: "Runs both workflows simultaneously in the background"
      }
    ],
    allowFreeformInput: false
  }
])
```

Proceed to **Phase 3** based on the answer.

---

## Phase 3 — Run the Workflow

### Option A: Find Prospects

Deploy a prospecting swarm using all three lead finder actors. Use the confirmed ICP from Phase 1.

**Target: 10 sample leads** — enough to validate quality before scaling.

```json
POST {BASE_URL}swarm/deploy/
{
  "query": "Find 10 sample B2B leads matching this ICP: [ICP summary from Phase 1]. Use all three lead finder tools in parallel:\n\nWorker 1: Use apify_leads_finder with ICP filters (personTitleIncludes, seniorityIncludes, companyIndustryIncludes, companySizeIncludes) to find 10 leads. Return linkedin_url, email, first_name, last_name, title, company, domain.\n\nWorker 2: Use moltsets_search_people with the same ICP title, seniority, and industry filters. Return linkedin_url, email, first_name, last_name, title, company.\n\nWorker 3: Use fullenrich_search_people with industry and seniority filters. Return linkedin_url, email, first_name, last_name, title, company.\n\nDedupe by linkedin_url. Return the top 10 most relevant leads across all three sources.",
  "swarmSize": 3
}
```

After deploying, show the plan and poll `{BASE_URL}swarm/status/` every 30 seconds until `allDone: true` (5-minute max).

When complete, render results as the standard lead table (linkedin_url · email · first_name · last_name · title · company). Then proceed to **Phase 4**.

---

### Option B: Find Signals

Trigger the full `sushi-signals` workflow — Phase 2 (Signal Scan) if a config already exists, or Phase 1 (Signal Config) if not.

> Read `../sushi-signals/SKILL.md` for the complete workflow. Proceed through it fully, then return to **Phase 4** once the signal scan results are delivered.

---

### Option C: Both (Parallel)

Deploy the prospecting swarm from Option A immediately. While it runs (polling), simultaneously begin the `sushi-signals` workflow from Option B.

Show the user two live progress indicators:

```
⏳ Prospect swarm — 1 / 3 workers done...
📡 Signals — scanning LinkedIn posts for job change signals...
```

Deliver prospect results and signal results together when both are complete. Then proceed to **Phase 4**.

---

## Phase 4 — What Would You Like to Do Next?

Once results are delivered, ask:

```
askQuestions([
  {
    header: "What's your next move?",
    question: "Now that you have data, what would you like to do with it?",
    multiSelect: true,
    options: [
      {
        label: "Send to HeyReach — add these leads to a LinkedIn outreach campaign",
        recommended: true
      },
      {
        label: "Export as a spreadsheet — download a CSV of the results"
      },
      {
        label: "Save to context lake — keep this data for future sessions"
      },
      {
        label: "Enrich further — get emails and phone numbers for all leads"
      },
      {
        label: "Scale up — run the same search for 100+ leads"
      },
      {
        label: "Something else"
      }
    ],
    allowFreeformInput: true
  }
])
```

Route based on their selection:

- **HeyReach** → proceed to **Phase 5**
- **Export as spreadsheet** → render a clean Markdown table and tell the user to copy it, or use the file system tools if available to write a CSV
- **Save to context lake** → run `sushi-save` workflow
- **Enrich further** → follow `sushi-research/provider-playbooks/enrichment-waterfall.md`
- **Scale up** → re-run the prospecting swarm from Phase 3 with a higher `totalResults` limit and ICP filters confirmed from Phase 1
- **Something else** → free-text; fulfill the request using the full Sushidata tool suite

---

## Phase 5 — HeyReach Connection Check

### Step 5A: Test the connection

Attempt to call `heyreach_list_campaigns` silently. Do not tell the user you are doing this.

**If the call succeeds:** HeyReach is connected. Auto-wire the Sushidata reply webhook immediately (see `sushi-research/provider-playbooks/heyreach.md` for the webhook setup), then:

```
✅ HeyReach is connected and reply alerts are active.

Here are your active campaigns:
[list campaigns from heyreach_list_campaigns]

Which campaign should I add these leads to?
```

Then add the leads to the selected campaign using `heyreach_add_to_campaign` in batches of ≤50.

**If the call fails or the tool is not available:** HeyReach is not connected. Proceed to **Step 5B**.

---

### Step 5B: HeyReach Setup Guide

```
## 🔗 Connect HeyReach

HeyReach isn't connected yet. Here's how to set it up in about 2 minutes:
```

Walk the user through these steps one at a time. After each step, ask "Done?" before continuing.

**Step 1 — Create your HeyReach account (skip if you already have one)**

> Create your HeyReach account to get started.

**Step 2 — Get your API key**

> In HeyReach: **Settings → API → Create API key**
> Copy the key — you'll need it in the next step.

**Step 3 — Find the HeyReach MCP server URL**

> In HeyReach: **Settings → Integrations → Claude / MCP**
> Copy the MCP server URL shown there. It will look like:
> `https://mcp.heyreach.io/sse?apiKey=YOUR_API_KEY`
>
> (If you don't see an MCP section, use: `https://mcp.heyreach.io/sse?apiKey=<your-api-key>` and replace `<your-api-key>` with the key you copied in Step 2.)

**Step 4 — Add the MCP server to Claude (Cowork)**

> In your Cowork / Claude interface:
> **Settings → Connectors → Add MCP Server**
>
> Enter:
> - **Name**: HeyReach
> - **URL**: the MCP URL from Step 3
>
> Save and reload the connector.

**Step 5 — Confirm**

> Once added, come back here and say "HeyReach is connected" — I'll verify the connection and set up reply alerts automatically.

After the user confirms HeyReach is connected, re-run **Step 5A** to verify and continue.

---

## Phase 6 — CRM and Tooling Check

After HeyReach is set up (or if the user skipped it), ask:

```
askQuestions([
  {
    header: "Any other tools to connect?",
    question: "Do you use a CRM or any other tools you'd like to sync this data with?",
    options: [
      { label: "HubSpot — sync contacts and companies to my CRM" },
      { label: "Salesforce — push leads to Salesforce" },
      { label: "Notion or Airtable — export data to my workspace" },
      { label: "Slack — get a summary posted to a channel" },
      { label: "I'll handle exports manually" },
      { label: "Skip for now" }
    ],
    allowFreeformInput: true
  }
])
```

**If HubSpot:** Follow `sushi-research/provider-playbooks/hubspot.md` to sync contacts.

**If Salesforce, Notion, Airtable, Slack:** These are not natively integrated. Offer to export the data as a CSV the user can import manually, or suggest using Zapier/Make as a bridge. Draft the export immediately — don't wait.

**If skip:** Acknowledge and move to **Phase 7**.

---

## Phase 7 — Handoff

Once onboarding is complete, close with:

```
## ✅ You're set up

Here's a quick summary of what we did:

- **ICP**: [one-line ICP summary]
- **Leads found**: [N leads — link to table above]
- **Signals found**: [N signals — link to signal table, or "not run this session"]
- **HeyReach**: [connected / not connected]
- **CRM**: [HubSpot connected / skipped]

**What I can do from here:**
- `/sushi-signals` — scan for buying signals on a schedule (Mon–Fri 9am)
- `/sushi-sales` — run your daily 40-lead pipeline
- Ask me to enrich any lead, build a TAM, research a company, or write outreach

What would you like to do next?
```

Stay in the session. Do not end the conversation — continue to fulfill whatever the user asks next using the full Sushidata tool suite.
