# collection-showcase-data

## Project Overview

Static JSON snapshots of the TCG collection, published automatically by the backend after every mutation or scheduled sync. Both frontends read from this repo via GitHub raw URLs.

**Do not manually edit the JSON files.** The backend owns all writes here.

## Related Repositories

All repos are siblings under the same parent directory:

```
FutureGadgetLabs/
├── collection-showcase-frontend/           ← Reads from this repo
├── collection-showcase-backend/            ← Writes to this repo via GitHub API
├── collection-showcase-immortal-frontend/  ← Reads from this repo
└── collection-showcase-data/              ← This repo
```

GitHub org: https://github.com/FutureGadgetCollections/

## How Data Gets Here

1. **On-demand:** admin frontend calls `POST /sync` (Firebase auth required); backend exports all four JSON files from BigQuery, pushes to GCS, and commits here.
2. **Daily:** Cloud Run Job `collection-showcase-data-sync` runs `cmd/syncdata/main.go`, which does the same export + commit.

Each commit is a full snapshot of the current BigQuery state.

## Data Files

### `products.json`
All TCG products (regardless of current inventory).

Fields: `product_id` (UUID PK), `name`, `game_category`, `game_subcategory`, `product_category`, `tcgplayer_id`, `pricecharting_url`, `listing_url`, `image_url`, `created_at`

### `transactions.json`
Every buy and sell event.

Fields: `transaction_id` (UUID PK), `product_id` (FK), `transaction_date`, `price`, `quantity`, `transaction_type` (`buy`/`sell`), `platform`, `notes`, `created_at`

### `collection.json`
Current inventory — products with net positive quantity. Includes gain/loss metrics.

Fields: `product_id` (FK), `quantity`, `avg_unit_cost`, `total_invested`, `realized_gain`, `unrealized_gain`, `latest_market_price`, `first_buy_date`, `days_held`, `roi`, `annualized_roi`

### `price_history.json`
Periodic market price snapshots from external sources.

Fields: `record_id` (UUID PK), `product_id` (FK), `snapshot_date`, `source`, `market_price`, `median_price`, `sell_through_rate`, `distinct_buyer_count`, `listed_count`, `created_at`

## How Frontends Read This Data

**Both frontends** (primary read path):
```
https://raw.githubusercontent.com/FutureGadgetCollections/collection-showcase-data/main/<file>.json
```

**Admin frontend** also has a GCS fallback:
```
https://storage.googleapis.com/collection-showcase-data/<file>.json
```

## GCP Infrastructure (for context)

The backend syncs to:
- GCS bucket: `collection-showcase-data` (project `future-gadget-labs-483502`)
- BigQuery (source of truth): project `future-gadget-labs-483502`, datasets `inventory` and `market_data`
