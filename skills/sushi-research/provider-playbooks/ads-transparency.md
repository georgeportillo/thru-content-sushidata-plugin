# Paid Ad Intelligence Playbook

Use this playbook when a Sushidata research or GTM task needs competitor paid ad intelligence — creative formats, longevity, recency, and messaging patterns — across **Google, Facebook/Meta, and LinkedIn**.

Two tools are available:
- **`ads_transparency`** — Google Ads Transparency Center (Google Search/Display Network, swarm-based)
- **Adyntel** (`search_facebook_ads`, `search_google_ads`, `search_linkedin_ads`) — Direct Ad Library queries across all three channels; faster, returns more detail per creative

Use Adyntel first for most ad intelligence tasks. Fall back to `ads_transparency` for deep Google creative history.

---

## Adyntel — Direct Multi-Channel Ad Research

Adyntel provides direct access to Facebook (Meta), Google, and LinkedIn ad libraries. These tools are called directly (no swarm required for single lookups).

| Tool name | Channel | Input |
| --- | --- | --- |
| `search_facebook_ads` | Facebook / Meta Ad Library | `keyword` (search term), `country_code` (2-letter, e.g. `"US"`) |
| `search_google_ads` | Google Ads Transparency | `company_domain` (e.g. `"hubspot.com"`) |
| `search_linkedin_ads` | LinkedIn Ad Library | `company_domain` OR `linkedin_page_id` |

### Adyntel: Facebook Ads
- Searches the Meta Ad Library by keyword + country
- Returns ad creatives, copy, format, and active status
- Use when you need to see what messaging a brand is running on Meta

### Adyntel: Google Ads
- Direct query to Google Ads Transparency by `company_domain`
- Faster alternative to `ads_transparency` for quick lookups

### Adyntel: LinkedIn Ads
- Returns sponsored content creatives, commentary, images, and carousel details
- Provide `company_domain` OR `linkedin_page_id` (numeric, e.g. `"15564"`)
- Use `continuation_token` from prior response to paginate

### Multi-channel competitive sweep (Adyntel)

To get a full paid media picture of a competitor, run all three Adyntel calls in parallel:

```json
POST /swarm/deploy/
{
  "query": "Run a full paid ad sweep for [company]. In parallel: (1) search_google_ads with company_domain = '[domain]', (2) search_linkedin_ads with company_domain = '[domain]', (3) search_facebook_ads with keyword = '[brand name]' and country_code = 'US'. For each channel: count of active creatives, recurring themes and messaging angles, format types used, and any evergreen (long-running) ads. Return a cross-channel comparison: where is this company most active? What messages are they repeating across channels? What CTAs recur?",
  "swarmSize": 3
}
```

---

## How Ads Transparency Works Inside Sushidata (Google Deep History)

Sushidata's `chainfuse-api` MCP server exposes Google Ads Transparency through `AdsTransparencyTool`.

| Capability | Sushidata tool name | Use for |
|---|---|---|
| Google ad creative search (deep history) | `ads_transparency` | Find ad creatives associated with a domain from Google's Ads Transparency Center |

**Input:** `domain` — a valid domain string (e.g. `"salesforce.com"`, `"hubspot.com"`). No protocol, no path, no `www.` prefix.

**Output fields per creative:**
- `position` — ranking position in results
- `id` — Google creative ID
- `target_domain` — the domain the ad targets
- `advertiser.name` / `advertiser.id` — the registered advertiser
- `first_shown_datetime` / `last_shown_datetime` — ISO timestamps of ad activity window
- `total_days_shown` — total number of days the creative has run
- `format` — ad format code: `"1"` = Image, `"2"` = Video, `"3"` = HTML/Responsive
- `img_src` — image URL if available
- `details_link` — preview URL if available

**Important constraints:**
- One call per domain — do not batch multiple domains in a single tool call
- Returns up to 100 creatives per call (first page)
- Google Search and Display Network only — LinkedIn Ads, Meta Ads, and DSP placements are not included
- `img_src` and `details_link` may be null for some creatives — this is normal
- `search_information.total_results` reflects Google's total index count, not just the returned page

---

## Agent Behavior

Claude should infer what the user needs, then run the appropriate ad intelligence tools.

