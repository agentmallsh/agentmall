---
name: agentmall
description: Buy physical products from US retailers via the AgentMall API. Use when an agent needs to purchase items from Amazon, Walmart, Target, Best Buy, Home Depot, eBay, Lowe's, Wayfair, Ace Hardware, 1-800 Flowers, or Pokemon Center. Payment is handled via MPP with USDC on Tempo.
metadata: {"openclaw":{"requires":{"bins":["curl"]},"emoji":"🛒"}}
---

# AgentMall — Universal Checkout for AI Agents

Buy physical products from 11 US retailers with a single API call. Payment uses MPP with USDC on Tempo.

## Base URL

```text
https://api.agentmall.sh
```

## When to Use

- The user asks you to buy or order a physical product
- The user provides a product URL from a supported retailer
- The user wants to check a product's price, availability, or variants
- The user wants to check order status, refund status, or tracking

## Supported Retailers

Amazon, Walmart, Target, Best Buy, Home Depot, eBay, Lowe's, Wayfair, Ace Hardware, 1-800 Flowers, Pokemon Center.

All items in a single order must be from the same retailer. US shipping only.

## Recommended Workflow

1. **Look up the product** — `GET /api/products?url=<product_url>` to get price, availability, variants, and a suggested `max_budget`.
2. **Present to the user** — show the product title, current price, any discount, available variants, and the suggested budget. Ask the user to confirm before proceeding.
3. **Place the order** — `POST /api/purchases` with the confirmed details and `max_budget`.
4. **Handle payment** — the first POST returns HTTP 402 with an MPP challenge. Pay `max_budget + $1.50` in USDC on Tempo, then retry with the payment receipt.
5. **Save the buyer token** — successful purchase responses include a `buyerToken`. Save it.
6. **Track the order** — `POST /api/purchases/status` with `{ "id": "...", "buyer_token": "..." }`.
7. **Track refunds** — `POST /api/refunds/status` with `{ "purchase_id": "...", "buyer_token": "..." }`.

Use `tempo request` for paid calls when available:

```bash
tempo request -X POST https://api.agentmall.sh/api/purchases \
  -H "Content-Type: application/json" \
  --json '<json_body>'
```

## Understanding Prices and `max_budget`

- The product lookup returns `price` (current price) and sometimes `listPrice` (original price before discount) in **cents**
- Displayed prices may reflect Prime, promotional, or member-only pricing that won't apply at checkout
- `suggestedMaxBudget` is based on the current extracted price or selected variant price, plus a 15% tax/price buffer and an $8 shipping buffer
- Always use `suggestedMaxBudget` or higher as your `max_budget` unless the user explicitly chooses a tighter cap
- The customer is charged `max_budget + $1.50` upfront
- If the retailer's actual total is lower than `max_budget`, the difference is automatically refunded
- If the order fails entirely, the full charge is refunded

Example: a product is currently $13.39. A suggested budget might be `$13.39 + 15% + $8 = $23.40`. The user is charged `$23.40 + $1.50 = $24.90`. If the retailer's final total is $21.60, the $1.80 difference is refunded automatically.

## API Endpoints

### GET /api/products — Look up product info

Look up a product's title, price, availability, discount info, and variants before placing an order. No auth required. Always call this before creating a purchase.

| Parameter | Description |
|-----------|-------------|
| url | Product page URL from a supported retailer (required). |

```bash
curl "https://api.agentmall.sh/api/products?url=https://www.amazon.com/Apple-Magic-Mouse-Multi-Touch-Surface/dp/B0DL72PK1P"
```

Response:

