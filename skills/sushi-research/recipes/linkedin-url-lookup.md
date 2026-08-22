---
name: linkedin-url-lookup
description: "Resolve LinkedIn profile URLs from name + company with strict identity validation to avoid false positives."
---

# LinkedIn URL Lookup

Find LinkedIn profile URLs when you have a name, with or without company context.

## When to use

- "Find LinkedIn URLs for the contacts in my CSV"
- "Resolve LinkedIn profiles from names and companies"
- "I only have names — find their LinkedIn profiles"
- "Verify these LinkedIn URLs match the right people"

## Provider sequence

Follow this order. Stop when you get a validated match.

### Step 1: WebSearch (free)

Search Google scoped to LinkedIn:

```
"Jane Smith" "Acme Corp" site:linkedin.com/in
```

Variations:
- Name + company (highest confidence): `"Jane Smith" "Acme Corp" site:linkedin.com/in`
- Name only: `"Jane Smith" site:linkedin.com/in`
- Name + title: `"Jane Smith" "VP Sales" site:linkedin.com/in`

Parse the LinkedIn URL from the top result. Skip results that aren't `linkedin.com/in/` URLs.

### Step 2: Profile validation

After finding a candidate URL via WebSearch, validate the profile with Browser Rendering, WebSearch snippets, or a focused Sushidata swarm. Sushidata does not currently expose a LinkedIn profile scraping Apify tool.

**Name-validate** the visible profile name or search-result name against the source name (see Post-lookup validation below). Company and title are supporting signals only.

If validation fails, try the next WebSearch result. If all results fail, move to Step 3.

### Step 3: Sushidata swarm (fallback for ambiguous names)

When WebSearch returns too many ambiguous results:

```
POST /swarm/deploy/
{
  "query": "Find the LinkedIn profile URL for [Name] who works at [Company] as [Title]. Return only verified linkedin.com/in/ URLs.",
  "swarmSize": 3
}
```

Verify any URLs returned by the swarm using the validation process in Step 2.

### Step 4: FullEnrich email finder (paid, email only)

If previous steps fail and you have a strong identity seed, FullEnrich can find an email for a named person, but email lookup does not return LinkedIn URLs directly.

```json
{
  "first_name": "Jane",
  "last_name": "Smith",
  "domain": "acmecorp.com"
}
```

Use `fullenrich_start_enrichment` only when email discovery is also useful. Do not use email finders as the primary path for LinkedIn URL resolution.

## Scenarios

### Name only

1. WebSearch: `"Jane Smith" site:linkedin.com/in` → validate the candidate profile
2. Add geography if too many results: `"Jane Smith" "New York" site:linkedin.com/in`
3. Sushidata swarm for disambiguation
4. Still ambiguous? Ask the user for more context before spending credits

### Name + company

1. WebSearch: `"Jane Smith" "Acme Corp" site:linkedin.com/in` → validate the candidate profile
2. Sushidata swarm if WebSearch misses
3. If email is also needed, use `fullenrich_start_enrichment` with company domain (or linkedin_url), then poll `fullenrich_get_enrichment`

### Name only (event attendees, RSVP lists)

Add event/role keywords to disambiguate:

```
"Jane Smith" (RevOps OR "Sales Operations" OR GTM OR Sales) site:linkedin.com/in
```

### Nickname handling

Common variants: Mike/Michael, Bob/Robert, Bill/William, Liz/Elizabeth, Alex/Alexander, Dan/Daniel.

WebSearch handles this well with OR queries:
```
("Mike" OR "Michael") "Smith" "Acme" site:linkedin.com/in
```

## Post-lookup name validation (mandatory)

After scraping, compare profile name to source name. **Null out any URL where first + last don't match.** Without this gate, ~26% of lookups return the wrong person.

Rules:
- **Last name**: exact or substring (handles hyphenated names)
- **First name**: exact, 3+ character prefix, or common nickname
- Normalize accents (`Rodríguez` → `Rodriguez`) and strip punctuation before comparing

## Key rules

- WebSearch first — free, fast, usually sufficient.
- Validate every URL — name match is mandatory.
- Sushidata swarm for disambiguation — better than random provider calls.
- FullEnrich only for email discovery on named contacts when you have a strong domain seed.
- **Name-validate every URL.** Company/title matching alone is not enough.
- Extract the `/in/username` slug — strip query params and trailing slashes.
- Without company context, add role keywords to the search query to disambiguate.
- If LinkedIn profile scraping is required, follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md), because that actor is not currently exposed in Sushidata.

---

## Save to Context Lake

Save resolved LinkedIn URLs after lookup. Name validation results are expensive to re-run — storing them avoids duplicate validation work on future sessions.

```json
POST /context/
{
  "serverId": "26",
  "content": "LinkedIn URL lookup complete. Input rows: {{count}}. URLs resolved: {{count}}. Name validation passed: {{count}}. Failed/nulled: {{count}}. Output: {{CSV path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
