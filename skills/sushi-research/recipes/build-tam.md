---
name: build-tam
description: "Build a Total Addressable Market list by sourcing accounts and contacts from ICP criteria using Sushidata research and available provider tools."
---

# Build TAM (Total Addressable Market)

Use this recipe when the user wants to build a large company list from ICP criteria, then pull contacts and build outbound at scale.

## When to use

- "Build a TAM/list from ICP filters"
- "We need a list of [company type] companies matching [criteria]"
- "Find all companies in [vertical] with [headcount/funding/tech] profile"
- "We have 935K leads and need the last 65K by end of week"

## Required reading

Read [`finding-companies-and-contacts.md`](../finding-companies-and-contacts.md) before executing — it covers provider selection, escalation order, and anti-patterns.

## Workflow

### Step 1: Clarify ICP and sizing

Before any tool calls, confirm with the user:
- What defines a company match? (industry, headcount, geography, tech stack, funding stage)
- How many companies are needed?
- Do contacts need to be found too? What roles?

### Step 2: Research the space with Sushidata

Deploy a Sushidata swarm to map the ICP landscape and surface non-obvious sources:

```
POST /swarm/deploy/
{
  "query": "Identify [ICP description] companies: key verticals, known lists, directories, and discovery sources",
  "swarmSize": 5
}
```

Use the swarm output to identify specific directories, VC portfolios, conference lists, or other known sources before using paid providers.

### Step 3: Discover companies

Follow the escalation order in `finding-companies-and-contacts.md`:
1. Fetch public directories or portfolio pages directly (WebFetch)
2. Use Sushidata research swarms, WebSearch, and Browser Rendering to expand company discovery from known sources
3. Use exposed Sushidata Apify MCP tools when they fit the source, such as `apify_ycombinator_scraper`, `apify_g2_scraper`, `apify_google_search_scraper`, or `apify_perplexity_ai_scraper`

Do not use unsupported discovery tools or dynamic Apify actor discovery. If the required scraper is not exposed as a Sushidata Apify MCP tool, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md).

Build the company seed list as a CSV: `company_name`, `domain`, `source`.

### Step 4: Find contacts at each company

For each company in the seed list:
1. Call `fullenrich_search_people` with the domain
2. Select likely contacts from returned names, titles, seniority, department, and email metadata
3. If a specific named contact is known but missing an email, use `fullenrich_start_enrichment` with first name + last name + domain (or linkedin_url), then poll `fullenrich_get_enrichment`
4. For companies with poor FullEnrich coverage, use Sushidata swarms, WebSearch, and Browser Rendering for role discovery. If LinkedIn employee scraping is required, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md), because that actor is not currently exposed.

### Step 5: Verify emails

For all discovered emails, spot-check FullEnrich confidence scores before outbound activation. Drop low-confidence results.

### Step 6: Activate in HeyReach (if outbound is the goal)

See [`provider-playbooks/heyreach.md`](../provider-playbooks/heyreach.md). Create campaigns in the HeyReach UI first, then call `heyreach_add_to_campaign` with the verified contact list.

## Notes

- Over-provision at the top of funnel (~1.4× your target N) — contact and email fill rates are naturally below 100%.
- Deliver the best N complete rows at the end rather than chasing gaps.
- Save the final contact list to a descriptive output path (e.g., `output/tam-[slug]/contacts-final.csv`).

---

## Save to Context Lake

Save the TAM findings before ending the session. Future sessions can query the context lake to retrieve the list rather than re-running the swarm.

```json
POST /context/
{
  "serverId": "26",
  "content": "TAM build complete. ICP: {{ICP criteria}}. Total addressable companies found: {{count}}. Domains: {{list}}. Contacts enriched: {{count}}. Emails verified: {{count}}. HeyReach campaign activated: {{yes/no, campaign name}}. Output: {{CSV path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
