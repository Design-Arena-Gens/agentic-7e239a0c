## Automation Architecture — Facebook Ads ➝ WhatsApp Ecommerce Pipeline

This repository defines a complete n8n automation system that replaces manual WhatsApp selling and Hifinox CRM for a Ghana-based ecommerce business. It contains workflow specifications, node definitions, trigger wiring, API call contracts, and data schemas so an agent can generate the n8n workflows programmatically and deploy them on a self-hosted n8n instance alongside WhatsApp Cloud API, Google Sheets, Telegram Bot, and OpenAI LLM integrations.

### Folder Layout
- `workflows/` — JSON workflow blueprints grouped by functional area.
- `schemas/` — Data models and Google Sheets layouts.
- `env/` — Environment variable contract.
- `docs/` — Supplemental logic charts and error handling notes.

### Deployment Checklist
1. Provision n8n with credentials listed in `env/variables.env`.
2. Import JSON workflow definitions under `workflows/`.
3. Create the Google Sheet defined in `schemas/orders_sheet.json` with worksheet name `ORDERS`.
4. Configure Telegram bot webhook pointing to the n8n Telegram Trigger node URL.
5. Populate OpenAI prompt assets from `docs/prompts/`.

Detailed workflow breakdowns, node ordering, conditional branches, and retry logic are documented in the `workflows/*.json` files and referenced diagrams in `docs/`.
