# Changelog

All notable changes to the sushidata-gtm plugin are documented here.

---

## [0.5.15] — 2026-08-06

### Changed

**`sushi-onboard` — removed free trial language and source name reference**

- HeyReach setup step 1 no longer mentions `heyreach.io` by name or references a free trial; updated to generic account creation language.
- `/context/` debug check now includes required `serverId: "26"` field in the request payload.

---

## [0.5.14] — 2026-08-05

### Added

**`sushi-debug` — live endpoint connectivity check skill**

New `/sushi-debug` skill that runs a 4-step live check against all Sushidata API endpoints and reports HTTP status codes and response validation:
- Check 1 `POST /context/` — verifies the write endpoint accepts data (expects HTTP 200/201)
- Check 2 `POST /query/` — verifies the read endpoint returns `summary` or `sources` (expects HTTP 200)
- Check 3 `POST /swarm/deploy/` — verifies a deploy returns at least one `doId` in the `workers` array
- Check 4 `POST /swarm/status/` — verifies the polling endpoint returns an `allDone` field (polls once, does not wait for completion)
- Renders a pass/fail results table with actual HTTP status codes and one-line response notes
- Includes a `⚠️ CAPABILITY NOTICE` section that explicitly prevents Claude from refusing to make API calls or claiming the endpoints are unreachable
- Distinguishes tool execution errors from HTTP errors to avoid false "Sushidata is down" conclusions

---

## [0.5.13] — 2026-07-22

### Added

**PredictLeads provider integration**

- Added `skills/sushi-research/provider-playbooks/predictleads.md` — full playbook covering all PredictLeads tools: company enrichment, buying signals (`news_events`, `financing_events`, `connections`), technology stack intelligence, hiring signals, cross-company discovery, portfolio companies, GitHub/SEC data, similar companies, and follow/unfollow webhook tracking.
- Updated `skills/sushi-research/SKILL.md`: added PredictLeads to provider routing, swarm tool reference, sub-doc routing table, and Provider Playbooks index.
- Updated `skills/sushi-research/provider-playbooks/enrichment-waterfall.md`: added PredictLeads to Company Tools table and signal-based discovery guidance.
- Updated `skills/sushi-research/finding-companies-and-contacts.md`: added `predictleads_discover_companies` and `predictleads_portfolio_companies` to the company search escalation order.
- Updated `README.md`: added PredictLeads to the Provider Playbooks table and requirements list.

---

## [0.5.12] — 2026-07-17

### Added

**`sushi-onboard` — end-to-end GTM onboarding skill**

New `/sushi-onboard` skill that walks a new user through the full Sushidata setup in a single guided session:
- ICP check: queries the context lake for a saved ICP; falls back to an elicitation form or LLM inference if none exists, and saves the result
- Use case selection: choose between finding prospects (10-lead sample swarm using all three lead finder actors), finding buying signals (full `sushi-signals` workflow), or running both in parallel
- Next-step routing: send to HeyReach, export as sheet, save to context lake, enrich further, or scale up
- HeyReach connection check: auto-detects if HeyReach MCP is connected; if not, walks the user through account creation → API key → MCP URL → Claude connector setup step by step
- CRM/tooling prompt: asks about HubSpot, Salesforce, Notion, Airtable, or Slack after HeyReach
- Handoff summary: closing recap with ICP, lead count, signal count, and CRM status; stays in session for follow-on requests

**`finding-companies-and-contacts.md` — lead finder actor prioritization**

- Restructured people search escalation order: `apify_leads_finder` is now #1 for ICP discovery, followed by `moltsets_search_people` and `fullenrich_search_people`
- Added "use all three actors together" directive and a swarm task template that names each actor explicitly, preventing workers from defaulting to Google search
- Separated lead discovery from row-level enrichment into distinct ordered sections

**`sushi-signals/SKILL.md` — named actor swarm tasks**

- Reddit signal worker task now explicitly names `apify_google_search_scraper` with `site:reddit.com/r/[subreddit]` query pattern
- News/press signal worker task now uses `apify_perplexity_ai_scraper` first (synthesized news) with `apify_google_search_scraper` as fallback — was "use web search" (unresolved, defaulted to Google)
- Updated Dependencies table to reflect both actors for news/press

---

## [0.5.11] — 2026-07-14

### Fixed

**Apify actor cleanup — removed unsupported tools, corrected actor IDs**

