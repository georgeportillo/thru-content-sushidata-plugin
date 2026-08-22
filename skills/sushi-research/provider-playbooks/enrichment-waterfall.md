# Enrichment Waterfall Playbook

Use this playbook when you need to find or enrich email addresses, person profiles, or company data using Sushidata's direct enrichment tools. These tools are called **directly** — no swarm needed for individual lookups — but they should be run **in parallel across multiple swarm workers** to maximize coverage when enriching more than a handful of contacts.

---

## Available Enrichment Tools

### Person / Contact Tools

| Tool name | Provider | Best for |
| --- | --- | --- |
| `fullenrich_start_enrichment` | FullEnrich | **Bulk** enrichment: work email + personal email + mobile for up to 100 contacts at once |
| `fullenrich_get_enrichment` | FullEnrich | Poll results from a bulk enrichment started with `fullenrich_start_enrichment` |
| `fullenrich_reverse_email` | FullEnrich | Start reverse email enrichment (email → person profile) |
| `fullenrich_get_reverse_email` | FullEnrich | Poll results from a reverse email enrichment |
| `fullenrich_search_people` | FullEnrich | Search people with rich filters |
| `moltsets_search_people` | MoltSets | Synchronous people search by name, title, company domain, industry, country, seniority |
| `moltsets_linkedin_to_best_email` | MoltSets | LinkedIn URL → best available email (business first, personal fallback) |
| `moltsets_linkedin_to_business_email` | MoltSets | LinkedIn URL → corporate email only |
| `moltsets_linkedin_to_personal_email` | MoltSets | LinkedIn URL → personal email (Gmail, iCloud, etc.) |
| `moltsets_linkedin_to_mobile_phone` | MoltSets | LinkedIn URL → carrier-verified mobile phone (10 tokens/result) |
| `moltsets_reverse_email_lookup` | MoltSets | Email → full profile: name, title, seniority, company, LinkedIn URL, firmographics |
| `moltsets_reverse_linkedin_lookup` | MoltSets | LinkedIn URL → complete profile + business email + personal email in one call |
| `moltsets_search_business_email_by_name` | MoltSets | Name + domain → business email in one synchronous call |
| `moltsets_search_business_profile_by_name` | MoltSets | Name + domain → full profile: title, company, firmographics |

### Company Tools

| Tool name | Provider | Best for |
| --- | --- | --- |
| `fullenrich_search_people` | FullEnrich | Contact discovery by company domain or filters |
| `fullenrich_search_company` | FullEnrich | Company search with rich filters |
| `moltsets_search_companies` | MoltSets | Synchronous company search by name, domain, industry, employee count, revenue range |
| `predictleads_company` | PredictLeads | Enrich one company by domain or ID — firmographics, descriptions, relationships |
| `predictleads_discover_companies` | PredictLeads | Search companies by location and size |
| `predictleads_news_events` | PredictLeads | Buying signals for one company: funding, hires, launches, partnerships, etc. |
| `predictleads_financing_events` | PredictLeads | Funding rounds for one company |
| `predictleads_technology_detections` | PredictLeads | Technology stack for one company |
| `predictleads_companies_using_technology` | PredictLeads | Find all companies using a specific technology |
| `predictleads_discover_job_openings` | PredictLeads | Cross-company hiring signal search |
| `predictleads_discover_news_events` | PredictLeads | Cross-company news event search |
| `predictleads_discover_financing_events` | PredictLeads | Cross-company funding event search |
| `predictleads_portfolio_companies` | PredictLeads | VC/accelerator portfolio companies |
| `predictleads_similar_companies` | PredictLeads | Find companies similar to a target |

---

## Multi-Agent Enrichment Strategy

For lists of 5+ contacts, **do not enrich serially** — deploy parallel swarm workers, then merge. This maximizes coverage without bottlenecking on any single provider's hit rate.

### Recommended agent decomposition

#### Agent 1 — Email Discovery (Work Email)
**Goal**: Find work emails for all contacts.

- **Single/small batches**: `fullenrich_start_enrichment` with first_name + last_name + domain (or linkedin_url), poll `fullenrich_get_enrichment`
- **Bulk (20–100 contacts)**: `fullenrich_start_enrichment` — submit all at once, poll with `fullenrich_get_enrichment`. **Always include `enrich_fields` inside each contact object** (not as a top-level field):
  ```json
  { "linkedin_url": "...", "enrich_fields": ["contact.work_emails"] }
  ```

