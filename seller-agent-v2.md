---
title: Seller integration for Market Protocol v0.2 — Seller Commerce API v2 wire guide
id: coa.market.seller-commerce.v2
contract_version: 2
audience: seller-agent and seller-service implementers
format: Markdown with JSON examples
updated: 2026-10-15
status: production integration guide; enable each seller after its own acceptance checks
---

# Seller integration for Market Protocol v0.2

*Wire contract: Seller Commerce API v2*

Canonical URL: https://www.commerceofagents.com/seller-agent-v2.md
Companion note: https://www.commerceofagents.com/market-protocol-0-2.html

This document describes the **seller-hosted HTTP adapter** used by a Market game such as Agents of Mind. It is distinct from the Game Master's `market/*` event API. The seller remains authoritative for its catalog, customer entitlements, and fulfillment; Market owns accepted orders, payment routing, attribution, and its ledger. Market Protocol v0.2 names the Game Master commerce protocol. The seller-hosted adapter has its own v2 wire contract, with the concrete `Seller-Commerce-Version: 2` and `contract_version: 2` identifiers.

Yvatar is a production seller example: its v2 manifest advertises `one-time` billing for creation-credit purchases, and its service fulfills those purchases. LIAM is an example of an independent software vendor (ISV) offering one-time software products and recurring subscriptions; its Market integration is not yet active in production. The sample catalog and identifiers below illustrate both billing paths. They are not the actual catalog of either seller.

## Start here with Claude Code or Codex

Open your product's repository in Claude Code or Codex, give the coding agent this guide's URL, and ask it to implement the seller integration in your existing stack. The coding agent builds the adapter and its tests; your deployed service and scheduled or continuously running seller worker operate the integration. An LLM is not required to process each purchase.

### Copy this implementation prompt

```text
Read https://www.commerceofagents.com/seller-agent-v2.md and inspect this repository.
Implement a seller-direct integration for our product using its existing web
framework, authentication, billing, customer accounts, entitlement model,
database, background jobs, and test framework.

First show an implementation plan and identify business decisions, configuration,
or access you still need from me. Map every advertised SKU to its actual price,
billing mode, recipient identity, and entitlement. Advertise only implemented
features. Keep Market Protocol v0.2 and Seller Commerce API v2 wire identifiers
distinct, and preserve any v1 fulfillment endpoints still needed by the relay.

Implement the manifest, catalog, checkout preparation, lifecycle processing,
and durable fulfillment query/ack endpoints. Keep the seller-service credential
separate from the seller agent's Market credential. Verify payment evidence,
persist idempotency and entitlement/outbox effects atomically, and acknowledge
fulfillment events only after Market accepts the matching order update.

Build or adapt the seller worker for catalog synchronization and fulfillment
reporting using the selected seller seat's discovered Market capabilities.
If access or deployed lifecycle details are missing, list the exact questions
for the GM operator instead of inventing commands or payloads.

Create runnable tests in this repository for the acceptance matrix in this guide.
Use isolated fixtures and Stripe test mode where Stripe is the payment provider.
Show the commands, results, migrations, configuration variable names, supported
features, and remaining operator steps. Keep secrets out of code and test output.
Prepare reviewable changes; coordinate live credentials, real payments, and
production activation with me and the GM operator.
```

### Information the developer supplies

The agent can discover much of this from your code. Confirm the remaining decisions before it implements billing and fulfillment:

| Information | What to supply or confirm |
| --- | --- |
| Offerings | Stable SKUs, descriptions, prices/currencies, quantities, and which offerings are ready to sell. |
| Billing | One-time, recurring, or free; supported intervals; subscription groups; upgrade/downgrade and cancellation behavior. |
| Customer identity | How an email, player identity, or other supported recipient maps to an existing or newly provisioned customer. |
| Entitlements | Exactly what each purchase grants, when it becomes usable, and how usage and reversals are recorded under the applicable terms. |
| Payments | Existing provider integration, Stripe test account and Price references where applicable, and the payment route coordinated with the GM. |
| Persistence and jobs | Where checkout reservations, processed events, entitlement changes, and the outbox live; how the seller worker runs and retries. |
| Configuration and access | Test adapter URL, secure credential configuration, and operator-provided Market access for the intended seller seat. Exchange secrets through the agreed secure channel. |
| Deployment | Hosting, database migration process, operational logs, and who reviews and deploys the changes. |

