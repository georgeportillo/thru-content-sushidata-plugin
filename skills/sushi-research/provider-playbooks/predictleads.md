# PredictLeads Playbook

Use this playbook when a Sushidata research or GTM task needs company intelligence, buying signals, technology stack data, hiring signals, funding events, or webhook-based company tracking via PredictLeads.

PredictLeads tools are called **directly** — they are synchronous GET or POST calls that return data immediately. All calls consume PredictLeads API credits unless noted as free.

---

## Available Tools

### Company Profile

| Tool name | Best for |
| --- | --- |
| `predictleads_company` | Enrich a single company by domain or PredictLeads ID — name, description, location, ticker, parent/subsidiary relationships, etc. |
| `predictleads_similar_companies` | Find up to 50 companies similar to a target, with AI-generated similarity reasons |

### Buying Signals & Intelligence

| Tool name | Best for |
| --- | --- |
| `predictleads_news_events` | Structured buying signals for one company: funding, hires, launches, partnerships, expansions, layoffs, etc. |
| `predictleads_financing_events` | Funding/financing rounds for one company: amount, round type, investors, effective date |
| `predictleads_connections` | Business relationships for one company: partners, integrations, clients, vendors, investors, competitors |
| `predictleads_products` | Product portfolio for one company |
| `predictleads_website_evolution` | Historical website structure and content changes |
| `predictleads_github_repositories` | Public GitHub repos owned by a company, with time-series forks/stars/watches |
| `predictleads_sec_filings` | SEC filings (6-K, 8-K, 10-K, 10-Q, 20-F, 40-F, 25) for public companies, returned as Markdown text |

### Technology Stack Intelligence

| Tool name | Best for |
| --- | --- |
| `predictleads_technology_detections` | Technology stack for one company (~54k detected technologies) |
| `predictleads_companies_using_technology` | Find all companies using a specific technology |
| `predictleads_extended_technology_detection` | Deep evidence for a single technology detection (subpages, phrases, DNS, sources) |
| `predictleads_technologies` | Search the technology catalog by fuzzy name |
| `predictleads_technology` | Full details for one technology by ID or fuzzy name |

### Hiring Signals

| Tool name | Best for |
| --- | --- |
| `predictleads_job_openings` | Current job openings for one company — titles, locations, categories, seniority, O*NET |
| `predictleads_job_openings_deleted` | Recently deleted/closed job openings for one company |
| `predictleads_discover_job_openings` | Cross-company job search by location, category, and date range |

### Cross-Company Discovery

| Tool name | Best for |
| --- | --- |
| `predictleads_discover_companies` | Search all companies by location and size |
| `predictleads_discover_news_events` | Search news events across all companies by category and location |
| `predictleads_discover_financing_events` | Search funding events across all companies by type and location |
| `predictleads_portfolio_companies` | Portfolio companies surfaced from VC/accelerator/incubator websites |
| `predictleads_startup_platform_posts` | Startup platform posts (currently Hacker News Show HN / Who is Hiring) |

### Follow / Subscription Management

| Tool name | Best for |
| --- | --- |
| `predictleads_followings` | List companies currently followed for webhook notifications (free) |
| `predictleads_follow_company` | Follow a company to receive webhook notifications (free) |
| `predictleads_unfollow_company` | Stop following a company (costs 1 credit) |
| `predictleads_api_subscription` | Check subscription status and remaining credits (free) |

---

## When to Use PredictLeads

| Task | Prefer |
| --- | --- |
| Enrich a company from its domain | `predictleads_company` |
| Find companies similar to a target account | `predictleads_similar_companies` |
| Detect buying signals at an account | `predictleads_news_events` |
| Find recently-funded companies | `predictleads_discover_financing_events` |
| Find companies hiring specific roles | `predictleads_discover_job_openings` |
| Map a company's tech stack or find competitors/users of a technology | `predictleads_technology_detections` / `predictleads_companies_using_technology` |
| Build a TAM list by location + size | `predictleads_discover_companies` |
| Build an investor portfolio list | `predictleads_portfolio_companies` |
| Track a target account continuously | `predictleads_follow_company` + `predictleads_followings` |
| Check remaining credits before a large run | `predictleads_api_subscription` |

