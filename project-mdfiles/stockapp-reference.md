# Stock Tracker App Reference

Covers the Streamlit GUI (`app.py`) built on top of your original `update_quotes.py`
and `generate_docs.py` scripts. This documents what each screen does and how
data flows — not a REST API (the app has no external API endpoints of its
own), just a reference for using and maintaining it.

**Access:** via the current Cloudflare Tunnel URL (changes on VM/tunnel restart —
see command below to fetch the current one).

```bash
gcloud compute ssh stockapp-vm --zone=us-central1-a --command="sudo journalctl -u cloudflared --no-pager | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | tail -1"
```

**Database:** `/opt/stockapp/data/stock_reports.db` (SQLite) — a single file
holding everything: companies, quotes, fundamentals, portfolios, transactions,
and groups.

---

## Pages

### Dashboard
Landing page. Shows a quick overview: list of your portfolios (name + owner),
list of groups (name + symbol count), and a count of every symbol ever tracked.

### Fetch Quotes
Pulls the latest price/quote data from Yahoo Finance for whichever symbols you
select. Choose **one** way to select symbols:
- **Everything tracked** — refreshes every symbol the database has ever seen
- **Specific symbols** — type them in directly
- **Group** — all symbols in a named group (see Groups below)
- **Sector / Industry / Exchange** — all symbols matching that filter
- **Portfolio** — all symbols currently held in one or more portfolios

This is the screen you use for your day-to-day "get updated quotes" task.

### Portfolios
Three tabs:
- **View / Buy / Sell** — pick a portfolio, see its current positions (shares,
  average cost, current value), and record new Buy/Sell transactions (symbol,
  quantity, price per share, date)
- **Transaction Log** — full history of buys/sells for a portfolio; delete a
  transaction by its ID if needed
- **New Portfolio** — create a new portfolio under one of the predefined owners

### Groups
Named collections of symbols (e.g. "Tech Watchlist," "Dow 30") independent of
any portfolio — useful for fetching quotes on a themed set of stocks without
owning them.
- **View** — quotes for every symbol in a group
- **Add / Remove Symbols** — manage group membership
- **Seed Built-ins** — one-click population of the Dow 30 or S&P 500 (the
  latter downloads a constituent list, takes a moment)

### Value Screens
Two stock-screening tools that filter your tracked universe:
- **P/E Value Screen** — filters by P/E ratio range, minimum price, minimum
  market cap, grouped by sector or industry
- **Deep Value Screen** — filters by fundamentals: max P/B ratio, min ROE,
  max debt/equity, max PEG ratio

### Company Lookup
Look up a single symbol across three tabs: **Company Info** (sector, industry,
exchange, etc.), **Quote History** (price history, toggle full history), and
**Fundamentals History** (P/E, market cap, ROE, etc. over time).

### Bulk Import
Upload a CSV to make many changes in one shot instead of clicking through the
UI. Click **Generate starter template** for an example CSV with one row per
supported action:

| Action | What it does |
|---|---|
| `BUY` | Record a purchase transaction |
| `SELL` | Record a sale transaction |
| `EDIT_TRANSACTION` | Modify an existing transaction by ID |
| `DELETE_TRANSACTION` | Remove a transaction by ID |
| `DELETE_PORTFOLIO` | Remove an entire portfolio |
| `WATCHLIST` | Add a symbol to a group without buying it |
| `REMOVE_FROM_GROUP` | Remove a symbol from a group |
| `ADD_COMPANY` | Add a new company/symbol to the tracked universe |
| `EDIT_COMPANY` | Update a company's stored details |
| `DELETE_COMPANY` | Remove a company entirely |

After upload, it reports how many rows succeeded and lists any row-level errors.

### Documentation PDF
Regenerates a PDF (from `generate_docs.py`) documenting `update_quotes.py`'s
features, and offers it as a download.

---

## Data model (SQLite tables, inferred from the app's functions)

- **Companies** — symbol, sector, industry, exchange, and other static info
- **Quotes** — historical price data per symbol, timestamped
- **Fundamentals** — historical P/E, P/B, ROE, debt/equity, PEG, market cap, etc. per symbol
- **Portfolios** — name + owner (owner chosen from a predefined list, `PORTFOLIO_OWNERS`)
- **Portfolio transactions** — buy/sell records: portfolio, symbol, type, date, quantity, price
- **Groups** — named symbol collections, many-to-many with companies

---

## Maintenance

**Restart the app after any code change:**
```bash
sudo systemctl restart stockapp
```

**Check it's actually running:**
```bash
sudo systemctl status stockapp --no-pager
```

**Watch logs live (useful if something's not loading):**
```bash
sudo journalctl -u stockapp -f
```

**Back up the database** (copies it from the VM down to Cloud Shell):
```bash
gcloud compute scp stockapp-vm:/opt/stockapp/data/stock_reports.db ~/stock_reports_backup.db --zone=us-central1-a
```

**Get the current tunnel URL** (see top of this doc):
```bash
gcloud compute ssh stockapp-vm --zone=us-central1-a --command="sudo journalctl -u cloudflared --no-pager | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | tail -1"
```

**Deploy an updated `app.py`:**
```bash
gcloud compute scp app.py stockapp-vm:~/app.py --zone=us-central1-a
gcloud compute ssh stockapp-vm --zone=us-central1-a
```
```bash
sudo cp ~/app.py /opt/stockapp/app.py
sudo chown stockapp:stockapp /opt/stockapp/app.py
grep -c "<known unique string from the new file>" /opt/stockapp/app.py   # confirm before restarting
sudo systemctl restart stockapp
```
