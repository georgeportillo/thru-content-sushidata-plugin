# MoltSets Playbook

Use this playbook when a Sushidata research or GTM task needs synchronous contact enrichment, LinkedIn-to-email conversion, reverse lookups, people/company search, or ad-audience identity resolution via MoltSets.

MoltSets tools are called **directly** — most return results synchronously with no polling required. Tokens are only charged when data is found.

---

## Available Tools

### Search & Discovery

| Tool name | Best for |
| --- | --- |
| `moltsets_search_people` | Find contacts by name, title, company, industry, country, seniority, or department — returns ranked records |
| `moltsets_search_companies` | Find companies by name, domain, industry, employee count, or revenue range |
| `moltsets_search_linkedin_profile` | Find LinkedIn profile URL from a name + company domain (use `count_only: true` first to check volume for free) |
| `moltsets_search_business_email_by_name` | Find business email from name + domain in one call — LinkedIn lookup + email in a single request |
| `moltsets_search_business_profile_by_name` | Full business profile from name + domain: title, company, firmographics |

### LinkedIn → Contact

| Tool name | Best for |
| --- | --- |
| `moltsets_linkedin_to_best_email` | LinkedIn URL → best available email; returns business if found, personal as fallback — use as default |
| `moltsets_linkedin_to_business_email` | LinkedIn URL → corporate email only (no personal fallback) |
| `moltsets_linkedin_to_personal_email` | LinkedIn URL → personal email (Gmail, iCloud, etc.) |
| `moltsets_linkedin_to_best_personal_email` | LinkedIn URL → highest-confidence personal email with validation date |
| `moltsets_linkedin_to_mobile_phone` | LinkedIn URL → carrier-verified mobile phone (single or batch up to 100) — costs 10 tokens per result |

### Reverse Lookups

| Tool name | Best for |
| --- | --- |
| `moltsets_reverse_email_lookup` | Email → full profile: name, title, seniority, company, LinkedIn URL, firmographics |
| `moltsets_reverse_linkedin_lookup` | LinkedIn URL → complete profile + valid business email + personal email — MoltSets' most complete enrichment endpoint |

### Identity & Audiences

| Tool name | Best for |
| --- | --- |
| `moltsets_email_to_linkedin` | Email → LinkedIn URL (highest-confidence match) |
| `moltsets_ip_to_company` | IPv4 → company domain (de-anonymise B2B web traffic) |
| `moltsets_email_to_maid` | Email → Mobile Advertising IDs (AAID + IDFA) for cross-device retargeting |
| `moltsets_business_email_to_sha256` | Business email → SHA256-hashed personal emails for privacy-safe ad audience matching |
| `moltsets_linkedin_to_sha256` | LinkedIn URL → SHA256 + MD5 hashed emails (business and personal) for ad platform uploads |

### Account Management (free — no tokens consumed)

| Tool name | Best for |
| --- | --- |
| `moltsets_get_account` | Check account details, plan, and credit balance |
| `moltsets_get_billing` | Check subscription status, token costs per tool, and billing period |
| `moltsets_get_usage` | Day-by-day usage breakdown — calls, credits, hit rate (today / week / month / billing_cycle) |

---

## When to Use MoltSets vs FullEnrich vs apify_leads_finder

| Task | Prefer |
| --- | --- |
| Known LinkedIn URL → best email (B2B outbound) | `moltsets_linkedin_to_best_email` |
| Known LinkedIn URL → full profile + email in one call | `moltsets_reverse_linkedin_lookup` |
| Known LinkedIn URL → mobile phone | `moltsets_linkedin_to_mobile_phone` |
| Name + domain → business email in one call | `moltsets_search_business_email_by_name` |
| Name + domain → full profile (title, company, firmographics) | `moltsets_search_business_profile_by_name` |
| Email → full profile | `moltsets_reverse_email_lookup` |
| Email → LinkedIn URL | `moltsets_email_to_linkedin` |
| IP address → company | `moltsets_ip_to_company` |
| Ad audience activation (MAID, SHA256 hashes) | MoltSets audience tools |
| Bulk 20–100 contacts, work email + phone | FullEnrich `fullenrich_start_enrichment` → poll |
| Large-scale cold prospecting (ICP filters, 200–50k leads) | `apify_leads_finder` |
| People or company search with full filters | Either `moltsets_search_people` / `moltsets_search_companies` (sync, up to 25 results) or `apify_leads_finder` (bulk, 200–50k results) |
| Company firmographic enrichment from domain | `moltsets_search_companies` (sync) or `fullenrich_search_company` |

**MoltSets strengths:** synchronous, no-polling, LinkedIn-to-email conversion, personal email, mobile phone, reverse lookups, ad audience identity.  
**FullEnrich strengths:** bulk async enrichment (up to 100 contacts per job), aggregator waterfall from 15+ providers, work email + personal email + phone in one job.  
**apify_leads_finder strengths:** large-scale ICP prospecting with fine-grained company/person filters across 250M+ contacts.

---

## moltsets_search_people — Input Guide

`query` is the full-text field — job title and role keywords belong here. Do not put country, seniority, or industry in `query`.

```json
{
  "query": "VP of Sales",
  "company_domain": "stripe.com",
  "limit": 10
}
```

By seniority and department:

```json
{
  "seniority": "C Suite",
  "department": "Marketing",
  "country": "United States",
  "limit": 25
}
```

Combined filters for ICP people search:

```json
{
  "query": "Head of Engineering",
  "industry": "Information Technology",
  "country": "United States",
  "seniority": "Director",
  "limit": 25
}
```

