# Recipe: GTM Competitor Report

**Use when**: Building a deep GTM analysis of a competitor — covering their SEO, social, paid media, events, PR, partnerships, and hiring signals. Produces a structured intelligence document the sales and marketing team can act on.

**Output**: A Markdown document with channel-by-channel GTM breakdown, verified source links, and strategic signals for the target company.

**Estimated time**: 30–60 minutes depending on swarm depth and link verification volume.

---

## Before you start

Ask the user for:

1. **Competitor name + domain** (e.g., "ZeroFox / zerofox.com")
2. **Your company name** — so "what this means for us" framing is accurate
3. **Any data already gathered** — LinkedIn Ads JSON export, prior reports, uploaded files
4. **Template to follow** — if the user has an existing competitor report they want to match structurally, ask them to share it

Save the request to the context lake before doing anything else:

```
POST /context/
{
  "serverId": "26",
  "content": "GTM competitor report requested for: [competitor] vs [our company]. Scope: [channels / depth].",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Step 1 — Query context lake for existing intelligence

Before deploying a swarm, check if Sushidata already has usable data on the competitor.

```
POST /query/
{ "query": "[competitor name] GTM strategy marketing channels funding hiring" }
```

If the response is comprehensive (covers 3+ channels with specific data), proceed to Step 3 using context lake data.
If sparse or outdated, proceed to Step 2.

---

## Step 2 — Deploy research swarm

```
POST /swarm/deploy/
{
  "query": "Comprehensive GTM analysis of [competitor name] ([domain]): SEO and content strategy, LinkedIn and social presence (follower count, posting cadence, ad campaigns), paid media (Google Ads, display, LinkedIn Sponsored Content), events and sponsorships, analyst coverage (Gartner, Forrester, IDC), PR cadence and media placements, partner and channel program, hiring signals and open roles. Focus on what this reveals about their go-to-market motion and how it compares to [your company].",
  "swarmSize": 10
}
```

Poll `/swarm/status/` every 30 seconds. Show the user the worker list and progress (e.g., "5 / 10 workers done...").

Once `allDone: true` (or 5-minute timeout), synthesize results directly from the `output` fields of completed workers collected during polling. Combine findings across workers, dedupe, and proceed to Step 3.

---

## Step 3 — Parse any uploaded data files

If the user has provided raw data (LinkedIn Ads JSON, traffic exports, event exhibitor lists), process them before writing.

### LinkedIn Ads JSON

LinkedIn's Ad Library can be pulled through Sushidata's `apify_linkedin_ad_library_scraper` MCP tool when the user provides prepared LinkedIn Ad Library search URLs. If the user provides an existing JSON export:

```bash
python3 -c "
import json, sys
from collections import Counter
data = json.load(open(sys.argv[1]))
print('Total ads:', len(data))
formats = Counter(ad.get('format', 'unknown') for ad in data)
print('Formats:', dict(formats))
" dataset_linkedin-ad-library-scraper_*.json
```

Then identify distinct campaigns by clustering ads around shared copy themes, images, or UTM patterns. For each campaign, extract:

- Campaign theme (product launch, analyst validation, thought leadership, etc.)
- Copy excerpt (first 80 characters of headline/body)
- Ad format (Single Image, Document, Video, InMail)
- Direct ad library URL if present in the JSON

**InMail ads** are especially high-signal — extract the full copy, sender name, and any personalization tokens (e.g., `%COMPANYNAME%`, win-back offers, gift card incentives).

### Other data files

Read and summarize any other uploaded files (traffic reports, event exhibitor PDFs, hiring data) before writing.

---

## Step 4 — Write the document

Create the output file at `output/[competitor-slug]-gtm-analysis-[month][year].md`.

**Document structure** (adapt to match a user-provided template if given):

```
# [Competitor] GTM Analysis — [Month Year]

## How to use this doc
## Competitor profile & positioning
### Company overview
### Funding & ownership
### Product positioning

# Channel Breakdown

## SEO & Content
### Blog / resource center
### Intelligence tools & resources
### Gated content

## Social Media
### LinkedIn
#### Executive visibility
### X / Twitter

## Paid Media
### LinkedIn Ads
### Google Ads

## Events & Sponsorships
### [Event 1]
### [Event 2]

## PR & Analyst Coverage
### Analyst placement
### PR cadence & syndication

## Partner & Channel
### [Partner type 1]
### [Partner type 2]

