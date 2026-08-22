---
name: clay-extraction
description: "How to extract Clay table configs via MCP or script. Read only when the user needs to extract from Clay — skip if they already provide an extract JSON."
---

# Clay Table Extraction

Use `scripts/clay-extract.py` to pull full table configs from Clay's internal API. Extracts: field definitions, action settings (prompts, models, webhook URLs), formula text, conditional run logic, and up to ~36–66 sample records.

## Two extraction paths

| Path | When | Steps |
|---|---|---|
| **Claude-in-Chrome MCP** | Running inside a Cowork session with the Chrome extension | Zero steps — use `javascript_tool` with `fetch(url, {credentials: 'include'})` directly from the authenticated browser session |
| **`clay-extract.py` script** | Standalone or no MCP | One-time cURL paste for auth, then zero-step extraction |

---

## Script setup (one-time)

```bash
pip install requests --break-system-packages

# Auth: paste a cURL from any api.clay.com request in Chrome DevTools
python3 scripts/clay-extract.py --auth
```

Session is saved to `.clay-session.json` and reused until it expires (~24h).

Alternatively, store your cookie in `.env.clay`:

```bash
# .env.clay (never commit — add to .gitignore)
CLAY_COOKIE='claysession=...; <other cookies>'
```

Use **single quotes** — Clay cookies often contain `$` characters that bash would otherwise expand.

---

## Extraction commands

```bash
# By table URL or ID
python3 scripts/clay-extract.py https://app.clay.com/workspaces/502058/workbooks/wb_xxx/tables/t_xxx
python3 scripts/clay-extract.py t_0t5pj9mqNnpxxjM6jaV

# By workbook URL (extracts all tables in the workbook)
python3 scripts/clay-extract.py https://app.clay.com/workspaces/502058/workbooks/wb_xxx

# By name (fuzzy matches workbook and table names)
python3 scripts/clay-extract.py --workspace 502058 "Demo Requests"
```

Output goes to `tmp/clay_extract_<table_name>.json`. Never overwrites existing files.

---

## What the extract contains

```json
{
  "_meta": { "extractedAt", "method", "tableId" },
  "table": { "id", "name", "workbookId", "workspaceId", "firstViewId", "tableSettings" },
  "fields": [
    {
      "id": "f_xxx",
      "name": "AI Message Generator",
      "type": "action",
      "typeSettings": {
        "actionKey": "use-ai",
        "inputsBinding": [
          { "name": "prompt", "formulaText": "You are writing..." },
          { "name": "model", "formulaText": "\"claude-sonnet-4-6\"" }
        ],
        "formulaText": "...",
        "conditionalRunFormulaText": "!!{{f_xxx}}"
      }
    }
  ],
  "tableSchema": { ... },
  "exampleRecords": [ ... ]
}
```

---

## Key Clay API endpoints

| Endpoint | Method | Returns |
|---|---|---|
| `/v3/tables/{TABLE_ID}` | GET | Full table config: fields, typeSettings, prompts, action bindings |
| `/v3/tables/{TABLE_ID}/views/{VIEW_ID}/table-schema-v2` | GET | Schema tree + example records (up to ~66 rows) |
| `/v3/workbooks/{WB_ID}/tables` | GET | List of tables in a workbook `[{id, name, ...}]` |
| `/v3/workspaces/{WS_ID}/resources_v2/` | POST | Top-level workspace resources (folders, workbooks) |
| `/v3/tables/{TABLE_ID}/views/{VIEW_ID}/records/ids` | GET | All record IDs (for full data pull) |
| `/v3/tables/{TABLE_ID}/bulk-fetch-records` | POST | Full cell data for specific record IDs |

All require `Cookie: claysession=...` + `origin: https://app.clay.com` headers.

---

## Important details

- **Formula text location**: `field.typeSettings.formulaText` (NOT `field.formulaText`)
- **Action prompts**: `field.typeSettings.inputsBinding` array → find entry with `name: "prompt"` → `.formulaText`
- **Model**: same array → `name: "model"` → `.formulaText` (e.g. `"claude-sonnet-4-6"`)
- **Field references in formulas**: `{{f_xxx}}` format — map to names via the fields array
- **Folder URLs** (`/home/f_xxx`): the `f_xxx` is a folder ID, not a field. Folder children aren't exposed via API — use workbook URLs or name search instead.
- **Cookie security**: `.clay-session.json` and `.env.clay` are gitignored. Never log or embed cookies in scripts.

---

## MCP extraction (Claude-in-Chrome)

When Claude-in-Chrome MCP is available, skip the script:

1. `tabs_context_mcp` → get tab context
2. `navigate` → any Clay table or workbook URL
3. `javascript_tool` with `credentials: 'include'`:

```javascript
fetch('https://api.clay.com/v3/tables/{TABLE_ID}', {
  headers: { 'accept': 'application/json' },
  credentials: 'include'
}).then(r => r.json()).then(data => { window.__clayConfig = data; });
```

4. Read result from `window.__clayConfig`

The browser already has the session cookie — `credentials: 'include'` sends it automatically. Without this flag, the request returns 401.

---

## Input data formats

When the user provides data directly (not via extraction), these are the possible formats ranked by richness:

| Input type | Key fields |
|---|---|
| **HAR file** | `bulk-fetch-records` responses with rendered formula cell values — richest |
| **clay-extract.py output** | `.fields[].typeSettings.inputsBinding` for prompts; `.exampleRecords` for samples |
| **Schema JSON** | Field names, IDs, action types. No cell values or prompts |
| **User description** | Weakest — must approximate everything |

**When `bulkFetchRecords` is null:** Fall back to `portableSchema`:
- Prompts: `.portableSchema.columns[].typeSettings.inputsBinding` → `{name: "prompt"}` → `.formulaText`
- JSON schemas: `{name: "answerSchemaType"}` → `.formulaMap.jsonSchema` (double-escaped — `JSON.parse` twice)
- Conditional run: `.typeSettings.conditionalRunFormulaText`

---

## Extract bulk-fetch-records from a HAR file

```python
import json, base64, gzip

with open('your-export.har') as f:
    har = json.load(f)

for entry in har['log']['entries']:
    url = entry['request']['url']
    if 'bulk-fetch-records' in url:
        body = entry['response']['content'].get('text', '')
        enc  = entry['response']['content'].get('encoding', '')
        data = base64.b64decode(body) if enc == 'base64' else body.encode()
        try:
            data = gzip.decompress(data)
        except Exception:
            pass
        print(json.dumps(json.loads(data), indent=2)[:5000])
```