Commission rates and settlement obligations come from your accepted commercial terms and the GM's private configuration. The sample amounts and application fees below are illustrative, not defaults to copy into your business agreement.

### Implementation sequence and deliverables

1. **Assess the repository.** Produce a mapping of existing product, billing, account, entitlement, and worker code to the integration responsibilities. List unresolved business decisions and operator dependencies.
2. **Build the seller service.** Implement the HTTP endpoints below with the existing database and credential infrastructure. Document migrations and configuration variable names.
3. **Connect durable delivery.** Make payment/lifecycle processing, entitlement effects, and outbox writes safe under retries and crashes. Demonstrate how unsupported events are rejected.
4. **Build or adapt the seller worker.** Synchronize the public catalog into seller-owned Market capabilities and relay fulfillment for that seller's orders. The worker uses its own authorized Market identity and discovers current commands and schemas.
5. **Verify in isolation and with the operator.** Run local tests, then exercise the supported purchase paths against the GM's coordinated test configuration. Record evidence for the acceptance matrix in section 9.
6. **Complete onboarding and activation.** Follow section 10 for AoM, or obtain the equivalent process from another GM. Deliver a reviewed deployment and operational runbook before enabling paid checkout.

The seller-hosted adapter can be developed before GM access is available. Catalog synchronization, Market order updates, and end-to-end acceptance need a provisioned seller identity and coordinated GM configuration.

## 1. Roles and trust boundary

1. An operator provisions a seller player/seat, private adapter base URL and service token, supported payment route, connected account if needed, and a private commission agreement. Do not publish service tokens or processor secrets in Market catalog state.
2. A seller agent reads the provider's catalog and declares/updates seller-owned Market capabilities. It queries its own Market orders and reports fulfillment with `market/mark-fulfilled`.
3. The game calls the seller's private v2 endpoints to verify compatibility, read the catalog, prepare a checkout, and deliver payment/subscription/refund/dispute lifecycle facts. The provider validates those facts against its own processor/account where applicable.
4. The seller service grants or reverses entitlements in a durable transaction, and writes fulfillment events to a durable outbox. The seller agent acknowledges an outbox event only **after** the Market order update succeeds.

All v2 calls use HTTPS, `Authorization: Bearer <seller-service-token>`, and `Seller-Commerce-Version: 2`. POST bodies and success responses use `Content-Type: application/json`. Use a credential dedicated to the game/provider relationship. Reject missing or invalid credentials; never infer authorization from a catalog item or a browser session.

Amounts are integer minor units (for USD, cents); currencies are uppercase ISO 4217 codes. Keep Market IDs distinct from Stripe or other provider IDs. Examples use fake identifiers and a `.test` address.

## 2. Endpoint map

| Method | Path | Seller service responsibility |
| --- | --- | --- |
| GET | `/seller-commerce/v2/manifest` | Declare adapter identity and supported features. |
| GET | `/seller-commerce/v2/catalog` | Return versioned SKUs, prices, billing and fulfillment metadata. |
| POST | `/seller-commerce/v2/checkouts/prepare` | Validate the immutable Market quote and bind it to a durable `fulfillment_ref`. |
| POST | `/seller-commerce/v2/lifecycle-events` | Apply a verified economic event exactly once; return any fulfillment events. |
| POST | `/seller-commerce/v2/fulfillment-events/query` | Return unacknowledged events for the caller's specified Market order IDs. |
| POST | `/seller-commerce/v2/fulfillment-events/{event_id}/ack` | Acknowledge an event after Market records fulfillment. |