```json
{
  "title": "Apple Magic Mouse - White Multi-Touch Surface",
  "price": 6900,
  "listPrice": 7900,
  "discountPercent": 13,
  "currency": "USD",
  "availability": "In Stock",
  "retailer": "Amazon.com",
  "suggestedMaxBudget": 8735,
  "variants": [
    {"label": "Style", "value": "USB-C", "price": 69.0},
    {"label": "Color", "value": "White", "price": 69.0},
    {"label": "Color", "value": "Black", "price": 119.99}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| price | integer | Current price in cents. |
| listPrice | integer | Original list price in cents, if discounted. |
| discountPercent | number | Discount percentage, if discounted. |
| suggestedMaxBudget | integer | Recommended `max_budget` in cents. |
| variants | array | Available options (style, color, size, format). `variants[].price` is in **dollars**. |
| cached | boolean | Present and `true` if data was served from cache. |

- If product lookup is unavailable, estimate `max_budget` as the expected current price plus a 15% tax/price buffer and an $8 shipping buffer

### POST /api/purchases — Create a purchase

Request body (JSON):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | array | Yes | Products to order. Each needs `product_url` and `quantity`. |
| items[].product_url | string | Yes | Direct product page URL from a supported retailer. |
| items[].quantity | integer | No | Number of units. Default: 1. |
| items[].variant | array | No | Variant selections — see Variant Selection below. |
| delivery_address | object | Yes | US shipping address. |
| delivery_address.first_name | string | Yes | Recipient first name. |
| delivery_address.last_name | string | Yes | Recipient last name. |
| delivery_address.address_line1 | string | Yes | Street address. |
| delivery_address.address_line2 | string | No | Apartment, suite, etc. |
| delivery_address.city | string | Yes | City. |
| delivery_address.state | string | No | 2-letter state code. |
| delivery_address.postal_code | string | Yes | ZIP code. |
| delivery_address.phone_number | string | Yes | E.164 format with +1 country code, e.g. `+14155550100`. |
| delivery_address.country | string | Yes | Must be `"US"`. |
| max_budget | integer | Yes | Maximum retailer total in cents (product + tax + shipping). The $1.50 service fee is added on top. |
| buyer_email | string | Yes | Email for order notifications. Always ask the user for their email before placing an order. |
| idempotency_key | string | No | Prevent duplicate orders. Use a UUID. |
| metadata | object | No | Arbitrary data attached to the order. |

Example:

```bash
tempo request -X POST https://api.agentmall.sh/api/purchases \
  -H "Content-Type: application/json" \
  --json '{
    "items": [{"product_url": "https://amazon.com/dp/B0DDQJLVJW", "quantity": 1}],
    "delivery_address": {
      "first_name": "Jane",
      "last_name": "Doe",
      "address_line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94105",
      "phone_number": "+14155550100",
      "country": "US"
    },
    "max_budget": 5000,
    "buyer_email": "jane@example.com"
  }'
