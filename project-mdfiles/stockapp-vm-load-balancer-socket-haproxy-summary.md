# Load Balancer + Socket Server + HAProxy — Summary

**Project:** `my-generic-project-504019`
**VM:** `stockapp-vm` (external IP `136.65.201.133`)
**Date:** 2026-08-15 / 2026-08-16
**Goal:** Learn load-balancing mechanics (round-robin, passive vs. active health checks) and raw socket communication for structured "order" data — all at $0 marginal cost, without touching the real app (`streamlit` on `/`, API on `/api/`).

---

## Part 1 — nginx Load Balancer (Round-Robin)

### Setup
Two disposable dummy backend HTTP servers were created to simulate "multiple backends" without spinning up new VMs:

```bash
mkdir -p ~/lb-demo && cd ~/lb-demo
# backend1.py -> HTTPServer on 127.0.0.1:9001, replies "Hello from BACKEND 1"
# backend2.py -> HTTPServer on 127.0.0.1:9002, replies "Hello from BACKEND 2"
nohup python3 backend1.py > backend1.log 2>&1 &
nohup python3 backend2.py > backend2.log 2>&1 &
```

Added to the real nginx config (`/etc/nginx/sites-enabled/stockapp`), alongside the existing `/` and `/api/` blocks — nothing production-facing was modified:

```nginx
upstream lb_demo {
    server 127.0.0.1:9001;
    server 127.0.0.1:9002 max_fails=1 fail_timeout=10s;
}

server {
    listen 80;
    location /lb-demo/ {
        proxy_pass http://lb_demo/;
        proxy_set_header Host $host;
    }
    # ...existing / and /api/ blocks unchanged...
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### What was learned
- **Round-robin confirmed:** `for i in {1..6}; do curl -s http://127.0.0.1/lb-demo/; done` alternated between BACKEND 1 and BACKEND 2.
- **Round-robin is per-worker, not global:** with `worker_processes auto` (multiple nginx workers), each worker keeps its own counter, so short bursts looked slightly uneven (`1,1,2,1,2,1`). Forcing `worker_processes 1` in `nginx.conf` produced strict alternation, confirming the cause.
- **Automatic failover is silent by default:** killing backend2 (`kill $(pgrep -f backend2.py)`) still produced all `200`s on the client side, because nginx's `proxy_next_upstream` (on by default) automatically retried the request against backend1 before responding. The real failure was only visible in `/var/log/nginx/error.log`:
  ```
  connect() failed (111: Connection refused) while connecting to upstream ... "http://127.0.0.1:9002/"
  ```
- **Raw failure exposed with `proxy_next_upstream off`:** temporarily disabling automatic retry surfaced a real `502` to the client for requests routed to the dead backend — proving the failover really was happening, just invisibly by default.
- **Passive health checks (`max_fails` / `fail_timeout`) are reactive:** nginx (open-source) only learns a backend is down after a real request fails against it — it does not proactively poll backends on its own.

---

## Part 2 — Raw Socket Server for Order Data

### Goal
Simulate the kind of persistent TCP socket used to send structured order data (a pattern used in real trading/order-entry systems), separate from the HTTP load-balancer demo.

### Design
- `backend1.py` extended with a second listener: a raw TCP socket on `127.0.0.1:9101`, running in a background thread alongside its existing HTTP server on `9001`.
- Protocol: newline-delimited JSON. Each order is one JSON object per line; server validates and responds with an accept/reject JSON message.

### Server-side order handling (`backend1.py`, socket portion)
```python
REQUIRED_FIELDS = {"order_id", "symbol", "side", "qty", "price"}

def handle_order(order):
    missing = REQUIRED_FIELDS - order.keys()
    if missing:
        return {"status": "rejected", "reason": f"missing fields: {sorted(missing)}"}
    if order["side"] not in ("buy", "sell"):
        return {"status": "rejected", "reason": "side must be 'buy' or 'sell'"}
    if not isinstance(order["qty"], (int, float)) or order["qty"] <= 0:
        return {"status": "rejected", "reason": "qty must be a positive number"}
    if not isinstance(order["price"], (int, float)) or order["price"] <= 0:
        return {"status": "rejected", "reason": "price must be a positive number"}
    return {
        "status": "accepted",
        "order_id": order["order_id"], "symbol": order["symbol"],
        "side": order["side"], "qty": order["qty"], "price": order["price"],
        "filled_price": order["price"],
    }
```

The socket loop reads from `conn.recv()`, buffers partial data, splits on `\n`, parses each complete line as JSON, and sends back a JSON response terminated by `\n`.

### Client (`order_client.py`)
```python
def send_order(order, host="127.0.0.1", port=9101):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host, port))
        s.sendall((json.dumps(order) + "\n").encode())
        return json.loads(s.recv(4096).decode())
```
Sends three sample orders, each over its own fresh connection.

### Result — confirmed working end to end
```
--> sending: {'order_id': 101, 'symbol': 'AAPL', 'side': 'buy', 'qty': 10, 'price': 227.5}
<-- received: {'status': 'accepted', ...}
--> sending: {'order_id': 102, 'symbol': 'TSLA', 'side': 'sell', 'qty': 5, 'price': 305.1}
<-- received: {'status': 'accepted', ...}
--> sending: {'order_id': 103, 'symbol': 'MSFT', 'side': 'hold', 'qty': 3, 'price': 410.0}
<-- received: {'status': 'rejected', 'reason': "side must be 'buy' or 'sell'"}
```

