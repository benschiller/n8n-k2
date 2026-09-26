# Live comps source: RapidAPI UAE Real Estate 3 (HappyEndpoint)

Replaces the Apify Bayut scraper (blocked by Bayut's DataDome challenge — both
`gio21` and `easyapi` actors return `[]`). This API serves **DLD-registered
Ejari rental transactions** as JSON: actual transacted rents, stronger
negotiation evidence than scraped asking prices.

## Access

- Hub: `https://rapidapi.com/happyendpoint/api/uae-real-estate3`
- Host: `https://uae-real-estate3.p.rapidapi.com`
- Subscribe to a plan on the hub first — without it every call 403s
  (`{"message":"You are not subscribed to this API."}`). Same `RAPID_API_KEY`
  works once subscribed; key lives in `.env` (gitignored).
- Every request needs **two** headers:
  - `x-rapidapi-key: <key>` (secret → n8n Header Auth credential)
  - `x-rapidapi-host: uae-real-estate3.p.rapidapi.com` (public → n8n Send Headers)

## Endpoints used

### `GET /autocomplete?query=<area name>`

Resolves a tenant-stated area to a location ID. Use the **`externalID`**
field, not `id`:

```json
{ "id": 36, "externalID": "5003", "name": { "en": "Dubai Marina" } }
```

Dubai Marina → `5003`. Cache per area — it never changes.

### `GET /transactions`

Registered sale/rent transactions. Verified working query (Sep 2026):

```
GET /transactions?purpose=for-rent&time_period=12m&page=1&location_ids=5003&beds=1
```

| Param | Notes (verified by probing — trust this, not guesses) |
|---|---|
| `purpose=for-rent` | Required. |
| `time_period=12m` | Lookback window. |
| `page=1` | 1-based, 24 results/page. |
| `location_ids` | **External IDs** from `/autocomplete` (e.g. `5003`). The numeric `id` (`36`) and the `locations_ids` spelling are silently ignored → city-wide results. |
| `beds` | Bedroom filter. `rooms` is silently ignored — do **not** use it. |
| `area_min` / `area_max` | **Square feet** (a 70–100 probe returned nothing; 700–1100 returned 20/20). Window ≈ ±20% around the tenant's sqft. |
| `baths` | **Unavailable.** Transaction records carry no bathroom field — cannot filter. |

## Response field map (per hit → comp)

| API field | Comp field | Example |
|---|---|---|
| `transaction_amount` | `rentAED` (annual) | `100000.00` |
| `beds` | `bedrooms` | `1` |
| `builtup_area_sqm` | `areaSqft` = `sqm × 10.7639` | `84.45` → `~909` |
| `bayut_leaf_location_name_en` | `tower` | `Sanibel Tower` |
| `date_transaction_nk` | `date` | `2026-09-24` |
| `rent_contract_renewal_status_name` | `status` | `New` / `Renewal` |
| `contract_monthly_amount` | cross-check | `2362.50` |

## Tight-comps recipe (the three categories)

1. **Area** — tenant area → `/autocomplete` → `location_ids=<externalID>`.
2. **Bed/bath** — `beds=<n>` (baths: skip, not in records).
3. **Square footage** — tenant sqft ±20% → `area_min`/`area_max` (sqft, not sqm).

Example tenant (Marina, 1-bed, 900 sqft, 160k/yr) → 20/20 Marina 1-beds,
700–1100 sqft, all registered within days. Broad median AED 87k;
size-matched median AED 100k — both far below 160k.

## Cost

One workflow run = 1 transactions call (+1 autocomplete, cacheable).
Billable per RapidAPI plan quota, not per result — far cheaper than
pay-per-event scraper actors, with zero bot-wall risk.

## Reliability notes (observed Sep 2026)

- The provider flaps: intermittent `403`/`502` bursts (`API unreachable`)
  amid healthy `200`s, with quota headers fine (e.g. 155+/200 remaining).
  Treat these as transient — retry, don't redesign.
- The n8n `Fetch Transactions` node therefore uses `onError:
  continueRegularOutput`, and the verdict prompt degrades gracefully on `[]`.
- Area-filtered queries showed denser flak bursts than unfiltered ones in
  one session; if this recurs, move the sqft window client-side (Shape)
  and query location+beds only.