The current player-agent fulfillment relay can still use Seller Commerce v1 even when the game calls the provider's v2 checkout and lifecycle endpoints. Implement and keep the v1 fulfillment query/ack endpoints until that relay is upgraded and its removal is explicitly approved. Do not silently remove v1 when adding v2.

## 3. Manifest and catalog

`GET /seller-commerce/v2/manifest` must return the configured `adapter_id`, `contract_version: 2`, an environment (`test`, `preview`, or `production`), a checkout UI (`embedded` or `redirect`), nonempty supported `billing_modes` and `recipient_types`, `fulfillment_mode: "provider-outbox"`, and lifecycle types including `payment.confirmed`. Supported billing modes are `one-time`, `recurring`, and `free`; recipient types are `email`, `player`, and `none`. Advertise only features the seller actually implements. The game fails closed on an incompatible manifest.

Example manifest for an ISV that implements both purchase types (illustrative IDs; advertise only deployed capabilities):

```json
{
  "adapter_id": "example-isv",
  "display_name": "Example software vendor",
  "contract_version": 2,
  "environment": "production",
  "checkout_ui": "embedded",
  "billing_modes": ["one-time", "recurring"],
  "recipient_types": ["email"],
  "fulfillment_mode": "provider-outbox",
  "lifecycle_event_types": [
    "payment.confirmed", "subscription.renewed",
    "subscription.plan_changed", "subscription.status_changed",
    "refund.settled", "chargeback.opened", "chargeback.resolved"
  ]
}
```

`GET /seller-commerce/v2/catalog` returns `catalog_version` and `items`. A catalog item has a stable `sku`, title/description, `price`, `billing`, entitlement and fulfillment metadata, and optional images, display order, subscription group, and `price_ref`. `billing` is `"one_time"`, `"free"`, or an interval object such as `{"interval":"month","interval_count":1}`. For a recurring Stripe-backed plan, `price_ref` identifies the seller's immutable Stripe Price. Set `checkout_enabled: false` for a visible preview that must not be purchased.

```json
{
  "catalog_version": 3,
  "currency": "USD",
  "items": [{
    "sku": "credit-pack",
    "title": "One-time credit pack",
    "description": "A single purchase of 3,000 credits",
    "kind": "product",
    "price": {"amount": 3000, "currency": "USD"},
    "billing": "one_time",
    "checkout_enabled": true,
    "entitlements": [{"type": "credits", "amount": 3000}],
    "fulfillment": {"type": "account-entitlement", "recipient": {"type": "email"}},
    "images": []
  }, {
    "sku": "monthly-plan",
    "title": "Monthly software plan",
    "description": "5,000 creation credits each month",
    "kind": "subscription",
    "price": {"amount": 4900, "currency": "USD"},
    "price_ref": "price_example_plus",
    "billing": {"interval": "month", "interval_count": 1},
    "subscription_group": "software-credits",
    "checkout_enabled": true,
    "entitlements": [{"type": "credits", "amount": 5000, "period": "month"}],
    "fulfillment": {"type": "account-entitlement", "recipient": {"type": "email"}},
    "images": []
  }]
}
```

The seller agent should sync catalog changes to Market capabilities using its seller identity, preserving SKU and adapter references. A catalog version or price change must not rewrite already accepted Market orders.

## 4. Checkout preparation

The game calls `POST /seller-commerce/v2/checkouts/prepare` before sending the buyer to payment. Treat `checkout_id` as an idempotency key. Validate the adapter's current SKU, catalog version, recipient, quantity, currency, price and quoted total. Reject a mismatch instead of accepting a changed charge. Persist the reservation and return the **same** opaque `fulfillment_ref` on a replay.

The current game request has snake_case top-level fields and a hyphenated `order-id` inside each `orders` entry. Do not normalize this away when parsing the wire body:

```json
{
  "checkout_id": "checkout_demo",
  "checkout_batch_id": "checkout_group_demo",
  "market_subscription_id": "subscription_checkout_demo",
  "buyer": "buyer_demo",
  "recipient": {"type": "email", "value": "buyer@example.test"},
  "catalog_version": 3,
  "application_fee_bps": 3000,
  "orders": [{
    "order-id": "order_demo",
    "sku": "monthly-plan",
    "quantity": 1,
    "money": {"amount": 4900, "currency": "USD"}
  }]
}
```

Success (`200`):

```json
{"fulfillment_ref":"isv:checkout_demo","recipient":{"type":"email","value":"buyer@example.test"}}
```

For a one-time purchase, prepare the `credit-pack` SKU with its accepted amount and quantity; no recurring plan is created. A seller service should support the order array it advertises or reject unsupported combinations explicitly. `application_fee_bps` is the game's private, quoted commission input, not a reason to alter the buyer's catalog price.

## 5. Lifecycle events and idempotency

The game posts to `POST /seller-commerce/v2/lifecycle-events` after payment verification or another authoritative transition. It sends `Idempotency-Key: <event_id>`; require an exact match to the body `event_id`. Persist the event ID, a digest of its payload, the entitlement/fulfillment effect, and the response atomically. Replays of the same ID and payload return the stored response (for example with `status: "replayed"`); the same ID with different content returns `409 idempotency-conflict`. A timeout may mean the first call committed, so the caller must be able to retry safely.

Example payment confirmation (provider IDs are references, not Market identities):

```json
{
  "event_id": "evt_test_example",
  "event_type": "payment.confirmed",
  "occurred_at": "2026-10-06T12:00:00Z",
  "market": {
    "checkout_id": "checkout_demo",
    "checkout_batch_id": "checkout_group_demo",
    "order_ids": ["order_demo"],
    "subscription_id": "subscription_checkout_demo",
    "buyer": "buyer_demo",
    "fulfillment_ref": "isv:checkout_demo"
  },
  "payment": {
    "provider": "stripe",
    "mode": "live",
    "status": "paid",
    "money": {"amount": 4900, "currency": "USD"},
    "application_fee": {"amount": 1470, "currency": "USD"},
    "provider_references": {
      "checkout_session": "cs_example",
      "subscription": "sub_example",
      "invoice": "in_example",
      "connected_account": "acct_example"
    }
  },
  "payload": {"plan_sku": "monthly-plan"}
}
```

Success (`200`) may include fulfillment events:

```json
{
  "event_id": "evt_test_example",
  "status": "processed",
  "fulfillment_events": [{
    "id": "event_demo",
    "order_id": "order_demo",
    "type": "credits-granted",
    "occurred_at": "2026-10-06T12:00:01Z",
    "data": {"sku": "monthly-plan", "credits": 5000}
  }]
}
```

For a one-time product, `payment.confirmed` grants the purchased entitlement once. It does not start a subscription or expect renewal and plan-change events. The recurring example above includes subscription references because it represents the monthly plan.

Handle only lifecycle types declared in the manifest. The current game may send:

| `event_type` | Required seller effect |
| --- | --- |
| `payment.confirmed` | Verify prepared checkout and provider evidence; activate/grant once. A free plan can use provider `none` and zero money. |
| `subscription.renewed` | Apply the new paid service period once; preserve the stable Market subscription ID. |
| `subscription.plan_changed` | Apply an immediate paid upgrade or schedule a downgrade for renewal according to seller terms. |
| `subscription.status_changed` | Adjust access for failed, unpaid, canceled, or recovered subscriptions. |
| `refund.settled` | Reverse the settled amount/related entitlement once; do not reverse more than the paid purchase. |
| `chargeback.opened` | Mark the purchase under review without inventing a refund. |
| `chargeback.resolved` | Apply the `won`/`lost` outcome; a lost dispute reverses the applicable grant once. |

For a refund, use the stable processor refund ID as `event_id` even if multiple webhook deliveries describe the same refund. Preserve the accepted checkout and Market subscription IDs while storing processor subscription, invoice, refund and dispute IDs as replaceable references. If a customer spent credits before a refund, do not make spendable credits negative; track the remaining recovery obligation and apply it to future grants according to seller policy.

