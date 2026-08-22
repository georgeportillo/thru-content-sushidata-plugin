# HeyReach Playbook

Use HeyReach for outbound activation after qualification and verification are complete.

## Key rules

- **Campaigns must be pre-created in the HeyReach UI.** The public API does not expose campaign creation — do not attempt to create campaigns via API tools. Create campaigns in the HeyReach UI first, then reference the resulting `campaign_id`.
- Always list campaigns first and resolve the exact campaign target before any inserts.
- Batch contact writes in small chunks (≤50 contacts) and validate response shape before scaling.
- Pull campaign stats after insert operations to confirm downstream effects.
- **On connection, auto-wire reply alerts.** The first time HeyReach is connected, register the Sushidata webhook (`{BASE_URL}webhook/ingest`) before any campaign work so the user is emailed the moment a prospect replies — see the next section.

## On connection: connect reply alerts (do this first, automatically)

The first time HeyReach is connected in a session — before listing campaigns or adding contacts — automatically wire up a webhook so the user is alerted the moment a prospect replies. Don't wait to be asked; set it up as part of connecting and tell the user it's done.

**Webhook target URL** (Sushidata ingest endpoint):

```
{BASE_URL}webhook/ingest
```

> `{BASE_URL}` is in `SETTINGS.md` and already includes the dataspace path, so the full URL is e.g. `https://dashboard.sushidata.ai/public/<resourceId>/webhook/ingest`. Replies and key lead events POSTed here flow into the context lake **and** trigger an email alert to every user in the tenant.

### Auto-connect via the HeyReach API (preferred)

If a HeyReach webhook tool is available (e.g. `heyreach_create_webhook`), create the webhook automatically — no need to ask first:

```json
{
  "webhookName": "Sushidata reply alerts",
  "webhookUrl": "{BASE_URL}webhook/ingest",
  "eventTypes": [
    "MESSAGE_REPLY_RECEIVED",
    "EVERY_MESSAGE_REPLY_RECEIVED",
    "INMAIL_REPLY_RECEIVED"
  ]
}
```

- Also enable any **connection-accepted** event your HeyReach plan exposes so accepted invites are captured too.
- After creating, list webhooks to confirm the Sushidata URL is registered.
- Tell the user: *"HeyReach is connected to Sushidata — you'll get an email alert whenever a prospect replies."*

### Manual fallback (if no webhook API tool)

If the connector doesn't expose webhook creation, ask the user to add it once in the HeyReach UI:

1. HeyReach → **Settings → Webhooks → Add webhook**
2. **Webhook URL**: `{BASE_URL}webhook/ingest` (paste the exact value from `SETTINGS.md`)
3. **Events**: enable the reply / message-received events and connection-accepted events
4. **Save**

Give the user the exact URL to paste and confirm once it's saved.

## Workflow

### 1. List available campaigns

Call `heyreach_list_campaigns` (no required payload) to retrieve all active campaigns and their IDs.

```json
{}
```

Identify the correct `campaign_id` from the response before proceeding.

### 2. Add contacts to a campaign

Call `heyreach_add_to_campaign` with the resolved campaign ID and contact list:

```json
{
  "campaign_id": "12345",
  "contacts": [
    {
      "linkedin_url": "https://www.linkedin.com/in/example",
      "first_name": "Ada",
      "last_name": "Lovelace",
      "email": "ada@example.com"
    }
  ]
}
```

**Only add contacts that have been verified** — do not pass unverified or catch-all emails.

### 3. Confirm stats

After inserting, call `heyreach_get_campaign_stats` (or equivalent) to confirm that the contact count and campaign state reflect the inserts correctly.

## Common pitfalls

- Attempting to create campaigns via API (not supported — always use the UI).
- Inserting contacts without resolving the campaign ID first.
- Skipping stat verification after inserts — downstream sync issues are silent.
- Passing catch-all or unverified emails; spot-check FullEnrich confidence scores before activating.

---

## Save to Context Lake

Save campaign activation results so future sessions can check campaign status without re-querying HeyReach:

```json
POST /context/
{
  "serverId": "26",
  "content": "HeyReach campaign activation complete. Campaign: {{campaign name, ID}}. Contacts added: {{count}}. LinkedIn account used: {{sender name}}. Contacts accepted: {{count from stats check}}. Any rejected: {{count and reason}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
