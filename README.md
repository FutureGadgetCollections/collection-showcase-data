# collection-showcase-data

Public, read-only data warehouse for the FutureGadgetCollections showcase. Contains static JSON snapshots of TCG collection inventory, transactions, market price history, and current holdings — published automatically by the backend after every mutation.

## Related Repositories

| Repo | Role |
|------|------|
| `collection-showcase-frontend` | Hugo admin UI — reads from this repo |
| `collection-showcase-backend` | Go/Cloud Run API — writes BigQuery, then publishes here |
| **This repo** | Static JSON snapshots consumed by the frontend |

---

## How the Data Gets Here

All writes originate from the **backend** ([collection-showcase-backend](https://github.com/FutureGadgetCollections/collection-showcase-backend)), which follows this flow on every mutation:

1. An authenticated admin performs a create, update, or delete operation via the frontend UI.
2. The frontend sends the request to the backend API on Cloud Run with a Firebase Auth Bearer token.
3. The backend verifies the token, applies the change to **BigQuery** (the source of truth), and re-exports all four JSON files.
4. The freshly exported JSON is committed and pushed to this repository **and** synced to a GCS bucket as a fallback.

This repo is never written to directly — it is always the output of a backend export. The git commit history reflects every data sync event.

---

## How the Frontend Uses This Data

The **frontend** ([collection-showcase-frontend](https://github.com/FutureGadgetCollections/collection-showcase-frontend)) is a Hugo static site that reads exclusively from these static JSON files — it never queries BigQuery directly. This keeps infrastructure costs near zero.

**Read path (primary):**
```
https://raw.githubusercontent.com/FutureGadgetCollections/collection-showcase-data/main/<file>.json
```

**Read path (GCS fallback):**
```
https://storage.googleapis.com/collection-showcase-data/<file>.json
```

The frontend fetches from GitHub Raw first and falls back to GCS if unavailable. All four files are fetched on page load and used to populate the collection browser, transaction log, and price history views.

---

## Data Files

### `products.json`

A flat array of every TCG product tracked in the system, regardless of whether it is currently in inventory.

| Field | Type | Description |
|-------|------|-------------|
| `product_id` | string (UUID) | Primary key |
| `name` | string | Product name (e.g. "Scarlet & Violet Booster Box") |
| `game_category` | string | Game family — e.g. `pokemon`, `mtg`, `yugioh`, `lorcana` |
| `game_subcategory` | string | Set or sub-series name |
| `product_category` | string | Product type — e.g. `booster_box`, `etb`, `single`, `tin`, `bundle` |
| `tcgplayer_id` | string | TCGPlayer product ID (empty if not listed) |
| `pricecharting_url` | string | PriceCharting URL for price lookups |
| `listing_url` | string | Active marketplace listing URL, if any |
| `created_at` | string (ISO 8601) | When the product was added to the system |

---

### `transactions.json`

A log of every buy and sell event across all products.

| Field | Type | Description |
|-------|------|-------------|
| `transaction_id` | string (UUID) | Primary key |
| `product_id` | string (UUID) | Foreign key → `products.json` |
| `transaction_date` | string (ISO 8601) | Date the transaction occurred |
| `price` | number | Per-unit price paid or received |
| `quantity` | number | Number of units in this transaction |
| `transaction_type` | string | `buy` or `sell` |
| `platform` | string | Where the transaction happened — e.g. `tcgplayer`, `ebay`, `facebook`, `local` |
| `notes` | string | Optional free-text notes |
| `created_at` | string (ISO 8601) | When the record was created |

---

### `collection.json`

The current inventory — a derived view showing only products with a net positive quantity on hand. This is computed by the backend from the transactions table (`buys − sells`) and re-exported after every change.

| Field | Type | Description |
|-------|------|-------------|
| `product_id` | string (UUID) | Foreign key → `products.json` |
| `quantity` | number | Units currently held (always > 0) |
| `avg_unit_cost` | number | Weighted average cost basis across all buy transactions |
| `total_invested` | number | `quantity × avg_unit_cost` — total capital deployed |

---

### `price_history.json`

Periodic market price snapshots for products, pulled from external sources such as TCGPlayer and PriceCharting.

| Field | Type | Description |
|-------|------|-------------|
| `record_id` | string (UUID) | Primary key |
| `product_id` | string (UUID) | Foreign key → `products.json` |
| `snapshot_date` | string (ISO 8601) | Date the snapshot was taken |
| `source` | string | Data source — e.g. `tcgplayer`, `pricecharting`, `ebay` |
| `market_price` | number | Market price at snapshot time |
| `median_price` | number | Median sale price, if available |
| `sell_through_rate` | number | Ratio of listings sold, if available |
| `distinct_buyer_count` | number | Number of unique buyers, if available |
| `listed_count` | number | Number of active listings, if available |
| `created_at` | string (ISO 8601) | When the record was created |

---

## License

See [LICENSE](./LICENSE).