- Removed `apify_google_maps_contact_details` from `apify.md` and all routing tables — this actor is not registered in `apify.mts` and was never a valid tool.
- Corrected `apify_ycombinator_scraper` actor ID: `memo23/y-combinator-scraper` → `michael.g/y-combinator-scraper` (matches live source).
- Removed `apify_reddit_scraper` from `sushi-research` SKILL.md swarm capabilities list and `sushi-signals` dependencies — does not exist. Reddit queries should use `apify_google_search_scraper` with `site:reddit.com/r/[subreddit]`.
- Updated `sushi-research` ⚠️ CRITICAL actor list to reflect the full, accurate set of 16 supported Apify actors.

---

## [0.5.10] — 2026-07-14

### Changed

**`sushi-signals` — added Phase 1 signal config onboarding**

First-time invocation now runs a 4-step elicitation form (via `askQuestions`) before scanning:
1. Signal history — what's worked and what hasn't
2. Signal sources — LinkedIn, Reddit, news, job boards, communities
3. Intelligence goals — what to extract per signal (pain, timing, budget, competitors, etc.)
4. Signal classification — custom tiers (Motivated Buyer, Buying Signal, Qualification, Watch & Wait, Cold Lead)

Config is saved to the Sushidata context lake and reused on every subsequent run. Scheduled runs skip config entirely.

---

## [0.5.9] — 2026-07-14

### Changed

**Slash command cleanup — reduced to 6 focused skills**

Removed: `niche-signal-discovery`, `sushi-help`, `sushi-matrix`, `sushi-research-quickstart`, `sushi-sales-quickstart`, `sushi-sequence`, `sushi-verify`.

Renamed: `sushi-sales-onboarding` → `sushi-sales` (shorter trigger, same workflow).

### Added

**`sushi-signals` — LinkedIn buying signal detection**

New scheduled skill that scans LinkedIn posts and comments for ICP accounts showing job change, pain, or hiring signals. Runs Monday–Friday at 9 AM. Maps each prospect to their detected signal with a recommended outreach action. Integrates with `apify_linkedin_post_search`, `apify_leads_finder`, and the Sushidata context lake. Can add flagged prospects directly to a HeyReach campaign or draft outreach on demand.

---

## [0.5.8] — 2026-07-09

### Fixed

**Corrected `apify_leads_finder` field notes in `apify.md`**

Three schema mismatches caught against the live zod tool definition in `apify.mts`:

1. **`functionIncludes` / `functionExcludes`**: `"hr"` is not a valid enum value — corrected to `"human_resources"`. Added missing values: `information_technology`, `support`, `education`, `consulting`.
2. **`seniorityIncludes` / `seniorityExcludes`**: added missing `"intern"` enum value.
3. **`totalResults` cap**: removed incorrect "capped at 200 by Sushidata" note. Actual cap is `$5` per run in charge (`maxTotalChargeUsd`). Default is 1,000; maximum is 50,000. `customOffset` can be used to paginate across runs.

---

## [0.5.7] — 2026-07-09

### Added

**MoltSets provider integration and `apify_leads_finder` documentation**

- Added `provider-playbooks/moltsets.md` — comprehensive playbook for all MoltSets tools: people search, company search, LinkedIn-to-email conversions (best / business / personal), mobile phone lookup, reverse email and LinkedIn lookups, ad audience identity (MAID, SHA256), IP-to-company, and account management.
- Documented MoltSets decision matrix (when to use MoltSets vs FullEnrich vs `apify_leads_finder`).
- Added swarm request patterns and field notes for `moltsets_search_people`, `moltsets_search_companies`, and all LinkedIn-to-contact endpoints.
- Updated `enrichment-waterfall.md`: added MoltSets to Person/Contact Tools table and Company Tools table; updated Merge and Dedupe Logic and Prospecting tables to include MoltSets and `apify_leads_finder`.
- Updated `finding-companies-and-contacts.md`: added MoltSets and `apify_leads_finder` to the People search escalation order; added `moltsets_search_companies` to the Company search escalation order.
- Updated `sushi-research/SKILL.md`: added MoltSets to Level 3 provider list, Contact & Email Enrichment tool reference, sub-doc routing table, and Provider Playbooks index.

---

## [0.5.6] — 2026-07-07

### Changed

**Consolidated email discovery and enrichment exclusively to FullEnrich**

- Replaced all legacy email-tool references with FullEnrich equivalents across all skill files.
- Domain contact discovery now uses `fullenrich_search_people` throughout.
- Company enrichment now uses `fullenrich_search_company` throughout.
- Person/profile enrichment now uses `fullenrich_reverse_email` / `fullenrich_start_enrichment`.
- Email verification step replaced with FullEnrich confidence score review across all recipes and playbooks.
- Removed the deprecated email-tool playbook from `provider-playbooks/`.
- Updated `enrichment-waterfall.md` Agent 2 to reflect FullEnrich-based quality checking.
- Updated `plugin.json` description and keywords.

