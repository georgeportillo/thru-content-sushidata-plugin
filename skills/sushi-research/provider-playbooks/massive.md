# Massive Browser Render Playbook

Use this playbook when you need to fetch rendered page content from URLs that actively block standard crawlers — such as PitchBook, LinkedIn, Crunchbase, or any site with aggressive bot-detection. The `massive_browser_render` tool routes requests through a **residential browser network** (Massive), making them indistinguishable from real user traffic.

Use this **instead of** `get_url_markdown` / `get_url_screenshot` when those tools return empty content, CAPTCHAs, or access-denied errors.

---

## Available Tools

| Tool name               | What it does                                                                                     |
|-------------------------|--------------------------------------------------------------------------------------------------|
| `massive_browser_render` | Fetches a URL through the Massive residential browser network, rendering JS and bypassing bot-detection. Returns page content in rendered HTML, Markdown, or plain text. |

---

## `massive_browser_render`

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` | string | — | The URL to render (required) |
| `format` | `rendered` \| `markdown` \| `text` | `rendered` | Output format |
| `country` | string (2-char ISO) | `US` | Exit node country code (e.g. `"DE"`, `"GB"`) |

**Format guide:**

| Format | Best for |
|--------|----------|
| `markdown` | Reading page text — articles, profiles, pricing pages, feature lists |
| `text` | Plain text extraction — faster, less noise than markdown |
| `rendered` | Full raw HTML — when you need to parse structure or extract specific elements |

---

## When to Use Massive

| Situation | Tool |
|-----------|------|
| Standard docs, blog posts, help center pages | WebFetch (built-in, no proxy overhead) |
| Pages returning CAPTCHA, 403, or empty content | `massive_browser_render` |
| PitchBook investor/company profiles | `massive_browser_render` |
| LinkedIn profiles or company pages | `massive_browser_render` |
| Crunchbase, G2, or similar gated/bot-protected pages | `massive_browser_render` |
| Need a specific country's version of a page | `massive_browser_render` (set `country`) |

> **Always try WebFetch first for standard public pages.** Only escalate to `massive_browser_render` if the first attempt returns an error, CAPTCHA, or clearly insufficient content.

---

## Examples

### Read a PitchBook investor profile

```json
{
  "url": "https://pitchbook.com/profiles/investor/41716-90",
  "format": "markdown",
  "country": "US"
}
```

### Read a LinkedIn company page

```json
{
  "url": "https://www.linkedin.com/company/stripe",
  "format": "markdown",
  "country": "US"
}
```

### Get a German-localized version of a pricing page

```json
{
  "url": "https://example.com/pricing",
  "format": "markdown",
  "country": "DE"
}
```

---

## Relationship with `apify_pitchbook_data_extractor`

For **structured** PitchBook investor data (firm details, deal counts, contact info, recent investments), prefer `apify_pitchbook_data_extractor` — it returns clean JSON you can directly use.

Use `massive_browser_render` for PitchBook when:
- You need a page type not covered by the Apify actor (e.g. company profiles, deal pages, custom searches)
- The Apify actor returns incomplete data for a specific investor
- You want to read the raw profile page text for qualitative research

---

## Rules

- Timeout is 60 seconds — Massive resolves residential exits which can be slow. Be patient and do not retry immediately on timeout.
- The tool returns full page content — extract only what you need before saving to the context lake
- Do not loop over large lists of URLs in a single agent turn — distribute across swarm workers (one URL or small batch per worker)
- For bulk PitchBook/LinkedIn research, use `apify_pitchbook_data_extractor` or the Apify LinkedIn actors first; fall back to Massive for gaps
- Verify important claims from rendered pages with a follow-up Sushidata swarm search before presenting as facts
