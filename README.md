# idealo MCP Server Guide

The idealo MCP server brings idealo's product intelligence directly into your AI assistant — search products, compare prices across hundreds of shops, and analyse 30-day price history across 6 European marketplaces (DE, AT, GB, FR, IT, ES) without ever leaving your editor.

> [!NOTE]
> Rate limits apply to all idealo MCP server tools.
> <br><br>
> The default **Trial** tier is self-serve via OAuth and gives you `3 req/s · 6 burst · 500 calls/day` per user (enforced per OAuth identity). Trial responses include a single product image, cap `get_product_offers` at 10 offers per product, and omit shop logos, test-report logos/links, and the full `userReviews` payload on `get_product_details` (search still returns `userReviewsSummary`).
> <br><br>
> **Partner tier** uses `10 req/s · 30 burst · 10,000 calls/day` and returns full image galleries (up to 7), shop logos, test-report assets, and user reviews. To request Partner access, contact us via mcp@idealo.de.

For the complete tool reference (input schemas, sample responses, tier-by-tier field differences), see **[docs/tools.md](./docs/tools.md)**.

## Features

- **Catalog search** — Full-text search with filters for category, brand, price range, availability, and bargains across 6 European marketplaces. Returns an encrypted `productRef` you pass to the other tools.

- **Product details** — Fetch a complete product profile: top offers, reviews summary, expert review, test reports, technical specifications, pros/cons, and a 30-day price trend in a single call.

- **Paginated offers** — Walk every offer for a product with shop name, ratings, pricing breakdown, delivery estimates, payment methods, free-return policy, and shipping costs. 5 offers per page.

- **Price history & analytics** — 30-day trend with formatted prices, min/max/avg statistics, period change %, sampling granularity, and confidence metadata (`get_product_price_history`; `get_product_details` includes a compact raw-cent snippet only).

## Installation & Setup

Different MCP clients require slightly different setups. Follow the instructions below for your specific client.

### Claude Desktop

The recommended way is via **Custom Connectors**:

1. Open Claude → **Settings → Connectors**.
2. Click **Add custom connector**.
3. Enter the URL `https://mcp.idealo.com/mcp` and click **Add**.
4. Authenticate when prompted — Claude opens a browser for the OAuth handshake.

<details>
<summary>Manual setup</summary>

Add the following to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or the equivalent on your platform:

```json
{
  "mcpServers": {
    "idealo": {
      "type": "url",
      "url": "https://mcp.idealo.com/mcp"
    }
  }
}
```

Restart Claude Desktop and confirm the `idealo` server is listed under MCP tools.

</details>

### Claude Code

Run the following command in your terminal:

```bash
claude mcp add --transport http idealo https://mcp.idealo.com/mcp --callback-port 6274
```

<details>
<summary>Manual setup</summary>

Add to `.mcp.json` at the project root (Claude Code reads it automatically):

```json
{
  "mcpServers": {
    "idealo": {
      "type": "url",
      "url": "https://mcp.idealo.com/mcp",
      "oauth": {
        "callbackPort": 6274
      }
    }
  }
}
```

Useful commands:

- `claude mcp list` — list configured servers
- `claude mcp get idealo` — show server details
- `claude mcp remove idealo` — remove the server

</details>

### VS Code

1. Use the shortcut `⌘ Shift P` to search for `MCP: Add Server`.
2. Select `HTTP`.
3. Paste `https://mcp.idealo.com/mcp` and hit `Enter`.
4. When prompted for a server ID, enter `idealo`.
5. Choose **global** or **workspace** scope.

The resulting `mcp.json` will look like:

```json
{
  "servers": {
    "idealo": {
      "type": "http",
      "url": "https://mcp.idealo.com/mcp"
    }
  }
}
```

Open the chat toolbar, switch to **Agent** mode, and type `#search_products` to confirm the tools are available. If nothing shows up, restart VS Code.

> [!NOTE]
> You must have [GitHub Copilot](https://github.com/features/copilot) enabled to use MCP in VS Code.

### Cursor

<details>
<summary>Manual setup</summary>

1. Open **Cursor → Settings → Cursor Settings**.
2. Go to the **MCP** tab.
3. Click **+ Add new global MCP server**.
4. Enter and save:

```json
{
  "mcpServers": {
    "idealo": {
      "url": "https://mcp.idealo.com/mcp"
    }
  }
}
```

</details>

### Other editors

Any client that supports the **Streamable HTTP** transport can connect. Use:

```json
{
  "mcpServers": {
    "idealo": {
      "url": "https://mcp.idealo.com/mcp"
    }
  }
}
```

## Authentication

The server uses OAuth 2.0 with **Dynamic Client Registration (DCR) + PKCE**. Spec-compliant MCP clients discover the auth flow automatically through `.well-known/oauth-protected-resource` — no `client_id` / `client_secret` setup required for the Trial tier. On first connect, your client opens a browser for login. Tokens are cached client-side and refreshed transparently.

Partner-tier credentials are issued separately by idealo upon request.

## Prompting your MCP client

Once connected, prompt your client in plain language. The four tools chain via an opaque `productRef` from `search_products`.

Recommended flow: broad `search_products` → refine with filter IDs from the response (`availableCategories`, `availableManufacturers`, `availableFilters`, `sizeAttributes`) → `get_product_details` → `get_product_offers` and/or `get_product_price_history`.

Examples:

- *"Find running shoes under €100 available in Germany."*
- *"Show me details, top offers, and pros/cons for the cheapest result."*
- *"Walk through every offer until I see one that ships free with PayPal accepted."*
- *"Give me the 30-day price history and tell me whether now is a good time to buy."*

## Tips & Best Practices

- **Use `productRef`.** `search_products` returns an encrypted `productRef` per result — pass that string directly to `get_product_details`, `get_product_offers`, and `get_product_price_history`. It's opaque, marketplace-aware, and short-lived.
- **`includeSearchFilters` and `includeSizeFilters` require `includeCategories`.** The server rejects calls that send attribute or size filters without at least one numeric category ID. Search first without filters to discover available `categoryId`s in the response.
- **Pick the right country.** Pass `country` as a 2-letter code (`DE`, `AT`, `GB` or `UK`, `FR`, `IT`, `ES`). Unknown codes silently fall back to Germany (`DE`).
- **Paginate offers properly.** `get_product_offers` returns 5 offers per call plus a `nextOffset`. Loop until `nextOffset === 0`.
- **Some shops are filtered out.** Compliance blocklist removes Amazon, eBay, and Otto offers from all tiers — this is intentional and applies regardless of tier.
- **Hit the same marketplace consistently.** A `productRef` from `country=DE` won't return useful data when reused with `country=GB`.

## Supported markets

🇩🇪 Germany · 🇬🇧 United Kingdom · 🇫🇷 France · 🇮🇹 Italy · 🇪🇸 Spain · 🇦🇹 Austria

## Support

- Website: [idealo.de](https://www.idealo.de)
- Tool reference: [docs/tools.md](./docs/tools.md)
- Partner-tier requests: mcp@idealo.de
- Request support for a new MCP client: mcp@idealo.de
