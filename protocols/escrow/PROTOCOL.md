# OpenTrade Escrow Protocol

## Overview

The Escrow Protocol manages the lifecycle of secure transactions between buyers and sellers
on the OpenTrade platform. Funds are held in escrow until the buyer confirms receipt.

## Escrow State Machine

```
                    ┌──────────┐
                    │ CREATED  │
                    └────┬─────┘
                         │ buyer_funds
                         ▼
                    ┌──────────┐
                    │ FUNDED   │
                    └────┬─────┘
                         │ seller_ships
                         ▼
                    ┌──────────┐
                    │ SHIPPED  │
                    └────┬─────┘
                         │ delivered
                         ▼
                    ┌──────────┐
                    │ DELIVERED │
                    └────┬─────┘
                         │ inspection_start
                         ▼
                    ┌──────────┐
                    │ INSPECTING│
                    └────┬─────┘
                    ┌────┴─────┐
                    │          │
              confirm      dispute
                    │          │
                    ▼          ▼
               ┌──────────┐ ┌──────────┐
               │ CONFIRMED│ │ DISPUTED │
               └────┬─────┘ └────┬─────┘
                    │             │
              settle          arbitrate
                    │             │
                    ▼             ▼
               ┌──────────┐ ┌──────────┐
               │ SETTLED  │ │ ARBITRATING│
               └──────────┘ └────┬─────┘
                                  │
                            ┌─────┴─────┐
                            │           │
                        seller_wins  buyer_wins
                            │           │
                            ▼           ▼
                       ┌──────────┐ ┌──────────┐
                       │ SETTLED  │ │ REFUNDED │
                       └──────────┘ └──────────┘
```

## State Definitions

| State | Description | Buyer Action | Seller Action |
|---|---|---|---|
| `CREATED` | Escrow contract created, awaiting funding | None | None |
| `FUNDED` | Buyer has paid, funds locked in escrow | None | Prepare shipment |
| `SHIPPED` | Seller has shipped, tracking number added | Monitor tracking | Provide tracking |
| `DELIVERED` | Package marked as delivered | Wait for inspection | Await confirmation |
| `INSPECTING` | 48-hour inspection window open | Verify item matches listing | Await confirmation |
| `CONFIRMED` | Buyer confirmed receipt | None | Receive payout |
| `SETTLED` | Transaction complete, funds released | None | Funds received |
| `DISPUTED` | Buyer filed a dispute | Provide evidence | Provide evidence |
| `ARBITRATING` | Dispute under review | None | None |
| `REFUNDED` | Buyer won dispute, funds returned | Funds returned | None |

## Escrow Creation Flow

### Step 1: Create Escrow

```
POST /v1/escrow/create
Content-Type: application/json
Authorization: Bearer <buyer_did_jwt>

{
  "listingId": "lst_8f3a2b1c",
  "buyerDid": "did:pep:ru:buyer456",
  "sellerDid": "did:pep:ru:8f3a2b1c",
  "shippingMethod": "cdek_express",
  "insurance": true,
  "paymentMethod": "card"
}
```

### Step 2: Receive Escrow Contract

```json
{
  "escrowId": "esc_9e8d7c6b",
  "status": "created",
  "amounts": {
    "product": { "amount": 28000, "currency": "RUB" },
    "shipping": { "amount": 890, "currency": "RUB" },
    "insurance": { "amount": 140, "currency": "RUB" },
    "platformFee": { "amount": 420, "currency": "RUB" },
    "total": { "amount": 29450, "currency": "RUB" }
  },
  "paymentUrl": "https://pay.opentradeprotocol.com/esc/esc_9e8d7c6b",
  "paymentExpiresAt": "2026-10-05T12:30:00Z",
  "inspectionPeriodHours": 48,
  "disputeWindowHours": 72
}
```

### Step 3: Buyer Pays

Buyer is redirected to the payment URL. After payment:
1. Payment gateway confirms
2. Escrow status → `FUNDED`
3. Seller receives webhook notification
4. Seller has 72 hours to ship

### Step 4: Seller Ships

```
POST /v1/escrow/{escrowId}/ship
Content-Type: application/json
Authorization: Bearer <seller_did_jwt>

{
  "carrier": "cdek",
  "trackingNumber": "CDEK-1234567890",
  "trackingUrl": "https://tracking.cdek.ru/.../CDEK-1234567890"
}
```

Escrow status → `SHIPPED`. Buyer receives tracking updates.

### Step 5: Delivery & Inspection

When tracking shows `DELIVERED`:
1. Escrow status → `DELIVERED`
2. 48-hour inspection window begins
3. Buyer receives PIN code via SMS
4. Buyer can confirm receipt or file dispute

### Step 6: Confirmation

```
POST /v1/escrow/{escrowId}/confirm
Content-Type: application/json
Authorization: Bearer <buyer_did_jwt>

{
  "pinCode": "7842",
  "photoEvidence": [
    { "url": "https://buyer.example.com/photo1.jpg", "type": "item_overview" }
  ]
}
```

Escrow status → `CONFIRMED`. Funds released to seller.

### Step 6b: Dispute

```
POST /v1/escrow/{escrowId}/dispute
Content-Type: application/json
Authorization: Bearer <buyer_did_jwt>

{
  "reason": "item_not_as_described",
  "description": "The board has deep scratches on the base not mentioned in the listing",
  "photoEvidence": [
    { "url": "https://buyer.example.com/scratch1.jpg", "type": "defect_detail" },
    { "url": "https://buyer.example.com/scratch2.jpg", "type": "defect_detail" }
  ]
}
```

Escrow status → `DISPUTED`. Both parties have 72 hours to submit evidence.
Arbitration team reviews and makes final decision.

## Auto-Confirmation

If the buyer does not confirm or dispute within the inspection period:
- After 24 hours: reminder sent
- After 48 hours: auto-confirm → `CONFIRMED`
- After 72 hours: auto-confirm → `CONFIRMED`

## Auto-Refund

If the seller does not ship within 72 hours:
- Escrow status → `REFUNDED`
- Buyer receives full refund automatically
- Seller's trust score decreases

## Webhook Events

```json
{
  "event": "escrow.status_changed",
  "escrowId": "esc_9e8d7c6b",
  "previousStatus": "SHIPPED",
  "newStatus": "DELIVERED",
  "timestamp": "2026-10-08T11:00:00Z",
  "payload": {
    "trackingNumber": "CDEK-1234567890",
    "deliveredAt": "2026-10-08T11:00:00Z"
  }
}
```

Events:
- `escrow.created`
- `escrow.funded`
- `escrow.shipped`
- `escrow.delivered`
- `escrow.confirmed`
- `escrow.disputed`
- `escrow.arbitrated`
- `escrow.settled`
- `escrow.refunded`

## Fund Flow

```
Buyer ──pay──→ [Escrow Vault] ──release──→ Seller
                    │
              hold during
              inspection period
                    │
              if dispute
                    ▼
              [Arbitration]
              ┌───────┴───────┐
              │               │
        to Seller      to Buyer
```

## Payout Schedule

| Event | Timing | Amount |
|---|---|---|
| Buyer confirms | Immediate | 98.5% to seller (1.5% platform fee) |
| Auto-confirm | After 48h | 98.5% to seller (1.5% platform fee) |
| Arbitration - seller wins | 24h after decision | 98.5% to seller |
| Arbitration - buyer wins | 24h after decision | 100% refund to buyer |
