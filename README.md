# Coffee Shop CRM: real-time dashboard

A live analytics dashboard for a single independent coffee shop. Every register sale appears on the dashboard the moment it is written to the database, with no polling and no page refresh. The shop owner's manual stock edits show up live too.

Built by [Shivam Dawar](https://github.com/legend12-rgb) as a portfolio project.

**Stack:** Next.js (App Router) · TypeScript · Tailwind CSS v4 · Recharts · Supabase (Postgres + Realtime) · Python 3.12 scripts

> **Live demo:** none at the moment. The dashboard reads from a Supabase project, and the demo project it was built against is no longer online. To run it, create your own free Supabase project and follow "Run it" below.

## What it does

| Page | What you see |
|---|---|
| **Overview** | Three findings about where the money really comes from, revenue and profit KPIs, profit by product (bar colour shows margin), revenue vs profit share by coffee type, order-value distribution, sales over time with `30D / 90D / 6M / ALL` range chips |
| **Customers** | One-time vs repeat vs dormant customers, a loyalty-card scorecard, and geography shown as a concentration-risk view with a ranked country and city list |
| **Inventory & live** | Live products table with low-stock flags, a live stock-movement ledger, orders by hour |
| **Live orders** | Full-page realtime feed of the latest 50 orders and today's running totals |

Every chart expands into a larger view with its own date-range control and a live recap strip. Colour encodes data (margin scale, stock health), not decoration. Dates, times and stock levels in the bundled sample data are synthetic and are labelled as demo data in the interface.

## How it is built

- **One realtime channel.** A single `DashboardProvider` (`dashboard/src/lib/DashboardContext.tsx`) holds all fetched data and one Supabase Realtime subscription. It sits in the root layout, so moving between pages never re-subscribes or re-fetches.
- **Read-only browser.** The browser uses the Supabase anon key. Row-level security and table grants give that role `SELECT` only (see section 5 of `db/schema.sql`). Bulk and admin writes run from trusted Python scripts with the service-role key, which never goes into the front end.
- **Inventory is a ledger, not app bookkeeping.** Placing an order decrements stock through a database trigger, and a second trigger writes a `sale` row to an append-only `stock_movements` table. The restock board, live feed, KPIs and low-stock alerts therefore all stay in sync, and a sale is never counted twice.
- **Money is `numeric(10,3)`.** The source prices have three decimals, and two-decimal columns would silently round them.
- **Honest nulls.** 366 of the 1,000 sample customers have never ordered, so their `customer_since` is `NULL` rather than a faked date.

## Data model

| Table | Purpose |
|---|---|
| `customers` | CRM rows. `customer_since` is the first-order time (`NULL` = never ordered). `created_at` is the row-insert time. |
| `products` | 48 SKUs (coffee type × roast × size). `stock_quantity` is a live on-hand count. |
| `orders` | One product line per order. `order_date` is business time, `created_at` is write time. |
| `stock_movements` | Append-only inventory ledger written by a trigger on `products` (`initial`, `sale`, `restock`). |

## Repository layout

```
db/schema.sql                    full schema, policies, triggers and functions
data/*.csv                       sample data the importer reads (1,000 customers, 1,000 orders, 48 products)
scripts/import_data.py           bulk seed through a direct Postgres connection
scripts/pos_simulator.py         live point-of-sale traffic generator (service-role key)
scripts/listen_realtime.py       realtime verification listener (anon key)
dashboard/                       Next.js dashboard
```

## Run it

You need a free [Supabase](https://supabase.com) project, Node.js and Python 3.12.

```bash
# 1. Python dependencies (a virtual environment is recommended)
pip install -r requirements.txt

# 2. Secrets: copy the template and fill in your own Supabase values
cp .env.example .env

# 3. Create the schema: paste db/schema.sql into the Supabase SQL editor and run it
#    (it drops and recreates the CRM tables)

# 4. Load the sample data
python scripts/import_data.py

# 5. Start the dashboard
cd dashboard
# create dashboard/.env.local with:
#   NEXT_PUBLIC_SUPABASE_URL=...
#   NEXT_PUBLIC_SUPABASE_ANON_KEY=...   (anon key only, never the service-role key)
npm install
npm run dev        # http://localhost:3000
npm run build      # production build (uses webpack; the Turbopack build failed on Google Fonts in testing)
```

To watch it update live, run the point-of-sale simulator in a second terminal. Orders, stock, customers and the inventory ledger will change on the dashboard with no refresh:

```bash
python scripts/pos_simulator.py
```

## Known limitations

- The "Restock" board on the Inventory page calls a Supabase function named `restock_products`. That function is not part of `db/schema.sql` yet, so restocking from the dashboard will fail on a fresh database until it is added. Everything else (live feed, KPIs, charts, ledger) works from the schema and sample data.
- There is no hosted demo, and no screenshots are included for the same reason.

## Security notes

- `.env` and `dashboard/.env.local` are git-ignored. `.env.example` contains placeholders only.
- Only the Supabase anon key is ever used in the browser, and row-level security limits it to reading.
- The service-role key is used only by the local Python scripts.

## Contact

Shivam Dawar · shivamdawar0031@gmail.com · +91 93551 70131
