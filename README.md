# Asset Catalog Query Layer

A self-serve query layer over an enterprise asset catalog, built on n8n. Three workflows that turn a daily CSV export from a corporate datalake into two consumable interfaces: an HTML form for humans and an authenticated JSON API for automations.

Built and shipped in production at a mid-sized cloud MSP. Sanitized version published here as a case study and reusable pattern.

## The problem

A corporate asset catalog of roughly 100,000 records lived in an enterprise datalake. Other teams needed access to it, but the access patterns were broken in two specific ways:

- **Security teams** wanted to enrich findings from external scanning tools with asset metadata (criticality, monitoring coverage, owner, business context). They had no programmatic access. Every request became a manual data dump from the automation team.
- **Operations and service delivery** wanted ad-hoc lookups for tickets and customer reviews. They were either asking the automation team to run queries or trying to learn unfamiliar internal tools.

In both cases, the automation team became the bottleneck. Every data request was a ticket. As volume grew, this didn't scale.

The narrower technical problem under that: the datalake export was a single 18 MB CSV with no filtering primitives. Anyone wanting a slice of the data had to either pull the whole thing or wait for someone to do it for them.

## The solution

Three n8n workflows.


```
                 ┌────────────────────────┐
                 │  Catalog Sync          │
                 │  (scheduled, daily)    │
                 │                        │
                 │  Datalake API → CSV    │
                 │  → parse → dedupe      │
                 │  → upsert to data table│
                 └───────────┬────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │     asset_catalog    │
                  │   (n8n data table)   │
                  │   ~88k unique rows   │
                  └──────┬─────────┬─────┘
                         │         │
              ┌──────────┘         └──────────┐
              ▼                               ▼
   ┌─────────────────────┐         ┌──────────────────────┐
   │   Query Form        │         │   Query API          │
   │  (public form)      │         │  (authenticated)     │
   │                     │         │                      │
   │  HTML + CSV export  │         │  JSON envelope       │
   │  for humans         │         │  for automations     │
   └─────────────────────┘         └──────────────────────┘
```

**Workflow 1: Catalog Sync.** Runs daily on a schedule. Triggers an asynchronous export job on the datalake, polls until ready, downloads the CSV, parses it, deduplicates on `asset_id`, and upserts every row into an n8n data table. No filtering at ingest; the full catalog is preserved.

**Workflow 2: Query Form.** A public n8n form trigger with twelve filter fields (text fields use partial match, dropdowns use exact match). On submit, reads the data table, applies filters in JavaScript, returns an HTML response with a results table and a CSV download button containing the full filtered set.

**Workflow 3: Query API.** A POST webhook with `x-api-key` Header Auth. Accepts a JSON body with `filters` and `limit`. Reads the same data table, applies the same filter semantics, returns a standardized JSON envelope (`success`, `query`, `result`, `meta`). Default limit 1000 rows, hard cap 5000.

Both consumer workflows read from the same data table, ensuring consistent semantics across the human and machine interfaces.

## Example: querying the API

```bash
# Coverage gap query: find productive assets without monitoring
curl -X POST "https://YOUR_N8N_HOST/webhook/YOUR_WEBHOOK_UUID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filters":{"status":"productive","monitoring_enabled":"false"},"limit":100}'
```

Response shape:

```json
{
  "success": true,
  "query": {
    "filters_applied": { "status": "productive", "monitoring_enabled": "false" },
    "limit": 100
  },
  "result": {
    "total_matched": 247,
    "returned": 100,
    "truncated": true,
    "data": [
      {
        "asset_id": "AS00001",
        "asset_name": "web-app-prod-01",
        "asset_type": "ManagedVM",
        "status": "productive",
        "owning_team": "Linux Platform Team",
        "service_group": "Internal Web Service",
        "business_unit": "Example Customer Inc",
        "monitoring_enabled": "false",
        "scanning_enabled": "true",
        "is_critical": "true",
        "is_external": "false"
      }
    ]
  },
  "meta": {
    "data_source": "asset_catalog",
    "refreshed_by": "Asset Catalog Sync",
    "freshness_note": "Refreshed weekday mornings",
    "generated_at": "2026-04-30T09:23:14.123Z"
  }
}
```

## Why this design

A few decisions that turned out to matter.

**Read-time filtering, not ingest-time.** It would have been faster to filter at sync time and store a smaller subset. But the consuming teams were unknown when the system was built, and their needs were heterogeneous: security wanted critical assets, ops wanted monitoring gaps, service delivery wanted by-customer slices. Storing the full catalog and filtering at query time meant the system worked for any future consumer without re-running the sync.

