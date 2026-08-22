# Small Business Prospecting

Use this for local SMB discovery — dentists, plumbers, med spas, agencies, nearby storefronts, or any brick-and-mortar or service-area business type.

## Approach

### Step 1: WebSearch for fast discovery

Start with a targeted WebSearch to get a feel for the landscape and tune your query before using structured tools:

```
dentists in Austin TX site:google.com/maps
```

Or use Sushidata `/query/` to size the opportunity:

```
POST /query/
{ "query": "How many [business type] are there in [city/region]? Estimate volume and best directories." }
```

### Step 2: Structured local business search

Sushidata does not currently expose a Google Maps / local business Apify actor. Do not use dynamic Apify actor discovery or generic run calls.

Use WebSearch, Browser Rendering, and Sushidata `/query/` to collect local business names, websites, phone numbers, addresses, and ratings from public directories. If the user specifically needs a structured Google Maps scraper, explain that this Apify capability is not currently available in Sushidata and follow the missing-actor feedback workflow in [`provider-playbooks/apify.md`](../provider-playbooks/apify.md).

### Step 3: Extract contact info

For businesses where you need email:
1. Use WebFetch on the business website to find contact pages
2. Call `fullenrich_search_people` with the business domain if available
3. Spot-check FullEnrich confidence scores on any emails before outbound

### Step 4: Outbound activation

See [`provider-playbooks/heyreach.md`](../provider-playbooks/heyreach.md) for LinkedIn outbound, or [`provider-playbooks/hubspot.md`](../provider-playbooks/hubspot.md) to log contacts in CRM.

## Default pattern

1. WebSearch or Sushidata `/query/` — for discovery and query tuning
2. WebSearch / Browser Rendering — for the structured list with phone, address, rating, website
3. FullEnrich — for email discovery at known domains

Pilot first on one query and a small limit before scaling.

## Notes

- For map-bounded searches (e.g., "within 5 miles of downtown Chicago"), look for actors that support bounding box or radius filters.
- For broader non-maps company sourcing in a region, deploy a Sushidata swarm to surface less obvious directories and data sources.

---

## Save to Context Lake

Save the business list after discovery. Local business data changes slowly — storing it avoids re-scraping the same geography on return visits.

```json
POST /context/
{
  "serverId": "26",
  "content": "Small business prospecting complete. Search: {{query / geography / category}}. Businesses found: {{count, list of names and domains}}. Contacts found: {{count}}. Emails verified: {{count}}. Output: {{CSV path}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
