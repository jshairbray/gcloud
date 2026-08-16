# Complete Guide: Creating a Load Balancer on GCP (Free/Low-Cost Approach)

**Environment:** GCP project `my-generic-project-504019`, VM `stockapp-vm` (`us-central1-a`, external IP `136.65.201.133`)
**Goal:** Understand and build a working load balancer from first principles, at $0 marginal cost, using nginx and HAProxy running on an existing VM — no managed GCP Load Balancer product involved.

---

## Part 0 — Concepts: Why This Approach

### What a load balancer actually does
A load balancer sits in front of two or more backend servers and distributes incoming requests across them, so no single backend is overwhelmed and traffic keeps flowing even if one backend fails. The three core mechanics it needs to handle:
1. **Distribution algorithm** — how it picks which backend gets the next request (round-robin, least-connections, weighted, etc.)
2. **Health checking** — how it knows a backend is dead so it can stop sending traffic there
3. **Failover** — what happens to a request when its chosen backend fails

### Why not GCP's managed Load Balancer product
GCP's HTTP(S)/TCP Load Balancing charges a **forwarding rule fee** (roughly $0.025/hour ≈ $18/month) plus data processing costs, regardless of traffic volume — there's no free tier for it. For a learning exercise or low-traffic project where cost matters, this is real recurring money for a feature you may barely use.

### The alternative used here: software load balancer on an existing VM
Instead of the managed product, this guide installs and configures **nginx** (and later **HAProxy**, for comparison) directly on a Compute Engine VM you already have. This costs nothing beyond the VM itself, which was already running. Two backend "servers" are simulated as separate local processes on different ports, so no additional VMs or billing were needed to complete the exercise.

**Trade-off, stated plainly:** this is a single point of failure (if the VM running nginx/HAProxy goes down, so does your load balancing), has no auto-scaling, and needs manual maintenance — acceptable for learning and low-stakes use, not a substitute for a managed LB in a production system with real uptime requirements.

---

## Part 1 — Environment Discovery (know what you're working with first)

Before adding anything, it's worth confirming what's already running on the VM so you don't accidentally collide with it.

### Check what's listening on network ports
```bash
sudo ss -tlnp
```
This lists every process bound to a TCP port. Example interpretation:
```
LISTEN  0  511  0.0.0.0:80    users:(("nginx",pid=423,fd=5))
```
means nginx is listening on port 80, reachable from anywhere (`0.0.0.0`).

### Check the existing nginx config
```bash
sudo nginx -T | grep "configuration file"
```
This prints every config file nginx actually loaded — useful because the real file in use might not be the default-named one you'd expect (in this case, it was `/etc/nginx/sites-enabled/stockapp`, not `sites-enabled/default`).

```bash
cat /etc/nginx/sites-enabled/stockapp
```
Existing config found:
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
    location /api/ {
        proxy_pass http://127.0.0.1:8503/api/;
        proxy_set_header Host $host;
    }
}
```
This told us nginx was already reverse-proxying two real services (a dashboard on 8501, an API on 8503) — the load-balancing exercise needed to be added **alongside** this without breaking it.

---

## Part 2 — Building the Load-Balanced Backends

Since there was only one instance of each real service, two disposable dummy HTTP servers were created to stand in as "multiple backends" to balance across — this is the cheapest possible way to practice the mechanics without cloning the real app or paying for new VMs.

### Create the working directory
```bash
mkdir -p ~/lb-demo && cd ~/lb-demo
```

### Backend 1
```bash
cat > backend1.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from BACKEND 1\n")

HTTPServer(('127.0.0.1', 9001), Handler).serve_forever()
EOF
```

### Backend 2
```bash
cat > backend2.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from BACKEND 2\n")