**PredictLeads strengths:** buying-signal detection, technology stack intelligence, hiring signals, funding events, continuous webhook-based company tracking.  
**Complementary tools:** pair PredictLeads company discovery with `fullenrich_search_people` or `moltsets_search_people` for contact enrichment, and route outbound activation through [`heyreach.md`](heyreach.md) or CRM sync through [`hubspot.md`](hubspot.md).

---

## Company Enrichment

Single company by domain:

```json
{
  "id_or_domain": "hubspot.com"
}
```

Call `predictleads_company`. Returns description, location, ticker, meta title/description, parent/subsidiary relationships, and more.

---

## Buying Signal Detection

### News events for one company

```json
{
  "company_id_or_domain": "deel.com",
  "categories": "receives_financing,hires,launches",
  "found_at_from": "2024-09-01",
  "limit": 100
}
```

Call `predictleads_news_events`. Event categories include: `acquires`, `closes_offices_in`, `declares_bankruptcy`, `decreases_headcount_by`, `expands_facilities`, `expands_offices_in`, `expands_offices_to`, `files_suit_against`, `goes_public`, `has_earnings`, `has_issues_with`, `has_revenue`, `has_valuation`, `hires`, `identified_as_competitor_of`, `increases_headcount_by`, `integrates_with`, `invests_into`, `invests_into_assets`, `is_developing`, `launches`, `loses_client`, `opens_new_location`, `partners_with`, `promotes`, `receives_financing`, `recognized_as`, `retires_from`, `spins_off_company`, `spins_off_division`, `wins_contract`.

### Financing events for one company

```json
{
  "company_id_or_domain": "deel.com",
  "limit": 50
}
```

Call `predictleads_financing_events`. Returns amount, round type (Seed, Series A-J, etc.), investors, effective date, and source URLs.

---

## Technology Stack & Competitive Intelligence

### Stack for one company

```json
{
  "company_id_or_domain": "adobe.com",
  "limit": 100
}
```

Call `predictleads_technology_detections`. Detects ~54k technologies via script tags, DNS records, job descriptions, cookies, and more.

### Find all companies using a technology

```json
{
  "technology_id_or_fuzzy_name": "HubSpot",
  "limit": 100
}
```

Call `predictleads_companies_using_technology`. Fuzzy match by name or use an exact technology ID from `predictleads_technologies`.

### Search the technology catalog

```json
{
  "fuzzy_name": "Salesforce",
  "order_by": "fuzzy_score_desc",
  "limit": 25
}
```

Call `predictleads_technologies`. Use `order_by: fuzzy_score_desc` when searching by name to surface the best matches first.

---

## Hiring Intelligence

### Current openings at one company

```json
{
  "company_id_or_domain": "hubspot.com",
  "categories": "software_development,engineering",
  "first_seen_at_from": "2024-09-01",
  "limit": 100
}
```

Call `predictleads_job_openings`.

### Cross-company job discovery

```json
{
  "company_location": "United States",
  "categories": "software_development,engineering",
  "first_seen_at_from": "2024-10-01",
  "limit": 100
}
```

Call `predictleads_discover_job_openings`.

---

## Cross-Company Discovery

### Companies by location + size

```json
{
  "location": "United States",
  "sizes": "51-200,201-500",
  "limit": 100
}
```

Call `predictleads_discover_companies`.

### Recently-funded companies

```json
{
  "financing_types_normalized": "seed,series_a,series_b",
  "company_location": "California",
  "limit": 100
}
```

Call `predictleads_discover_financing_events`.

### News events across all companies

```json
{
  "categories": "receives_financing,hires,launches",
  "company_location": "United States",
  "limit": 100
}
```

