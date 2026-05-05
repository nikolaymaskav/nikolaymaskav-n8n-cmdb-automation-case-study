# n8n Automation Platform: Case Study

A year of building and shipping automation on n8n inside a mid-sized cloud MSP. This repo is a portfolio overview of the work, with sanitized versions of the production workflows and a deeper case study on the headline project.

## What this is

I'm a Cloud Operations Specialist who spent the last year of my role going deep on automation engineering with n8n. The work covered three rough buckets: building production pipelines that other teams depend on, helping non-developers across the company build their own workflows, and writing the documentation and tutorials that made it stick.

The numbers below are the platform's aggregate output across many workflows, not a single project. The asset catalog case study (linked below) is one example, the most substantial one, broken out in detail.

## Impact from my time with tooling in Q1-Q2  



- ~4,800 production executions across the platform
- ~382 hours of manual work removed
- 0.8% failure rate
- ~9 second average runtime

The shift behind the numbers matters more: people who had never built automation before started shipping their own workflows instead of opening tickets and waiting.

## What I built and shipped

The work fell into three categories.

### 1. Production pipelines

End-to-end automation that other teams depend on daily. Things like:

- An asset catalog query layer that pulls roughly 100K records from a datalake every morning and exposes them via a self-serve form for analysts and a JSON API for automations. Used by security teams for scanner enrichment and by operations for ad-hoc lookups. 
- Scheduled coverage and compliance reports against the same data, delivered to Teams channels.
- Various smaller integrations between internal systems (ticketing, monitoring, datalakes) that used to require ticket-based requests.

### 2. Enablement for non-developers

A lot of teams wanted automation but had no way to build it themselves. I worked alongside them to scope, build, and hand off workflows so they could maintain and extend them without me.

- Direct pairing sessions with service delivery, operations, and security teams
- Documentation on the company knowledge base covering workflow patterns, n8n basics, and the specific platforms (datalake, internal APIs, Teams) that matter for our environment
- Tutorial videos on common workflow patterns
- Office-hours-style availability for ad-hoc questions in a dedicated Teams channel

### 3. Cross-team coordination

The harder half of the work, and the half that doesn't show up on a CV unless you spell it out.

