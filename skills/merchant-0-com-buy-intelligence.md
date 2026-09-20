---
name: merchant-0-com-buy-intelligence
description: Discover a Merchant-0 intelligence SKU, negotiate it, sign, execute with your trade question, and read the Grok deliverable back — with the quota, review-gate, dispute and privacy rules the contract states.
api: openapi/merchant-0-com-openapi.json
operations:
  - get_service_catalog_api_catalog_get
  - ap2_negotiate_alias_api_ap2_negotiate_post
  - ap2_sign_alias_api_ap2_sign_post
  - ap2_execute_alias_api_ap2_execute_post
  - get_intel_result_api_intel__execution_id__get
  - get_ap2_review_status_api_ap2_review__contract_id__get
  - get_subscription_usage_by_id_api_subscriptions__subscription_id__usage_get
  - ap2_dispute_file_api_ap2_dispute_post
method: generated
generated: '2026-09-19'
grounding: Every operationId above exists verbatim in openapi/merchant-0-com-openapi.json. Quoted behaviour comes from that contract's descriptions and from the provider's agent card.
---

# Buy an intelligence query or market report from Merchant-0

Base URL `https://api.merchant-0.com`. No API key exists. You identify yourself by sending your own `did:web` DID in the request body; the provider never echoes it back (every read returns `buyer_did_hash`).

## Before you spend anything

- The A2A card's `url` answers **405** to JSON-RPC and `/message/send` returns a fixed acknowledgement for every method. Purchases happen over these REST calls only.
- Prices in the agent card are stale (the card says intel-001 costs 0.99; the live catalog says 2.99). Always price from the catalog.
- **There is no refund, cancel or void operation and no stated window.** The only recourse after execute is a dispute. Do not execute anything you are not prepared to pay for.
- Deals **>= USD 100** are held for a "Strategist war-game" review before sign/execute; the recommendation arrives in the negotiate response and the state is readable at `GET /api/ap2/review/{contract_id}`.

## 1. Discover

`GET /api/catalog` (`get_service_catalog_api_catalog_get`), optionally `?category=intelligence`. Use only rows with `"purchasable": true`. On 2026-09-19 the intelligence SKUs were `merchant0-intel-001` (2.99/query, max 8k input tokens), `merchant0-report-001` (4.90/report), `merchant0-hs-monitor-001`, `merchant0-freight-intel-001`, `merchant0-bilateral-flow-001`.

If you hold a subscription, check `GET /api/subscriptions/{subscription_id}/usage` (`get_subscription_usage_by_id_api_subscriptions__subscription_id__usage_get`) first — negotiate returns **429** when the plan's 200 monthly credits are exhausted.

## 2. Negotiate

`POST /api/ap2/negotiate` (`ap2_negotiate_alias_api_ap2_negotiate_post`)

```json
{"buyer_did": "did:web:your-agent.example", "item_id": "merchant0-report-001", "quantity": 1}
```

- **404** = unknown `item_id`; re-read the catalog.
- **402** = you are a FLAGGED buyer and received a proof-of-work challenge (SHA-256, difficulty 4); retry with the solution in the `X-PoW-Solution` header.
- **403** `sentinel_blocked` = the semantic firewall matched an injection pattern in your request; it fails closed only on a confirmed match.
- **200** = a PENDING contract with Diplomat-personalised terms (`unit_price_usd`, `total_price_usd`). Keep `contract_id`.

## 3. Sign

`POST /api/ap2/sign` (`ap2_sign_alias_api_ap2_sign_post`) with `{"contract_id": "...", "buyer_signature": "...", "buyer_did": "did:web:your-agent.example"}`.

The contract does not define what `buyer_signature` signs or with which key; it says only that the signature's length is logged, never the material. Sending `buyer_did` lets the Advocate run its BUYER_MISMATCH check. **This step is idempotent** — re-signing an already SIGNED contract is a documented no-op. It is the only idempotent write in the API.

## 4. Execute

`POST /api/ap2/execute` (`ap2_execute_alias_api_ap2_execute_post`)

```json
{"contract_id": "...", "query": "Q3 2026 palm-oil supply-chain risk, Indonesia to India", "report_type": "regulatory"}
```

- `query` is **required for report-001** to dispatch Grok; `report_type` is `trade_route | regulatory | arbitrage` and "currently advisory only".
- `dry_run: true` "skips invoice / subscription side effects; delivery handlers still run" — a rehearsal that still runs the model. Whether it consumes a credit is not stated.
- Execute is **not** documented as idempotent. If the call times out, do not blindly retry; read the contract's review state and your usage before sending it again.
- The response carries `settlement_confirmed` (Wise auto-confirmation) per the card. Keep `execution_id`.

## 5. Read the deliverable

`GET /api/intel/{execution_id}` (`get_intel_result_api_intel__execution_id__get`). A report contains `report_title, executive_summary, market_analysis, risk_assessment, opportunities, regulatory_context, recommended_actions, data_sources, validity_window` (field list from the card's capability description).

## 6. If something is wrong

`POST /api/ap2/dispute` (`ap2_dispute_file_api_ap2_dispute_post`) with `{"buyer_did", "contract_id", "execution_id", "dispute_type", "description"}`. Only the contract owner can file. The contract does not say what a resolution can do or how long you have.

## Error envelope

FastAPI `{"detail": ...}`; 422 carries `detail[]` of `{loc, msg, type}`. Not RFC 9457. No rate-limit headers exist; 429 is the only quota signal.
