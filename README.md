# Dubai Rent Renewal Negotiator

An [n8n](https://n8n.io/) chat workflow, powered by **K2 Horizon**, that helps Dubai tenants negotiate their lease renewal: it scrapes live comparable listings from Bayut, reasons about whether the current rent is fair, drafts a polite negotiation email, pauses for your review in the chat, then delivers a bilingual (Arabic + English) email ready to send.

**Live workflow (published):** https://benschiller.app.n8n.cloud/workflow/355PFlJoG8wOOL7e

## How it works

```text
Chat Trigger ──▶ K2 Parse Request ──▶ Apify Fetch Bayut Comps ──▶ Shape Market Data
                                                                      │
                       (pauses until you reply: APPROVE or your edit) │
K2 Fairness Verdict ◀─────────────────────────────────────────────────┘
      │
      ▼
K2 Draft Email ──▶ Review Draft (Chat, sendAndWait) ──▶ Approved or Edited? (IF)
                                                            │ true: reuse draft
                                                            │ false: use edited text
                                                            ▼
                              K2 Translate Arabic ──▶ K2 Safety Check ──▶ Send Final Email
```

| Node | Type | What it does |
| --- | --- | --- |
| On New Chat Message | Chat Trigger (v1.5) | Hosted chat, `responseMode: responseNodes`; receives the free-text description of your home and rent |
| K2 Parse Request | HTTP Request | K2 Horizon extracts `area`, `areaSlug`, `bedrooms`, `bathrooms`, `areaSqft`, `annualRentAED` and builds the Bayut `searchUrl` (city-wide fallback if the area can't be resolved) |
| Apify Fetch Bayut Comps | HTTP Request | Runs `gio21/bayut-property-scraper` synchronously (flat actor input `{searchUrl, maxItems: 24}`); returns the dataset items themselves |
| Shape Market Data | Set (`executeOnce`) | Single-item choke point: compact comps JSON (tolerates a bare array **or** a `{data:{items}}` envelope), the parse JSON, and the original message |
| K2 Fairness Verdict | HTTP Request | Verdict (fair / above / below market), % vs the comps median, 3–5 cited listings with prices, a recommended target range |
| K2 Draft Email | HTTP Request | Polite, culturally appropriate English negotiation email referencing the real listings |
| Review Draft | Chat (`sendAndWait`, freeText) | Sends the draft into the chat and **pauses the execution** until you reply |
| Approved or Edited? | IF | True when the trimmed, uppercased reply is `APPROVE` → reuse the draft; otherwise the reply is the edited email |
| K2 Translate Arabic | HTTP Request | Modern Standard Arabic translation + an English gloss after `---ENGLISH GLOSS---` |
| K2 Safety Check | HTTP Request | Quality gate: checks tone/formality and translation accuracy, returns the final approved plain text |
| Send Final Email | Chat (`send`) | Delivers the Arabic email + English gloss as a copy/paste message |

## Example run (verified live)

Input typed into the chat:

```text
i live in dubai marina, 1 bed / 1 bath, 900 sq ft. i pay aed 160k / yr
```

K2 parse output:

```json
{"area":"Dubai Marina","areaSlug":"dubai-marina","bedrooms":1,"bathrooms":1,"areaSqft":900,"annualRentAED":160000,"searchUrl":"https://www.bayut.com/for-rent/property/dubai/dubai-marina/"}
```

After typing `APPROVE` at the review step, the final delivery is the safety-checked bilingual email (Arabic first, then `---ENGLISH GLOSS---`, then the English gloss) — 2,307 characters in the verified run.

## Setup (from a clean n8n instance)

1. In n8n, import [`workflow.json`](workflow.json) (workflow menu → **Import from File**).
2. Create two credentials (**Credentials → Add credential → "Header Auth"** — create from the Credentials page, not from inside a node):
   - **IFM K2 Horizon API** — Name: `Authorization`, Value: `Bearer <your K2 Horizon API key>`
   - **Apify API** — Name: `Authorization`, Value: `Bearer <your Apify token>`
3. Open each HTTP Request node and select the matching credential in the dropdown (the 5 K2 nodes → IFM credential; the Apify node → Apify credential).
4. Open the chat (the workflow's chat panel, or n8n Chat Hub) and describe your home, e.g. the message above.
5. At the review step, paste your edited email or type `APPROVE` (the wait limit is 45 minutes by default).
6. Copy the final bilingual email into your own mail client.

> **Why credentials and not env vars?** Verified live on n8n Cloud: `$env` access in expressions is blocked (`access to env vars denied`) and environment variables cannot be set from the Cloud dashboard. Header Auth credentials keep the keys out of the workflow JSON, so this repository contains no secrets.

## Design notes

- **K2 Horizon**: all 5 LLM steps POST to `https://api.ifm.ai/v1/chat/completions` with model `IFM/K2-Horizon-375B-A23B` (OpenAI-compatible). Request bodies are built with a single `JSON.stringify({...})` expression so arbitrary user text cannot break the JSON. 240 s timeout per call.
- **Apify**: the actor runs synchronously via `POST /v2/actors/gio21~bayut-property-scraper/run-sync-get-dataset-items` — the POST payload is passed to the actor as its input directly (verified against the Apify API docs), and the response is the dataset items themselves. The node carries `alwaysOutputData` so an empty/captcha scrape still flows into a graceful model answer instead of dead-ending the chat. 300 s timeout (Apify answers 408 at 300 s).
- **Robust parsing**: the parse output's `searchUrl` is extracted with a regex (not `JSON.parse`), so markdown fences or model chatter cannot break the scrape request.
- **Fallback scraper**: Bayut uses DataDome anti-bot protection; `gio21` has no stated bypass. If the scrape returns a captcha page or no items, swap the actor for `get_anything/bayut-property-scraper` (Camoufox-based, ~$6 per 1,000 results) and keep the same flat input.

## What was tested

- **Graph tests** with pinned data: comps shaping (array and envelope shapes) and both gate branches (`APPROVE` and an edited reply) route correctly.
- **Live end-to-end run** (5 real K2 Horizon calls; scraper pinned to sample data): success in ~3.5 minutes — parse, verdict, draft, translation, and the final safety-checked bilingual email.
- **Key validity check**: the K2 API was called directly from the local machine with the Bearer header (HTTP 200).

## Repository layout

```text
.
├── workflow.json                        # Importable n8n workflow (the actual code)
├── docs/
│   ├── dubai-rent-negotiator-plan.md    # Build spec, kept up to date with build-time findings
│   └── K2-Horizon-Starter-Workflow.json # Minimal K2 request starter (placeholders only)
├── .env                                 # Local secrets; gitignored
├── .gitignore
└── README.md
```

## Security checklist

- `.env` is gitignored — never commit API keys or bearer tokens.
- `workflow.json` contains credential **references** (names/ids) only; credential values live encrypted in the n8n instance.
- Before sharing any n8n export publicly, strip secrets and rotate anything that was previously committed.