### What was learned
- A socket is a persistent, bidirectional byte stream — unlike HTTP's per-request model, the same connection can be used for multiple send/receive round trips.
- Newline-delimited JSON is a simple, common framing method for structured messages over a raw TCP stream (data must be buffered and split on a delimiter, since `recv()` gives no guarantee of message boundaries).
- Server-side validation (required fields, valid enum values, numeric sanity checks) mirrors the basic shape of real order-entry validation, just simplified.

---

## Part 3 — HAProxy (Active Health Checks)

### Why HAProxy, alongside nginx
nginx's open-source version only does **passive** failure detection (reacts after a real request fails). HAProxy was installed to compare against **active** health checking — proactively polling each backend on a timer, independent of real traffic.

### Install & config
```bash
sudo apt update && sudo apt install -y haproxy
sudo nano /etc/haproxy/haproxy.cfg   # (must be edited with sudo — plain nano without sudo hit "Permission denied" writing to /etc)
```

Added:
```
frontend lb_demo_front
    bind *:8080
    default_backend lb_demo_back

backend lb_demo_back
    balance roundrobin
    option httpchk GET /
    server backend1 127.0.0.1:9001 check
    server backend2 127.0.0.1:9002 check

listen stats
    bind *:8404
    stats enable
    stats uri /
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # validate before restarting
sudo systemctl restart haproxy
```

### Result — active health check confirmed via stats page (`:8404`)
With backend2 killed, the HAProxy stats page showed, **without any client traffic being sent**:

| Backend | Status | Detail |
|---|---|---|
| backend1 (`9001`) | 🟢 UP | `L7OK/200 in 0ms` |
| backend2 (`9002`) | 🔴 DOWN | `L4CON in 0ms — Layer4 connection problem: Connection refused` — `Dwn: 1`, `Dwntme: 20s` |

This proved HAProxy had already detected and quarantined the dead backend on its own schedule (`option httpchk GET /`, default ~2s interval) — it didn't need a real request to fail first, unlike nginx's `max_fails` approach.

### Viewing the stats page from a local browser (SSH tunnel)
Since HAProxy's stats bind (`127.0.0.1:8404`) only exists on the VM, an SSH local port-forward was used to view it from a Windows browser:
```bash
# Run FROM THE LOCAL MACHINE (not from inside an existing VM session):
ssh -i ~/.ssh/google_compute_engine -L 8404:127.0.0.1:8404 shairj@136.65.201.133
```
Then browsing to `http://127.0.0.1:8404` locally (while that SSH session stays open) rendered the live stats dashboard.

**Common mistake caught during this step:** running the `-L` tunnel command *from inside* an already-open SSH session on `stockapp-vm` (i.e., SSH-ing to itself) instead of from the local machine — the fix was exiting back to the local prompt first, then initiating the tunnel from there.

### What was learned
- **Active vs. passive health checking is the core practical difference** between nginx (free) and HAProxy for failure detection speed.
- HAProxy's built-in stats page (`stats enable` / `stats uri /`) gives direct visibility into per-backend status, check history, and traffic — genuinely useful for understanding load-balancer behavior, not just a toy.
- Config files under `/etc/` require `sudo` to edit **and to save** — opening `nano` without `sudo` will let you edit in-memory but fail on write; either restart with `sudo nano ...` or write out via `Ctrl+O` then `|sudo tee /path/to/file`.
- SSH local port-forwarding (`-L local_port:remote_host:remote_port`) is the standard way to reach a service bound to `127.0.0.1` on a remote box from a local browser, without exposing it publicly — must be run from the local machine, and the session must stay open for the tunnel to work.

---

## Tooling Notes (Windows/PuTTY-specific issues encountered)

- `gcloud compute ssh` / `gcloud compute scp` use PuTTY's `plink`/`pscp` under the hood on Windows by default — this caused `~` not expanding correctly in remote paths (`pscp: unable to open ~/lb-demo/backend1.py`) and general paste/heredoc corruption in PuTTY-based sessions.
- **Fix used:** switched to Git Bash's bundled OpenSSH (`ssh`/`scp` directly, not through `gcloud`), which behaves like standard Linux SSH — reliable heredocs, normal copy/paste, no `~` expansion issues.
- PuTTY selection model: click-drag or click + Shift-click to select (auto-copies), right-click to paste — no `Ctrl+C`/`Ctrl+V`. Awkward on trackpads; Shift-click is the trackpad-friendly option.
- Git Bash's `ssh.exe` must be run **from within Git Bash itself**, not from PowerShell — running it from PowerShell produced `failed to initialize w32posix wrapper` (an MSYS2 environment-initialization error).
- Windows' built-in OpenSSH client requires an optional feature that may be blocked by admin/group policy in managed corporate environments — Git Bash's bundled OpenSSH is a working alternative that doesn't require that feature or admin rights.

---

## Cleanup (if none of this should persist)

```bash
# Stop dummy backends
kill $(pgrep -f backend1.py) $(pgrep -f backend2.py) 2>/dev/null

# Remove the /lb-demo/ block + upstream from nginx
sudo nano /etc/nginx/sites-enabled/stockapp   # delete upstream lb_demo{} and location /lb-demo/{}
sudo nginx -t && sudo systemctl reload nginx

# Stop and remove HAProxy if no longer needed
sudo systemctl stop haproxy
sudo apt remove -y haproxy

# Remove demo files
rm -rf ~/lb-demo
```
