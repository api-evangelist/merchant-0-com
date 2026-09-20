---
name: merchant-0-com-verify-agent-did
description: Buy a USD 0.25 verification of another agent's did:web identity and A2A registry standing from Merchant-0, or issue a 24-hour signed proof of your own subscription tier for a counterparty.
api: openapi/merchant-0-com-openapi.json
operations:
  - get_service_catalog_api_catalog_get
  - ap2_negotiate_alias_api_ap2_negotiate_post
  - ap2_sign_alias_api_ap2_sign_post
  - ap2_execute_alias_api_ap2_execute_post
  - get_intel_result_api_intel__execution_id__get
method: generated
generated: '2026-09-19'
grounding: Every operationId above exists verbatim in openapi/merchant-0-com-openapi.json. The per-SKU execute fields are quoted from the AP2ExecuteBody schema description and the agent card's capability descriptions.
---

# Verify a counterparty DID, or prove your own tier

Both products run through the same negotiate -> sign -> execute flow described in `merchant-0-com-buy-intelligence.md`; only the SKU and one execute field differ. Base URL `https://api.merchant-0.com`, no API key, identify yourself by `buyer_did` in the body.

## A. Verify another agent (`merchant0-verify-001`, USD 0.25 per check)

1. Confirm the SKU is still `purchasable` in `GET /api/catalog` (`get_service_catalog_api_catalog_get`).
2. `POST /api/ap2/negotiate` (`ap2_negotiate_alias_api_ap2_negotiate_post`) with `{"buyer_did": "did:web:you.example", "item_id": "merchant0-verify-001", "quantity": 1}` -> `contract_id`.
3. `POST /api/ap2/sign` (`ap2_sign_alias_api_ap2_sign_post`) with `{"contract_id", "buyer_signature", "buyer_did"}`.
4. `POST /api/ap2/execute` (`ap2_execute_alias_api_ap2_execute_post`) with `{"contract_id": "...", "target_did": "did:web:counterparty.example"}`.
   - `target_did` is "The DID to be verified. **NOT** the buyer's DID." Sending your own DID here verifies yourself and wastes the check.
5. Read the result at `GET /api/intel/{execution_id}` (`get_intel_result_api_intel__execution_id__get`). Per the card the verifier "Resolves /.well-known/did.json and /.well-known/agent-card.json (with agent.json fallback) from the target DID domain, then queries a2aregistry.org", returning `target_did, did_resolved, did_document_valid, agent_card_found, agent_name, a2a_conformant, is_healthy, verification_timestamp, warnings, verified_by`. The execution is persisted with status `verify_delivered`.

Treat `a2a_conformant` as Merchant-0's opinion, not a certification: Merchant-0's own card fails an A2A 1.0.0 hard check (no `protocolVersion`).

## B. Prove your subscription tier (`merchant0-proof-001`, USD 0.05 per proof)

Same steps 1-3 with `item_id: "merchant0-proof-001"`, then execute with `{"contract_id": "...", "subscription_id": "<optional>"}` — "if absent the most-recent active subscription for the buyer is used".

The proof is **stateless** ("each execution generates a fresh proof; not persisted") and returned in the execute response: `proof_id, subject_did, issued_by, issued_at, expires_at` (24 hours from issuance), `subscription {plan_sku, status, usage_count, usage_limit, queries_remaining, next_billing}` or `null`, `lifetime_queries, proof_hash` (sha256 truncated to 32), `merchant_signature`. The card does not publish the key that produces `merchant_signature`; the DID document at `https://merchant-0.com/.well-known/did.json` carries one Ed25519 key (`#key-1`) that a verifier would try first.

## Rules that apply to both

- Each negotiate creates a new PENDING contract; there is no cancel. Sign is the only idempotent write.
- No refund path and no stated window. A wrong `target_did` is a spent USD 0.25.
- 402 = proof-of-work challenge for flagged buyers (retry with `X-PoW-Solution`); 403 = `sentinel_blocked`; 404 = unknown SKU; 429 = plan quota exhausted.