Call `predictleads_discover_news_events`.

### VC portfolio companies

```json
{
  "first_seen_at_from": "2024-09-01",
  "limit": 100
}
```

Call `predictleads_portfolio_companies`. Returns connections categorized as `investor`, showing the VC firm and portfolio company.

---

## Following Companies (Webhooks)

Following is free and enables webhook notifications for new PredictLeads events on a company.

### Follow a company

```json
{
  "company_id_or_domain": "hubspot.com",
  "custom_company_identifier": "crm-12345"
}
```

Call `predictleads_follow_company`.

### List followed companies

```json
{
  "limit": 100
}
```

Call `predictleads_followings`.

### Unfollow a company

```json
{
  "company_id_or_domain": "hubspot.com"
}
```

Call `predictleads_unfollow_company`. Costs 1 credit.

---

## Swarm Request Patterns

**Multi-signal account brief:**

```json
POST /swarm/deploy/
{
  "query": "Build a buying-signal brief for [domain]. Use predictleads_company for firmographics, predictleads_news_events for recent signals (limit 50), predictleads_job_openings for hiring trends (limit 50), predictleads_technology_detections for tech stack, and predictleads_connections for partners/integrations. Synthesize into: company summary, top 5 recent signals, hiring themes, key technologies, and notable relationships.",
  "swarmSize": 5
}
```

**Funding-based prospecting:**

```json
POST /swarm/deploy/
{
  "query": "Find recently-funded B2B SaaS companies in [location/stage]. Use predictleads_discover_financing_events with financing_types_normalized=seed,series_a,series_b and company_location filter. For each returned company, call predictleads_company to get domain and description. Return: company name, domain, funding type, location, and a short description.",
  "swarmSize": 3
}
```

**Technology install-base prospecting:**

```json
POST /swarm/deploy/
{
  "query": "Find companies using [technology] that were detected recently. Use predictleads_technologies with fuzzy_name to resolve the technology ID, then predictleads_companies_using_technology with that ID and first_seen_at_from filter. Return company name, domain, first_seen_at, and detection source.",
  "swarmSize": 3
}
```

**Hiring-signal prospecting:**

```json
POST /swarm/deploy/
{
  "query": "Find companies actively hiring [role categories] in [location] using predictleads_discover_job_openings. Filter by categories and first_seen_at_from. For each company, use predictleads_company to get the domain. Return company name, domain, job titles sampled, and first_seen_at.",
  "swarmSize": 3
}
```

---

## Rules

- Always check `predictleads_api_subscription` before large discovery runs to confirm remaining credits.
- Use company domain (`hubspot.com`) rather than a full URL where accepted — most endpoints treat the domain as a valid identifier.
- Filter dates with ISO-8601 strings (e.g. `2024-09-25`). Use `_from` and `_until` together to bound time windows and avoid unbounded credit spend.
- `limit` defaults are small; raise `limit` up to 1000 for broad discovery, but pair with tight filters to avoid wasted credits.
- `predictleads_follow_company` is free and idempotent — call it freely for target accounts the user wants to track.
- `predictleads_unfollow_company` costs 1 credit — confirm before unfollowing many accounts.
- PredictLeads gives company- and signal-level intelligence, not individual contact data. Pair with FullEnrich or MoltSets to find contacts at discovered companies.
- When building prospect lists from PredictLeads, follow the output format in [`finding-companies-and-contacts.md`](../finding-companies-and-contacts.md): `company_name`, `domain`, and source lineage before moving to contact enrichment.

---

## Save to Context Lake

After any PredictLeads discovery or enrichment run, save the output summary:

```json
POST /context/
{
  "serverId": "26",
  "content": "PredictLeads run complete. Task: {{company enrichment / signal scan / discovery}}. Companies processed: {{count}}. Signals found: {{count}}. Tools used: {{list}}. Output: {{file path or inline table}}.",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
