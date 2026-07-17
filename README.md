```
lautaro@v0id:~/n8n-automation-suite$
```

# n8n Automation Suite for MSP Operations

A set of n8n workflows, built on PostgreSQL, MySQL, Redis and LangChain, extracted from a managed service provider (MSP) automation stack that runs in production. They cover asset inventory reconciliation across multiple monitoring tools, on-demand report generation with a webhook/callback pattern, ticket sync from a legacy CRM, proactive notifications, monitoring ingestion, self-backup of the automation layer itself, and a WhatsApp-based AI agent stack (conversational NOC assistant, LLM-as-judge inventory reconciliation, dedup subworkflow).

This is a curated, sanitized subset of a larger private system. All hostnames, credentials, and tenant-specific data have been replaced with placeholders. The SQL, control flow, and data-shaping logic are unmodified and reflect how the workflows actually run.

## Why this exists

MSPs typically run several independent tools per client: an inventory agent (OCS Inventory), an EDR/antivirus console (ESET PROTECT), a monitoring system (Zabbix), and a CRM/ticketing system (vTiger). None of these agree with each other on device identity, and none of them expose a unified view on their own. This suite is the reconciliation and reporting layer that sits on top: it merges inventory sources into one row per physical device, keeps ticket data warm in a reporting-friendly schema, and turns all of it into on-demand client reports without touching the source systems.

## Architecture

```
                     ┌─────────────────┐   ┌──────────────────┐
                     │   OCS Inventory  │   │  ESET PROTECT API │
                     │   (agent-based)  │   │  (multi-tenant)   │
                     └────────┬─────────┘   └─────────┬────────┘
                              │ REST                   │ OAuth2
                     ┌────────▼─────────────────────────▼────────┐
                     │        inventory-sync-* workflows          │
                     │   (fetch, normalize, upsert per source)    │
                     └────────────────────┬────────────────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │  inventory-merge-dedupe   │
                              │  (3-tier match: hardware  │
                              │   UUID → MAC → hostname)  │
                              └────────────┬──────────────┘
                                           │
   ┌───────────────────┐        ┌──────────▼──────────┐        ┌────────────────────┐
   │  legacy CRM (MySQL) │──────▶│   PostgreSQL cache    │◀──────│  Zabbix JSON-RPC    │
   │  crm-ticket-sync    │       │   (reporting schema)  │       │  monitoring-sync    │
   └───────────────────┘        └──────────┬──────────┘        └────────────────────┘
                                           │
                              ┌────────────▼──────────────┐
                              │   report-generation         │
                              │   webhook → SQL modules →   │
                              │   assemble → callback PATCH │
                              └────────────┬────────────────┘
                                           │
                              ┌────────────▼────────────┐
                              │   report/CRM backend      │
                              │   (external HTTP consumer)│
                              └──────────────────────────┘

   ┌──────────────────────┐        ┌───────────────────────────┐
   │ cron-ticket-          │        │ workflow-git-backup        │
   │ notifications          │        │ (n8n API → GitHub content  │
   │ (WhatsApp via          │        │  API, diff-aware upsert)   │
   │  Evolution API)        │        └───────────────────────────┘
   └──────────────────────┘
```

Everything downstream of the sync workflows reads from Postgres, never from the source APIs directly. That keeps report generation fast and decouples it from third-party rate limits and outages.

## Workflows

### `inventory-sync-ocs.json`
Pulls the full computer inventory from an OCS Inventory server (paginated REST API) and normalizes it in a single Code node before upserting into Postgres. Handles a long tail of real-world OCS data quality issues: virtual/duplicated MAC prefixes (VPN adapters, hypervisor NICs), generic OEM serial numbers ("To Be Filled By O.E.M.", "0123456789"), SMART disk health parsing, and SSD/HDD classification from model-name regexes when the OS doesn't report a type. Trigger: schedule + manual.

### `inventory-sync-eset-devices.json`
Fetches endpoint device inventory from ESET PROTECT for every active tenant stored in a `tenants` table, authenticating per-tenant against ESET's OAuth2 token endpoint before paging through the devices API. Interesting pattern: credentials live in the database, not in the workflow, so onboarding a new tenant is a SQL insert, not a workflow edit.

### `inventory-merge-dedupe.json`
The reconciliation core. A single Postgres `DO` block that upserts OCS-only rows, then enriches them with ESET data using a three-tier matching strategy, tried in order until one hits:

1. **Hardware UUID** (SMBIOS UUID from BIOS, the most reliable identifier)
2. **MAC address**, with known virtual/hypervisor OUI prefixes filtered out and `DISTINCT ON` dedup on both sides to avoid `ON CONFLICT` violations from duplicate matches
3. **Hostname**, scoped to the same client (so `PC01` at Client A never matches `PC01` at Client B)

Runs guarded by a Postgres advisory lock so overlapping schedule triggers can't race each other, and skips truncating the detections cache if the upstream ESET sync looks stale, to avoid wiping good data on a bad run.

### `report-generation.json`
The report engine, triggered by a webhook from the CRM. Resolves which report modules apply to the client's contract, runs one SQL query per module (inventory, security, SLA, servers, availability, backups) in parallel, transforms each into report-ready shape, merges them back together, and assembles the final payload. It then calls back into the report backend over HTTP with the assembled data (PATCH) rather than returning it synchronously in the webhook response, since some modules can take longer to compute than an HTTP client wants to wait. A companion HTTP call handles maintenance-log fetches by scope (workstations/servers/both) for a given period.

### `crm-ticket-sync.json`
Syncs tickets, entity relationships, and change-tracking events from a legacy vTiger CRM (MySQL) into the Postgres reporting cache, then recomputes aggregates. Three independent MySQL → Code → Postgres pipelines feeding the same cache schema, so a failure in one (e.g. ticket sync) doesn't block the others.

### `cron-ticket-notifications.json`
A scheduled job that finds technicians with active tickets assigned and sends each of them a single grouped WhatsApp message (via Evolution API) instead of one message per ticket, capped and summarized when the list is long. Groups by phone number, translates CRM status codes to human-readable labels, and dedupes tickets already seen in the same run.

### `workflow-git-backup.json`
Nightly self-backup of the n8n instance: lists all workflows via the n8n API, then for each one, checks whether a matching file already exists in a GitHub repo (via the Contents API), diffs the base64-encoded content, and only pushes a commit if something actually changed. Skips no-op writes, so the repo's commit history reflects real workflow edits rather than a daily bot commit.

### `monitoring-zabbix-sync.json`
Pulls open problems from Zabbix's JSON-RPC API on a schedule and upserts them into Postgres for use in the reporting layer, keeping report generation independent of Zabbix's own uptime.

### `ai-technician-agent.json`
A conversational NOC agent triggered by an incoming WhatsApp message. A LangChain AI Agent node (Google Gemini, with OpenRouter wired in as a second model) carries Redis-backed chat memory and calls out to four tool sub-workflows exposed as callable tools: an ad-hoc SQL runner against the Postgres reporting cache (tickets, FS, projects, companies, contacts), a Zabbix alerts query tool, a domain-expiry lookup, and an n8n error-metrics reporter. The system prompt pins down exact column names and known query pitfalls for both the Postgres cache and the legacy MySQL CRM, plus a strict per-report-type JSON contract (tickets/projects/FS/alerts/etc), so a downstream Code node can reliably regroup rows (e.g. one row per FS or per project task) into a readable WhatsApp reply before it goes out through the messaging gateway. Trigger: sub-workflow, called from the WhatsApp inbound handler.

### `ai-data-reconciliation.json`
An LLM-as-judge pass over the asset inventory, run on demand from the report backend. It pulls the pre-merged OCS/ESET device rows for a client plus the contracted device counts, then asks an AI agent, constrained to a structured output schema, to collapse rows that represent the same physical machine using a stated identity heuristic (hardware UUID with endianness tolerance, a non-junk BIOS serial, or adjacent MAC addresses sharing a serial), while telling it up front which UUID/serial values are known junk values to disregard. The agent returns a canonical device list, a diff against contracted counts, and a list of detected inconsistencies, which get merged into the report payload and PATCHed back to the report backend. Trigger: webhook.

### `whatsapp-dedup-subworkflow.json`
A small reusable dedup gate: takes a message ID, attempts a `SET NX` in Redis with a TTL, and reports back whether the message had already been seen. Meant to sit at the top of any WhatsApp-triggered workflow so retried or duplicate deliveries from the messaging gateway's webhook don't get processed twice. Trigger: sub-workflow.

## Interesting patterns, if you're skimming

