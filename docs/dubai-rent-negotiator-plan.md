# Build Spec: Dubai Rent Renewal Negotiator
## Instructions for AI coding agent (n8n MCP)

Build this entire workflow in n8n. Do not ask me to write code — write it yourself
based on the node purposes described below. Use n8n's native node types wherever
one exists for the job (HTTP Request, Code, IF, Set, Form, Chat Trigger); only
use a Code node where no native node fits.

## Credentials / environment (assume these already exist as n8n credentials or env vars)
- IFM_BASE_URL — K2 Horizon API endpoint
- IFM_API_KEY — K2 Horizon API key
- IFM_MODEL — model identifier string for API calls (e.g. "k2-horizon-375b-a23b")
- APIFY_API_KEY — Apify token

## Overview of what this workflow does
A user describes their Dubai residence and current annual rent in plain
English. The workflow finds real comparable rental listings for that area,
uses K2 Horizon to reason about whether the rent is fair and draft a
culturally-sensitive negotiation email in English, lets the user review and
edit that draft, translates the approved version into Arabic with an English
gloss, runs a safety check on the result, and outputs final plain text
(Arabic email + English gloss) for the user to copy and paste into their own
email client.

## Instructions for AI coding agent (n8n MCP)
**Status: v4 (built & tested)** — Chat (not Form) trigger; HITL verified against node source; Apify body corrected to the documented actor-input shape; credential-based auth for n8n Cloud / public repo.

## Workflow (built)
- n8n workflow **Dubai Rent Renewal Negotiator** — `355PFlJoG8wOOL7e` — https://benschiller.app.n8n.cloud/workflow/355PFlJoG8wOOL7e (personal project)
- Graph tests passed (comps shaping, APPROVE and edited-text gate branches); live K2 test pending the two credentials being filled.

## Credentials (n8n Cloud-safe; no secrets in the repo)
Verified live on n8n Cloud: `$env` access in expressions is **blocked** (`access to env vars denied`) and env vars cannot be set from the dashboard, so the workflow uses **Templated Custom Auth credentials** instead — keys live only in encrypted n8n credentials, so the exported workflow JSON is safe for a public GitHub repo:
- Credential `IFM K2 Horizon API` — Custom Auth (templated), default template `{"headers":{"Authorization":"Bearer {{api_key}}"}}`, paste the K2 API key into the **API key** field. Attached to all 5 K2 HTTP nodes.
- Credential `Apify API` — same kind, paste the Apify token. Attached to the Apify node.
- K2 endpoint and model are hardcoded in the nodes: `https://api.ifm.ai/v1/chat/completions`, model `IFM/K2-Horizon-375B-A23B` (no env needed).
- `N8N_ACCESS_TOKEN` — n8n MCP access (not used inside the workflow itself; keep in `.env`, which is gitignored).
- On self-hosted n8n, real env vars (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) remain a valid alternative.

## Key API decisions (settled)

### K2 Horizon call (confirmed against starter workflow)
- URL: `https://api.ifm.ai/v1/chat/completions`
- Method: POST
- Headers: `Authorization: Bearer {{ $env.IFM_API_KEY }}`, `Content-Type: application/json`
- Body: `{"model": "IFM/K2-Horizon-375B-A23B", "messages": [{"role":"user","content": <prompt>}]}`
- Response content read from: `$json.choices[0].message.content`
- All 4 K2 steps use this same node shape; only the prompt differs.

### Apify actor: `gio21/bayut-property-scraper`
Run it **synchronously** so one HTTP Request returns the dataset items directly
(no polling, no second call). Verified against the Apify API docs and the
actor's input schema: the POST payload is passed to the Actor **as its input
directly** — no `body` wrapper, no `waitForFinish` body field (the endpoint
itself waits for the run to finish, up to 300 s, then answers 408):
- URL: `https://api.apify.com/v2/actors/gio21~bayut-property-scraper/run-sync-get-dataset-items`
- Method: POST
- Headers: `Authorization: Bearer {{ $env.APIFY_API_KEY }}`, `Content-Type: application/json`
- Body (flat actor input):
  ```json
  {
    "searchUrl": "https://www.bayut.com/for-rent/property/dubai/<area-slug>/",
    "maxItems": 24
  }
  ```
- Returns: the dataset items themselves — a JSON array of
  `{price, priceFormatted, bedrooms, bathrooms, area, listingUrl, ...}`. n8n's
  HTTP Request splits a JSON-array response into N items, so the next node must
  be a single-execution choke point. The comp-shaping Set node is built to also
  tolerate an enveloped `{ "data": { "total": N, "items": [...] } }` shape, and
  the Apify node carries `alwaysOutputData` so an empty/captcha scrape still
  flows into a graceful model answer instead of silently dead-ending the chat.
- HTTP Request timeout: 300000 ms (Apify answers 408 at 300 s).
- Risk: Bayut uses DataDome anti-bot; `gio21` has no stated bypass. If captcha
  returns, fall back to the hardened `get_anything/bayut-property-scraper`
  (Camoufox-based, ~$6/1,000 results) — keep the same flat input shape.

### Native human-in-the-loop text review (n8n standard pattern)
The **Chat node** (`@n8n/n8n-nodes-langchain.chat`) with:
- `operation: sendAndWait`
- `responseType: freeTextChat`
- `message: "<draft text + instruction to review/edit>"`

