# Ship24 Tracking MCP tools

Server: `https://api.ship24.com/mcp` (Streamable HTTP), authenticated with `Authorization: Bearer <api key>`.
Without a key the server answers only the discovery methods (`initialize`, `notifications/initialized`,
`tools/list`, `ping`); every `tools/call` returns `401`.
Tools are registered per session according to the plan detected for the key; reconnect after a plan change.
Source: https://docs.ship24.com/integrate-with-ai and the server README.

| Tool | Endpoint | Plan | Use for |
| --- | --- | --- | --- |
| `search_tracking` | `POST /tracking/search` | per-call | One-off synchronous lookup, no tracker created |
| `track` | `POST /trackers/track` | per-shipment | Create a tracker and return results in one call (up to one minute); idempotent on the payload |
| `get_tracking_results` | `GET /trackers/{trackerId}/results` | per-shipment | Results of an existing tracker (`searchBy=clientTrackerId` supported) |
| `search_tracking_by_number` | `GET /trackers/search/{trackingNumber}/results` | per-shipment | Results for every tracker with that number |
| `create_tracker` | `POST /trackers` | per-shipment | Register a tracker for later webhooks or polling |
| `bulk_create_trackers` | `POST /trackers/bulk` | per-shipment | Up to 100 trackers; returns `{ status, summary, data, error }`, not the `{ data }` envelope |
| `list_trackers` | `GET /trackers` | per-shipment | Page through trackers (`page`, `limit`, `sort`) |
| `get_tracker` | `GET /trackers/{trackerId}` | per-shipment | Tracker metadata only, no results |
| `update_tracker` | `PATCH /trackers/{trackerId}` | per-shipment | Subscribe or unsubscribe, add courier codes, correct destination before results exist |
| `resend_webhooks` | `POST /trackers/{trackerId}/webhook-events/resend` | per-shipment | Replay every webhook message of a tracker (1 request per second) |
| `download_webhook_history` | `GET /trackers/{trackerId}/webhook-history/download` | per-shipment | Every sent push with request, response and status code; pending (unsent) webhooks are excluded |
| `get_couriers` | `GET /couriers` | all | Full courier list (large, 1 request per second); fetch once and reuse |

Plan visibility: per-call only → `search_tracking`, `get_couriers`; per-shipment only → everything except
`search_tracking`; both → all tools; no active subscription or undetectable plan → all tools with fallback
instructions.

Not available as tools: webhook URL configuration, account and plan management, API key creation. Those live
in the dashboard.
