# sushidata-gtm

A Claude Cowork plugin that turns Claude into a Sushidata-native GTM agent. Every research session, prospect list, and competitive finding is saved to the Sushidata context lake so future sessions build on what was already learned — not from scratch.

## What this plugin does

Four overlapping use cases, all backed by the Sushidata context lake:

**GTM Prospecting & Outbound** — Discover ICP-fit companies with a Sushidata research swarm, find contacts via FullEnrich, verify emails, and activate campaigns in HeyReach. Sync everything to HubSpot. The full pipeline runs inside Cowork without switching tools.

**Competitor & Market Intelligence** — Research competitors channel by channel (SEO, paid, social, events, PR, hiring). Parse LinkedIn Ads JSON exports. Verify every claim and link before the document is shared. Save findings to the context lake for all future sessions.

**ICP Signal Discovery** — Upload Closed Won and Closed Lost domain lists. The `niche-signal-discovery` skill extracts multi-page website content and job listings, then computes Laplace-smoothed lift scores to surface what actually differentiates buyers from non-buyers.

**Community & Campaign Intelligence** — Ingest signals from Discord, Slack, forums, and campaign data into the context lake. Query them later to inform outreach, content, or product decisions.

---

## Skills

| Skill | Trigger |
|-------|---------|
| `sushi-research` | Any research, prospecting, enrichment, or GTM execution task |
| `sushi-sales-quickstart` | "Show me what Sushidata can do", "find me some leads", "quick demo" |
| `sushi-research-quickstart` | "Research my competitors", "get started", first use with no prior context |
| `niche-signal-discovery` | ICP signal analysis from won/lost domain lists |

---

## Recipes

Step-by-step playbooks inside `skills/sushi-research/recipes/`:

| Recipe | Use for |
|--------|---------|
| `gtm-competitor-report.md` | Full GTM competitor analysis — channels, ads, events, PR, hiring |
| `document-accuracy-review.md` | Verifying factual accuracy and link integrity in any research document |
| `account-orgchart.md` | Mapping decision makers and warm intro paths for a target account |
| `build-tam.md` | Building a total addressable market list from ICP criteria |
| `portfolio-prospecting.md` | Finding companies backed by a specific investor, then contacts |
| `linkedin-url-lookup.md` | Resolving LinkedIn profile URLs from names and companies |
| `scheduled-tasks.md` | Setting up recurring or one-time automated GTM and research tasks |
| `small-business-prospecting.md` | Finding local small businesses via Maps-style search |

---

## Provider Playbooks

Inside `skills/sushi-research/provider-playbooks/`:

| Playbook | Covers |
|----------|--------|
| `heyreach.md` | LinkedIn campaign insertion (≤50 contacts per batch) |
| `hubspot.md` | Contact, company, note, and deal upsert patterns |
| `apify.md` | Actor quality ranking, job listing scraping, LinkedIn data |
| `predictleads.md` | Company intelligence, buying signals, technology stack, hiring/funding events, follow webhooks |

---

## Sushidata API

All skills connect to:

```
BASE URL:  https://dashboard.sushidata.ai/public/<YOUR_PUBLIC_URL_ID>/
Tenant:    Sushidata
Dataspace: Sushidata Internal
```

Key endpoints:

| Endpoint | Purpose |
|----------|---------|
| `POST /context/` | Save any finding to the context lake |
| `POST /query/` | Fast lookup — always try before deploying a swarm |
| `POST /swarm/deploy/` | Parallel research agents (swarmSize 2–20) |
| `POST /swarm/status/` | Poll swarm progress |
| `POST /verify/` | Validate evidence links before presenting to the user |

---

## Validation Scripts

Two pure-Python (stdlib only) data quality scripts in `skills/sushi-research/scripts/`:

**`validate-emails.py`** — Flags rows where the enriched email domain doesn't match the company domain. Warns if mismatch rate exceeds 20%.

```bash
python3 skills/sushi-research/scripts/validate-emails.py enriched.csv \
  --email-col email --domain-col domain
```

**`validate-linkedin-names.py`** — Validates scraped LinkedIn profile names against source names. Handles accents, hyphenated names, 50+ nickname pairs, and initials. Eval mode available.

```bash
python3 skills/sushi-research/scripts/validate-linkedin-names.py enriched.csv \
  --source-first first_name --source-last last_name --profile-name-col profile_name
```

---

## ICP Signal Discovery Scripts

Two pure-Python scripts in `skills/niche-signal-discovery/scripts/`:

**`analyze_signals.py`** — Computes Laplace-smoothed lift scores across website content and job listing keywords. Takes a CSV with `website` and `jobs` columns (JSON format), outputs a ranked signal table with evidence snippets.

**`dedupe_utils.py`** — Deduplicates prospect lists against an existing customer/CRM list. Primary match on apex domain (public-suffix-aware, handles 80+ multi-label TLDs). Fallback on fuzzy company name (SequenceMatcher, 0.85 threshold).

---

## Assets

Drop screenshots, diagrams, and visual references in `assets/`. These are not bundled into the `.plugin` file — they're for your reference while editing skills.

---

## Packaging

To rebuild the `.plugin` file after editing source files:

```bash
cd /path/to/sushi-gtm
zip -r ../sushi-gtm.plugin . -x "*.DS_Store" -x "assets/*" -x "*.plugin"
```

The ZIP root must contain `.claude-plugin/plugin.json` and `skills/` directly — no wrapping folder.

---

## Requirements

- Claude Cowork (desktop app) with the plugin installed
- Sushidata account with access to the `Sushidata Internal` dataspace
- MCP connectors for the providers you want to use: HeyReach, HubSpot, Apify, FullEnrich, PredictLeads
