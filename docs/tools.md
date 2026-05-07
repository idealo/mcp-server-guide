# idealo MCP — Tool Reference

Detailed reference for every tool exposed by the idealo MCP server: input schema, sample request, sample response, and tier-by-tier field differences.

> [!NOTE]
> All response samples are illustrative. Actual values depend on the marketplace, product, and current shop offers.

## Tiers at a glance

| Capability | Trial (default) | Partner |
|---|---|---|
| Rate limit | 4 req/s · 8 burst · 200/day (per user) | Higher (contact mcp@idealo.de) |
| Product images per response | 1 | up to 7 |
| `offers[].shop.logoUrl` | omitted | included |
| `offers[].shop.rating.url` | omitted | included |
| `testReports[].logoUrl` | omitted | included |
| `testReports[].externalUrl` | omitted | included |
| `userReviews` (full payload + comments) | omitted | included |

Trial vs Partner is selected server-side based on the OAuth principal — the same tools and inputs work for both tiers; only the response shape differs.

---

## `search_products`

Search the idealo product catalog across 6 European marketplaces. Returns product results plus filter metadata you can use to refine subsequent searches.

### Input schema

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `query` | `string` (min 1) | ✅ | — | Free-text search query |
| `country` | `string` (length 2) | ❌ | `DE` | Two-letter ISO code: `DE`, `AT`, `GB`, `FR`, `IT`, `ES`. Unknown codes fall back to `DE`. |
| `filters.minPrice` | `number` | ❌ | — | Minimum price (whole EUR) |
| `filters.maxPrice` | `number` | ❌ | — | Maximum price (whole EUR) |
| `filters.availableOnly` | `boolean` | ❌ | `false` | Show only in-stock products |
| `filters.bargainsOnly` | `boolean` | ❌ | `false` | Show only discounted offers |
| `filters.includeCategories` | `string[]` (numeric) | ❌ | — | Numeric category IDs, e.g. `["30311"]`. **Required when using `includeSearchFilters` or `includeSizeFilters`.** |
| `filters.includeManufacturers` | `string[]` | ❌ | — | Manufacturer / brand IDs |
| `filters.includeSearchFilters` | `string[]` | ❌ | — | Attribute filter IDs (color, material, etc.). **Requires `includeCategories`.** |
| `filters.includeSizeFilters` | `string[]` | ❌ | — | Size filter IDs. **Requires `includeCategories`.** |

### Sample request

```json
{
  "query": "macbook",
  "country": "DE",
  "filters": {
    "minPrice": 800,
    "maxPrice": 1500,
    "availableOnly": true,
    "includeCategories": ["30311"]
  }
}
```

### Sample response

```json
{
  "query": "macbook",
  "results": [
    {
      "productRef": "pr.v1.k1.base64encodedtoken123==",
      "itemType": "PRODUCT",
      "siteId": 1,
      "name": "Apple MacBook Air 13\" M2",
      "price": "999,00 €",
      "imageUrl": "https://cdn.idealo.com/folder/Product/12345/1/12345_1.jpg",
      "url": "https://www.idealo.de/preisvergleich/OffersOfProduct/12345.html",
      "totalOfferCount": 42,
      "userReviewsSummary": { "averageRating": 4.5, "totalReviewCount": 128 },
      "testReportsSummary": { "grade": "gut (2,1)" }
    }
  ],
  "availableCategories": [
    { "categoryId": "30311", "categoryName": "Notebooks", "amount": 245 }
  ],
  "availableManufacturers": [
    { "manufacturerId": "356", "manufacturerName": "Apple", "amount": 3, "searchFilterId": null }
  ],
  "availableFilters": [
    {
      "attributeId": "12345",
      "name": "Farbe",
      "filters": [
        { "filterId": "f1", "content": "Space Grau", "amount": 2 },
        { "filterId": "f2", "content": "Silber", "amount": 1 }
      ]
    }
  ],
  "sizeAttributes": [],
  "suggestedFilters": [{ "id": "356", "type": "MANUFACTURER" }],
  "searchResultsUrl": "https://www.idealo.de/preisvergleich/?q=macbook"
}
```