---

## [0.5.5] — 2026-07-06

### Changed

**Email discovery routes through FullEnrich**

- Email finder capability uses `fullenrich_start_enrichment` → poll `fullenrich_get_enrichment` throughout.
- Updated enrichment-waterfall.md, niche-signal-discovery SKILL.md, finding-companies-and-contacts.md, sushi-research SKILL.md, sushi-sales-onboarding SKILL.md, fullenrich.md, and all recipe files (account-orgchart, build-tam, linkedin-url-lookup, portfolio-prospecting).
- Cleaned up email discovery references in provider playbooks.

---

## [0.5.4] — 2026-06-24

### Fixed

**Playbook accuracy audit — removed tools, cost cap, and polling**

- Removed all references to deleted tools (`web-search`, `get_url_markdown`, `get_url_screenshot`, Browser Rendering) from `apify.md` and `massive.md`. Replaced with `massive_browser_render` or WebFetch as appropriate.
- Corrected Apify cost cap in `apify.md`: was documented as `$0.20` with a global `maxItems=200` — actual cap is `$5` per run with no global item limit.
- Expanded FullEnrich polling status table in `fullenrich.md` from 3 to 7 statuses: added `CANCELED`, `CREDITS_INSUFFICIENT`, `RATE_LIMIT`, and `UNKNOWN` with explicit stop-poll instructions for terminal error states.
- Fixed `ads-transparency.md` pagination pitfall: "Don't paginate unless asked" replaced with a clear note that pagination is not supported — `ads_transparency` has no pagination input parameter despite the response schema including `next_page_token`.
- Updated `massive.md` comparison table: removed `get_url_markdown` and `get_url_screenshot` rows; updated fallback guidance to WebFetch for standard public pages.

---

## [0.5.3] — 2026-06-20

### Added

**Auto-connect HeyReach reply alerts on connection**

- On the first HeyReach connection, the playbook now wires up the Sushidata webhook (`{BASE_URL}webhook/ingest`) automatically — before any campaign work — so users get an email alert the moment a prospect replies.
- Documented both the API-based auto-connect path (`heyreach_create_webhook` with reply event types) and a manual HeyReach UI fallback with the exact ingest URL to paste.
- Reply / message-received and connection-accepted events flow into the context lake and trigger tenant email notifications.

---

## [0.5.2] — 2026-06-19

### Changed

**Updated the Y Combinator Apify actor and playbook**

- Switched `apify_ycombinator_scraper` to the `memo23/y-combinator-scraper` actor (from `michael.g/y-combinator-scraper`).
- Documented the new schema in the Apify provider playbook: companies and jobs modes, `startUrls` URL auto-routing, filter fields (`mode`, `queries`, `batch`, `role`, `location`), and `scrapeFounderDetails` / `scrapeOpenJobs` enrichment.

---

## [0.5.1] — 2026-06-18

### Fixed

**Aligned provider playbooks with actual MCP tool schemas to prevent invalid payloads**

Audited every provider playbook against the live zod tool schemas and corrected field names, payload shapes, enum values, and costs that would otherwise cause Claude to send invalid requests:

- **FullEnrich** — `inputs` → `data`; `job_id` → `enrichment_id`; terminal status `completed` → `FINISHED`; reverse-email requires `name` + `data: [{ email }]`; people/company search use `{ value }` filter arrays (`current_position_titles`, `current_company_domains`, `names`, `industries`, `headcounts`).
- **LimaData** — `find_profiles` uses `full_name` + `company_domain`; `prospect_people_search_url` takes a `search_url` and returns people (it does not generate URLs from filters); `search_posts` paginates with `page` (there is no `limit`); corrected capability descriptions.
- **Datagma** — `job_change_detection` uses name + required `companyName` (no `data` field); forward phone uses `email` / social URL; reverse phone uses `number` (not `phone`).
- **AI ARK** — company filters must be nested under `account`; `reverse_people_lookup` uses a single `search` field; `mobile_phone_finder` uses `linkedin` / `name` (not `linkedin_url` / `full_name`).
- **Lusha** — `prospect_contacts` requires the `filters` wrapper with plural keys; `lookalike_contacts` / `lookalike_companies` require an `ids` array; `signals_contacts` uses `signalTypes` array with valid enums (`promotion`, `companyChange`, `allSignals`).
- **ContactOut** — `people_search` uses `job_title` / `company` / `location` arrays; corrected `email_enrich` cost.
- **Adyntel** — `search_facebook_ads` uses `country_code`; `search_google_ads` uses `company_domain`.
- **Browser Rendering** — screenshot `fullPage` / `omitBackground` nested under `screenshotOptions`; `waitUntil` is an array.
- **Dropleads** — corrected email-verifier cost (0.1 credits, not 1).
- **PDL** — corrected record-count claim (3B+ person records).
- **Tool reference** — fully-qualified `predictleads_discover_companies` / `theirstack_company_search`; added FullEnrich, Lusha company-enrich, and company-prospecting tools to the SKILL.md reference and the enrichment-waterfall tables.