HTTPServer(('127.0.0.1', 9002), Handler).serve_forever()
EOF
```

**Why `127.0.0.1` and not `0.0.0.0`:** binding to loopback only means these dummy servers are never reachable from outside the VM — only nginx (running on the same machine) can reach them. This is good practice generally: backend servers behind a reverse proxy usually shouldn't be directly internet-facing.

### Start both in the background, detached from the terminal
```bash
nohup python3 backend1.py > backend1.log 2>&1 &
nohup python3 backend2.py > backend2.log 2>&1 &
```
- `nohup` — keeps the process running even if your SSH session closes. It will print `nohup: ignoring input` on start; that's normal, not an error.
- `> backend1.log 2>&1` — redirects both standard output and error output into a log file, so you can inspect what happened later.
- `&` — runs the command in the background so your terminal isn't blocked.

### Verify both are actually listening
```bash
ss -tlnp | grep -E '9001|9002'
```
Expect two lines, both showing `127.0.0.1:900X` with a `python3` process attached.

### Sanity check each one directly, before involving nginx at all
```bash
curl http://127.0.0.1:9001
curl http://127.0.0.1:9002
```
Expected: `Hello from BACKEND 1` and `Hello from BACKEND 2` respectively. If either fails here, the problem is the Python server itself — fix that before touching nginx.

---

## Part 3 — Configuring nginx as the Load Balancer

### Step 1: Locate the real config file
```bash
sudo nginx -T | grep "configuration file"
```
Confirm it points to the file you expect to edit (in this environment: `/etc/nginx/sites-enabled/stockapp`).

### Step 2: Open it with sudo
Files under `/etc/` are root-owned. **Always open with `sudo` from the start** — opening without it lets you edit in the buffer but fails silently (or with a permission error) when you try to save:
```bash
sudo nano /etc/nginx/sites-enabled/stockapp
```

### Step 3: Add an `upstream` block
The `upstream` directive defines a named pool of backend servers nginx can route to. It must be placed **outside** the `server {}` block (at the same level), so add it near the top of the file, before `server {`:
```nginx
upstream lb_demo {
    server 127.0.0.1:9001;
    server 127.0.0.1:9002;
}
```
By default, nginx distributes requests to entries in an `upstream` block using **round-robin** — no extra configuration needed for that baseline behavior.

### Step 4: Add a new `location` block that uses the upstream
Inside the existing `server { ... }` block — **alongside**, not replacing, the existing `location /` and `location /api/` blocks:
```nginx
location /lb-demo/ {
    proxy_pass http://lb_demo/;
    proxy_set_header Host $host;
}
```
`proxy_pass http://lb_demo/;` tells nginx: for any request matching this location, forward it to the pool named `lb_demo`, letting nginx's built-in round-robin decide which specific backend server actually handles it.

### Full resulting file shape
```nginx
upstream lb_demo {
    server 127.0.0.1:9001;
    server 127.0.0.1:9002;
}

server {
    listen 80;

    location /lb-demo/ {
        proxy_pass http://lb_demo/;
        proxy_set_header Host $host;
    }

    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8503/api/;
        proxy_set_header Host $host;
    }
}
```

Save and exit (`nano`: `Ctrl+O`, `Enter`, `Ctrl+X`).

**A real typo caught during this step, worth watching for:** writing `127.0.1.1:9002` instead of `127.0.0.1:9002` in the upstream block — a single-character typo in an IP address that silently points nginx at a nonexistent backend. Always double-check IPs character by character when editing configs like this.

### Step 5: Validate syntax before reloading — every time, no exceptions
```bash
sudo nginx -t
```
Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
If this errors, **do not reload** — fix the reported line first. This step is what catches typos (like the `127.0.1.1` example above) before they can affect the live app.

### Step 6: Reload nginx — not restart
```bash
sudo systemctl reload nginx
```
`reload` re-reads the configuration without dropping existing connections. `restart` would briefly take nginx down entirely — unnecessary and riskier when a real app (the streamlit dashboard and API) is being served by the same nginx instance.

### Step 7: Confirm the real app is still working, untouched
```bash
curl -I http://127.0.0.1/
curl -I http://127.0.0.1/api/
```
Both should return normal HTTP responses (not errors) — this confirms editing the file for the LB demo didn't break the production routes.

### Step 8: Test round-robin on the new path
```bash
for i in {1..6}; do curl -s http://127.0.0.1/lb-demo/; done
```
Expected: alternating output —
```
Hello from BACKEND 1
Hello from BACKEND 2
Hello from BACKEND 1
Hello from BACKEND 2
Hello from BACKEND 1
Hello from BACKEND 2
```
This confirms nginx is genuinely load-balancing between the two backends via round-robin.

**A subtlety observed here:** the output isn't always perfectly alternating (e.g. `1,1,2,1,2,1`) if nginx is running multiple worker processes (`worker_processes auto`, the default — typically one per CPU core). Each worker keeps its **own independent round-robin counter**, so short bursts of requests can look unevenly distributed even though the algorithm is working correctly; it evens out statistically at real traffic volumes. This can be directly observed by temporarily setting `worker_processes 1;` in `/etc/nginx/nginx.conf`, reloading, and re-running the test — output becomes perfectly ordered. Revert to `auto` afterward, since running with only 1 worker is worse for real traffic handling.

---

## Part 4 — Failure Handling and Failover Behavior

### Step 1: Kill one backend to simulate a real failure
```bash
kill $(pgrep -f backend2.py)
ss -tlnp | grep 9002   # should return nothing — confirms it's actually down
```

### Step 2: Test the load-balanced path again
```bash
for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/lb-demo/; done
```
**Surprising result observed:** every request still returned `200`, despite backend2 being confirmed dead.

**Why:** nginx has a directive called `proxy_next_upstream`, which is **on by default**. When a request to one backend fails, nginx automatically and silently retries the *same request* against the next server in the `upstream` list, before ever responding to the client. This is desirable behavior in production — users shouldn't see backend failures — but it means you can't detect a dead backend from client responses alone.

### Step 3: Prove the failure is really happening, via the error log
```bash
sudo tail -20 /var/log/nginx/error.log
```
Found:
```
2026/08/15 19:51:22 [error] 25804#25804: *477 connect() failed (111: Connection refused) while connecting to upstream, client: 127.0.0.1, server: , request: "GET /lb-demo/ HTTP/1.1", upstream: "http://127.0.0.1:9002/", host: "127.0.0.1"
```
The timestamp matched exactly when the test loop was run — confirming nginx tried backend2, got refused, and silently rerouted to backend1.

### Step 4: See the raw, unmasked failure (diagnostic only, not for production)
Temporarily disable automatic failover on this one location:
```nginx
location /lb-demo/ {
    proxy_pass http://lb_demo/;
    proxy_set_header Host $host;
    proxy_next_upstream off;
}
```
```bash
sudo nginx -t && sudo systemctl reload nginx
for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/lb-demo/; done
```
Result observed:
```
200
200
502
200
200
200
200
200
```
The single `502` is the request that landed on the round-robin slot pointing at the dead backend, with nginx no longer allowed to reroute it — the real, unmasked failure.

**Afterward:** remove `proxy_next_upstream off;` (or leave it out) to restore normal production behavior — automatic failover is what you want in real use; this was purely to observe the failure directly.

### Step 5: Add passive failure detection so nginx stops retrying a known-dead backend on every request
```nginx
upstream lb_demo {
    server 127.0.0.1:9001;
    server 127.0.0.1:9002 max_fails=1 fail_timeout=10s;
}
```
This tells nginx: after 1 failed attempt against this server, mark it "down" and skip it entirely for the next 10 seconds, rather than trying it again on every single incoming request.
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Step 6: Bring the backend back and confirm automatic recovery
```bash
cd ~/lb-demo
nohup python3 backend2.py > backend2.log 2>&1 &
sleep 12
for i in {1..6}; do curl -s http://127.0.0.1/lb-demo/; done
```
After the `fail_timeout` window passes, nginx resumes trying backend2 on its own — no reload required — and alternation returns.

**Key limitation of this approach, worth naming explicitly:** nginx's free/open-source version only performs **passive** health checking — it only learns a backend is dead by having a real request fail against it first. It never proactively checks backend health on its own schedule. That gap is what Part 5 (HAProxy) addresses.

---

## Part 5 — HAProxy: Active Health Checking

### Why add HAProxy alongside nginx
To directly compare **passive** (nginx, reactive) vs. **active** (HAProxy, proactive) health checking — HAProxy periodically polls each backend on its own timer, independent of real client traffic, and can mark a backend down before any real user ever hits it.

### Step 1: Install
```bash
sudo apt update && sudo apt install -y haproxy
```

### Step 2: Edit the config — with sudo, from the start
```bash
sudo nano /etc/haproxy/haproxy.cfg
```
(Opening without `sudo` will let you type edits but fail with "Permission denied" trying to save, since the file is root-owned.)

### Step 3: Add a frontend, backend, and stats listener
Append to the bottom of the file:
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

**What each piece does:**
- `frontend lb_demo_front` / `bind *:8080` — HAProxy listens on port 8080 (chosen to avoid colliding with nginx on port 80).
- `backend lb_demo_back` / `balance roundrobin` — same distribution algorithm as nginx's default.
- `option httpchk GET /` — this is the active health check: HAProxy will independently send a real `GET /` request to each backend on a recurring interval (default ~2 seconds), regardless of real client traffic.
- `server backend1 127.0.0.1:9001 check` — the `check` keyword is what enables active health checking for that specific server entry.
- `listen stats` / `stats enable` / `stats uri /` — exposes a built-in live dashboard on port 8404 showing each backend's real-time status.

### Step 4: Validate before restarting
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
Fix any reported errors before proceeding.

### Step 5: Restart HAProxy
```bash
sudo systemctl restart haproxy
```
(HAProxy doesn't support the same graceful `reload` semantics as nginx by default in all configurations — `restart` is the standard approach here.)

### Step 6: Test round-robin through HAProxy
```bash
for i in {1..6}; do curl -s http://127.0.0.1:8080/; done
```
Should alternate between the two backends, same as the nginx test earlier.

### Step 7: Kill a backend and observe active detection — no client traffic needed
```bash
kill $(pgrep -f backend2.py)
```
View the stats dashboard:
```bash
curl -s http://127.0.0.1:8404 | grep -A5 "backend2"
```
Observed result: backend2 flipped to **DOWN** status (`L4CON — Layer4 connection problem: Connection refused`) with a `Dwn: 1`, `Dwntme: 20s` counter — **before any client request was sent through the load-balanced port at all.** This is the core proof that HAProxy detects failures proactively, unlike nginx's passive/reactive approach.

### Step 8: Bring the backend back and confirm self-healing
```bash
cd ~/lb-demo
nohup python3 backend2.py > backend2.log 2>&1 &
```
Wait ~5-10 seconds (past the check interval), then re-check the stats page — backend2 should flip back to UP automatically, with no HAProxy restart needed.

### Viewing the HAProxy stats dashboard in a local browser
Since `127.0.0.1:8404` only exists on the VM, an SSH local port-forward makes it viewable from a local Windows browser:

**Run this from your local machine (not from inside an already-open VM session):**
```bash
ssh -i ~/.ssh/google_compute_engine -L 8404:127.0.0.1:8404 shairj@136.65.201.133
```
Leave that terminal window open and logged in — it's holding the tunnel open. Then, in a browser on your local machine, go to:
```
http://127.0.0.1:8404
```
This renders the live HAProxy stats table, showing per-backend status, check history (`L7OK/200` vs `L4CON`), request counts, and response times in real time.

**Common mistake to avoid:** running the `-L` tunnel command from *inside* an SSH session already on the VM (i.e., trying to SSH from the VM back to itself) — the `-L` flag must be part of the initial connection command run from your local machine.

---

## Part 6 — Comparison Summary: nginx vs. HAProxy for This Exercise

| Aspect | nginx (open-source) | HAProxy |
|---|---|---|
| Distribution algorithm | Round-robin by default | Round-robin (and others) via `balance` directive |
| Health checking | Passive only — reacts after a real request fails | Active — proactively polls backends on a timer (`option httpchk`) |
| Detection speed | Only as fast as real traffic reveals a failure | Detects independently of traffic, typically within seconds |
| Built-in visibility | Only via `error.log` | Built-in live stats dashboard (`stats enable`) |
| Already installed here | Yes (serving the real app) | Added specifically for this comparison |
| Config reload | `reload` — no dropped connections | `restart` used here |

---

## Cleanup (if none of this should persist on `stockapp-vm`)

```bash
# Stop the dummy backend processes
kill $(pgrep -f backend1.py) $(pgrep -f backend2.py) 2>/dev/null

# Remove the load-balancing additions from nginx
sudo nano /etc/nginx/sites-enabled/stockapp
# delete the `upstream lb_demo { ... }` block and the `location /lb-demo/ { ... }` block
sudo nginx -t && sudo systemctl reload nginx

# Stop and optionally remove HAProxy
sudo systemctl stop haproxy
sudo apt remove -y haproxy

# Remove the demo directory entirely
rm -rf ~/lb-demo
```