### Tier differences

| Field | Trial | Partner |
|---|---|---|
| `results[].imageUrl` | single image (already a single field) | single image (same) |

`search_products` itself returns the same shape across tiers — the divergence shows up downstream in `get_product_details` and `get_product_offers`.

### Errors & quirks

- Sending `includeSearchFilters` without `includeCategories` throws `FilterValidationError`.
- Category IDs must match `^\d+$` (numeric strings). Non-numeric IDs are rejected.

---

## `get_product_details`

Fetch a complete profile for a single product: top offers, reviews, expert review, test reports, specifications, pros/cons, and a 30-day price trend.

### Input schema

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `productRef` | `string` (min 1) | ✅ | — | Encrypted reference from `search_products`, format `pr.v1.<kid>.<base64>` |
| `country` | `string` (length 2) | ❌ | `DE` | Marketplace code |
| `type` | `"PRODUCT" \| "CLUSTER" \| "SINGLE_OFFER"` | ❌ | `PRODUCT` | Item type |

### Sample request

```json
{
  "productRef": "pr.v1.k1.base64encodedtoken123==",
  "country": "DE"
}
```

### Sample response (Partner tier — full payload)

Fields marked **⭐ Partner only** are omitted or truncated for Trial.

```json
{
  "productRef": "pr.v1.k1.base64encodedtoken123==",
  "itemType": "PRODUCT",
  "siteId": 1,
  "name": "Apple MacBook Air 13\" M2",
  "url": "https://www.idealo.de/preisvergleich/OffersOfProduct/12345.html",

  // ⭐ Partner: full array. Trial: truncated to 1 image.
  "imageUrls": [
    "https://cdn.idealo.com/folder/Product/12345/1/12345_1.jpg",
    "https://cdn.idealo.com/folder/Product/12345/2/12345_2.jpg",
    "https://cdn.idealo.com/folder/Product/12345/3/12345_3.jpg"
  ],

  "nextOffset": 0,
  "offers": [
    {
      "key": "offer1",
      "relocatorUrl": "https://www.techstore.de/macbook-air",
      "price": "999,00 €",
      "basePrice": "999,00 €",
      "shop": {
        "shopId": "123",
        "merchantId": "456",
        "name": "TechStore",
        "merchantName": "TechStore GmbH",
        "logoUrl": "https://cdn.idealo.com/shops/techstore.png",  // ⭐ Partner only
        "isMarketplace": false,
        "rating": {
          "avg": 4.8,
          "count": 1234,
          "url": "https://www.idealo.de/preisvergleich/Shop/123.html"  // ⭐ Partner only
        }
      },
      "payments": [
        { "costCode": "pp", "name": "PayPal" },
        { "costCode": null, "name": "Standard" }
      ],
      "delivery": {
        "deliveryText": "1-2 Werktage",
        "deliveryStatus": "SHORT",
        "earlyDeliveryDate": "2024-01-05",
        "lateDeliveryDate": "2024-01-06"
      },
      "freeReturnDays": 30,
      "cheapestTotalPriceOffer": true,
      "freshness": "1h",
      "minShippingCost": "0,00 €",
      "hasAlwaysFreeShipping": true
    }
  ],

  // ⭐ Partner only — entire `userReviews` block omitted for Trial.
  "userReviews": {
    "summary": {
      "averageRating": 4.5,
      "totalReviewCount": 128,
      "totalReviewWithCommentCount": 64,
      "ratingDistribution": [
        { "count": 80, "rating": 5, "roundedPercentage": 63 },
        { "count": 30, "rating": 4, "roundedPercentage": 23 }
      ],
      "providerDistribution": [
        { "count": 100, "displayName": "idealo", "group": "IDEALO" }
      ]
    },
    "list": {
      "userReviews": [
        {
          "comment": "Great laptop, very fast!",
          "createdAt": 1704067200000,
          "language": "de",
          "ratingValue": 5,
          "title": "Excellent product",
          "verified": true
        }
      ]
    }
  },

  "prosCons": {
    "pros": ["Sehr schneller M2 Chip", "Lange Akkulaufzeit"],
    "cons": ["Nur 2 USB-C Ports", "Speicher nicht erweiterbar"]
  },

  "testReports": {
    "summary": {
      "grade": "gut (2,1)",
      "averageRating": 2.1,
      "rating": "gut",
      "ratingCount": 3
    },
    "list": [
      {
        "providerName": "Stiftung Warentest",
        "shortDescription": "Leichtes und schnelles Notebook",
        "summaryText": "Das MacBook Air überzeugt durch Performance.",
        "ratingValue": "2,0",
        "publishedAt": 1704067200000,
        "language": "de",
        "logoUrl": "https://cdn.idealo.com/logos/stiftung-warentest.png",  // ⭐ Partner only
        "issue": "01/2024",
        "externalUrl": "https://www.test.de/macbook-air",                  // ⭐ Partner only
        "testedProductsCount": 15
      }
    ]
  },

  "expertReview": {
    "author": "Max Mustermann",
    "authorTitle": "idealo Experte",
    "rating": 4.5,
    "teaser": "Ein ausgezeichnetes Notebook",
    "text": "Das MacBook Air M2 ist ein hervorragendes Notebook..."
  },

  "productSpecification": {
    "groups": [
      {
        "name": "Prozessor",
        "type": "DEFAULT",
        "attributes": [
          { "name": "Prozessor", "value": "Apple M2" },
          { "name": "Anzahl Kerne", "value": "8" }
        ]
      }
    ]
  },

  "productOverview": ["8-Core CPU, 8-Core GPU", "8 GB RAM", "256 GB SSD"],
  "productDetailsUrl": "https://www.idealo.de/preisvergleich/OffersOfProduct/12345.html",

  "priceTrend": {
    "history": [
      { "dateTime": 1701388800000, "price": 105900 },
      { "dateTime": 1706745600000, "price": 99900 }
    ]
  }
}
```

