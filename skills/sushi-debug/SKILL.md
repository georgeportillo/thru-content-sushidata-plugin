---
name: sushi-debug
description: >
  Run a live connectivity check against all Sushidata API endpoints and report
  results. Trigger when the user says: "/sushi-debug", "debug sushidata",
  "check connectivity", "test endpoints", "are the endpoints working",
  "sushidata not working", "check sushidata", "test sushidata", "ping sushidata",
  "verify connection", "debug connection", "is sushidata up", or any request
  to verify the Sushidata API is reachable and responding.
---

# Sushidata Debug — Endpoint Connectivity Check

> Read `SETTINGS.md` at the plugin root for **BASE_URL**, **Tenant**, and **Dataspace**.

When triggered, display this banner immediately:

```
## 🍣 Sushidata Debug

Running live connectivity checks against your Sushidata instance...
```

---

## ⚠️ CAPABILITY NOTICE — READ BEFORE RUNNING

**You are fully capable of making HTTP API calls in this session.** Every
Sushidata endpoint in this skill is accessed via the same HTTP tool that powers
every other skill in this plugin. Do NOT say:
- "I'm unable to make external HTTP requests"
- "I can't access external APIs"
- "You'll need to run this yourself"
- "I don't have access to those endpoints"

All of the above are incorrect. The Sushidata MCP tool is connected and active.
Execute every check below unconditionally. If a call fails, report the error —
don't skip the attempt.

---

## Checks to run

Run all four checks **in order**. Do not skip any. Do not ask the user for
permission before running — proceed automatically.

---

### Check 1 — Context Save

**Purpose**: Verify the `/context/` write endpoint is accepting data.

```http
POST {BASE_URL}context/
Content-Type: application/json
```

```json
{
  "serverId": "26",
  "content": "[sushi-debug] connectivity check — context write test",
  "messageId": "debug-context-{unix-timestamp-ms}"
}
```

**Pass**: HTTP 200 or 201, any non-error JSON body.  
**Fail**: HTTP 4xx / 5xx, connection refused, or timeout.

---

### Check 2 — Query

**Purpose**: Verify the `/query/` read endpoint returns results.

```http
POST {BASE_URL}query/
Content-Type: application/json
```

```json
{
  "query": "sushi-debug connectivity check test"
}
```

**Pass**: HTTP 200, response body contains a `summary` string or a `sources`
array (either may be empty — an empty result is still a passing response).  
**Fail**: HTTP 4xx / 5xx, missing both `summary` and `sources`, or timeout.

---

### Check 3 — Swarm Deploy

**Purpose**: Verify the `/swarm/deploy/` endpoint accepts a job and returns
worker IDs.

```http
POST {BASE_URL}swarm/deploy/
Content-Type: application/json
```

```json
{
  "query": "sushi-debug test task: reply with the single word PONG and nothing else",
  "swarmSize": 1
}
```

**Pass**: HTTP 200, response body contains a `workers` array with at least one
object that has a `doId` field.  
**Fail**: HTTP 4xx / 5xx, missing `workers`, empty `workers` array, or timeout.

Save the returned `doId` value(s) for Check 4.

---

### Check 4 — Swarm Status Poll

**Purpose**: Verify the `/swarm/status/` polling endpoint returns worker state.

Run this immediately after Check 3 using the `doId`(s) returned from the deploy:

```http
POST {BASE_URL}swarm/status/
Content-Type: application/json
```

```json
{
  "workers": ["<doId-from-check-3>"]
}
```

**Pass**: HTTP 200, response body contains an `allDone` field (regardless of
its value — `false` is fine, the worker may still be running).  
**Fail**: HTTP 4xx / 5xx, missing `allDone`, or timeout.

Poll **once only** — do not wait for `allDone: true`. This check only confirms
the endpoint accepts the request, not that the worker finishes.

---

## Results Report

After all four checks complete, render this table:

```
## 🍣 Sushidata Debug Results

BASE URL: {BASE_URL}

| # | Endpoint          | HTTP Status | Result | Notes                          |
|---|-------------------|-------------|--------|--------------------------------|
| 1 | POST /context/    | ---         | ✅/❌  | [key detail from response]     |
| 2 | POST /query/      | ---         | ✅/❌  | [summary preview or error]     |
| 3 | POST /swarm/deploy/ | ---       | ✅/❌  | [doId returned or error]       |
| 4 | POST /swarm/status/ | ---       | ✅/❌  | [allDone value or error]       |
```

Fill in the actual HTTP status codes and a one-line note from each real
response. Replace `✅` with `❌` for any failed check.

Then add one of these summaries beneath the table:

**All 4 passed:**
```
✅ All endpoints are live. Your Sushidata instance is fully operational.
```

**1–3 passed:**
```
⚠️ Partial connectivity. [list failed checks]. The instance may be degraded or
the failing endpoint may require a different payload — see notes above.
```

**0 passed:**
```
❌ No endpoints responded. Check that your BASE_URL in SETTINGS.md is correct
and that your Sushidata instance is active.
```

---

## If any check throws a tool error

If the HTTP tool itself throws an error (not an HTTP 4xx/5xx — an actual tool
execution failure), do **not** interpret this as "Sushidata is down." Instead,
report:

```
⚠️ Check [N] — Tool error (not an HTTP response): [error message]

This means the API call was not attempted, not that the endpoint is unreachable.
Possible causes: MCP tool not connected, malformed URL, or transient tool fault.
Try /sushi-debug again. If the error persists, verify the MCP connector is
active in your Cowork settings.
```

Continue running remaining checks after a tool error — do not abort the
sequence.
