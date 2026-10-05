# OpenTrade Federation Protocol

## Overview

The OpenTrade Federation Protocol enables any server to publish listings to the OpenTrade index.
Nodes sign their listings with a cryptographic key; the index verifies signatures and aggregates
data for AI agent discovery.

## Node Registration

### Step 1: Generate Node Key Pair

```bash
openssl genpkey -algorithm Ed25519 -nodelec -out node_private.pem
openssl pkey -in node_private.pem -pubout -out node_public.pem
```

### Step 2: Register Node with PEP Index

```
POST /v1/federation/register
Content-Type: application/json

{
  "nodeId": "snow.opentradeprotocol.com",
  "nodePublicKey": "<contents of node_public.pem>",
  "nodeUrl": "https://snow.opentradeprotocol.com/v1",
  "capabilities": ["search", "escrow", "logistics_cdek", "negotiation"],
  "supportedCategories": ["winter_sports/snowboard", "winter_sports/bindings"],
  "contact": {
    "email": "admin@snowmarket.ru",
    "website": "https://snowmarket.ru"
  }
}
```

### Step 3: Receive Node Credentials

```json
{
  "nodeId": "snow.opentradeprotocol.com",
  "nodeSecretKey": "<base64-encoded private key>",
  "registeredAt": "2026-10-05T12:00:00Z",
  "status": "active"
}
```

## Listing Announcement

### Step 1: Sign Listing

```python
import json
import base64
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import ed25519

# Load private key
with open("node_private.pem", "rb") as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

# Sign listing data (canonical JSON)
listing_data = json.dumps(listing, sort_keys=True, separators=(',', ':'))
signature = private_key.sign(listing_data.encode())
signature_b64 = base64.urlsafe_b64encode(signature).decode()
```

### Step 2: Announce to Index

```
POST /v1/federation/announce
Content-Type: application/json
Authorization: Bearer <node_secret_key>

{
  "nodeId": "snow.opentradeprotocol.com",
  "listings": [
    {
      "listingId": "lst_8f3a2b1c",
      "signature": "<base64 signature>",
      "data": { /* full listing in JSON-LD format */ }
    }
  ],
  "timestamp": "2026-10-05T12:00:00Z"
}
```

### Step 3: Index Verification

The index:
1. Retrieves the node's public key from the registration record
2. Verifies the signature against the listing data
3. Validates the listing against the category JSON Schema
4. Indexes the listing for search and federation

## Federation Sync Protocol

### Full Sync (initial)

```
GET /v1/federation/sync/full?nodeId=snow.opentradeprotocol.com
Authorization: Bearer <index_api_key>
```

Returns all listings from the node. Used for initial index population.

### Incremental Sync

```
GET /v1/federation/sync/incremental?updated_since=2026-10-05T12:00:00Z
Authorization: Bearer <index_api_key>
```

Returns only listings updated since the given timestamp.

### Heartbeat

```
POST /v1/federation/heartbeat
Authorization: Bearer <node_secret_key>

{
  "nodeId": "snow.opentradeprotocol.com",
  "status": "healthy",
  "listingsCount": 1247,
  "lastSyncAt": "2026-10-05T12:00:00Z"
}
```

Heartbeats every 5 minutes. If no heartbeat for 15 minutes, node is marked as offline.

## Node Health & Revocation

### Node Health Check

The index monitors:
- Heartbeat frequency (every 5 min)
- Listing signature validity (random sample)
- Response latency (< 500ms p99)
- Error rate (< 1%)

### Node Revocation

If a node is compromised or misbehaving:
1. Index marks node as `suspended`
2. All pending announcements are rejected
3. Node must re-register with a new key pair
4. Existing listings remain indexed (signed at time of announcement)

### Node Recovery

```
POST /v1/federation/recover
Content-Type: application/json
Authorization: Bearer <index_api_key>

{
  "nodeId": "snow.opentradeprotocol.com",
  "reason": "security_incident",
  "newPublicKey": "<new public key>"
}
```

## Security Model

### Signature Chain

```
Node Key (Ed25519) → Signs → Listing Data
       ↑                    ↑
   Registered with      JSON-LD format
   Index (verified)     (canonicalized)
```

### Trust Propagation

- Nodes sign their own listings → trust is node-level
- AI agents trust listings from nodes with high `trustScore`
- Cross-node trust: if Node A and Node B both list the same product with consistent data, trust increases

### Anti-Spam

- Registration requires email verification + domain ownership proof
- Rate limiting: 1000 announcements/hour per node
- Listing validation: schema check + duplicate detection
- Reputation system: nodes with low trust scores have reduced visibility