### Tier differences

| Field | Trial | Partner |
|---|---|---|
| `imageUrls` | truncated to 1 entry | up to 7 entries |
| `userReviews` | omitted | full summary + reviews list |
| `offers[].shop.logoUrl` | omitted | included |
| `offers[].shop.rating.url` | omitted | included |
| `testReports[].logoUrl` | omitted | included |
| `testReports[].externalUrl` | omitted | included |

All other fields (specifications, pros/cons, expert review, price trend, offer pricing/delivery) are identical across tiers.

---

## `get_product_offers`

Paginated list of offers for a product. Use after `get_product_details` (which only returns 3 offers) to walk every shop selling the product.

### Input schema

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `productRef` | `string` (min 1) | ✅ | — | Encrypted reference from `search_products` |
| `country` | `string` (length 2) | ❌ | `DE` | Marketplace code |
| `type` | `"PRODUCT" \| "CLUSTER" \| "SINGLE_OFFER"` | ❌ | `PRODUCT` | Item type |
| `offset` | `number` (≥ 0) | ❌ | `0` | Pagination offset; pass `nextOffset` from previous response |

**Page size** is fixed at 5 offers per call. `nextOffset === 0` in the response signals the end of results.

### Sample request

```json
{
  "productRef": "pr.v1.k1.base64encodedtoken123==",
  "country": "DE",
  "offset": 0
}
```

### Sample response (Partner tier)

```json
{
  "nextOffset": 5,
  "offers": [
    {
      "key": "offer1",
      "relocatorUrl": "https://www.techstore.de/macbook-air",
      "price": "999,00 €",
      "basePrice": "999,00 €",
      "shop": {
        "shopId": "123",
        "merchantId": "456",
        "name": "TechStore",
        "merchantName": "TechStore GmbH",
        "logoUrl": "https://cdn.idealo.com/shops/techstore.png",  // ⭐ Partner only
        "isMarketplace": false,
        "rating": {
          "avg": 4.8,
          "count": 1234,
          "url": "https://www.idealo.de/preisvergleich/Shop/123.html"  // ⭐ Partner only
        }
      },
      "payments": [
        { "costCode": "pp", "name": "PayPal" },
        { "costCode": null, "name": "Versandkostenfrei" }
      ],
      "delivery": {
        "deliveryText": "1-2 Werktage",
        "deliveryStatus": "SHORT",
        "earlyDeliveryDate": "2024-01-05",
        "lateDeliveryDate": "2024-01-06"
      },
      "freeReturnDays": 30,
      "cheapestTotalPriceOffer": true,
      "freshness": "1h",
      "minShippingCost": "0,00 €",
      "hasAlwaysFreeShipping": true
    }
  ]
}
```

