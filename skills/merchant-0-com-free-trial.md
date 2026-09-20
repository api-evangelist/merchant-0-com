---
name: merchant-0-com-free-trial
description: Claim Merchant-0's one-per-DID, 24-hour free trial of the Grok intelligence query and run one real query at USD 0.00 before committing to a paid contract.
api: openapi/merchant-0-com-openapi.json
operations:
  - post_trial_grant_api_ap2_trial_grant_post
  - post_trial_execute_api_ap2_trial_execute_post
  - get_service_catalog_api_catalog_get
method: generated
generated: '2026-09-19'
grounding: Every operationId above exists verbatim in openapi/merchant-0-com-openapi.json. Terms are quoted from the two trial operations' descriptions.
---

# Try the intelligence query for free

The trial is the only zero-cost way to see a real deliverable. It covers `merchant0-intel-001` only. Base URL `https://api.merchant-0.com`; no API key.

## 1. Claim the grant

`POST /api/ap2/trial-grant` (`post_trial_grant_api_ap2_trial_grant_post`)

```json
{"buyer_did": "did:web:your-agent.example", "referral_code": ""}
```

Terms, quoted: "Public (any buyer agent). One trial per DID, expires 24h, not for FLAGGED DIDs. Rule #11: only buyer_did_hash in response / logs." Keep the returned `trial_id` and `trial_nonce` — the nonce is the credential for step 2 and the response shape is otherwise undeclared. A second grant for the same DID is refused; the status code is not documented.

## 2. Run the query

`POST /api/ap2/trial-execute` (`post_trial_execute_api_ap2_trial_execute_post`)

```json
{"trial_id": "...", "trial_nonce": "...", "query": "Novelty scan: EV battery component flows Thailand -> Vietnam, last 90 days"}
```

Quoted: "Use a trial grant: real Grok intel, $0.00 audit row ... Public (authenticated by trial_nonce). Grok only -- never Perplexity. Does NOT increment buyer_reputation.total_executions (acquisition discount preserved)."

The paid product caps input at 8k tokens (agent card); assume the same here.

## 3. Decide

Compare the output with the paid SKU's list price in `GET /api/catalog` (`get_service_catalog_api_catalog_get`) — 2.99 per query on 2026-09-19, or 200 credits/month for 49.00 on `merchant0-intel-sub-monthly`. Then follow `merchant-0-com-buy-intelligence.md`.

## What the trial does not give you

- No trial for reports, verification or proofs.
- No sandbox: the trial hits the production Grok pipeline, and everything else you touch is live.
- No way to check your own trial state — `GET /api/trial/status` is an operator route (401 without the operator token).
