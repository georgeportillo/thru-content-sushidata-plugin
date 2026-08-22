# Account Org Chart Builder

Build an interactive HTML org chart for account mapping — find decision makers, map reporting structures, and identify warm intro paths.

## When to use

- User asks to "map an account" or "build an org chart"
- User wants to find "who reports to X" or "decision makers at Y"
- User has a warm connection and wants to find paths through the org

## Inputs

One of:
- LinkedIn URL (+ optional company/domain)
- Name + company name or domain
- Just a company domain (maps the full GTM org)

If only a name is given with no company context, ask before proceeding.

## Quick reference

| Step | What                        | Source                                              | Cost    |
| ---- | --------------------------- | --------------------------------------------------- | ------- |
| 1    | Resolve target identity     | WebSearch, Browser Rendering, Sushidata swarm, or `fullenrich_start_enrichment` for known names | Low     |
| 2a   | LinkedIn company enrichment | `apify_linkedin_company_scraper` when a company URL is known | Low     |
| 2b   | FullEnrich people search    | `fullenrich_search_people`                              | Low     |
| 2c   | Sushidata swarm (gap fill)  | Research swarm for execs and key roles              | Free    |
| 3    | Classify + infer hierarchy  | Title-based seniority + tenure signals              | 0       |
| 4    | Generate HTML org chart     | Write output file                                   | 0       |

**Cost-optimal order:** WebSearch / Browser Rendering for public identity resolution, then FullEnrich for email discovery, then Sushidata swarm for executive gap-fill. Use `apify_linkedin_company_scraper` for company profile enrichment, not employee scraping.

## Pipeline

### 1. Resolve target identity

**If LinkedIn profile URL given:** Sushidata does not currently expose a LinkedIn profile scraper. Use the URL as a candidate identity signal, then validate the person's name, company, and title with WebSearch, Browser Rendering, or a focused Sushidata swarm. If profile scraping is required, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md).

Then resolve the company domain if missing — use WebSearch for "Company Name official website".

**If name + company given:** Resolve the LinkedIn URL first using the `linkedin-url-lookup` recipe, then scrape the profile.

### 2. Find relevant employees (cost-optimal waterfall)

**Source 1: LinkedIn company enrichment**

First resolve the LinkedIn company slug via WebSearch: `"COMPANY_NAME LinkedIn company page site:linkedin.com/company"`

Then call `apify_linkedin_company_scraper`:
```json
{
  "profileUrls": ["https://www.linkedin.com/company/SLUG/"]
}
```

Use returned company profile data as context for the org chart. Sushidata does not currently expose a LinkedIn employee-list scraper. For employee discovery, use WebSearch, Browser Rendering, Sushidata swarms, and FullEnrich people search. If bulk employee scraping is required, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md).

**Source 2: FullEnrich people search (low cost)**

Call `fullenrich_search_people` with the company domain. The Sushidata schema exposes only `domain`; do not pass department or seniority filters. Use returned titles, seniority, department, and email metadata to identify GTM roles (Sales, Marketing, RevOps, CS).

```json
{
  "domain": "example.com"
}
```

**Source 3: Sushidata swarm (executive gap-fill, free)**

After the first two sources, identify gaps (e.g., "5 Sales Directors but no VP Sales"). Deploy a targeted swarm:

```
POST /swarm/deploy/
{
  "query": "Who is the VP of Sales / Head of Sales at [Company]? Find their name, LinkedIn, and any public contact info.",
  "swarmSize": 3
}
```

Merge all results, deduplicate by slugified name.

### 3. Classify seniority + infer hierarchy

**Seniority classification (check in order, first match wins):**

| Rank | Level       | Patterns                                  |
| ---- | ----------- | ----------------------------------------- |
| 0    | ceo         | "ceo", "chief executive"                  |
| 1    | c-level     | "chief" + any (cto, cfo, coo, cmo, cro)   |
| 2    | evp         | "evp", "executive vice president"         |
| 3    | svp         | "svp", "senior vice president"            |
| 4    | vp          | "vice president", "vp"                    |
| 5    | sr-director | "senior director"                         |
| 6    | director    | "head of", "director"                     |
| 7    | sr-manager  | "senior manager"                          |
| 8    | manager     | "manager"                                 |
| 9    | lead        | "lead"                                    |
| 10   | senior      | "senior", "sr."                           |
| 11   | ic          | everything else                           |

**For display, simplify to 4 levels:**
- **exec**: ceo, c-level, evp, svp, vp → color `#a78bfa`
- **director**: sr-director, director → color `#60a5fa`
- **manager**: sr-manager, manager, lead → color `#34d399`
- **ic**: senior, ic → color `#525252`

**Team consolidation (if >8 teams):**

| Original                            | Simplified        |
| ----------------------------------- | ----------------- |
| Executive Leadership, Office of CEO | Leadership        |
| Sales, GTM, Business Development    | Sales             |
| Revenue Operations, Sales Operations | Revenue Ops      |
| Marketing, Demand Gen               | Marketing         |
| Customer Success, Support           | Customer Success  |
| AI, Product, Engineering, Data      | Product & Eng     |
| HR, People, Finance, Legal          | Other             |

### 4. Generate HTML org chart

**Design system:**
- Dark mode: `--bg: #0a0a0a`, `--surface: #141414`, `--border: #262626`
- Text: `--text: #e5e5e5`, `--text-muted: #a3a3a3`
- Inter font only

**Required UX elements:**
- **Priority score column** (0–100, see scoring below)
- **Team sidebar** (left): collapsed teams with counts, click to filter
- **Table layout**: sortable by priority — Priority | Name/Title | Team | Level | Contact
- **Empty state**: "No contacts match — Clear all filters" button
- **Keyboard**: `/` to focus search, `Esc` to clear filters
- `🆕` badge for anyone with tenure < 90 days

Save to `output/orgchart/{company}-{date}.html` and present as a file link.

## Priority scoring

```
score = title_score + contact_score + recency_bonus
```

| Factor         | Condition                                    | Score |
| -------------- | -------------------------------------------- | ----- |
| Title          | exec level                                   | +40   |
| Title          | director level                               | +25   |
| Title          | manager level                                | +10   |
| Contact        | has email                                    | +25   |
| Contact        | has phone                                    | +10   |
| Recency        | exec, tenure < 90 days (newly hired)         | +20   |
| Recency        | any level, tenure < 30 days                  | +10   |

Newly-hired execs get a +20 bonus because they are actively remapping their vendor stack in the first 90 days — highest buying authority, lowest incumbent loyalty.

## When NOT to use

- User already has a CSV and wants enrichment → go directly to provider tools
- User needs email/phone for outbound without the org chart → use `build-tam.md` or `linkedin-url-lookup.md` instead

---

## Save to Context Lake

Save the org chart after building it. AEs and SDRs returning to this account in a future session can retrieve the decision-maker map without re-scraping.

```json
POST /context/
{
  "serverId": "26",
  "content": "Org chart complete — {{company name}} ({{domain}}). Decision makers found: {{count}}. Key contacts: {{names, titles, emails of top 3-5}}. Economic buyer identified: {{name, title}}. Champion candidate: {{name, title}}. Warm intro paths: {{count}}. Output: {{HTML/CSV path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
