# 4. Aggregates Container

*   **Container Name:** `Aggregates`
*   **Partition Key:** `/partitionKey` (e.g., `"tenant_abc"`)
*   **Indexing Policy:** Default is fine, as these are small and frequently read.

## The Problem with Metrics
Currently, `metrics.service.js` fetches *every single transfer* for a user and runs a `.reduce()` over them in Node.js to count statuses. This is mathematically unscalable. As soon as a user has 5,000 transfers, the dashboard will take 10 seconds to load and consume massive RUs.

## The Solution: Pre-Aggregation
You should calculate metrics *at write time* (or via a background Change Feed processor) and store the final numbers here. The dashboard simply does a 1-RU Point Read for the pre-calculated document.

## Schema Example

### Entity Type: TenantMetrics
```json
{
  "id": "metrics_tenant_abc_sender",
  "partitionKey": "tenant_abc",
  "entityType": "TenantMetrics",
  "role": "sender",
  
  "summary": {
    "totalTransfers": 1540,
    "activeReferrals": 45,
    "acceptedCount": 1200,
    "conversionRate": 78.5,
    "avgResponseTimeHours": 4.2
  },
  
  "statusCounts": {
    "sent": 10,
    "delivered": 5,
    "pending_info": 2,
    "acknowledged": 8,
    "accepted": 1200,
    "declined": 30,
    "expired": 15,
    "closed": 270
  },
  
  "volumeTrend": {
    "2026-05-11": 12,
    "2026-05-10": 8,
    "2026-05-09": 15
  },
  
  "lastUpdated": "2026-05-11T10:15:00Z"
}
```

### How Queries Work Now:
`container.item("metrics_tenant_abc_sender", "tenant_abc").read()`
Cost: 1 RU. Time: 2ms. No matter how many transfers the tenant has.