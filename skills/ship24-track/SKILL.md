---
name: ship24-track
description: Look up a parcel or shipment right now with the Ship24 Tracking API. Use when asked "track this package", "where is my parcel", "what is the status of tracking number <X>", "check delivery status", "look up a tracking number with Ship24", or when a SHIP24_SAMPLE_* number is mentioned. Picks the right Ship24 MCP tool for the account's plan, summarizes the result without inventing events, and falls back to curl when no MCP server is connected. Not for building an integration (that is ship24-integration).
license: MIT
metadata:
  author: Ship24
  docs: https://docs.ship24.com/integrate-with-ai
---

# Track a parcel now

Sources: [Integrate with AI](https://docs.ship24.com/integrate-with-ai), [Trackers](https://docs.ship24.com/trackers),
[Per-call API](https://docs.ship24.com/per-call-api). Tool table in `references/mcp-tools.md`.

## With the bundled MCP server

The plugin connects to `https://api.ship24.com/mcp` as the `ship24-tracking` server. The tools visible depend
on the plan attached to the API key.

| Situation | Tool | Why |
| --- | --- | --- |
| Per-call plan | `search_tracking` | Synchronous lookup, no tracker created. |
| Per-shipment plan, number not tracked yet | `track` | Creates the tracker and returns results in one call; up to one minute. |
| Per-shipment plan, tracker exists | `get_tracking_results` (by `trackerId`) or `search_tracking_by_number` | Reads stored results; the second may return several trackers because tracking numbers are not unique. |
| Need a courier code | `get_couriers` | Once per session, then reuse; the list is large and rate limited to 1 request per second. |

Do not call `create_tracker` and then poll in a conversation: `track` does both in one step. Do not force a
`courierCode` unless the user named the courier; Ship24 detects it.

Every call spends the account's quota exactly like a direct API call.

If `tools/list` works but every call answers `401`, the key is missing or invalid: set a valid `SHIP24_API_KEY`
where the host tool reads it and reconnect. If `search_tracking` is absent, the key has no per-call plan; use `track`.

## Reporting the result

MCP tools return `{ trackings: [...] }`; the curl fallback wraps it as `data.trackings`. For each item:

1. `shipment.statusMilestone` (see `ship24-tracking-statuses` for meanings) and `shipment.statusCode` when set.
2. Latest event: `status` (raw courier text), `occurrenceDatetime`, `location`. Keep the datetime as returned;
   it may be courier local time or a bare date.
3. Delivery estimate only when present: `shipment.delivery.estimatedDeliveryDate`,
   `shipment.delivery.courierEstimatedDeliveryDate` (`from`, `to`), `shipment.delivery.aiPredictiveDeliveryDate`
   (add-on; absent, not `null`, when unavailable).
4. Timeline from `statistics.timestamps` (first occurrence of each milestone) or from `events[]` sorted by
   `occurrenceDatetime` and `order`.
5. Origin and destination from `shipment.originCountryCode` and `shipment.destinationCountryCode`.

Never invent events or dates. If `events` is empty, say the courier has not published events yet
(`statusMilestone` will be `pending`). Tracking numbers come back uppercased.

## Without an MCP server

```bash
# Per-shipment plan: create the tracker and get results synchronously (up to one minute)
curl -sS -X POST https://api.ship24.com/public/v1/trackers/track \
  -H "Authorization: Bearer $SHIP24_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "trackingNumber": "SHIP24_SAMPLE_DELIVERED_000" }'

# Per-call plan: synchronous lookup, no tracker created
curl -sS -X POST https://api.ship24.com/public/v1/tracking/search \
  -H "Authorization: Bearer $SHIP24_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "trackingNumber": "SHIP24_SAMPLE_DELIVERED_000" }'
```

Both answer `{ "data": { "trackings": [ ... ] } }`. Per-call results have no `tracker` object. Add
`courierCode`, `destinationCountryCode` or `destinationPostCode` only when the user provided them.

## Demo numbers

`SHIP24_SAMPLE_<MILESTONE>_000` returns a fixed result for that milestone (`DELIVERED`, `IN_TRANSIT`,
`OUT_FOR_DELIVERY`, `EXCEPTION`, `PENDING`, `INFO_RECEIVED`, `FAILED_ATTEMPT`, `AVAILABLE_FOR_PICKUP`).
Change the last three digits to create a fresh tracker. Full list at
https://docs.ship24.com/status#ship24-sample-tracking-numbers.

## Sibling skills

`ship24-integration` to build a persistent integration, `ship24-couriers` for courier codes,
`ship24-troubleshooting` when a call fails.