**Two interfaces, one source of truth.** The form is for analysts who just need to look something up. The API is for automations that need programmatic access. Both read from `asset_catalog`, so the freshness and semantics are identical. Drift between human and machine views was a real risk if these had been separate pipelines.

**POST with JSON body for the API, not GET with query params.** With twelve filter fields plus a limit, GET with query string would have been ugly and broken on values containing spaces or special characters. POST with a typed JSON body is what consumers expect from query APIs at this complexity (Elasticsearch, GitHub Search, Algolia all use this pattern).

**Memory-safe binary handling.** The naive way to read a CSV in n8n is `Buffer.from(input.binary.data.data, 'base64')`. For files that n8n stores in filesystem mode (anything over 5 MB), this returns a 6-byte stub instead of the full file. The fix is `await this.helpers.getBinaryDataBuffer(0, 'data')`, which reads the actual binary regardless of storage mode. Took an embarrassing amount of time to debug; documented prominently in the parse code so the next person doesn't trip on it.

**Public API contract, internal fields stripped.** The data table has internal fields (`id`, `createdAt`, `updatedAt`) that consumers shouldn't depend on. The API explicitly whitelists 17 public fields and strips everything else from the response. This makes the contract stable and lets the underlying schema evolve without breaking consumers.

**Error branch with clean JSON output.** Without a dedicated error trigger and 500-response path, consumers would see raw n8n stack traces on failure. Adding a clean `{success: false, error: {...}}` shape makes the API behave like a real API.

## What changed as a result

- Security built their scanner-enrichment integration against the API in days, not weeks. They went from waiting on data dumps to self-serving via webhook.
- Operations got a non-technical-friendly form interface. Service delivery managers can pull customer-specific lists without involving anyone.
- The automation team went from 100% of asset data requests routed through them to roughly 0%. Every new consumer is decoupled.
- Daily refresh confirmed via Microsoft Teams notifications (success path) and error notifications (failure path) keep the team informed without alert fatigue.

## My role

Built and owned this end to end:

- Designed the architecture (three workflows, shared data table, twin interfaces)
- Built and shipped all three workflows
- Negotiated requirements with security, operations, and the datalake team
- Wrote internal documentation and tutorial videos
- Set up monitoring (Teams notifications for success and failure paths)
- Handed off to consumer teams with API key distribution via the company secrets manager

Beyond the code, a meaningful part of the work was cross-team coordination: scoping requirements with stakeholders who had different mental models of "what the data should do," handling a curveball mid-project (an assumed integration field turned out not to exist in the source, which broke the original plan and required renegotiating scope), and getting alignment between three teams that didn't normally talk to each other.

## What's in this repo

```
.
├── workflows/
│   ├── Asset_Catalog_Sync.public.json          # Workflow 1: scheduled sync
│   ├── Asset_Catalog_Query_Form.public.json    # Workflow 2: human-facing form
│   └── Asset_Catalog_Query_API.public.json     # Workflow 3: machine-facing API
├── images/
│   └── architecture.png                        # high-level flow diagram
└── README.md
```

All three workflows are sanitized for public release. Internal URLs, credential IDs, webhook UUIDs, data table IDs, team and people names, and Microsoft Teams identifiers have been replaced with placeholders.

## How to reuse these workflows

If you want to adapt this to your own organization:

1. **Set up an n8n data table** named `asset_catalog` (or rename throughout) with appropriate columns. The default schema includes 17 string fields covering identity, type, status, ownership, service grouping, and security flags.
2. **Create credentials**:
   - Header Auth credential for your datalake API
   - Header Auth credential for your query API key (`x-api-key`)
   - Replace `REPLACE_WITH_YOUR_CREDENTIAL` references with your credential IDs
3. **Replace placeholders**:
   - `YOUR_DATALAKE_API_HOST` with your datalake URL
   - `YOUR_DATALAKE_QUERY_NAME` with your export query name
   - `REPLACE_WITH_YOUR_DATA_TABLE_ID` in Data Table nodes
   - `REPLACE_WITH_YOUR_TEAM_ID`, `REPLACE_WITH_YOUR_CHANNEL_ID`, `REPLACE_WITH_YOUR_TENANT_ID` in Microsoft Teams nodes
4. **Generate an API key** for the query webhook:
   ```bash
   openssl rand -hex 32
   ```
   Store in your secrets manager.
