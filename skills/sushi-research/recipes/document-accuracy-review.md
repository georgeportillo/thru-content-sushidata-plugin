# Recipe: Document Accuracy Review

**Use when**: A research document (competitor report, GTM analysis, market overview, battlecard) needs to be verified for factual accuracy, link integrity, correct labeling, and citation quality before being shared with stakeholders.

**Output**: An audit report (usable-yield score + flagged issues) and an updated document with corrections applied.

**Complements**: The `document-accuracy-review` skill, which provides structured audit output. This recipe adds the Sushidata context lake query step, the link-chase loop, and the correction workflow.

---

## Before you start

Identify:
1. **The document to review** — file path or paste the content
2. **The subject domain** — what area (competitor GTM, market sizing, analyst landscape, etc.) so you can target the right verification queries
3. **What "accurate" means here** — factual claims only, or also link validity and label correctness?

Save the review request to the context lake:

```
POST /context/
{
  "serverId": "26",
  "content": "Document accuracy review requested for: [document title / path]. Domain: [subject area]. Focus: [factual / links / labels / all].",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Step 1 — Read the document

Read the full document before doing anything else. Do not skip to link verification without understanding the claims first.

For Markdown files:
```bash
cat [document-path].md | wc -l   # check length
```

Then Read the file. Note:
- All named facts (counts, percentages, dates, named events)
- All links (inline and reference-style)
- All analyst/report citations (Gartner, Forrester, IDC, etc.)
- All named job titles, roles, and personnel
- All event names, dates, and sponsorship tiers

---

## Step 2 — Query Sushidata for prior research

Before running the skill or doing WebSearch, check whether Sushidata's context lake already holds verified facts about the subject.

```
POST /query/
{ "query": "[subject of document] verified facts data sources" }
```

Cross-check the document's claims against the context lake response. Flag any discrepancy as a candidate correction.

---

## Step 3 — Run the document-accuracy-review skill

Invoke the `document-accuracy-review` skill on the document content. This produces a structured audit with:
- **Usable-yield score** (0–100): percentage of content that is accurate and usable as-is
- **Flagged issues per section**: factual errors, broken links, incorrect labels, language quality issues, consistency gaps

Work through each flagged issue in order of severity (factual errors first, then links, then language).

---

## Step 4 — Verify flagged links

For every link flagged by the skill or manually identified during Step 1, run the following verification chain:

### 4a. Direct WebFetch

```
WebFetch: [URL]
```

- If the page loads and contains the expected content: mark verified (✅)
- If the page returns a login wall, paywall, or empty shell: mark auth-gated or paywalled (⚠️)
- If the page 404s: the link is broken — find a replacement

### 4b. For client-rendered pages (empty WebFetch result)

WebFetch returns raw HTML without executing JavaScript. If the fetched page looks like a shell with no real content, the site is client-rendered. Use the Claude in Chrome tools if available:

```
mcp__Claude_in_Chrome__navigate → URL
mcp__Claude_in_Chrome__get_page_text
```

If Chrome is not available, search for a cached or alternative source.

### 4c. Finding replacement links

When a link is broken or missing:

```
WebSearch: "[organization] [claim or report title] [year] site:[authoritative-domain.com]"
WebSearch: "[claim keywords] site:globenewswire.com OR site:prnewswire.com"
```

**Analyst reports (Gartner, Forrester, IDC):**
- Search the analyst's own site for the exact document — note that document type matters (MarketScape ≠ Survey Spotlight ≠ Wave)
- If paywalled, link to the abstract page and classify as ⚠️ paywalled

**Job postings:**
```
WebSearch: "[company] [job title] site:apply.workable.com"
WebSearch: "[company] [job title] site:greenhouse.io"
WebSearch: "[company] [job title] site:linkedin.com/jobs"
```

**LinkedIn and X posts:**
```
WebSearch: "[company] [post topic] site:linkedin.com/posts"
WebSearch: "[company] [post topic] site:x.com"
```
Note: These will always remain ⚠️ (auth-gated) even if the URL is found.

**Event sponsorships:**
```
WebSearch: "[event name] [year] sponsors exhibitors site:[event-domain]"
WebFetch: [event sponsor page URL]
```
Always check the exact sponsorship tier — the document may have the wrong label (e.g., "Gold sponsor" vs. "Lanyard sponsor").

---

## Step 5 — Verify factual claims

For each named fact in the document:

### Analyst and report citations

Common errors to look for:
- **Wrong document type**: IDC Survey Spotlight labeled as IDC MarketScape; Forrester Wave labeled as Forrester Report
- **Wrong year**: citing a 2024 report as 2025
- **Wrong scope**: a regional report cited as global

Verification:
```
WebSearch: "[Report title] [Year] [Vendor] site:[analyst-domain].com"
WebFetch: [abstract or press release URL]
```

Correct the label in the document if wrong. If the document is paywalled, link the abstract and classify as ⚠️.

### Personnel and hires

For named executive hires:
```
WebSearch: "[company] [person name] [title] [year] announcement"
```

Look for a first-party PR or press release (globenewswire.com, prnewswire.com, or the company's own newsroom). If found, link the PR and mark verified (✅).

### Event attendance and booth details

For events and booth numbers:
- Booth numbers are often only available in pre-event exhibitor directories — these may 404 post-event
- If unverifiable, keep the claim but flag as ⚠️ and note the source (e.g., "sourced from Sushidata context lake")

### Metrics (follower counts, ad counts, headcount)

- Social follower counts change daily — do not WebFetch these unless Claude in Chrome is available
- Ad counts from JSON files are authoritative — do not override with swarm estimates
- Headcount figures from LinkedIn or company sites as of a specific date should be dated explicitly

---

## Step 6 — Apply corrections

For each confirmed error, edit the document:

1. **Wrong label** (e.g., "IDC MarketScape" → "IDC Survey Spotlight"): Update the text and the link
2. **Wrong sponsor tier** (e.g., "Networking Reception sponsor" → "Lanyard sponsor"): Update text and source link
3. **Broken link** → replace with the verified URL found in Step 4
4. **Upgraded link** (⚠️ → ✅): Update the classification symbol
5. **Unverifiable claim**: Add ⚠️ inline and a brief note on why (e.g., "⚠️ paywalled — abstract only")

Run a final check after editing:

```bash
# Confirm no em-dashes remain (if the document style prohibits them)
grep -n " -- " [document-path].md | wc -l

