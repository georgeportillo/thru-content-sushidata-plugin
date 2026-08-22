# Finding Companies and Contacts

Use this doc for discovery, sourcing, TAM/list building, known-source extraction, contact discovery, and hiring-qualified company search before any row-level enrichment.

This doc does **not** cover email waterfalls or row-level enrichment mechanics. If you already have rows and need to fill or transform columns, move to the enrichment phase using available provider tools.

## Core rules

**Companies first, then people.** When the task involves finding contacts at companies matching criteria (ICP, portfolio, accelerator, hiring signal), always discover the company set first, then search for people at those companies. Do not start with broad people-search queries — you will get noisy, unaffiliated results. The only exception is when the user provides a specific named company list and only needs contacts.

## Research with Sushidata

For any discovery task involving market intelligence — ICP sizing, niche verticals, investor-backed companies, technology usage — deploy a Sushidata swarm first:

```
POST /swarm/deploy/
{
  "query": "Find B2B SaaS companies in [vertical] with [ICP criteria]",
  "swarmSize": 5
}
```

Use swarm results to build the initial company seed list, then move to direct provider calls for structured contact discovery.

## Company search: provider escalation order

Escalate only when you need a filter the current step lacks.

1. **Web fetch / known source URL** — if the data lives on a public page (VC portfolio, accelerator directory, conference attendee list), fetch it directly with WebFetch before using any paid provider. This is faster, cheaper, and more complete.
2. **Sushidata swarm** — for concept-based or niche discovery ("find companies that use X technology", "find startups in this space").
3. **Exposed Sushidata Apify MCP tools** — use only when a supported actor fits the source, such as `apify_ycombinator_scraper`, `apify_g2_scraper`, `apify_google_search_scraper`, or `apify_perplexity_ai_scraper`.
4. **Direct enrichment providers for company data** — use when you need firmographics or structured company profiles:
   - `fullenrich_search_company` — company search with rich filters
   - `moltsets_search_companies` — synchronous company search by name, domain, industry, employee count, or revenue
   - [`predictleads_discover_companies`](provider-playbooks/predictleads.md) — search companies by location and size
   - [`predictleads_portfolio_companies`](provider-playbooks/predictleads.md) — VC/accelerator portfolio companies from public portfolio pages
   - For full detail, read [`provider-playbooks/enrichment-waterfall.md`](provider-playbooks/enrichment-waterfall.md)
5. **Missing actor feedback** — if the needed scraper is not exposed, follow `provider-playbooks/apify.md` instead of improvising actor IDs.

## People search: provider escalation order

> **Use all available lead finder actors together, not sequentially.** For any ICP prospecting or discovery task, run `apify_leads_finder`, `moltsets_search_people`, and `fullenrich_search_people` in combination — each covers different parts of the contact database. Never rely on a single actor for lead finding.

### Lead discovery (finding new people matching ICP criteria)

1. **`apify_leads_finder`** — primary tool for all ICP prospecting (200–50k leads). Rich filters: title, seniority, company domain, industry, company size, funding stage, tech stack, geography, verified/unverified email. Read [`provider-playbooks/apify.md`](provider-playbooks/apify.md) for the Leads Finder section. **Start here for any prospecting task.**
2. **`moltsets_search_people`** — synchronous people search by name, title, company domain, industry, seniority, or country. Up to 25 results per call; paginate with `offset`. Best for targeted lookups at specific companies. Read [`provider-playbooks/moltsets.md`](provider-playbooks/moltsets.md).
3. **`fullenrich_search_people`** — structured contact discovery by company domain. Best when you have a domain list and need contacts with emails.
4. **WebSearch / WebFetch** — for public directories, event attendee lists, or LinkedIn Sales Navigator exports the user already has.
5. **Sushidata swarm** — deploy a swarm naming all three lead finder actors explicitly when broad ICP discovery is needed. Use the swarm task template below.
6. **Missing actor feedback** — if bulk LinkedIn employee scraping is required, follow `provider-playbooks/apify.md`.