FullEnrich costs: work email = 1 credit, personal email = 3 credits, mobile phone = 10 credits per contact found. Only request the fields you need.

#### Agent 2 — Email Quality Check (mandatory before outbound)
**Goal**: Review FullEnrich confidence scores on all discovered emails before any outbound activation.

Use: Review confidence scores from `fullenrich_get_enrichment` results. Treat low-confidence results as non-send by default — flag results without a work email or with no confident match as risky. Spot-check a sample with `fullenrich_reverse_email` → poll `fullenrich_get_reverse_email` when confidence is uncertain.

#### Agent 3 — Deep Profile Enrichment
**Goal**: Fill in title, seniority, department, employment history, and any missing firmographic data.

Use: `fullenrich_reverse_email` → poll `fullenrich_get_reverse_email` (email → full profile), `fullenrich_start_enrichment` → poll `fullenrich_get_enrichment` (email + phone from LinkedIn)

#### Agent 4 — Company Enrichment
**Goal**: Enrich all companies on the list with firmographics.

Use: `fullenrich_search_company` (from domain or filters)

---

## Merge and Dedupe Logic

After all agents complete, merge per-contact using this priority order:

| Field | Priority |
| --- | --- |
| `email` (work) | FullEnrich highest confidence first; MoltSets business email as secondary; drop low-confidence results |
| `email` (personal) | MoltSets personal email; FullEnrich |
| `mobile_phone` | MoltSets |
| `title` | MoltSets reverse lookup → FullEnrich |
| `company` firmographics | MoltSets → FullEnrich |

Dedupe contacts by: `email` → `linkedin_url` → `(full_name + company_domain)` in that order.

---

## Swarm Request Pattern

When deploying agents for enrichment, structure each swarm worker with a narrow, concrete task:

```json
POST /swarm/deploy/
{
  "query": "Email discovery for the following contacts: [list of name + domain pairs]. For each contact, use fullenrich_start_enrichment with first_name + last_name + domain (or linkedin_url). Poll fullenrich_get_enrichment until done. Return per contact: email found, confidence or status if available. Leave email blank if the provider returns nothing.",
  "swarmSize": 3
}
```

```json
POST /swarm/deploy/
{
  "query": "Bulk email enrichment for the following contacts: [list]. Use fullenrich_start_enrichment. Include linkedin_url per contact where available. Set enrich_fields INSIDE each contact object in the data array (e.g. {\"linkedin_url\": \"...\", \"enrich_fields\": [\"contact.work_emails\"]}). Get the enrichment_id, then poll fullenrich_get_enrichment until status is FINISHED. Return per contact: work email, personal email, phone if found.",
  "swarmSize": 2
}
```

---

## Bulk Enrichment (FullEnrich)

For batches of 20–100 contacts, use FullEnrich instead of per-contact calls:

1. Call `fullenrich_start_enrichment` with up to 100 contacts. Include `linkedin_url` per contact whenever available — it significantly improves hit rates.
2. Inside each contact object in the `data` array, include `enrich_fields` specifying what to retrieve (e.g. `["contact.work_emails"]`). This field is **per-contact, not top-level** — placing it at the top level will be ignored.
3. Get back an `enrichment_id`.
4. Poll `fullenrich_get_enrichment` with that ID until `status` is `FINISHED`.
5. Merge FullEnrich results with results from other agents.

---

## Prospecting with People/Company Search

When you have ICP criteria and need a list of matching contacts or companies:

| Goal | Use |
| --- | --- |
| Search contacts by domain or filters | `fullenrich_search_people` or `moltsets_search_people` (synchronous, up to 25/call) |
| Search people with rich filters | `fullenrich_search_people` or `moltsets_search_people` |
| Search companies with rich filters | `fullenrich_search_company` or `moltsets_search_companies` |
| Signal-based company discovery (funding, hiring, tech usage, news) | `predictleads_discover_*` tools — see `provider-playbooks/predictleads.md` |
| Large-scale ICP prospecting (200–50k leads) | `apify_leads_finder` — see `provider-playbooks/apify.md` |

---

## Save to Context Lake

After completing any enrichment run, save a summary:

```json
POST /context/
{
  "serverId": "26",
  "content": "Enrichment waterfall complete. Contacts processed: {{count}}. Emails found: {{count}}. Providers used: {{list}}. Output: {{file path}}.",
  "messageId": "msg-<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