# Confirm no hedging language
grep -n "appears to\|seems to\|might be\|could be" [document-path].md
```

---

## Step 7 — Produce the audit summary

After corrections are applied, summarize the review:

```
## Accuracy Review Summary

**Document**: [title / path]
**Review date**: [date]
**Usable-yield score**: [N]/100

### Changes made
- [N] factual corrections (list the most significant)
- [N] links upgraded from ⚠️ to ✅
- [N] links confirmed ⚠️ (auth-gated or paywalled)
- [N] broken links replaced

### Remaining ⚠️ items
- [list each with reason]

### Items that could not be verified
- [list with explanation]
```

---

## Step 8 — Save to context lake

```
POST /context/
{
  "serverId": "26",
  "content": "Document accuracy review completed: [document title]. Usable-yield score: N/100. Corrections: N factual, N links upgraded, N paywalled confirmed. Key corrections: [2-3 most significant]. Document path: [path].",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

---

## Link classification reference

| Symbol | Meaning | Action |
|--------|---------|--------|
| ✅ | Publicly accessible, content verified via WebFetch | No change needed |
| ⚠️ paywalled | Abstract visible, full content requires payment | Link abstract page, note price/access method |
| ⚠️ auth-gated | Requires account login (LinkedIn, X, Gartner peer insights) | Include URL if findable, note auth requirement |
| ⚠️ unverifiable | Claim sourced from context lake or swarm without a primary source | Keep claim, add note on source |
| (broken) | URL 404s or redirects to unrelated content | Replace or remove |

---

## Common errors by document type

### Competitor GTM reports

- **LinkedIn follower counts**: often stale from swarm research — verify against direct observation or leave undated
- **Ad format descriptions**: only assert what the raw JSON confirms
- **Event sponsorship tiers**: always check the event's own sponsor press release, not just what was reported
- **Analyst report types**: verify document type on the analyst's own site before including

### Market sizing / TAM reports

- **Source year**: TAM figures age quickly — always include the report year
- **Definition scope**: "global TAM" vs. "US TAM" vs. "serviceable market" are different — be precise

### Battlecards

- **Competitor pricing**: changes frequently — date all pricing claims explicitly
- **Feature comparisons**: only include features you can verify via the competitor's own docs or marketing pages