- **Dedupe via `DISTINCT ON` + `ON CONFLICT`**: several workflows normalize noisy upstream data (MACs in different cases/separators, empty-string vs. NULL) into a single canonical row before upsert, using `DISTINCT ON` to guarantee at most one candidate per conflict key so Postgres doesn't reject the batch with "ON CONFLICT DO UPDATE command cannot affect row a second time."
- **Webhook + callback instead of synchronous response**: `report-generation` responds to its trigger immediately and reports results asynchronously via an HTTP callback, which is the only way to reconcile "fast webhook ack" with "some reports take 30+ seconds to compute."
- **Per-tenant credentials from the database**: the ESET sync reads OAuth credentials per tenant from Postgres instead of hardcoding one set of n8n credentials per client, so onboarding a new tenant doesn't require touching the workflow.
- **Diff-aware git backup**: the backup workflow fetches the existing file before writing, compares content, and skips the write if nothing changed, so the backup repo's history is meaningful instead of a wall of identical daily commits.
- **Advisory locks for idempotent cron jobs**: the merge workflow takes a Postgres advisory lock before running, so a slow run and its next scheduled trigger can't stomp on each other.
- **AI Agent node + memory + tools**: the technician agent is a single LangChain Agent node wired to a chat model, Redis-backed session memory, and several `toolWorkflow` nodes, each pointing at a separate n8n workflow described only by its natural-language tool description. Adding a new capability to the agent is "write a sub-workflow and describe it," not "touch the agent's code."
- **LLM-as-judge for data QA**: rather than trying to get a fully deterministic dedup rule to cover every messy real-world case, the reconciliation workflow hands the AI agent a pre-filtered candidate set plus explicit identity rules and known-junk values, and constrains its output to a JSON schema so the result can be merged back programmatically. The LLM is used for judgment on ambiguous cases, not as a free-form chatbot.
- **Reusable dedup subworkflow**: the WhatsApp dedup gate is a 3-node subworkflow (Redis `SET NX` + a code node to interpret the result) meant to be called from any inbound-message workflow, so retried gateway webhooks don't get processed twice without duplicating that logic per workflow.

## Stack

- **Orchestration**: n8n (self-hosted)
- **Database**: PostgreSQL (reporting cache), MySQL (legacy CRM, read-only source)
- **Monitoring/inventory sources**: OCS Inventory NG, ESET PROTECT Cloud API, Zabbix
- **Messaging**: Evolution API (WhatsApp)
- **AI/LLM**: Google Gemini and OpenRouter as LangChain chat models, Redis for agent session memory and message dedup
- **Backup target**: GitHub Contents API

## Setup

These are n8n workflow exports, not a standalone application. To run them:

1. Import each `workflows/*.json` file into an n8n instance (`Import from File`). All eleven ship with `active: false`, since they reference placeholder credentials that don't exist yet in a fresh instance.
2. Create the following credentials in n8n and reassign them on each imported workflow:
   - `postgres-main` / `postgres-crm` (PostgreSQL)
   - `mysql-crm-legacy` (MySQL)
   - `github-token` (GitHub personal access token with repo contents write)
   - `zabbix-api-token`, `whatsapp-api-token` (HTTP Header Auth)
   - `n8n-api` (n8n public API key, for the backup workflow)
   - `google-gemini-api`, `openrouter-api` (LangChain chat model credentials)
   - `redis-main` (Redis, for agent memory and WhatsApp dedup)
3. Replace the placeholder hosts (`your-ocs-server.example.com`, `your-zabbix-server.example.com`, `your-crm.example.com`, `your-report-server.example.com`) with your real endpoints.
4. Replace `CHANGE_ME_CALLBACK_SECRET` and `CHANGE_ME_USER:CHANGE_ME_PASSWORD` with real values, ideally moved into n8n credentials rather than left inline.
5. Provision the referenced Postgres schema (`cache_ocs_computers`, `cache_eset_devices`, `cache_inventario`, `cache_tickets`, etc). A schema migration is not included in this repo.
6. Once credentials and endpoints are in place, activate each workflow manually from the n8n UI.

There's no `.env.example` here on purpose: these are n8n workflow exports, not application code reading `process.env`. Every secret (DB connection, API tokens, callback secret) lives in n8n's own credential store, referenced by name from each node.

## Status

Portfolio/demo project. This is a sanitized excerpt of a private, in-production automation stack, published to show real automation design patterns (dedupe strategy, async callback reporting, per-tenant credential handling). It is not runnable as-is without the underlying database schema and live service endpoints, and it is not maintained as an open-source project.

## License

MIT, see [LICENSE](LICENSE).