### Tier differences

| Field | Trial | Partner |
|---|---|---|
| `offers[].shop.logoUrl` | omitted | included |
| `offers[].shop.rating.url` | omitted | included |

Pricing, delivery, payments, return policy, and freshness are identical across tiers.

### Pagination workflow

1. First call: omit `offset` (defaults to 0).
2. Read `nextOffset` from the response.
3. If `nextOffset > 0`, call again with `offset: <nextOffset>`.
4. Stop when `nextOffset === 0`.

---

## `get_product_price_history`

30-day price history with statistics, trend analysis, ASCII chart, and buying-time insights.

### Input schema

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `productRef` | `string` (min 1) | ✅ | — | Encrypted reference from `search_products` |
| `country` | `string` (length 2) | ❌ | `DE` | Marketplace code |
| `type` | `"PRODUCT" \| "CLUSTER" \| "SINGLE_OFFER"` | ❌ | `PRODUCT` | Item type |

### Sample request

```json
{
  "productRef": "pr.v1.k1.base64encodedtoken123==",
  "country": "DE"
}
```

### Sample response

```json
{
  "priceTrend": {
    "history": [
      {
        "dateTime": 1701388800000,
        "price": "1.059,00 €",
        "dateTimeISO": "2023-11-30T12:00:00.000Z",
        "priceChangePercent": null
      },
      {
        "dateTime": 1706745600000,
        "price": "999,00 €",
        "dateTimeISO": "2024-02-01T12:00:00.000Z",
        "priceChangePercent": -2.92
      }
    ],
    "statistics": {
      "minPrice": "950,00 €",
      "maxPrice": "1.099,00 €",
      "avgPrice": "1.020,00 €",
      "currentPrice": "999,00 €",
      "priceChangePercent": -5.66,
      "lowestPriceDate": "2024-01-15T10:30:00.000Z",
      "highestPriceDate": "2023-12-01T08:15:00.000Z"
    },
    "samplingGranularity": "daily",
    "dataSourceConfidence": 0.95,
    "periodDays": 30,
    "totalDataPoints": 30
  }
}
```

If no history is available (e.g. brand-new product or invalid `productRef` for the given marketplace):

```json
{ "priceTrend": null }
```

### Tier differences

`get_product_price_history` returns the same shape across tiers — pricing analytics are not gated.

### Currency formatting

| Marketplace | Format |
|---|---|
| `DE`, `AT`, `FR`, `IT`, `ES` | `"999,00 €"` (comma decimal, EUR after) |
| `GB` | `"£999.99"` (period decimal, GBP before) |

---

## Common parameter notes

### `productRef`

Always pass it through unchanged from the source response (`search_products` results) to other tools. It is marketplace-aware — a `productRef` produced for `country=DE` will not yield meaningful data when reused with `country=GB`.

### `country` → `siteId` mapping

| Country | `siteId` |
|---|---|
| `DE` | 1 |
| `AT` | 2 |
| `GB` (alias `UK`) | 3 |
| `FR` | 4 |
| `IT` | 10 |
| `ES` | 11 |

### `ItemType` enum

```
PRODUCT       — standard product
CLUSTER       — variant cluster (e.g. multiple sizes/colors of the same item)
SINGLE_OFFER  — single shop's offer
```

### Payment `costCode` values

| Code | Method |
|---|---|
| `pp` | PayPal |
| `cbc` | Credit card |
| `dd` | Direct debit |
| `cia` | Advance payment |
| `inv` | Invoice |
| `amzp` | Amazon Pay |
| `apple_pay` | Apple Pay |
| `google_pay` | Google Pay |