---

## [0.5.0] — 2026-06-18

### Fixed

**Corrected `apify_leads_finder` field values based on live actor validation**

Four issues discovered through live testing:

1. **`emailStatusIncludes`**: removed invalid `"deliverable"` and `"catch_all"` values. Valid values are `"verified"` and `"unverified"` only.
2. **`companyIndustryIncludes`**: values must be title-cased (e.g. `"Computer Software"`, `"Information Technology & Services"`). Lowercase values return no results. Also removed `"saas"` which is not a valid actor enum.
3. **`annualRevenueIncludes`**: corrected enum values from `"50m_100m"`, `"100m_500m"`, `"gt_500m"` to `"50m_200m"`, `"200m_1b"`, `"gt_1b"`.
4. **`functionIncludes` `"it"`**: flagged as conflicting with the actor's schema. Added warning to avoid `"it"` and use `companyIndustryIncludes` for IT industry targeting instead.

Updated files:

- `sushi-research/provider-playbooks/apify.md` — all three example JSON blocks and the field notes section

---

## [0.4.9] — 2026-06-18

### Changed

**Updated `apify_leads_finder` actor ID**

Replaced old `code_crafter/leads-finder` actor reference with the correct `pipelinelabs/leads-finder-with-emails-apollo-lusha-zoominfo` actor and updated the capability description to match.

Updated files:

- `sushi-research/provider-playbooks/apify.md` — corrected actor ID and description in the capabilities table

---

## [0.4.8] — 2026-06-18

### Added

**New Apify capability: Google Maps Email Extractor (`apify_google_maps_contact_details`)**

Adds `lukaskrivka/google-maps-with-contact-details` as a new Apify-backed tool. Scrapes Google Maps places and extracts contact details from their websites: email addresses, phone numbers, and social media links. Supports keyword + location search, direct Maps URLs, star-rating and website filters, and an optional per-place employee leads enrichment add-on.

Updated files:

- `sushi-research/provider-playbooks/apify.md` — added capability table row, Capability Selection Guide row, and full Google Maps Contact Details workflow section with three example patterns (keyword search, direct URL, leads enrichment)

---

## [0.4.7] — 2026-06-09

### Changed

**Removed `/swarm/summary/` endpoint — synthesize directly from `/swarm/status/` outputs**

`/swarm/summary/` has been removed from the workflow. It was slow and added unnecessary latency. Claude now synthesizes results directly from the `output` fields collected on completed workers during the `/swarm/status/` polling loop — no extra API call required.

Updated files:

- `sushi-research/SKILL.md` — replaced section 5 (swarm/summary) with a direct synthesis step; updated polling rules and decision flow diagram
- `sushi-research-quickstart/SKILL.md` — updated post-poll step to synthesize from worker outputs
- `sushi-sales-quickstart/SKILL.md` — updated ICP prospecting recipe to synthesize from worker outputs
- `sushi-sales-onboarding/SKILL.md` — updated dependencies table and polling step
- `sushi-research/recipes/gtm-competitor-report.md` — removed swarm/summary call, replaced with synthesis step

---

## [0.2.1] — 2026-05-20

### Added

**New skills**

- `sushi-sales-quickstart` — Three hardcoded live-demo recipes: ICP prospecting (swarm → FullEnrich email waterfall → HeyReach offer), single-account deep dive (swarm → org chart → account brief), and competitor displacement (swarm → tier scoring → contacts for Tier 1). Uses Sushidata API endpoints throughout. Every recipe saves its request and results to the context lake.

- `sushi-research-quickstart` — Competitive intelligence entry point for new users. Given a company name, deploys a Sushidata research swarm to identify and profile at least three competitors, then delivers a structured battlecard. Designed to demonstrate Sushidata value in under five minutes.

- `niche-signal-discovery` — Sushidata-native niche signal discovery skill. Discovers differential signals (website content, job listings, tech stack, maturity markers) between Closed Won and Closed Lost accounts using Laplace-smoothed lift scores. Uses Sushidata swarm, WebSearch/WebFetch, and Apify MCP tools. Includes two pure-Python stdlib scripts (`analyze_signals.py`, `dedupe_utils.py`) and nine reference documents.

