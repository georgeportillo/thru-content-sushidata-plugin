# Writing Outreach

Per-row copy generation, sequence design, qualification language, and scoring. The deliverable is personalized text — cold emails, multi-step sequences, ICP tier classifications, or lead scores — for rows that already have research and identity context.

This doc does **not** cover finding people or filling emails — those live in `finding-companies-and-contacts.md` and must complete before this doc is useful. Outreach generation without research columns produces mail-merge.

## When you are in this doc

You have rows with name, title, company, and **at least one research or context column** — "what they sell," "recent funding," "pain points," "tier." User phrases that land here:

- "write a cold email for each of these leads"
- "draft a 4-step sequence for the Tier 1 contacts"
- "write qualification copy for each row explaining why they fit"
- "personalize a first line per contact based on the research column"

If rows do not have research columns yet, use Sushidata `/swarm/deploy/` to research each company first (swarmSize 3–5 per company), producing a `company_research` column. Then return here.

## Research first, copy second — always

A common failure: writing research and copy in the same pass — "Look up Acme and write me a personalized email." That conflates two jobs:

1. **Research** — What does Acme do? What pain do they have? What's their tech stack? → Sushidata swarm, swarmSize 3–5, output saved to `/context/` and as a `company_research` CSV column.
2. **Copy** — Write an email that references the research column. → Claude in this session, reading the `company_research` column per row.

Combining them produces shallow research because the prompt is optimized for copy, not factual depth. Separate the stages. The research stage is cheap to cache (Sushidata saves it to the context lake automatically); the copy stage can then be re-run with different angle or tone without paying for research again.

## Durable rules

### Personalization requires a row-specific signal

Every email must reference something specific to that row: a product the company makes, a pain point in the research column, recent news, a hiring signal, a use case. `{{first_name}}` and `{{company_name}}` in a template is not personalization. If the only row-specific data is name + title + company, the output is mail-merge regardless of how the prompt is phrased. Fix upstream: enrich with a research column first.

### Structured output for anything downstream will read

When the output is emails, sequences, or qualification objects, return structured JSON — not free text. Free-text output for sequences requires the user (or the next stage) to parse N emails out of a blob; any malformed run silently breaks the parse. A schema enforces shape at write time and gives downstream stages a clean contract.

```json
// 4-step sequence schema — ask Claude to return this shape
{
  "steps": [
    {
      "step": 1,
      "subject": "string",
      "core_value_prop": "string",
      "email": "string"
    }
  ]
}
// Require exactly 4 items in steps array
```

### Qualification is evidence-based with high recall

When classifying rows against an ICP, build the prompt around *only* the provided context — do not let the model hallucinate evidence. Mark `unknown` when evidence is missing, not a low score. Default to high recall: when evidence is borderline, lean toward the higher tier. Outreach is cheaper than missing a real prospect; noise is filtered downstream by reply rate.

### Carry lineage end-to-end

Preserve research source columns and identity columns alongside the generated copy. A CSV of generated emails with no link to which research informed each one is impossible to QA, debug, or A/B test. Every row should carry the inputs that produced its copy, not just the copy itself.

### Save research to Sushidata before generating copy

After research swarms complete (but before copy generation), save the research to the Sushidata context lake:

```json
POST /context/
{
  "serverId": "26",
  "content": "Company research batch for outreach: {{research_summary_or_CSV_row}}",
  "messageId": "msg-{{timestamp}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```

This makes the research reusable across sessions — if the user asks "what did we find about Acme?" later, the context lake answers without re-running the swarm.

---

## Patterns

### One-shot per-row personalization

For each row, prompt Claude with the research column and ask for structured output. For small batches (<20 rows), do this inline in the conversation. For larger batches, write a Python script that calls Claude's API per row.

**Prompt template:**

```
Write a cold email to {{first_name}} {{last_name}} ({{title}}) at {{company_name}}.

Context about their company:
{{company_research}}

Rules:
- Open with a one-sentence reference to a specific pain point from the research above
- Keep under 90 words
- No generic openers ("Hope this finds you well", "I wanted to reach out")
- No fabricated statistics or case studies

Return JSON:
{
  "subject": "...",
  "first_line": "...",
  "body": "..."
}
```

The prompt explicitly requires `{{company_research}}` as the personalization signal — that column was produced by a prior Sushidata swarm.

### Multi-step sequence

Generate a full 3-5 email sequence as one structured output rather than calling the model N times. The model plans the arc better when it sees all steps at once.

**Prompt template:**