**For a quick ad lookup on a single competitor**: call `search_linkedin_ads`, `search_google_ads`, and `search_facebook_ads` directly (or via a swarm) — no need for the `ads_transparency` deep-history tool for a fast lookup.

**For deep Google creative history** (precise longevity, `total_days_shown`, complete output fields): use a swarm with `ads_transparency`.

**For multi-competitor comparison**: deploy a swarm with all three Adyntel tools in parallel across all domains.

Do not improvise unavailable capabilities. If the capability is missing entirely, name it and help the user send feedback to `support@sushidata.com`.

---

## Swarm Request Pattern

When Ads Transparency data is needed, deploy a Sushidata swarm with a concrete task description. Include:

- target domains to search
- what intelligence to extract (creative count, longevity, format mix, messaging themes)
- how to classify paid maturity
- output schema

**Example — single competitor:**

```json
POST /swarm/deploy/
{
  "query": "Use the ads_transparency tool to research [competitor_domain]'s Google ad activity. For each creative returned: note the format (1=image, 2=video, 3=HTML), first_shown and last_shown dates, total_days_shown, and img_src or details_link if available. Then produce: (1) total creatives on record, (2) count of active creatives with last_shown within the last 30 days, (3) format mix breakdown, (4) list of evergreen creatives with total_days_shown > 90 and their messaging if img_src is available, (5) paid maturity classification — Active & Scaled (50+ creatives, recent, mixed formats) / Active & Early (<20, recent, mostly image) / Paused (last_shown >90 days ago) / Minimal (<5 results). Save findings to context lake.",
  "swarmSize": 3
}
```

**Example — multi-competitor comparison:**

```json
POST /swarm/deploy/
{
  "query": "Use ads_transparency to research paid ad activity for these competitors: [domain_a], [domain_b], [domain_c]. For each: total creatives, active count (last 30 days), format mix, longest-running creative, and paid maturity classification. Then compare: which competitor has the most sustained paid investment? Which has the freshest creative? Which appears to have paused? Return a structured comparison table plus a summary of messaging themes where img_src data is available.",
  "swarmSize": 5
}
```

---

## How to read the output

### Volume signal
`search_information.total_results` indicates sustained paid investment. 100+ creatives = active, scaled program. <10 = early stage, paused, or primarily other channels.

### Longevity signal
`total_days_shown > 90` = evergreen creative. Companies don't run underperforming ads for 90+ days. These creatives contain their most validated messaging — note format, CTA language, and landing page offer type.

### Recency signal
`last_shown_datetime` within 30 days = currently active. More than 90 days ago = likely paused or shifted budget.

### Format mix
Count by format code. Mixed `"1"` + `"2"` signals mature paid program. Only `"1"` = performance-only or early stage. `"3"` = Display Network / responsive investment.

### Paid maturity classification

| Classification | Signal |
|---|---|
| Active & Scaled | 50+ total, recent last_shown, mixed formats |
| Active & Early | <20 total, recent last_shown, mostly image |
| Paused or winding down | last_shown >90 days ago |
| Minimal paid presence | <5 total results |

---

## GTM competitor report integration

When building a GTM competitor report (see `recipes/gtm-competitor-report.md`), the Paid Advertising section should be populated from this playbook. Use the swarm output to populate:

- Total creatives on record and active count
- Format mix
- Evergreen creative descriptions and messaging themes
- Paid maturity classification
- Notable shifts in format or recency vs. prior research

If `total_results` is 0, note it explicitly: *"No Google ad creatives on record — company may rely on organic, LinkedIn Ads (see apify.md), or other paid channels not visible in Google Ads Transparency."*

---

## Common pitfalls

- **Don't include protocol or path in domain** — `"hubspot.com"` not `"https://www.hubspot.com/products"`
- **Don't treat 0 results as no paid activity** — the company may be active on LinkedIn Ads, Meta, or programmatic channels
- **Don't skip the maturity classification** — raw creative counts without context aren't useful in a report
- **Don't claim an ad is "active" without checking last_shown_datetime** — total_results can be high even for a company that paused paid 6 months ago
- **Pagination is not supported** — `ads_transparency` returns up to 100 creatives per call with no way to fetch subsequent pages. The response may include a `next_page_token` field but there is no corresponding input parameter — additional pages cannot be requested