5. **Adjust the parse logic** in the sync workflow if your source CSV has different columns or different upstream typos to fix.
6. **Activate** the sync workflow first, wait for one successful run, then activate the form and API.

## Patterns worth noting

A few things that generalize beyond this specific data type.

### The "full mirror, filter on read" pattern

When consumer needs are heterogeneous or unknown at build time, ingest the full source and filter at query time. More storage, vastly more flexibility. Premature filtering at ingest is a common mistake that locks the system to a single use case.

### The "twin interfaces" pattern

Pair a form for humans with an API for machines, both reading the same source. Same filter semantics in both. Same data freshness in both. Documentation and trust both improve when consumers can verify that the form and the API return identical data for the same query.

### The "stable contract, internal flexibility" pattern

The API explicitly whitelists what fields it returns. The underlying data table can grow new columns without breaking consumers. Adding a new filter field is non-breaking; renaming or removing existing fields is breaking. Document this contract clearly.

### The "every request becomes a ticket" antipattern

If your team is being asked the same data question more than three times, build a self-serve interface. The marginal cost of building one form is lower than the cumulative cost of answering ten ad-hoc requests. The harder argument is convincing yourself that the form is worth building when each individual request takes only ten minutes.

## Stack

- **n8n** for workflow orchestration, scheduling, form rendering, webhook hosting, and data tables
- **JavaScript** in n8n Code nodes for parsing, filtering, and response building
- **Microsoft Teams** for success and error notifications (replaceable with email, Slack, or any other channel)
- **A datalake of your choice** as the upstream source (the pattern works for any system that exposes a CSV export via API)

No external services, no Python, no database beyond n8n's built-in data tables. Everything runs inside one n8n instance.

## Limitations

- **Performance.** With 88k rows, full-table scans for every query take a few seconds. For larger catalogs (millions of rows) you'd want a proper database with indexes. n8n data tables are fine for tens of thousands of rows, not millions.
- **Sync latency.** The pattern refreshes once a day. Real-time data requires a different architecture (CDC, webhooks from the source system, or polling at higher frequency).
- **Single-tenant auth.** Authentication is a shared API key, not per-consumer keys. For multi-tenant access with audit trails, you'd add a small auth layer in front.
- **Source-specific limitations carry through.** Whatever the source dataset doesn't carry, this layer can't expose. If a needed field isn't in the upstream export, it has to be added there first.

## Known issues in this version

A few things I'd fix if I were shipping this fresh today, kept here transparently rather than silently patched.

- **The "polling" loop isn't really polling.** The current sync uses a single 5-second wait, then checks the export status, then either proceeds or logs an error. If the export takes longer than 5 seconds (which an 18 MB export occasionally does), the workflow falsely reports an error when it should keep waiting. A real polling loop with a maximum retry count and exponential backoff is the correct fix.
- **The error branch has dual semantics.** The error notification fires both for genuine datalake failures and for "report not ready yet" cases. The notification message references `$json.error.message`, which doesn't exist on the not-ready path, so you'd see "Error: undefined" in Teams. Should be split into two branches or renamed to reflect what it actually catches.
- **CSV parser is naive about quoted newlines.** The parser splits on `\r?\n` first and then parses each line, which breaks if a field contains an embedded newline inside quotes. Robust fix is a proper streaming CSV parser (`csv-parse` library or a stateful character-by-character parser). For the source data this hasn't bitten us yet, but the risk is real.
- **Limit semantics differ between Form and API.** Form default 100 / cap 1000. API default 1000 / cap 5000. Field name is `result_limit` in the form and `limit` in the API. Anyone using both interfaces will trip over this. Should be unified.
- **No top-level error trigger on the sync.** The sync workflow only catches errors on specific branches. If the very first HTTP request fails (auth, timeout, 5xx), there's no global notification. A workflow-level Error Trigger would close that gap.

## What I'd improve if rebuilding

- **Better observability**: structured logging with request IDs, latency tracking per query, separation between sync failures and query failures
- **Workflow versioning**: Git-backed workflow definitions instead of n8n's internal versioning, with PR-based change review
- **Integration tests**: synthetic queries running on a schedule to catch silent breakage (schema drift, auth misconfiguration, datalake export format changes)
- **Per-consumer API keys**: with rate limits and audit logs per consumer team, not just a shared key
- **A small caching layer**: for the most common queries to avoid full table scans on every request

## License

MIT. See LICENSE file. Use freely. Attribution appreciated but not required.