```
Design a 4-step outbound sequence for {{first_name}} at {{company_name}}.

Persona: {{title}}
Research: {{company_research}}

Arc:
- Step 1: Hook + value prop tied to their specific pain
- Step 2: Proof / reference customer with a similar profile
- Step 3: Low-friction CTA (15-min call, async Loom, forward to the right person)
- Step 4: Breakup email — honest, no guilt

Each step under 90 words. Return JSON:
{
  "steps": [
    {"step": 1, "subject": "...", "core_value_prop": "...", "email": "..."},
    {"step": 2, "subject": "...", "core_value_prop": "...", "email": "..."},
    {"step": 3, "subject": "...", "core_value_prop": "...", "email": "..."},
    {"step": 4, "subject": "...", "core_value_prop": "...", "email": "..."}
  ]
}
```

### ICP tier classification

Qualify rows against an ICP by returning an enum tier with evidence-grounded reasoning.

**Prompt template:**

```
Using ONLY the context below, classify {{company_name}} for our ICP.

ICP description: {{icp_description}}
Company context: {{company_research}}

If the context does not contain enough evidence to decide, return tier "unknown" — do not infer.

Return JSON:
{
  "tier": "high_fit" | "medium_fit" | "low_fit" | "unknown",
  "rationale": "...",
  "evidence_used": ["specific fact 1 from context", "specific fact 2"]
}
```

`evidence_used` is an accountability field — it surfaces which context facts the model actually used, which catches hallucinated reasoning.

### Multi-criteria lead scoring (0–100)

When the user wants a composite score, return both the score and the per-criterion breakdown for auditability.

**Prompt template:**

```
Score this lead 0–100 against our ICP.

ICP criteria: {{icp_description}}
Lead context: {{company_research}}
Role: {{title}}

Scoring tiers: 60+ = Tier 1 immediate outreach | 35–59 = Tier 2 trigger-based | <35 = nurture or skip

Return JSON:
{
  "score": <integer 0-100>,
  "fit_band": "high" | "medium" | "low" | "unknown",
  "criteria": [
    {"criterion": "...", "met": true/false, "note": "..."}
  ]
}
```

For larger lists, save scored rows to Sushidata after scoring so swarms in future sessions can reference which accounts are already Tier 1:

```json
POST /context/
{ "content": "Lead scoring run — Tier 1 accounts: {{list}}. Scoring criteria: {{icp}}." }
```

### Positioning-led LinkedIn multi-variant outreach

Use this pattern when the user wants LinkedIn-native outreach (connection notes + InMail) rather than email sequences, and when they have a positioning framework or point of view to lead with — not just a feature list.

The core principle: **arrive with a POV, don't discover pain live.** Each message teaches the buyer how to think about the market before asking for anything. Research the account first, then write copy that references something specific to that company. Mail-merge is not this pattern.

---

#### Framework structure (adapt to the user's product and ICP)

Before writing any copy, establish:

1. **The market framing** — What are the 2–3 ways buyers currently solve this problem? (e.g. "most teams use X, some build Y, a few use Z.") Name the camps honestly, including alternatives that are legitimate.
2. **The core positioning line** — One sentence that names who the product is for and what it replaces. Sharp, specific, no jargon.
3. **The proactive objection** — What's the biggest reason a buyer would say no? State it yourself in the message before they do. Buyers trust sellers who raise objections unprompted.
4. **The signal** — One sentence per account describing the specific pain their team faces, derived from Sushidata swarm research. This populates the hook in every variant.

---

#### Buyer persona map

Map the user's ICP to at least two buyer types before writing:

| Role type | Angle |
|---|---|
| **Economic buyer** (VP, CRO, C-suite) | Budget, team efficiency, strategic risk |
| **Technical champion** (IC engineer, ops, RevOps) | Practitioner pain, workflow friction, build vs buy |
| **Marketing / narrative buyer** (CMO, PMM, Head of Content) | Market positioning, category narrative, competitive angle |

Each persona gets a different message — same signal, different frame.

---

#### 5-variant message structure

For each account, produce exactly 5 variants as structured JSON. Adapt the persona labels to the user's actual buyer map.

| Variant | Format | Persona | Character limit |
|---|---|---|---|
| `connection_economic` | LinkedIn connection note | Economic buyer (VP/C-suite) | ≤300 chars, no subject |
| `connection_champion` | LinkedIn connection note | IC practitioner / ops | ≤300 chars, no subject |
| `inmail_economic` | LinkedIn InMail | Economic buyer | 1200–1700 chars, has subject |
| `inmail_marketing` | LinkedIn InMail | Marketing / narrative buyer | 450–900 chars, has subject |
| `connection_technical` | LinkedIn connection note | Technical buyer | ≤300 chars, no subject |

**Hard formatting rules (apply to all variants):**
- No em dashes anywhere — use commas, colons, or periods instead
- Connection notes: strictly ≤300 chars including spaces
- InMail body: stay within the char range for that variant
- Every variant must include a `chars` field with the actual character count
- No generic openers ("Hope this finds you well", "I wanted to reach out")
- No fabricated statistics, case studies, or attributed quotes

---

#### InMail structure (for the long-form economic variant)

Follow this arc for the 1200–1700 char InMail:

1. **Specific observation** — One sentence about something real at their company (product, team, recent news, hiring signal). Must be derived from the research signal, not generic.
2. **Market framing** — "Most teams in your position do X, Y, or Z. Here's how they compare..." — name the camps honestly.
3. **Proactive objection** — "The obvious question is whether you'd just [build it / use X instead]. Here's when that makes sense and when it doesn't."
4. **Ask** — Specific, low-friction. A 15-minute call, an async question, a forward to the right person. Not "let me know if you're interested."

---

#### JSON output schema

```json
{
  "company": "CompanyName",
  "signal": "One sentence: the specific pain this account's team faces, from swarm research.",
  "outreach": {
    "connection_economic": {
      "to": "Name, Title",
      "subject": "",
      "body": "...",
      "chars": 0,
      "note": "Day 0 LinkedIn connection note — economic buyer"
    },
    "connection_champion": {
      "to": "Name, Title",
      "subject": "",
      "body": "...",
      "chars": 0,
      "note": "Day 0 LinkedIn connection note — IC practitioner"
    },
    "inmail_economic": {
      "to": "Name, Title",
      "subject": "Subject line here",
      "body": "...",
      "chars": 0
    },
    "inmail_marketing": {
      "to": "Name, Title",
      "subject": "Subject line here",
      "body": "...",
      "chars": 0
    },
    "connection_technical": {
      "to": "Name, Title",
      "subject": "",
      "body": "...",
      "chars": 0,
      "note": "Day 0 LinkedIn connection note — technical buyer"
    }
  }
}
```

---

#### Research prerequisite for this pattern

Before writing any variant, deploy a Sushidata swarm against the account:

```
swarmSize: 5–10
query: "What specific research, enrichment, or data pain does [company]'s sales or marketing team face? Look for signals in job listings, LinkedIn posts, product pages, and community mentions."
```

Use the swarm output to populate the `signal` field. The signal is what makes all 5 variants specific — without it, the output collapses to mail-merge.

---

#### Validation checklist (run before returning output)

1. Every connection note is ≤300 chars
2. `inmail_economic` is 1200–1700 chars
3. `inmail_marketing` is 450–900 chars
4. No em dashes in any variant
5. Each `chars` field matches the actual body length
6. The `signal` field references something specific from swarm research — not a generic description of the company

---

## Batch processing

For lists >20 rows, write a Python script that loops over a CSV and calls Claude's API for each row. Keep the research and copy stages separate:

```python
import csv, json, anthropic

client = anthropic.Anthropic()

with open("enriched.csv") as f:
    rows = list(csv.DictReader(f))

results = []
for row in rows:
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",   # fast + cheap for copy generation
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": f"""Write a cold email to {row['first_name']} {row['last_name']} ({row['title']}) at {row['company']}.
Company research: {row['company_research']}
Rules: specific pain reference, under 90 words, no generic openers.
Return JSON: {{"subject": "...", "first_line": "...", "body": "..."}}"""
        }]
    )
    try:
        copy = json.loads(resp.content[0].text)
    except Exception:
        copy = {"subject": "", "first_line": "", "body": resp.content[0].text}
    results.append({**row, **copy})

with open("outreach_ready.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=results[0].keys())
    writer.writeheader()
    writer.writerows(results)
```

Use `claude-haiku-4-5-20251001` for copy generation (fast, cheap) and reserve `claude-sonnet-4-6` for research swarms where depth matters.

---

## Activation via HeyReach

Once the outreach CSV is ready (`name, linkedin_url, email, sequence_step_1...N`), activate via HeyReach:

1. Read `provider-playbooks/heyreach.md` for the campaign insertion workflow.
2. The campaign must be pre-created in the HeyReach UI with the sequence steps already configured.
3. Add contacts in batches of ≤50 using the `add_leads_to_campaign` MCP tool.
4. Confirm via `get_overall_stats` that contacts were accepted.

HeyReach sends the sequence steps — the deliverable from this doc is the CSV the user uploads, not the send itself.

---

## Validation

### Novelty check

After generating copy across a list, sample a few rows and check: does each email reference something specific to that row, or do the emails read like a template with name swaps? If the latter, the upstream research column is too thin — return to the Sushidata swarm and produce richer research before regenerating.

A simple check: count rows where the email body overlaps >80% with another row's body. High overlap = model is template-completing = research signal is too weak.

### Spot-check before sending

Generated copy should be read by a human before going to a sender. Models occasionally produce confidently wrong claims (wrong company description, fabricated case study, misattributed quote) that are caught immediately by reading but invisible to programmatic checks.

---

## Exit

- Copy and qualification columns are ready → hand off to HeyReach (see `provider-playbooks/heyreach.md`) or export CSV directly to the user.
- Output reads as mail-merge → return to Sushidata swarm to produce a richer `company_research` column, then regenerate.
- ICP classifications have low recall → check that the prompt references `unknown` as the fallback and that the research column has enough depth.