> **Swarm task template for lead finding** — always name all three actors explicitly so workers do not default to Google search:
> ```
> Worker 1: "Use apify_leads_finder with [title/seniority/industry/company-size filters] to find [N] leads matching ICP. Return linkedin_url, email, first_name, last_name, title, company, domain."
> Worker 2: "Use moltsets_search_people with [title, companyDomain, seniority] to find contacts at [specific company list]. Paginate with offset if needed. Return linkedin_url, email, first_name, last_name, title, company."
> Worker 3: "Use fullenrich_search_people with companyDomain for each domain in [domain list] to discover contacts. Return linkedin_url, email, first_name, last_name, title, company."
> ```

### Row-level enrichment (you have a person's identity, need email/phone)

7. **`moltsets_search_business_email_by_name`** — when you already have a person's name and company domain and just need their email: one synchronous call, tokens only charged on hit.
8. **`fullenrich_start_enrichment`** — named-person email lookup; submit first name + last name + domain (or linkedin_url), then poll `fullenrich_get_enrichment`.
9. **Multi-provider enrichment waterfall** — for maximum coverage, run providers in parallel (read [`provider-playbooks/enrichment-waterfall.md`](provider-playbooks/enrichment-waterfall.md)):
   - **Work email from LinkedIn URL**: `moltsets_linkedin_to_best_email` (synchronous) or `fullenrich_start_enrichment` → poll
   - **Bulk enrichment (20–100 contacts)**: `fullenrich_start_enrichment` → poll `fullenrich_get_enrichment`
   - **Mobile phone**: `moltsets_linkedin_to_mobile_phone`

## Discovery workflow

| Step | What to do                                                              | Why                                        |
| ---- | ----------------------------------------------------------------------- | ------------------------------------------ |
| 0    | Check if the data exists on a known public URL                          | Avoid unnecessary provider calls           |
| 1    | Shortlist 1–2 providers from the table above                            | Prevent random provider thrash             |
| 2    | Run a count-like or narrow first pass                                   | Cheaply confirm fit before full pull       |
| 3    | Execute the full discovery pass                                         | Build the seed list                        |
| 4    | Hand off to enrichment/contact-finding phase with the seed list         | Keep phases clean                          |

## Anti-patterns

- **Starting with people-search before having a company list** — searching for "GTM Engineer at YC startup" without a company list produces noisy, unaffiliated results. Find companies first, then find people at each.
- **Trying to discover portfolio companies through paid provider tools** — VC portfolio data is public. Fetch the portfolio page directly.
- **Manually looping calls for every row without dedupe** — for large batches, write a small script that calls the exposed Sushidata tools with deduplication and retry handling.

## Output format

### Company seed list

Deliver a CSV or structured list with at minimum: `company_name`, `domain`, and any available `linkedin_url`. Preserve source lineage. Do not start row-level enrichment until the seed list is complete and reviewed.

### Contact / lead output — required columns

Whenever returning a list of people (prospects, leads, contacts, enriched rows), **always include these four columns first, in this order**:

| # | Column | Notes |
| --- | --- | --- |
| 1 | `linkedin_url` | Full profile URL. Leave blank if not found — never fabricate. |
| 2 | `email` | Address + a status emoji (see key below). Leave blank if not found. |
| 3 | `first_name` | Given name only. Split from full name if needed. |
| 4 | `last_name` | Family name only. Split from full name if needed. |

Additional columns (title, company, domain, phone, seniority, etc.) follow after these four. You decide which additional columns are useful for the task — but the four above are **always present and always first**.

**Email status emoji key** — append directly after the address, no space:

| Emoji | Meaning |
| --- | --- |
| ✅ | Verified — safe for outbound |
| ⚠️ | Catch-all — deliverable but domain accepts everything; use with caution |
| ❓ | Unknown — not yet verified |
| ❌ | Invalid — bad address, do not send |

---

## Save to Context Lake

After completing any company or contact discovery run, save the results before ending the session:

```json
POST /context/
{
  "serverId": "26",
  "content": "Company/contact discovery complete. Task: {{task description}}. Companies found: {{list of domains or count}}. Contacts discovered: {{count, names, titles}}. Filters applied: {{ICP criteria used}}. Output file: {{CSV path if saved}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

**Save twice for multi-phase work:** once after company discovery (with the list of domains) and again after contact enrichment (with names, titles, and emails). This way, if a future session needs to continue from where you left off, it can retrieve the company list without re-running the discovery phase.