- Negotiated requirements with stakeholders who had different mental models of what the data should do
- Got alignment between teams that didn't normally talk to each other (security, ops, datalake, monitoring, infra)
- Skipped the ticket-based comms model where I could and went directly to team leads
- Kept the conversation going when scope shifted mid-project (one assumed integration approach turned out to require fields the source didn't carry, which forced a renegotiation rather than a silent rebuild)

## How the platform is set up

```
   Datalake          Internal systems     Email / Teams
        │                  │                   │
        └──────────┬───────┴───────────────────┘
                   ▼
            ┌────────────┐
            │    n8n     │  ← runs on Kubernetes / OpenShift
            │  (shared)  │
            └─────┬──────┘
                  │
            ┌─────▼──────┐
            │ PostgreSQL │  ← state, queue, data tables
            └────────────┘
                  │
            ┌─────▼──────┐
            │  Consumers │  ← humans (forms), automations (webhooks)
            └────────────┘
```

The default is one shared n8n instance with workflows organized by team or domain. Isolated instances exist for specific compliance reasons but the shared instance carries most of the volume.

The pattern across most workflows is the same: pull from a source, store in a queryable layer (n8n data tables backed by Postgres), expose through either a form or an authenticated webhook depending on who's consuming.

## Featured case study: Asset Catalog Query Layer

The most substantial single project from this period. Three workflows that turn a corporate asset catalog from a datalake export nobody could query into a self-serve query layer with two interfaces.

**Why it matters as a case study:**

- Real production scope (~88k records, refreshed daily, used across multiple teams)
- Demonstrates the "twin interfaces, one source of truth" pattern that generalizes to almost any data product
- Forced real engineering decisions (read-time vs ingest-time filtering, GET vs POST for query APIs, memory-safe binary handling for large CSVs)
- Required cross-team negotiation when an assumed integration field turned out to be missing from the source


## Where the time actually went

Honest breakdown, in case anyone reading this is trying to estimate effort for a similar role:

- ~30% on building production pipelines
- ~25% on cross-team coordination, scoping, and stakeholder alignment
- ~20% on documentation and tutorials
- ~15% on supporting non-developer teams who were trying to build their own workflows
- ~10% on platform care: monitoring, error handling, schema changes, dependency upgrades

The "build the thing" portion is the smallest of the five. That surprised me too.

## Things that turned out to matter

A few lessons that aren't obvious until you're a year in.

**Self-serve beats faster ticket response.** Teams asked me to "speed up the data pull request turnaround." Wrong target. The right move was building a form anyone could use, which dropped the request count to roughly zero. Optimizing the wrong layer is easy to do.

**Documentation is part of the deliverable, not a follow-up.** A workflow without docs gets used by you and nobody else. A workflow with a knowledge-base page gets used by the rest of the team. The marginal hour spent writing the page is probably the most leveraged hour in the project.

**Tickets are a tax on collaboration.** Most cross-team work moved 5x faster when I went directly to a team lead in Teams instead of opening a ticket. Tickets exist for compliance and audit, not for getting things done.

**Trust compounds across projects.** One team had been burned by a previous internal project and refused to engage on a new initiative. Spending real time with them early, including them in decisions, and crediting their work publicly turned them into one of the most reliable collaborators by the end. That trust then carried into the next two projects with them.

**n8n's quirks are real but workable.** Memory limits on large binaries, sequential upserts being slow, the difference between filesystem and memory mode for binary data, these eat hours if you don't know about them. I documented every gotcha I hit so the next person on the platform doesn't lose the same days I did.

## Limitations of the platform (honest)

- **n8n data tables don't scale to millions of rows.** Tens of thousands is fine. Past that you want a real database with indexes.
- **Sequential upserts are slow.** 88k rows takes roughly 50 minutes in n8n's data tables. For workflows where freshness matters more, you'd want batch inserts or skip n8n's data tables entirely.
- **Workflow versioning is weak.** n8n has internal version history but nothing like Git-backed change review. If I rebuilt the platform from scratch, the workflows would live in Git with PR-based review.
- **Multi-tenancy auth is shared-key, not per-consumer.** Fine for trusted internal consumers, not fine if you need rate limiting or audit per team.

## What I'd do differently if I started over

- **Git-backed workflow definitions from day one.** n8n supports JSON exports; the discipline to commit them and review changes via PR was missing in our setup, and I'd insist on it next time.
- **Integration tests for the consumer-facing APIs.** A scheduled synthetic query that hits the API once an hour and pages me on failure would have caught at least two silent breakages this year.
- **Per-consumer API keys with rate limits.** Shared key works until it doesn't; setting up a small auth layer in front would cost a day and save a quarter of "who's hammering the API today" debugging.
- **Better metrics from the start.** The dashboard at the top of this README came together late. Earlier visibility into execution counts, failure rates, and per-workflow trends would have made platform health easier to argue for.

## Tech stack

- **n8n** for orchestration, scheduling, forms, webhooks, and data tables
- **PostgreSQL** for n8n state and the data tables that back the workflows
- **Kubernetes / OpenShift** for hosting
- **Microsoft Teams** for notifications and team-facing comms
- **A datalake** as the upstream source for most data pipelines (your source can be anything that exposes an API)
- **JavaScript** in n8n Code nodes for the parsing, filtering, and response logic

No external services beyond what's listed. No Python in this stack, though I use it elsewhere.

## Repository layout

```
.
├── asset-catalog/
│   ├── README.md                                 # detailed case study
│   ├── workflows/
│   │   ├── Asset_Catalog_Sync.public.json
│   │   ├── Asset_Catalog_Query_Form.public.json
│   │   └── Asset_Catalog_Query_API.public.json
│   └── images/
├── images/
│   └── metrics.png                               # n8n execution dashboard
├── LICENSE
└── README.md                                     # this file
```

All workflows in this repo are sanitized. Internal URLs, credential IDs, webhook UUIDs, data table IDs, Microsoft Teams identifiers, and team or people names have been replaced with placeholders. The structure and logic match the production versions.

## Contact

Open to remote automation engineering and adjacent roles. Email and LinkedIn in my GitHub profile.