**New recipes (inside `sushi-research`)**

- `gtm-competitor-report.md` — End-to-end workflow for building a GTM competitor analysis document: context lake query, swarm research, LinkedIn Ads JSON parsing, channel-by-channel document writing, source verification loop (event sponsorship tiers, analyst citation types, job posting URLs, LinkedIn/X post URLs), structural review checklist, and context lake save. Derived directly from the ZeroFox GTM analysis session.

- `document-accuracy-review.md` — Verification workflow wrapping the `document-accuracy-review` skill with Sushidata context lake queries, WebFetch link verification, client-rendered page escalation (Claude in Chrome), link classification system (verified / paywalled / auth-gated / unverifiable / broken), factual claim verification by domain (analyst reports, personnel, events, metrics), and per-document-type error patterns.

- `scheduled-tasks.md` — Playbook for setting up recurring and one-time automated GTM tasks via Cowork's scheduler (cron and fire-at patterns).

**New job doc**

- `jobs/writing-outreach.md` — Sushidata outreach-writing workflow. Uses a Python loop with the Anthropic SDK (`claude-haiku-4-5-20251001`) for per-row copy generation. Covers one-shot emails, four-step sequences, ICP tier classification, multi-criteria lead scoring, and HeyReach activation. Includes context lake saves for research batch and scored results.

### Changed

**`sushi-research` SKILL.md**

- BASE URL updated to `https://dashboard.sushidata.ai/public/019dff6e-988f-71e2-8aa0-1be949e8421b/`
- Tenant updated to `Sushidata`, Dataspace set to `Sushidata Internal`
- Frontmatter description expanded to cover: community signals, campaign performance, competitor battlecards, GTM competitor reports, outreach copy, and document accuracy review
- Recipes routing table updated with two new entries: `gtm-competitor-report.md` and `document-accuracy-review.md`

**Context lake audit — all actionable docs**

- Added `POST /context/` save blocks to every recipe, playbook, quickstart, and job doc
- Each save is tailored to that workflow's output (domain lists, contact counts, campaign IDs, actor names, etc.)
- Pattern: save the user's request before starting, save results with evidence links after completing

**`niche-signal-discovery` reference docs**

- `keyword-catalog.md`: prompt examples use `Sushidata swarm:` equivalents
- `quality-gate.md`: enrichment timing guidance uses "WebFetch and Apify actor runs complete synchronously..."
- `pitfalls.md`: buffer flush note updated for Sushidata tooling

**`plugin.json`**

- Added `homepage`, `repository`, and `license` fields (required for validation)
- Version bumped to `0.2.1`

### Fixed

- Plugin ZIP structure: previously packed with a `sushidata-gtm/` wrapper directory, causing validation failure. Rebuilt so `.claude-plugin/plugin.json` and `skills/` are at the root of the ZIP.

---

## [0.1.0] — 2026-05-19

### Added

Initial plugin build for Sushidata GTM workflows.

**`sushi-research` skill** — Core research and GTM execution skill. Covers the full Sushidata API workflow: `/context/` (save), `/query/` (fast lookup), `/swarm/deploy/` + `/swarm/status/` + `/swarm/summary/` (parallel research), `/verify/` (link validation). Includes decision flow, context saving rules, and transparency guidelines.

**Provider playbooks**

- `fullenrich.md` — email enrichment, people search, and company search via FullEnrich
- `heyreach.md` — LinkedIn campaign insertion (≤50 contacts per call, list-first pattern)
- `hubspot.md` — contact/company/note upsert patterns, deal creation
- `apify.md` — curated Sushidata MCP actor tools exposed by `ApifyTools`

**Recipes**

- `account-orgchart.md` — Map decision makers and warm intro paths for a target account
- `build-tam.md` — Build a total addressable market list from ICP criteria
- `linkedin-url-lookup.md` — Resolve LinkedIn profile URLs from names and companies
- `portfolio-prospecting.md` — Find companies backed by a specific investor, then find contacts
- `small-business-prospecting.md` — Find local small businesses using Maps-style search

**Validation scripts**

- `validate-emails.py` — Flag rows where enriched email domain doesn't match company domain
- `validate-linkedin-names.py` — Validate scraped LinkedIn profile names against source names (handles accents, hyphenated names, 50+ nickname pairs, initials); includes eval mode against 52 fixture test cases

**`finding-companies-and-contacts.md`** — Phase doc for the companies-first discovery order, over-provisioning rule, and approval gate for paid actions.