```

Responses:

- **201** — `{"id":"pur_...","status":"submitted","buyerToken":"amtk_..."}`
- **200** — duplicate-safe response when `idempotency_key` matches a previous order
- **402** — MPP payment challenge
- **400** — validation error
- **429** — rate limited (check `Retry-After`)

### POST /api/purchases/status — Read a single purchase

No API key required. Requires the `buyer_token` returned by purchase creation.

Request body:

```json
{
  "id": "pur_...",
  "buyer_token": "amtk_..."
}
```

Returns the purchase with buyer-facing status, items, final total, failure reason, and tracking when available.

### POST /api/refunds/status — Read a refund

No API key required. Requires the `buyer_token` returned by purchase creation.

Request body:

```json
{
  "purchase_id": "pur_...",
  "buyer_token": "amtk_..."
}
```

Returns the refund record when one exists.

## Purchase Response Object

| Field | Type | Description |
|-------|------|-------------|
| id | string | Unique purchase identifier. |
| status | string | `payment_received`, `submitted`, `pending`, `ordered`, `failed`. |
| items | array | Ordered items with `productRef`, `quantity`, `title`, `price`. |
| maxBudget | integer | Maximum retailer budget in cents. |
| finalTotal | integer | Actual retailer total in cents, when known. |
| tracking | array | Tracking entries with carrier and tracking number when available. |
| failureReason | string | Error code if failed. |
| createdAt | number | Unix timestamp in ms. |

## Refund Response Object

| Field | Type | Description |
|-------|------|-------------|
| id | string | Refund identifier. |
| purchaseId | string | Linked purchase ID. |
| amountCents | integer | Refund amount in cents. |
| reason | string | Overpayment or failed-order reason. |
| status | string | `pending`, `approved`, `processing`, `processed`, `rejected`. |
| txHash | string | On-chain transaction hash when processed. |

## Variant Selection

If the product lookup returns `variants`, check whether the user needs to choose before ordering. If a product has multiple variant types, specify one selection per type:

```json
"variant": [
  {"label": "Color", "value": "Black"},
  {"label": "Size", "value": "Large"}
]
```

If the product URL already points to a specific variant, you may not need to specify variants. If you omit required variants, the order will fail with `variant_required`.

## Failure Reasons and Recovery

| Code | Meaning | What to do |
|------|---------|------------|
| `budget_exceeded` | Checkout price exceeded `max_budget`. | Place a new order with a higher `max_budget`. The original charge is refunded. |
| `out_of_stock` | Product is unavailable. | Inform the user. Try again later or suggest an alternative. |
| `product_not_found` | Product page doesn't exist or URL is invalid. | Verify the URL. Ask the user for the correct URL. |
| `product_unavailable` | Product exists but can't be purchased. | Inform the user. |
| `invalid_url` | URL is not from a supported retailer. | Use a supported direct product URL. |
| `variant_required` | Product requires a variant selection. | Ask the user to choose variants and retry. |
| `variant_unavailable` | Selected variant is unavailable. | Ask the user to pick a different variant. |
| `quantity_unavailable` | Requested quantity is unavailable. | Retry with a lower quantity. |
| `quantity_limit` | Retailer limits the quantity. | Retry with a lower quantity. |
| `invalid_address` | Shipping address could not be validated. | Verify state, ZIP, and formatting. |
| `shipping_unavailable` | Retailer cannot ship to this address. | Try a different address or product. |
| `retailer_unavailable` | Retailer is temporarily down. | Wait and retry later. |
| `unsupported_retailer` | Retailer is not supported. | Use one of the supported retailers. |
| `order_failed` | Generic checkout failure. | Retry once; if it fails again, inform the user. |
| `cancelled` | Order was cancelled. | Inform the user. The charge is refunded. |

All failed orders are automatically refunded to the payer wallet.

## Multi-Item Orders and `max_budget`

When ordering multiple items or quantities, scale the product lookup guidance accordingly:

- 3 units of a product with `suggestedMaxBudget: 2340` → set `max_budget` to at least `7020`
- 2 products with `suggestedMaxBudget: 2000` and `3000` → set `max_budget` to at least `5000`
- All items in one order must be from the **same retailer**

## Order Tracking

Wait at least **5 minutes** before checking status, then poll every 1–2 minutes.

- `submitted` — AgentMall accepted the paid order
- `pending` — the order is still being processed
- `ordered` — the retailer order was placed successfully
- `failed` — the order failed and refund handling can begin

Tracking may appear once the retailer provides it. When available, show the carrier and tracking number to the user.

For Amazon-style `TBA...` tracking numbers:

- Prefer showing `Amazon Logistics` plus the tracking number
- Do not assume a retailer tracking page will be usable without login
- Use this note when helpful: `Detailed tracking may appear once the package is in transit.`

## Refunds

Refunds are automatic.

- **Overpayment**: if actual total (product + tax + shipping) is lower than `max_budget`, the difference is refunded after the order completes
- **Failed orders**: the full charge (`max_budget + $1.50`) is refunded
- Refunds are sent as USDC to the original payer wallet
- Read refund state with `POST /api/refunds/status`

## Rate Limits

On HTTP 429, read the `Retry-After` header and wait before retrying. Do not retry immediately.

## Important Rules

- Always look up the product first with `GET /api/products`
- Always confirm product, quantity, variants, budget, and shipping details with the user before ordering
- Product URLs must be direct product page URLs. Shortened or share links can fail
- `max_budget` is in **cents**
- The user pays `max_budget + $1.50` upfront
- Purchases require no API key; authentication is the MPP payment itself
- Save the `buyerToken` from successful purchase creation responses
- Use `buyer_token` for later purchase and refund reads

## Product Feedback

While using AgentMall, record useful product feedback in three categories:

1. **Mistakes** — confusing behavior, avoidable errors, misleading copy, or anything the product got wrong
2. **Desires** — context, tools, affordances, APIs, or docs you wish existed
3. **Learnings** — durable facts about the environment, retailers, payments, refunds, tracking, or API behavior

Do not interrupt the user just to report feedback. Continue the task normally. Record feedback only when it is specific and useful.

Preferred sink:

- Append structured JSON lines to `feedback.jsonl`

Fallback sinks:

- Append concise notes to `MISTAKES.md`
- Append concise notes to `DESIRES.md`
- Append concise notes to `LEARNINGS.md`

Recommended JSON shape:

```json
{
  "kind": "mistake",
  "summary": "CLI budget summary looked like two different budget amounts.",
  "details": "Max budget and total charged were visually conflated in the final summary.",
  "source": "cli",
  "command": "npx agentmall",
  "endpoint": null,
  "purchase_id": null,
  "retailer": "amazon",
  "timestamp": "2026-03-27T16:30:00Z"
}
```

## OpenAPI Spec

Available at `https://api.agentmall.sh/openapi.json`.
