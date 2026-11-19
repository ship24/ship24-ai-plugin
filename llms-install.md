# Installing the Ship24 Tracking MCP server

These instructions are for AI agents (such as Cline) setting up Ship24's hosted MCP server for a user.
Nothing needs to be cloned, built or run locally: the server is hosted by Ship24.

## What you need from the user

A Ship24 API key. It starts with `apik_`. If the user doesn't have one, they can create it in the
Ship24 dashboard at https://dashboard.ship24.com/integrations/api-keys (in the side menu, Developers,
then the API Keys tab). Suggest a dedicated
key for AI tools so it can be revoked on its own.

Ask the user for the key before writing any configuration. Do not invent or guess one.

## Configuration

Add this entry to the MCP settings file (for Cline, `cline_mcp_settings.json`), replacing
`apik_your_key_here` with the user's key:

```json
{
  "mcpServers": {
    "ship24-tracking": {
      "type": "streamableHttp",
      "url": "https://api.ship24.com/mcp",
      "headers": {
        "Authorization": "Bearer apik_your_key_here"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Keep `"type": "streamableHttp"`. If it is left out, Cline falls back to the legacy SSE transport and
the connection fails.

If the file already has other servers under `mcpServers`, add `ship24-tracking` alongside them rather
than replacing the file.

## Check it works

1. The `ship24-tracking` server should show as connected, with its tools listed. Which tools appear
   depends on the plan attached to the API key.
2. Ask the user for a tracking number, or use one they've already mentioned, and call the tracking
   tool. A working setup returns the parcel's courier and tracking events.

## If something goes wrong

- **401 Unauthorized:** the key is missing, mistyped or revoked. Check the `Authorization` header is
  exactly `Bearer ` followed by the key.
- **Connection fails straight away:** check `"type"` is `"streamableHttp"` and the URL is
  `https://api.ship24.com/mcp`.
- **No tracking tools listed:** the key's plan may not include them. The user can check their plan in
  the Ship24 dashboard.

Calls made through the MCP server count against the user's Ship24 plan quota, exactly like direct API
calls.

More detail: https://docs.ship24.com/integrate-with-ai
