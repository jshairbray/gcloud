# Stock Tracker CLI & API Reference

Two ways to run this outside the web GUI: the **CLI** (direct on the VM) and
a small **JSON API** (over HTTP, for scripts/automation). Both call the exact
same underlying functions as the Streamlit app — nothing's reimplemented.

**⚠️ Path note:** `update_quotes.py` defaults to Android paths
(`/sdcard/Download/...`) that don't exist on the VM. Running it directly
would fail or write to the wrong place. Use the `cli.py` wrapper below
instead — it points at the real data directory, same as the web app does.

**Where to run CLI commands:** SSH into the VM, then:

```bash
gcloud compute ssh stockapp-vm --zone=us-central1-a
```
```bash
cd /opt/stockapp
sudo -u stockapp ./venv/bin/python3 cli.py <arguments>
```

(`sudo -u stockapp` runs it as the same low-privilege user the web app uses,
so it reads/writes the same database correctly. See the bottom of this doc
for one-time setup of `cli.py` on the VM if it's not there yet.)

---

## Fetching quotes

```bash
# Specific symbols
cli.py AAPL MSFT GOOGL

# By group
cli.py --group "Dow 30"

# By sector / industry / exchange
cli.py --sector "Healthcare"
cli.py --industry "Software - Infrastructure"
cli.py --exchange "NASDAQ"

# Every symbol held in one or more portfolios
cli.py --portfolio Growth Retirement
```
Only pick **one** filter at a time — combine with `--owner` if a portfolio name is ambiguous between owners.

---

## Groups (management only — doesn't fetch quotes)

```bash
cli.py --create-group "Tech Watchlist"
cli.py --add-to-group "Tech Watchlist" AAPL MSFT NVDA
cli.py --remove-from-group "Tech Watchlist" NVDA
cli.py --list-groups
cli.py --list-group "Tech Watchlist"
cli.py --show-group "Tech Watchlist"
cli.py --seed-dow30
cli.py --seed-sp500
```

---

## Portfolios (management only — doesn't fetch quotes)

```bash
cli.py --create-portfolio Growth --owner Josh

# Buy: portfolio, symbol, quantity, price — in that order
cli.py --buy Growth AAPL 10 150.25 --date 2026-01-05

# Sell: same shape
cli.py --sell Growth AAPL 5 200.00 --date 2026-03-01

cli.py --list-portfolios
cli.py --show-portfolio Growth
cli.py --show-portfolio Growth Retirement   # multiple at once

cli.py --list-transactions Growth
cli.py --delete-transaction 42
cli.py --edit-transaction 42 --quantity 8 --price 155.00

cli.py --delete-portfolio Growth --owner Josh
```
`--date` defaults to today if omitted on `--buy`/`--sell`.

---

## Companies (management only — doesn't fetch quotes)

```bash
cli.py --add-company TSLA --company-name "Tesla, Inc." --sector "Consumer Cyclical" --industry "Auto Manufacturers" --exchange NASDAQ
cli.py --edit-company TSLA --sector "Industrials"
cli.py --delete-company TSLA
cli.py --show-company AAPL
```

### Symbol quote / fundamentals history

```bash
cli.py --show-symbol AAPL
cli.py --show-symbol AAPL --full-history
cli.py --show-symbol AAPL --start-date 2026-01-01 --end-date 2026-06-30
cli.py --show-symbol AAPL --fields price pe volume

cli.py --show-fundamentals AAPL
cli.py --show-fundamentals AAPL --full-history
```
Valid `--fields` values: `price`, `market_cap` (`mcap`), `pe_ratio` (`pe`), `dividend_yield` (`div`), `day_range`, `week52_range`, `volume` (`vol`), `error`.

---

## Bulk import from CSV (doesn't fetch quotes)

```bash
cli.py --import-template starter.csv
cli.py --import-file my_changes.csv
```
Generates a starter file with example rows for every supported action
(buy/sell, transaction edit/delete, watchlist/group changes, company
add/edit/delete), then applies a filled-in one.

---

## Value screening (doesn't fetch quotes)

```bash
# P/E screen
cli.py --value-screen --group-by sector --min-pe 3 --max-pe 60 --min-price 5 --min-market-cap 1000000000 --limit 25
cli.py --value-screen --group-by industry --scope "Software - Infrastructure"

# Fundamentals-based deep value screen (requires fundamentals data —
# run a normal quote fetch first to populate it)
cli.py --deep-value-screen --max-pb 3 --min-roe 0.10 --max-debt-equity 150 --max-peg 2
```
All numeric flags above show their defaults — omit any you don't want to change.

---

## PDF export

Add `--pdf <name>.pdf` to `--show-portfolio`, `--show-symbol`, `--show-fundamentals`, or `--show-group` to also save a PDF. The original script hardcodes `/sdcard/documents/` — the `cli.py` wrapper redirects this to `/opt/stockapp/data/pdfs/` instead, same as the web app's Documentation PDF button.

```bash
cli.py --show-portfolio Growth --pdf growth_report.pdf
```

Add `--fundamentals` to `--show-portfolio` or `--show-group` to show fundamentals metrics (P/B, ROE, debt/equity, PEG, margins, growth) instead of price/gain-loss — combine with `--fields` to pick specific ones, or `--pdf` to export.

---

## Quick reference: everything at a glance

```bash
cli.py --help
```
Prints the full, authoritative flag list directly from the script — the source of truth if anything above ever drifts from the actual code.

---

## Setting up `cli.py` on the VM (one-time, if not already there)

```bash
gcloud compute scp cli.py stockapp-vm:~/cli.py --zone=us-central1-a
gcloud compute ssh stockapp-vm --zone=us-central1-a
```
```bash
sudo cp ~/cli.py /opt/stockapp/cli.py
sudo chown stockapp:stockapp /opt/stockapp/cli.py
```

---

# JSON API (for scripts, automation, curl)

A small Flask API (`api.py`) runs alongside Streamlit on the same VM,
reusing the exact same functions and database. Every request needs an
`Authorization: Bearer <token>` header.

## One-time setup

```bash
gcloud compute scp api.py api.service stockapp-vm:~ --zone=us-central1-a
gcloud compute ssh stockapp-vm --zone=us-central1-a
```
```bash
sudo cp ~/api.py /opt/stockapp/api.py
sudo chown stockapp:stockapp /opt/stockapp/api.py

# Generate a real token and put it in the service file
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
sudo sed -i "s/CHANGE_ME_SEE_DEPLOY_GUIDE/<paste-the-token-here>/" ~/api.service
sudo cp ~/api.service /etc/systemd/system/api.service
sudo systemctl daemon-reload
sudo systemctl enable --now api
sudo systemctl status api --no-pager   # confirm "active (running)"
```

## Route it through nginx (so it's reachable at /api/ over the same tunnel)

The working nginx config on this VM actually lives at
`/etc/nginx/sites-enabled/stockapp` (not `sites-available/` — check with
`sudo nginx -T | grep "configuration file"` if that ever seems off again).

```bash
sudo nano /etc/nginx/sites-enabled/stockapp
```

Add this location block inside the existing `server { ... }` block, alongside the existing `location /` block:
```
    location /api/ {
        proxy_pass http://127.0.0.1:8503/api/;
        proxy_set_header Host $host;
    }
```

**Port 8503, not 8502** — 8502 is already used by a separate, pre-existing
`stockapp-api` service (FastAPI/uvicorn, `quotes_api.py`) that predates this
one. It has **no authentication** and different routes (no `/api/` prefix —
e.g. `/quote/{symbol}`, `/portfolios`, `/screen/value`). It was left running
as-is rather than replaced. It's not routed through nginx/the tunnel — only
reachable from inside the VM:
```bash
curl -s "http://localhost:8502/quote/AAPL?latest_only=true"
```

Save (`Ctrl+O`, Enter, `Ctrl+X`), then test and reload:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

## Getting `$BASE` and `$TOKEN` (from your local PC / PowerShell)

Neither is looked up automatically — both are plain values you fetch from
the VM and store in local shell variables each session.

**`$BASE`** — the current Cloudflare Tunnel URL (changes if the VM or the
`cloudflared` service has restarted since you last checked):
```powershell
$BASE = (gcloud compute ssh stockapp-vm --zone=us-central1-a --command="sudo journalctl -u cloudflared --no-pager | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | tail -1").Trim()
```
Note: a plain `grep trycloudflare` (no `-oE` pattern) breaks over time — every
tunnel restart logs *two* matching lines ("Requesting new quick Tunnel..." and
the actual URL), so as restarts accumulate in the log history, plain grep
returns multiple lines jammed into one variable, which then breaks curl's URL
parsing ("bad range in position..."). The version above extracts only the
URL pattern itself and keeps just the most recent one with `tail -1`, so it
stays correct no matter how many times the tunnel has restarted over the
VM's lifetime.

**`$TOKEN`** — the API token, read directly from the VM's service file:
```powershell
$TOKEN = (gcloud compute ssh stockapp-vm --zone=us-central1-a --command="grep API_TOKEN /etc/systemd/system/api.service").Split('=')[-1]
```

Both are lost when you close PowerShell — re-run these two commands at the
start of each new session. To avoid re-fetching every time, you can persist
either as a permanent environment variable via your PowerShell profile
(`notepad $PROFILE`, add `$env:TOKEN = "..."`) — reasonable for a personal
machine, just know it then sits in plain text on disk.

### Rotating the token later

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
sudo sed -i "s/API_TOKEN=.*/API_TOKEN=<paste-new-token-here>/" /etc/systemd/system/api.service
sudo systemctl daemon-reload
sudo systemctl restart api
```
The old token stops working the moment you restart the service — update
`$TOKEN` on any machine that was using it.

### PowerShell-specific curl notes

PowerShell's built-in `curl` is actually `Invoke-WebRequest` in disguise —
use `curl.exe` explicitly to get real curl. Line continuation is a backtick
(`` ` ``) in PowerShell, not backslash — easiest to just keep POST commands
on one line, with inner JSON quotes escaped:

```powershell
curl.exe -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{\"symbols\":[\"AAPL\",\"MSFT\",\"GOOGL\"]}' "$BASE/api/fetch"
```

Or skip curl entirely and use PowerShell's native HTTP client, which handles JSON escaping automatically:
```powershell
$headers = @{ Authorization = "Bearer $TOKEN"; "Content-Type" = "application/json" }
$body = '{"symbols":["AAPL","MSFT","GOOGL"]}'
Invoke-RestMethod -Method Post -Uri "$BASE/api/fetch" -Headers $headers -Body $body
```



Set your token and base URL once per terminal session:
```bash
TOKEN="<the token you generated above>"
BASE="https://<your-current-cloudflared-url>"
```

```bash
# Health check (no auth needed)
curl "$BASE/api/health"

# List portfolios
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/portfolios"

# Show one portfolio's holdings
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/portfolio/Growth"

# List a portfolio's transactions
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/portfolio/Growth/transactions"

# Record a buy
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"portfolio":"Growth","symbol":"AAPL","quantity":10,"price":150.25,"date":"2026-01-05"}' \
  "$BASE/api/buy"

# Record a sell
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"portfolio":"Growth","symbol":"AAPL","quantity":5,"price":200.00}' \
  "$BASE/api/sell"

# List groups
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/groups"

# Show a group's quotes
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/group/Dow%2030"

# Fetch fresh quotes -- by explicit symbols
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"symbols":["AAPL","MSFT","GOOGL"]}' \
  "$BASE/api/fetch"

# Fetch fresh quotes -- by group instead
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"group":"Dow 30"}' \
  "$BASE/api/fetch"

# Fetch everything ever tracked (omit body entirely)
curl -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/fetch"

# Company info
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/company/AAPL"

# Symbol quote history
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/symbol/AAPL?full_history=true"

# Fundamentals history
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/fundamentals/AAPL"

# P/E value screen
curl -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/value-screen?group_by=sector&min_pe=3&max_pe=60&min_price=5&limit=10"

# Deep value screen
curl -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/deep-value-screen?max_pb=3&min_roe=0.10"
```

## Notes on this API

- **Read endpoints return the same formatted text the CLI/GUI would print**, wrapped in a JSON `output` field — not fully structured data for every field. Good enough for scripting/automation that just needs the numbers on screen; if you want fully structured JSON (e.g. `{"price": 191.23, "pe_ratio": 31.2}` instead of formatted text), that's a further enhancement, not what's built here yet.
- **No rate limiting** on this API currently, unlike the photo site/shopping list — since it's SSH/token-gated rather than a public link, this is lower risk, but worth adding if you ever expose it more broadly.
- **The token lives in `api.service`**, not Secret Manager — this project uses the VM's systemd files directly rather than Secret Manager (that pattern was used for the Cloud Run projects instead). Rotate it by editing the service file and running `sudo systemctl daemon-reload && sudo systemctl restart api`.

