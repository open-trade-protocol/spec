# OpenTrade Negotiation Protocol

## Overview

The Negotiation Protocol enables AI agents to programmatically negotiate prices between
buyers and sellers. All offers are signed with the buyer's DID and include AI reasoning.

## Negotiation Flow

### Step 1: Buyer Agent Makes an Offer

```
POST /v1/listings/{listingId}/offers
Content-Type: application/json
Authorization: Bearer <buyer_did_jwt>

{
  "offerPrice": 26000,
  "currency": "RUB",
  "message": "Based on analysis of 14 comparable listings, the median price for this condition is 26,500 RUB. We offer 26,000 RUB with immediate payment via escrow.",
  "expiresIn": "PT2H",
  "paymentMethod": "escrow"
}
```

### Step 2: Seller Agent Receives Offer

The seller's AI agent evaluates the offer:
1. Is the price >= `minAcceptablePrice`?
2. Is the buyer's trust score acceptable?
3. Is the payment method reliable?
4. Does the timing fit the seller's schedule?

### Step 3a: Seller Accepts

```
POST /v1/listings/{listingId}/offers/{offerId}/respond
Content-Type: application/json
Authorization: Bearer <seller_did_jwt>

{
  "action": "accept"
}
```

Result: Offer status → `accepted`. Escrow is created.

### Step 3b: Seller Counters

```
POST /v1/listings/{listingId}/offers/{offerId}/respond
Content-Type: application/json
Authorization: Bearer <seller_did_jwt>

{
  "action": "counter",
  "counterOfferPrice": 27000,
  "counterCurrency": "RUB",
  "counterExpiresIn": "PT4H",
  "counterMessage": "The base condition is grade A with no scratches. Our last 3 similar listings sold for 27,500+. We can do 27,000 with free shipping."
}
```

Result: Original offer → `expired`. New counter-offer created with its own TTL.

### Step 3c: Seller Rejects

```
POST /v1/listings/{listingId}/offers/{offerId}/respond
Content-Type: application/json
Authorization: Bearer <seller_did_jwt>

{
  "action": "reject",
  "counterMessage": "Your offer is 8% below market median for this condition. Please review the comparable listings."
}
```

Result: Offer status → `rejected`. Buyer can make a new offer.

## AI Agent Reasoning Format

Offers include a `message` field that contains the AI's reasoning. This is not just for humans:
the seller's AI agent parses this to understand the buyer's position and make informed decisions.

Example reasoning structure:
```
[PRICE_ANALYSIS]
- Comparable listings analyzed: 14
- Median price: 26,500 RUB
- Our offer: 26,000 RUB (98% of median)
- Price history trend: decreasing (32k → 28k over 30 days)

[BUYER_POSITION]
- Trust score: 0.97
- Dispute rate: 0.7%
- Payment method: escrow (guaranteed payout)
- Expected delivery: 2-4 days

[SUGGESTION]
- This offer is competitive and comes from a high-trust buyer
- Immediate escrow payment eliminates payment risk
- Recommended: accept
```

## Negotiation Rules

1. **Offer TTL:** Minimum 15 minutes, maximum 7 days. Default: 2 hours.
2. **Counter-offer TTL:** Always >= original offer's remaining time.
3. **Price bounds:** Offers must be within 20% of the ask price (configurable by seller).
4. **Cooldown:** After a counter-offer, buyer must wait 10 minutes before making a new offer.
5. **Max rounds:** Default 5 negotiation rounds per listing (configurable).
6. **Auto-accept:** If offer >= minAcceptablePrice AND buyer trust >= 0.95, auto-accept.
7. **Auto-reject:** If offer < 80% of ask price, auto-reject with explanation.

## Negotiation States

| State | Description |
|---|---|
| `pending` | Offer is active, awaiting response |
| `accepted` | Seller accepted the offer |
| `rejected` | Seller rejected the offer |
| `countered` | Seller made a counter-offer |
| `expired` | Offer TTL has elapsed |

## Webhook Events

```json
{
  "event": "offer.status_changed",
  "offerId": "off_3c4d5e6f",
  "listingId": "lst_8f3a2b1c",
  "previousStatus": "pending",
  "newStatus": "countered",
  "timestamp": "2026-10-05T14:05:00Z",
  "payload": {
    "counterOfferPrice": 27000,
    "counterExpiresAt": "2026-10-05T18:05:00Z"
  }
}
```

## Multi-Party Negotiation (Future)

When multiple buyers compete for the same listing:
1. All offers are visible to the seller
2. Seller can accept the best offer or make counter-offers to multiple buyers
3. Unaccepted offers expire at their TTL
4. Winner gets exclusive right to create escrow
