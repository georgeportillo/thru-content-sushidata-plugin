---
name: sushi-save
description: >
  Save current session outputs to the Sushidata context lake on demand.
  Trigger when the user says: "save", "save to sushidata", "save my session",
  "save context", "write to context lake", "save this", "save everything",
  or any request to explicitly persist the current session's work.
---

# Sushidata Save

When triggered, give the user a choice of what to save, then write their
selection to the Sushidata context lake.

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

---

## Step 1 — Scan the session

Re-read the current conversation and identify everything worth saving. Group
findings into the following categories:

- **Accounts & Contacts** — companies researched, people found, org charts built
- **Competitor Intelligence** — battlecards, GTM analysis, channel breakdowns,
  positioning
- **Documents Produced** — reports, TAM analyses, outreach sequences, any
  written deliverable
- **ICP & Signals** — scoring models, niche signals, won/lost patterns
- **Campaign & Outreach** — lists built, campaigns launched, messages drafted
- **Other** — anything that doesn't fit the above

Omit categories with nothing worth saving.

---

## Step 2 — Present the save menu

Show the user a summary of what was found, then ask what they want to save:

---

### 🍣 Sushidata Save

Here's what I found worth saving from this session:

[List each category with a one-line summary of what's in it. Example:]
- **Accounts & Contacts** — 3 accounts researched: Acme Corp, Initech, Globex
- **Competitor Intelligence** — ZeroFox GTM report, channel breakdown
- **Documents Produced** — TAM analysis for enterprise security segment

---

**What would you like to save?**

> 💬 **"Save all"** — Persist everything listed above to your context lake
> 💬 **"Save [category name]"** — Save just that category (e.g. *"Save the competitor intel"* or *"Save the contacts"*)

Wait for the user's response before writing anything.

---

## Step 3 — Confirm selection

If the user says **All**, proceed to Step 4 with everything.

If the user says **Choose**, ask them to specify which categories or items.
Accept plain language — "just the competitor intel" or "the accounts and the
TAM report" are both fine. Confirm back what you're about to save in one line,
then proceed to Step 4.

---

## Step 4 — Get the session ID

Run:
```bash
echo $PWD | grep -oP 'local_[a-f0-9-]+'
```

Use the result as the `threadId` for all saves.

---

## Step 5 — Write to the context lake

For each item being saved, make a POST request:

```
POST {BASE_URL}context/
Content-Type: application/json

{
  "serverId": "26",
  "content": "<concise summary of the item — what it is, key findings, date>",
  "messageId": "msg-<unix-timestamp-ms>",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

Write one save per distinct item or document. Do not bundle unrelated items
into a single save — each save should be retrievable on its own.

Use a fresh `messageId` timestamp for each save (increment by 1ms if saving
multiple items in sequence).

---

## Step 6 — Confirm to the user

After all saves are complete, confirm with a short receipt:

---

### Saved to Sushidata ✅

[List each item saved with a checkmark. One line each.]

> These will be available in any future session when you run a restore.

**What's next?**

> 💬 **"Restore"** — Pull this context back at the start of any future session
> 💬 **"Help"** — See everything Sushidata can do
> 💬 **"Research [company]"** — Keep the momentum going

---

## Rules

- Always scan first and present the menu. Never write to the context lake
  before showing the user what you found.
- If the session has nothing worth saving, tell the user plainly:
  "Nothing substantial found in this session to save yet."
- Keep each saved `content` value concise — 2–4 sentences max per item.
  Include enough detail that it's useful when retrieved cold in a future session.
- Use a fresh timestamp for each `messageId`. Never reuse the same value.
- Always confirm saves with the receipt in Step 6.
