# Orchestrator 01 — Lead Discovery

An automated lead-generation pipeline built in n8n that scrapes local businesses matching a target profile, extracts structured contact details with an AI model, and pushes clean, deduplicated records into HubSpot CRM as Contacts, Companies, and Deals.

🎥 **Demo video:** [add your Loom link here]

## What it does

1. **Search** — Queries [Firecrawl](https://firecrawl.dev)'s search API for businesses matching a configurable service + city + state + country (e.g. "skin clinics in Hyderabad").
2. **Filter** — Excludes aggregator/directory domains (Practo, JustDial, YouTube, social platforms, etc.) so only businesses' own websites are scraped.
3. **Scrape** — Fetches each remaining site's content via Firecrawl.
4. **Extract** — Uses Mistral AI (via n8n's Information Extractor node) to pull business name, phone, email, address, and services from the raw page content.
5. **Route** — Every lead with a business name is kept; leads with an email get a full Contact record, leads without one still get a Company + Deal record so no lead is dropped.
6. **Push to CRM** — Creates/updates the corresponding Contact, Company, and Deal in HubSpot, with owner auto-assignment.

## Tech stack

- **n8n** — workflow orchestration
- **Firecrawl** — web search + scraping
- **Mistral AI** (`mistral-medium-latest`) — structured data extraction from unstructured page content
- **HubSpot CRM API** (Private App token) — Contacts, Companies, Deals

## CRM fields populated

| Object | Fields |
|---|---|
| Company | Name, Phone, Website, Street Address, Email (custom property), Owner |
| Contact | Email, Phone, Company Name, Website, Owner |
| Deal | Deal Name, Stage, Owner |

## Setup

1. Import `Orchestrator-01-Lead-Discovery.json` into n8n.
2. Connect credentials:
   - Firecrawl API key
   - Mistral Cloud API key
   - HubSpot Private App token (scopes: `crm.objects.companies.read/write`, `crm.objects.contacts.read/write`, `crm.objects.deals.read/write`, `crm.objects.owners.read`)
3. In HubSpot, create a custom **single-line text** company property named `email` (HubSpot doesn't ship one by default) if you want email visible on Company records.
4. Adjust the `Edit Fields` node at the start of the workflow to target your desired service/city/state/country.
5. Run manually via the "Execute workflow" trigger.

## Notable engineering details

- **Cross-node field scoping**: HubSpot "create" nodes return their own API response as the item's data, which silently breaks any downstream node that reads fields by implicit `$json` reference. Fixed by explicitly sourcing lead fields via `$('prepare lead record').item.json.*` in every node after the first HubSpot call, rather than relying on the previous node's output.
- **Graceful degradation**: contact creation requires an email (HubSpot upserts contacts by email), but company/deal creation doesn't — the workflow branches so leads without an email still land in the CRM instead of being silently dropped.
- **Rate-limit resilience**: the AI extraction step retries with backoff (5 tries, 5s apart) on transient 429s from the LLM provider instead of failing the whole run.
- **Real filtering**: aggregator/directory URLs and failed scrapes are dropped before reaching the AI extraction step, rather than being passed through and cleaned up later.
