# Dubai Rent Renewal Negotiator

An [n8n](https://n8n.io/) chat workflow, powered by **K2 Horizon**, that helps Dubai tenants negotiate their lease renewal: it pulls live DLD-registered rental transactions for the tenant's area via RapidAPI, reasons about whether the current rent is fair, drafts a polite negotiation email in Arabic, then posts it back into the chat ready to copy into your mail client.

**Live workflow (published):** https://benschiller.app.n8n.cloud/workflow/355PFlJoG8wOOL7e

## How it works

```text
Chat Trigger ──▶ K2 Parse Request ──▶ Resolve Location ──▶ Fetch Transactions ──▶ Shape Market Data
                                                                                        │
                                                                                        ▼
Copy Email ◀── Output Arabic Email Text ◀── K2 Draft Email ◀── K2 Fairness Verdict ◀────┘
```

The canvas is organised into four labeled groups: **Parse & Locate**, **Market Scan**, **Fairness Analysis & Draft**, **Deliver negotiation email**.

> **No human-in-the-loop review — intentional.** An earlier build paused at a review gate (Chat `sendAndWait`: type `APPROVE` or paste your edited email). For ease of judging and to speed the demo to conclusion that step was eliminated; the drafted email is delivered straight to the chat. A sticky note on the canvas says the same.

| Node | Type | What it does |
| --- | --- | --- |
| On New Chat Message | Chat Trigger (v1.5) | Hosted chat, `responseMode: responseNodes`; receives the free-text description of your home and rent |
| K2 Parse Request | HTTP Request | K2 Horizon extracts `area`, `bedrooms`, `bathrooms`, `areaSqft` and `annualRentAED` from the message |
| Resolve Location | HTTP Request | `GET /autocomplete` resolves the stated area to a location externalID (e.g. Dubai Marina → `5003`); city-wide fallback if the area can't be resolved |
| Fetch Transactions | HTTP Request | Queries DLD-registered Ejari rental transactions (`GET /transactions`: `purpose=for-rent`, `location_ids=<externalID>`, `beds`, `area_min/max` in sqft); returns registered rents with tower, date and New/Renewal status |
| Shape Market Data | Set (`executeOnce`) | Single-item choke point: compact comps JSON (`rentAED`, `bedrooms`, `areaSqft`, tower, date), the parse JSON, and the original message |
| K2 Fairness Verdict | HTTP Request | Verdict (fair / above / below market), % vs the transactions median, 3–5 cited transactions with registered rents, a recommended target range |
| K2 Draft Email | HTTP Request | Polite, culturally appropriate Arabic negotiation email referencing the real transactions (subject line first, no preamble) |
| Output Arabic Email Text | Set | Unwraps the drafted email out of the K2 response into a plain `emailText` field |
| Copy Email | Chat (`send`) | Posts the finished Arabic email back into the chat for copy/paste |

## Example run (verified live)

Input typed into the chat:

```text
i live in dubai marina, 1 bed / 1 bath, 900 sq ft. i pay aed 160k / yr
```

K2 parse output:

```json
{"area":"Dubai Marina","areaSlug":"dubai-marina","bedrooms":1,"bathrooms":1,"areaSqft":900,"annualRentAED":160000}
```

The chat reply is the finished Arabic email (subject line first) — copy it into your mail client.

## Setup (from a clean n8n instance)

1. In n8n, import [`workflow.json`](workflow.json) (workflow menu → **Import from File**).
2. Create two credentials (**Credentials → Add credential → "Header Auth"** — create from the Credentials page, not from inside a node):
   - **IFM K2 Horizon API** — Name: `Authorization`, Value: `Bearer <your K2 Horizon API key>`
   - **RapidAPI Key** — Name: `x-rapidapi-key`, Value: `<your RapidAPI key>` (subscribed to `happyendpoint/uae-real-estate3` on the hub, otherwise every call 403s)
3. Open each HTTP Request node and select the matching credential in the dropdown (the K2 nodes → IFM credential; Resolve Location and Fetch Transactions → RapidAPI Key). Both RapidAPI nodes also enable **Send Headers** with `x-rapidapi-host: uae-real-estate3.p.rapidapi.com` (the host is public, not a secret — Header Auth only carries one header).
4. Open the chat (the workflow's chat panel, or n8n Chat Hub) and describe your home, e.g. the message above.
5. Copy the Arabic email the chat replies with into your own mail client.

> **Why credentials and not env vars?** Verified live on n8n Cloud: `$env` access in expressions is blocked (`access to env vars denied`) and environment variables cannot be set from the Cloud dashboard. Header Auth credentials keep the keys out of the workflow JSON, so this repository contains no secrets.

## Design notes

- **K2 Horizon**: all 3 LLM steps POST to `https://api.ifm.ai/v1/chat/completions` with model `IFM/K2-Horizon-375B-A23B` (OpenAI-compatible). Request bodies are built with a single `JSON.stringify({...})` expression so arbitrary user text cannot break the JSON. Capped `max_tokens` per step (parse 500, verdict 800, draft 1500) with low temperatures (0 / 0.2 / 0.6). 240 s timeout per call; all three retry 3× (5 s apart) on transient failures.
- **RapidAPI transactions**: `GET /transactions` on `uae-real-estate3.p.rapidapi.com` with `purpose=for-rent`, `location_ids=<autocomplete externalID>`, `beds`, `area_min/max` in sqft (see [`docs/rapidapi-transactions.md`](docs/rapidapi-transactions.md)). Bedroom filter uses `beds` (`rooms` is ignored); records carry no bathroom field, so baths can't filter. Auth needs two headers (`x-rapidapi-key` credential + `x-rapidapi-host` send-header). 24 results/page. No scraping, no bot wall — DLD-registered Ejari data.
- **Location resolution**: the area name is resolved via `GET /autocomplete` and its `externalID` (e.g. Dubai Marina → `5003`); the numeric `id` is ignored by the transactions filter.
- **Tight comps**: same area + same beds + sqft window ±20% around the tenant's size (e.g. 700–1100 for 900 sqft). Marina 1-beds, Sep 2026: broad median AED 87k, size-matched median AED 100k.

## What was tested

- **Graph tests** with pinned data: comps shaping (array and envelope shapes) feeds the verdict prompt correctly.
- **Live data check**: RapidAPI `/transactions` queried directly from the local machine — Dubai Marina 1-beds, 20/20 area-matched, all registered within days (broad median AED 87k, size-matched median AED 100k). Full contract in [`docs/rapidapi-transactions.md`](docs/rapidapi-transactions.md).
- **K2 step tests** (each node's exact prompt, direct calls): parse ~7s, verdict cites live comps (60% above market, median 100k, towers named), draft ~15s. All three K2 nodes retry 3× (5s apart) on transient 503s.
- **End-to-end run** (exec 20, live, earlier build that still had the review gate): parse 6.0s → resolve 0.4s → fetch 2.1s → verdict 12.8s → draft 14.0s, zero errors. Draft needed one fix first: at temp 0.7 the model burned 1000 tokens on exposed reasoning (`finish_reason=length`, no email); temp 0.2 + an explicit output rule now yields a clean Subject-first email.
- **Key validity check**: the K2 API was called directly from the local machine with the Bearer header (HTTP 200).

## Repository layout

```text
.
├── workflow.json                        # Importable n8n workflow (the actual code)
├── docs/
│   ├── dubai-rent-negotiator-plan.md    # Build spec, kept up to date with build-time findings
│   ├── rapidapi-transactions.md         # Live comps API contract (params, field map, verified queries)
│   └── K2-Horizon-Starter-Workflow.json # Minimal K2 request starter (placeholders only)
├── .env                                 # Local secrets; gitignored
├── .gitignore
└── README.md
```

## Security checklist

- `.env` is gitignored — never commit API keys or bearer tokens.
- `workflow.json` contains credential **references** (names/ids) only; credential values live encrypted in the n8n instance.
- Before sharing any n8n export publicly, strip secrets and rotate anything that was previously committed.