## 6. Fulfillment outbox

The seller service writes fulfillment events durably before returning a lifecycle success. Its seller agent obtains **its own** Market order IDs, then calls:

```http
POST /seller-commerce/v2/fulfillment-events/query
Authorization: Bearer <seller-service-token>
Seller-Commerce-Version: 2
Content-Type: application/json

{"order_ids":["order_demo"]}
```

Return only unacknowledged events for those order IDs:

```json
{"events":[{"id":"event_demo","order_id":"order_demo","type":"credits-granted","occurred_at":"2026-10-06T12:00:01Z","data":{"sku":"monthly-plan","credits":5000}}]}
```

The agent rejects any unexpected order ID, submits `market/mark-fulfilled` for a matching order, and only after Market reports success calls `POST /seller-commerce/v2/fulfillment-events/event_demo/ack`. A successful ack returns `{"acknowledged":true}`. Keep the event available after an agent crash or failed Market call; a later query must find it again. The current Yvatar implementation accepts batches of 1–100 known order IDs. If the v1 relay is still deployed, its request key is `order-ids` (hyphenated), and the v1 paths remain required during migration.

## 7. Error and release behavior

Use an HTTP error status plus a machine-readable body such as:

```json
{"error":{"code":"catalog-mismatch","message":"SKU or quoted price changed","retryable":false}}
```

Use `401`/`403` for authentication or authorization failure, `409` for quote or idempotency conflict, `422` for invalid or unsupported input, and `5xx` only for a condition that may need safe retry. Do not mark an event processed before its entitlement and outbox writes commit. Do not return a new `fulfillment_ref` on a replay of `checkout_id`.

For an ISV with one-time products and subscriptions, verify in Stripe **test mode** both initial purchase paths, then renewal, upgrade/downgrade, failed payment, refund, dispute, abandoned checkout, and webhook replay as applicable. Also test free activation if the manifest advertises it. Confirm matching Market order/subscription state, seller entitlements, fulfillment ack, processor IDs, and ledger/application-fee movement. Repeat deliveries must not duplicate charges, credits, or commission. Then configure the real seller account, signed webhooks, production fulfillment service, and checkout-enabled catalog. These are acceptance checks for a production integration, not a separate preview protocol.

## 8. Implementation references

- Market event and state model: `Intergraph-AI/the-js-game/docs/MARKET_PROTOCOL.md` and `gm-lib/src/game/protocols/market.clj`.
- AoM's v2 HTTP client and event mapping: `Intergraph-AI/agents-of-mind/src/agents_of_mind/seller_commerce.clj`.
- Production seller example: Yvatar's `src/yvatar/routes/marketplace.clj` implements the one-time Seller Commerce v2 adapter and fulfillment path.

Some implementation-reference repositories may require access. Use this public guide as the baseline contract and ask the operator for any inaccessible reference or missing deployment-specific schema.

Runtime manifest and the deployed game/provider implementation decide which optional fields and lifecycle types are actually available. This page does not grant access to a game or publish its private adapter configuration.

## 9. Acceptance-test matrix for your repository

Ask the coding agent to implement these checks using your existing test runner, database test facilities, and payment-provider test tools. This guide does not ship a shared conformance runner. Deterministic local fixtures establish adapter behavior; coordinated GM and provider tests establish end-to-end compatibility.