## Hiring Signals
### Open roles
### Recent GTM hires
### Strategic signal
```

**Writing rules:**
- No em-dashes — use commas, colons, or restructure the sentence
- No hedging constructions ("it seems," "appears to," "might be")
- No AI language patterns ("delve," "leverage," "robust")
- No traffic volume figures from third-party tools (Semrush, SimilarWeb) — these are inaccurate
- No ad format conclusions from scrapers without raw data backing
- Every factual claim needs either a context lake source, swarm-returned URL, or a `(?)` flag

**Link classification system:**

| Symbol | Meaning |
|--------|---------|
| (verified) | Publicly accessible, verified via WebFetch |
| (?) | Paywalled, auth-gated (LinkedIn, X), or unverifiable — include URL if discoverable |

---

## Step 5 — Source verification loop

After drafting, work through every `(?)` item and attempt to find or upgrade it.

### For event sponsorships

Check the official event website for exhibitor or sponsor lists. Sponsorship type often differs from what is reported (e.g., "Lanyard sponsor" vs. "Networking Reception sponsor" — get the exact tier from the event's own press release or sponsor page).

```
WebSearch: "[Event name] [year] sponsor exhibitor site:[event-domain.com]"
```

### For analyst citations (Gartner, Forrester, IDC)

Check the analyst's site directly for the exact document title and type. Common errors to catch:
- Confusing IDC Survey Spotlight with IDC MarketScape (different products entirely)
- Calling a Forrester Wave a Forrester Report
- Wrong year on recurring reports

```
WebSearch: "[Competitor] Gartner Magic Quadrant [year] site:gartner.com OR site:globenewswire.com"
WebSearch: "IDC [report title] [US-document-number] site:my.idc.com"
```

If the document is paywalled, link to the abstract page (still useful, keep `(?)`).

### For job postings

Search Workable (most tech companies use it), LinkedIn, and Greenhouse:

```
WebSearch: "[Company name] [job title] site:apply.workable.com"
WebSearch: "[Company name] [job title] site:linkedin.com/jobs"
```

If the posting is live and fetchable, link it as verified. If not findable, leave as text with no link.

### For LinkedIn and X posts

LinkedIn and X posts require authentication — WebFetch will return an empty page. These stay as `(?)`. However, find the post URL via Google:

```
WebSearch: "[Company name] [post topic keyword] site:linkedin.com/posts"
WebSearch: "[Company name] [post topic keyword] site:x.com"
```

Include the URL even though it's auth-gated — it's navigable for the user.

### For PR mentions (news wires, trade press)

GlobeNewswire, PRNewswire, and BusinessWire are fully public. BankInfoSecurity, Dark Reading, SC Media are also publicly crawlable. Search and verify with WebFetch. If accessible, upgrade to verified.

---

## Step 6 — Structural review

Before sharing, verify:

- [ ] Heading hierarchy is consistent (H1 / H2 / H3, no skipped levels)
- [ ] No em-dashes in the document (`grep " -- " file.md | wc -l` should be 0)
- [ ] All `(?)` items have been chased and either upgraded or confirmed unresolvable
- [ ] LinkedIn follower counts, ad counts, and other numbers match the raw data (not swarm estimates)
- [ ] Analyst product types are correct (MarketScape ≠ Survey Spotlight ≠ Wave)

---

## Step 7 — Save to context lake

Save the completed document summary and key findings:

```
POST /context/
{
  "serverId": "26",
  "content": "GTM competitor report completed: [competitor] vs [our company]. Key findings: [3-5 bullet summary of top signals]. Document: [output path]. Total ads analyzed: N. Verified links: N. Paywalled: N.",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Common pitfalls

**Wrong analyst document type** — Gartner Magic Quadrant, Forrester Wave, IDC MarketScape, and IDC Survey Spotlight are distinct products. Never substitute one for another. Verify on the analyst's own site.

**Stale swarm data on social follower counts** — Swarm agents often report rounded or cached numbers. Always cross-check against a direct page visit or trust the user's direct observation over swarm estimates.

**LinkedIn Ads section without real data** — Do not write ad campaign descriptions from swarm inference alone. Either parse the user's uploaded JSON, or flag the section as `(?) unverified` and note that the LinkedIn Ad Library requires authentication.

**Conflating job title with role signal** — "Hiring a Marketing Operations Analyst" is not the same signal as "Hiring 12 AEs." Read the role level and quantity before drawing GTM conclusions.

**Event booth number sourced from context lake only** — Flag booth numbers that cannot be confirmed on the official event exhibitor directory as `(?) sourced from Sushidata context lake`.
