# BizNetAI MCP Server

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server that routes
natural-language shopping queries to live, independent merchant storefronts and
returns normalized product and merchant results — built for AI agents and shopping
assistants that need real-time commerce data without integrating each merchant
individually.

This repository documents the **hosted service** — there is nothing to install or
run locally. Point your MCP client at the endpoint below with an API key and start
calling tools.

---

## Endpoint

| | |
|---|---|
| **URL** | `https://biznetaimcp.consumergenie.net/mcp` |
| **Transport** | `streamable-http` |
| **Auth** | Required — `Authorization: Bearer <api_key>` on every request |

The server is stateless per request — there is no session handshake to perform first.

---

## Getting an API Key

Access is self-serve:

1. Submit a request with your email, name, and a short description of your use case:
   ```bash
   curl -X POST https://api.merchant.registration.consumergenie.net/api/developer-keys \
     -H "Content-Type: application/json" \
     -d '{"email": "you@example.com", "name": "Your Name", "reason": "Building an AI shopping assistant"}'
   ```
2. Once approved, you'll receive an email with your key (`bnai_live_...`). It's shown
   once and never stored in plaintext anywhere — if you lose it, request a new one.

Each key has its own rate limit (default 60 requests/minute). Exceeding it returns
`429` with a `Retry-After` header; a missing, invalid, or revoked key returns `401`.

---

## Connecting

### MCP client (Claude Desktop, Claude Code, etc.)

Most clients speak stdio, so bridge through
[`mcp-remote`](https://www.npmjs.com/package/mcp-remote), passing your key as a header:

```json
{
  "mcpServers": {
    "biznetai": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://biznetaimcp.consumergenie.net/mcp",
        "--header", "Authorization:Bearer ${BIZNETAI_API_KEY}"
      ]
    }
  }
}
```

### Raw HTTP

```bash
BASE_URL="https://biznetaimcp.consumergenie.net/mcp"
API_KEY="bnai_live_..."

curl -X POST "$BASE_URL" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer $API_KEY" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"list_categories","arguments":{}}}'
```

---

## Tools

### `find_products`
Search for products across merchant storefronts, ranked by relevance.

```
query                   str   required   Natural language product search query
country                 str   required   ISO country code (e.g. US, CA)
limit                   int   0          Page size (0 = server default)
offset                  int   0          Results to skip, for paging beyond the first page
merchant_cap            int   0          Max merchants to consider (0 = all matching merchants)
merchant_product_limit  int   0          Max products considered per individual merchant
```

Returns a list of normalized product objects:
```json
{
  "product_id": "gid://shopify/Product/103382155290",
  "title": "Vitamin C Brightening Serum",
  "description": "...",
  "price_min": 24.60,
  "price_max": 24.60,
  "currency": "USD",
  "available": true,
  "url": "https://merchant.com/products/vitamin-c-serum",
  "image_url": "https://cdn.shopify.com/...",
  "store_domain": "merchant.com",
  "merchant_position": 0
}
```

`available` reflects whether at least one product variant is in stock (boolean only —
exact stock counts aren't available from all merchant backends).

### `find_merchants`
Find live merchants matching a query — useful when you want merchant identity before
doing a custom product lookup.

```
query    str   required   Natural language search query
country  str   required   ISO country code (e.g. US, CA)
limit    int   0          Max merchants to return (0 = all live matches)
```

### `list_categories`
Return the full BizNetAI merchant category vocabulary. Useful for understanding what
kinds of merchants are available before querying.

---

## Rate Limits & Errors

| Status | Meaning |
|---|---|
| `401` | Missing, malformed, invalid, or revoked API key |
| `429` | Rate limit exceeded — see `Retry-After` header for when to retry |

---

## Support

Questions or issues with the API — email the address you used to request your key,
or open an issue on this repository.

## License

[MIT](./LICENSE)