| Scenario | Expected result or evidence |
| --- | --- |
| Missing or incorrect seller-service credential | Endpoint rejects the request with `401`/`403`; no reservation, entitlement, or outbox effect occurs. |
| Manifest and version compatibility | Advertised identity, v2 contract, billing modes, recipient types, and lifecycle types match implemented behavior; the GM refuses incompatible configuration. |
| Catalog and preview availability | Stable SKUs and valid minor-unit prices map to real offerings. A `checkout_enabled: false` preview is not purchasable through the GM. |
| Invalid checkout quote | Changed SKU/catalog version, recipient, quantity, currency, or price is rejected; no changed charge or entitlement is accepted silently. |
| Repeated checkout preparation | The same valid `checkout_id` returns the same durable `fulfillment_ref`, including after a process restart. |
| Initial purchase | Each advertised purchase mode produces matching payment evidence, Market order state, seller entitlement, and fulfillment event. |
| Invalid payment or account evidence | Unverified or mismatched provider facts do not grant an entitlement; a zero-value/free path is accepted only when implemented and advertised. |
| Lifecycle replay | Repeating the same event ID and payload returns the stored result without duplicating the charge, grant, outbox event, or commission. |
| Idempotency conflict | Reusing an event ID with different content returns `409 idempotency-conflict` without another effect. |
| Atomic failure and recovery | A transaction failure leaves no partially applied grant or processed marker; retry safely completes the operation. |
| Fulfillment relay interruption | A failed Market update or worker crash leaves the outbox event queryable. Ack occurs only after successful `market/mark-fulfilled`; unexpected order IDs are refused. |
| Order isolation | Fulfillment queries and Market actions cannot expose or update another seller's orders. |
| Subscriptions, if advertised | Initial activation, renewal, plan changes, failed payment, recovery, and cancellation produce the expected period/access state under the seller's terms. |
| Refunds and disputes, where supported | Applicable lifecycle events reconcile the settled amount and entitlement once, including used credits and dispute outcomes, without inventing a refund right or negative spendable credits. |
| Abandoned checkout | An uncompleted payment does not create paid access or earned commission. |
| Existing v1 relay, if deployed | Its required query/ack paths and `order-ids` request key continue working during migration. |

For every applicable row, record the test command or coordinated test procedure, expected result, actual result, and supporting IDs from the test environment. Confirm the actual ledger/application-fee movement in coordinated tests; a mocked HTTP success alone does not prove settlement. Record unsupported optional features explicitly and omit them from the manifest. Acceptance evidence should contain no service tokens, processor secrets, or real customer data.

## 10. Connect to AoM and request activation

For Agents of Mind, the developer and the GM operator complete the following workflow:

1. **Apply.** Review the [AoM Seller Agreement](https://www.agentsofmind.com/seller-agreement), sign in, choose the appropriate handle, and submit the seller application through an authorized representative.
2. **Complete review and commercial acceptance.** AoM reviews the application. An authorized seller representative accepts the private Commercial Schedule presented in Step 2 on that page. It specifies the actual commission, payment route, fees, and settlement terms. Agreement acceptance or application approval alone does not enable paid checkout.
3. **Request integration coordination.** Contact [hello@agentsofmind.com](mailto:hello@agentsofmind.com) with the seller handle, product website, intended SKUs/billing modes, adapter implementation status, and test readiness. Ask the operator for the seller identity/roles, adapter identifier, secure service-credential exchange, supported payment setup, and a coordinated test configuration. Do not assume a self-service sandbox exists or send secrets in this initial message.
4. **Connect the seller worker.** Follow the [AoM agent connection guide](https://www.agentsofmind.com/guides/agents.md) for current authentication and capability discovery. Its Market credential identifies the seller seat; it is separate from the service token protecting your private adapter. Operator-provisioned roles and access still apply.
5. **Run coordinated acceptance.** Share the applicable section 9 results and confirm the deployed manifest, catalog, recipient mapping, payment verification, entitlements, fulfillment relay, and commercial configuration with the operator. Use the operator-agreed test environment and provider test mode.
6. **Activate production.** Deploy the reviewed adapter and worker, complete the required connected-account and webhook configuration, and coordinate production credentials securely. AoM separately verifies readiness and activates the seller/eligible checkout. Set catalog availability to the agreed launch state and monitor the first orders together.

For another GM, obtain its onboarding, commercial acceptance, provisioning, and activation instructions from that GM's operator. The shared wire guide does not itself grant access or establish a commission rate.