It sends the draft into the chat and **pauses the workflow execution** until the user replies with their edited text. The reply is available on the node's output and feeds the next step. Requires Chat Trigger `options.responseMode: "responseNodes"`. No separate form, webhook, or approval node needed — this is the built-in, industry-standard way.

Verified against the node source (`Chat.node.ts`): for node version >= 1.2 the resumed output puts the user reply in top-level `$json.chatInput` (it does **not** nest under a `data` key), so the following IF reads `($json.chatInput || '').trim().toUpperCase()`.

## Revised workflow (all native nodes: Chat Trigger, Chat, HTTP Request, Set, IF)

1. **Chat Trigger** — hosted chat (n8n's generated chat UI / Chat Hub). Receives a free-text description of the residence + current annual rent, e.g. `i live in dubai marina, 1 bed / 1 bath, 900 sq ft. i pay aed 160k / yr`. `options.responseMode: "responseNodes"`, `public: false`.

2. **HTTP Request → K2 #1 (Parse & locate).** Prompt extracts structured data from the free text and builds the Bayut rent search URL:
   `{"area":"...", "areaSlug":"...", "bedrooms":N, "bathrooms":N, "areaSqft":N, "annualRentAED":N, "searchUrl":"https://www.bayut.com/for-rent/property/dubai/<areaSlug>/"}`
   (fall back to city-wide `/for-rent/property/dubai/` if the area can't be resolved; downstream reads `searchUrl` out of the parse output with a regex extraction plus a city-wide fallback, so markdown fences or chatter cannot break the scrape).

3. **HTTP Request → Apify** (sync, above) — scrapes real comps for that area; returns the dataset items (an array response is split into N items by n8n; `alwaysOutputData` keeps an empty/captcha scrape flowing).

4. **Set → Shape Market Data** (`executeOnce: true`) — single-item choke point after the scrape: compact comps JSON (array or `{data:{items}}` envelope tolerated), the raw K2 #1 parse JSON, and the original chat message.

5. **HTTP Request → K2 #2 (Fairness reasoning).** Inputs: comps + lease data. Output: verdict (fair / above / below market), a % figure vs the comps median, and 3–5 cited comparable listings with their prices.

6. **HTTP Request → K2 #3 (Draft email).** Inputs: reasoning + comps + original message. Output: culturally-sensitive English negotiation email referenced to real listing evidence.

7. **Chat node — REVIEW (HITL).** `sendAndWait` / `freeTextChat`: sends "Here is the draft. Review and paste your edited version (or type APPROVE): <draft>". **Pauses until user replies.** The resumed output exposes the reply as top-level `$json.chatInput` (verified).

8. **IF → Approved or Edited?** True when the reply (trimmed, uppercased) is `APPROVE`: reuse the draft unchanged; else use the user's edited text. Both branches wire into K2 #4, whose prompt conditionally picks the draft or the edited reply — no extra Set nodes needed, and the canvas stays within the top-level box budget.

9. **HTTP Request → K2 #4 (Translate).** Inputs: approved English text. Output: Arabic negotiation email plus an English gloss (side by side / in-line).

10. **HTTP Request → K2 #5 (Safety check).** Inputs: Arabic + gloss. Verifies tone/formality, translation accuracy against the English source, and flags any problematic content. Returns the final approved plain text.

11. **Chat node — FINAL OUTPUT.** `send` (or end-of-flow reply): emits the final plain text — Arabic email + English gloss — as a simple copy/paste message the user drops into their own email client.

## Node inventory
- `@n8n/n8n-nodes-langchain.chatTrigger` (v1.5) — hosted chat, `options.responseMode: "responseNodes"`
- `@n8n/n8n-nodes-langchain.chat` (v1.3) ×2 — sendAndWait/review + final send
- `n8n-nodes-base.httpRequest` ×6 — Apify + 5 K2 calls (K2 timeout 240 s, Apify timeout 300 s; all authenticated via Templated Custom Auth credentials)
- `n8n-nodes-base.set` ×1 — Shape Market Data (single-execution choke point after the scrape: comps JSON, K2 #1 parse JSON, original message)
- `n8n-nodes-base.if` ×1 — if the user typed APPROVE, reuse the draft unchanged; else use their edited text
- Sticky notes ×3 — how to run, scraping risk / fallback actor, how the review wait works

No Code node, no Form trigger, no DB/data-table persistence (the review round-trip is handled in-flow by the Chat node's wait). The comp-shaping Set uses `executeOnce` because n8n splits an array HTTP response into one item per listing — without it every downstream node (including the review chat) would fire once per listing.

## Resolved at build time
- K2 `/chat/completions` request body confirmed against `K2-Horizon-Starter-Workflow.json`: built with a single `JSON.stringify({model, messages:[{role:"user",content}]})` expression; response read from `$json.choices[0].message.content`.
- Apify body corrected from the v2 sketch (wrapped `{"body":...}` + `waitForFinish`) to the documented flat actor input `{searchUrl, maxItems}`; the response is the dataset items themselves (array → n8n splits into items; the `{data:{items}}` envelope is tolerated defensively).
- Chat node `sendAndWait` resumed output confirmed to carry the reply in top-level `$json.chatInput` (node version >= 1.2).
- Env-var auth abandoned after a live test returned `access to env vars denied` on n8n Cloud; replaced with Templated Custom Auth credentials.