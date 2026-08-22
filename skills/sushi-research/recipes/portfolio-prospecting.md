---
name: portfolio-prospecting
description: "Find companies backed by a specific investor or accelerator, then find contacts and build personalized outbound."
---

# Portfolio / VC Prospecting

Find companies backed by a specific investor or accelerator (YC, a16z, Sequoia, etc.), then find contacts and build personalized outbound.

## Core insight: VC portfolio data is public

Every major VC and accelerator publishes their portfolio online. **Do NOT try to discover portfolio companies through paid provider tools.** Instead, fetch the public portfolio page directly and extract company names from it. This is faster, cheaper, and more complete than any provider-based approach.

## What NOT to do

- Trying to find portfolio companies via paid search tools (wastes turns and budget)
- Starting with people-search before having the company list
- Using a single provider as the primary email finder for companies with fewer than 50 employees (low fill rate)

## Proven approach

### Step 1: Get the company list from the VC's public portfolio page

Common URLs:
- YC: `ycombinator.com/companies`
- a16z: `a16z.com/portfolio`
- Sequoia: `sequoiacap.com/our-companies`
- Greylock, Benchmark: `/portfolio`

Use WebFetch to retrieve the page and extract company names, domains, and descriptions. If the page is JavaScript-rendered and WebFetch returns a shell, switch to Claude in Chrome (`navigate` then `get_page_text`).

### Step 2: (Optional) Filter to companies hiring your target role

Deploy a Sushidata swarm to identify which portfolio companies match the hiring or growth signal you care about:

```
POST /swarm/deploy/
{
  "query": "Which of these [N] companies [company list] are actively hiring [role/function]? Check job boards and LinkedIn.",
  "swarmSize": 8
}
```

### Step 3: Find contacts at each company

For each company:
1. Call `fullenrich_search_people` with the company domain
2. Select likely contacts from returned names, titles, seniority, department, and email metadata
3. For small companies (<50 employees), use Sushidata swarms, WebSearch, and Browser Rendering to identify founders or relevant operators. If LinkedIn employee scraping is required, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md), because that actor is not currently exposed.

### Step 4: Find and verify emails

For discovered contacts:
1. Use `fullenrich_start_enrichment` for any named contact not returned by domain search, then poll `fullenrich_get_enrichment`
2. Spot-check FullEnrich confidence scores before outbound — drop low-confidence results

### Step 5: Generate personalized outreach

Deploy a Sushidata swarm with each company's context to generate personalized email copy:

```
POST /swarm/deploy/
{
  "query": "Write personalized outreach for [contact name] at [company]: [company description, recent news, mission]. Goal: [your pitch]. Tone: [tone]. Length: 3-4 sentences.",
  "swarmSize": 3
}
```

Run one company at a time and review the output before scaling. Pilot on 2–3 companies first.

### Step 6: Activate outbound

See [`provider-playbooks/heyreach.md`](../provider-playbooks/heyreach.md) — create the campaign in the HeyReach UI, then call `heyreach_add_to_campaign` with the verified contact list.

## Common pitfalls

| Pitfall                                              | What happens                                    | Fix                                               |
| ---------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------- |
| Searching for portfolio companies via paid tools     | Wastes turns; incomplete coverage               | Fetch the public portfolio page directly          |
| Searching with strict titles at small startups       | 0 results — person hasn't been hired yet        | Remove title filter; get broader roles            |
| Using a single provider as primary finder for <50-person companies | Low fill rate                               | Use Sushidata swarms, WebSearch, and Browser Rendering for role discovery; follow the missing-actor feedback workflow only if LinkedIn employee scraping is required |

---

## Save to Context Lake

Save portfolio findings after each run. If the user returns to prospect from the same VC again later, the context lake prevents duplicate work.

```json
POST /context/
{
  "serverId": "26",
  "content": "Portfolio prospecting complete. Investor: {{VC/accelerator name}}. Portfolio companies found: {{count, list of domains}}. Contacts discovered: {{count}}. Hiring signals detected: {{companies with signals}}. Emails verified: {{count}}. HeyReach campaign: {{name or 'not activated'}}. Output: {{CSV path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