Field notes:
- `seniority` accepted values: `Intern`, `Entry`, `Senior`, `Manager`, `Director`, `VP`, `Head`, `C Suite`, `Owner`, `Partner`. Use `"C Suite"` (with space) for CEO/CTO/CFO.
- `company_domain` is more precise than `company` for known accounts.
- `limit` max is 25 per call; use `offset` for pagination.
- Results are sorted by `_score` (higher = stronger match).

---

## moltsets_search_companies — Input Guide

```json
{
  "domain": "stripe.com"
}
```

By industry and size:

```json
{
  "industry": "Computer Software",
  "employee_range": "201-500",
  "limit": 25
}
```

Field notes:
- `employee_range` accepted values: `1-10`, `11-20`, `21-50`, `51-200`, `201-500`, `501-1000`, `1001-5000`, `5001+`, `Small`, `Mid-Market`, `Enterprise`.
- `revenue_range` accepted values: `Below $500k`, `$500k - $1M`, `$1M - $5M`, `$5M - $10M`, `$10M - $20M`, `$20M - $50M`, `Above $50M`, and larger bands.
- Prefer `domain` over `query` when you have a known domain — it is an exact filter.
- `limit` max is 25; use `offset` for pagination.

---

## LinkedIn → Email Workflows

**Default — best available email:**

```json
{
  "linkedin_url": "https://linkedin.com/in/janedoe"
}
```

Call `moltsets_linkedin_to_best_email`. Returns `email` + `type` (`"business"` or `"personal"`). Use as the default endpoint unless email type is a hard requirement.

**B2B outbound — business email only:**

Call `moltsets_linkedin_to_business_email` when you must have a corporate address.

**Enrichment + email together:**

Call `moltsets_reverse_linkedin_lookup` when you also need title, seniority, company, and firmographics alongside the email. This is the most complete single-call enrichment endpoint.

---

## Reverse Lookup Patterns

**Email → profile:**

```json
{
  "email": "jane@stripe.com"
}
```

Call `moltsets_reverse_email_lookup`. Returns name, title, seniority, company URL, LinkedIn URL, firmographics, and any additional emails on file.

**LinkedIn URL → full profile + email:**

```json
{
  "linkedin_url": "https://linkedin.com/in/janedoe"
}
```

Call `moltsets_reverse_linkedin_lookup`. Returns full name, title, seniority, business email, personal email, company name, industry, revenue, and LinkedIn URL.

---

## Mobile Phone Lookup

Single profile:

```json
{
  "linkedin_url": "https://linkedin.com/in/janedoe"
}
```

Batch (up to 100):

```json
{
  "linkedin_urls": [
    "https://linkedin.com/in/janedoe",
    "https://linkedin.com/in/johndoe"
  ]
}
```

Call `moltsets_linkedin_to_mobile_phone`. Returns carrier-verified mobile numbers only — no landlines or VoIP. Costs 10 tokens per result returned. Only use when phone is a hard requirement.

---

## Ad Audience Identity

**Email → MAID (mobile ad ID):**

Use `moltsets_email_to_maid` to resolve a known email to AAID/IDFA for cross-device retargeting.

**Business email → SHA256 hashes for ad platforms:**

Use `moltsets_business_email_to_sha256` to get SHA256-hashed personal emails from a known business email. Upload hashes to Meta, Google, LinkedIn, TikTok, etc.

**LinkedIn → all hashes (business + personal):**

Use `moltsets_linkedin_to_sha256` to get SHA256 and MD5 hashes for all known emails on a LinkedIn profile.

---

## Swarm Request Patterns

**People search → email enrichment pipeline:**

```json
POST /swarm/deploy/
{
  "query": "Find and enrich VP-level Sales contacts at the following accounts: [company domains]. Use moltsets_search_people with company_domain and query='VP Sales' for each domain. For each match, call moltsets_linkedin_to_best_email on the linkedin_url field. Return per contact: full name, title, seniority, linkedin_url, email, email type.",
  "swarmSize": 3
}
```

**Name + domain → email (named account outreach):**

```json
POST /swarm/deploy/
{
  "query": "Find business emails for the following contacts: [name, domain list]. For each, call moltsets_search_business_email_by_name with name + company (domain). Return per contact: name, domain, email found (or blank if not found), linkedin_url if returned.",
  "swarmSize": 3
}
```

**Reverse enrichment from an email list:**

```json
POST /swarm/deploy/
{
  "query": "Enrich these email addresses using moltsets_reverse_email_lookup: [email list]. For each, return full name, job title, seniority, company, LinkedIn URL, and any firmographic data returned. Flag addresses where no data was found.",
  "swarmSize": 3
}
```

---

## Rules

- Check `moltsets_get_account` or `moltsets_get_billing` at the start of large runs to confirm credit balance.
- Most MoltSets tools are synchronous — no polling required. Call and read the response.
- `moltsets_linkedin_to_mobile_phone` costs 10 tokens per result — always get approval before running on a large list.
- `moltsets_search_people` and `moltsets_search_companies` return a max of 25 results per call — paginate with `offset` for larger sets. For large-scale ICP prospecting (200+ leads), use `apify_leads_finder` instead.
- `count_only: true` on `moltsets_search_linkedin_profile` is free — use it first to validate volume before committing tokens.
- Tokens are only charged when data is found — empty results cost nothing.
- Do not use MoltSets people search as a substitute for bulk enrichment. For 20+ contacts where you already have names + domains, `fullenrich_start_enrichment` is more cost-effective.

---

## Save to Context Lake

After any MoltSets enrichment run, save the output summary:

```json
POST /context/
{
  "serverId": "26",
  "content": "MoltSets enrichment complete. Task: {{people search / LinkedIn-to-email / reverse lookup}}. Contacts processed: {{count}}. Emails found: {{count}}. Phone numbers found: {{count if applicable}}. Output: {{file path or inline table}}.",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString()>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
