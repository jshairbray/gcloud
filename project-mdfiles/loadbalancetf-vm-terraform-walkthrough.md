# Load Balancer Demo — Terraform Build, Deploy & Test Walkthrough

**Project:** `my-generic-project-504019`
**Instance name:** `loadbalancetf-vm`
**Zone:** `us-central1-a`
**External IP (this run):** `104.155.153.172`
**Original manual VM (`stockapp-vm`):** left untouched throughout — never referenced by Terraform state

---

## 1. What this rebuilds

The original setup was built by hand on a GCE VM called `stockapp-vm`:
- **nginx** — round-robin load balancing at `/lb-demo/`, proxying to two backend HTTP servers
- **HAProxy** — round-robin load balancing on `:8080` with active health checks, plus a live stats dashboard on `:8404`
- **backend1.py** — HTTP server on `:9001`, plus a raw TCP socket server on `:9101` that accepts newline-delimited JSON "order" messages (mimicking how order data is sent over sockets in production), validates them, and sends back a JSON ack
- **backend2.py** — a simple second HTTP server on `:9002`, purely for load-balancing demonstration

This walkthrough recreates the entire thing as Terraform-managed infrastructure, under a new instance name (`loadbalancetf-vm`) so the original VM is never touched.

---

## 2. File layout

```
C:\Temp\gcloud\source-code\load-balance\
├── main.tf                  # firewall rules, static IP, the VM resource
├── variables.tf             # all configurable inputs (project, zone, names, etc.)
├── versions.tf              # Terraform + google provider version constraints
├── outputs.tf                # URLs printed after apply
├── terraform.tfvars          # actual values used for this deployment (see below)
├── README.md                 # original usage README
└── scripts\
    ├── backend1.py            # HTTP :9001 + order socket :9101
    ├── backend2.py            # HTTP :9002
    ├── nginx-lb.conf          # nginx round-robin config, mounted at /lb-demo/
    ├── haproxy.cfg            # HAProxy round-robin + health checks + stats
    └── startup.sh.tpl         # boot-time script: installs packages, writes configs,
                                # starts everything as systemd services
```

`terraform.tfvars` (values used for this deployment):
```hcl
project_id    = "my-generic-project-504019"
instance_name = "loadbalancetf-vm"
```

Because this file exists, no `-var=...` flags are needed on any `plan` / `apply` / `destroy` command — Terraform reads it automatically.

---

## 3. Resources Terraform manages

| Resource | GCP name | Purpose |
|---|---|---|
| `google_compute_instance.stockapp_vm` | `loadbalancetf-vm` | The demo VM (Debian 12, `e2-small`) |
| `google_compute_address.stockapp_ip` | `loadbalancetf-vm-ip` | Reserved static external IP |
| `google_compute_firewall.allow_ssh` | `loadbalancetf-vm-allow-ssh` | Opens port 22 |
| `google_compute_firewall.allow_lb_http` | `loadbalancetf-vm-allow-lb-http` | Opens ports 80, 8080, 8404 |

Backend ports (9001, 9002, 9101) are **not** opened externally — nginx/HAProxy proxy to them over `127.0.0.1`, matching the original design. The order socket is only reachable from inside the VM (over SSH).

---

## 4. Deploy steps actually run

```powershell
cd C:\Temp\gcloud\source-code\load-balance
terraform init
terraform plan
terraform apply
```

Typed `yes` at the confirmation prompt. Result:

```
Plan: 4 to add, 0 to change, 0 to destroy.
...
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:
external_ip          = "104.155.153.172"
haproxy_frontend_url = "http://104.155.153.172:8080/"
haproxy_stats_url    = "http://104.155.153.172:8404/stats"
nginx_lb_demo_url    = "http://104.155.153.172/lb-demo/"
order_socket_ssh_tip = "ssh into the instance, then: nc 127.0.0.1 9101"
```

Apply took under a minute for the GCP resources; the startup script (package installs + service starts) finished shortly after in the background.

---

## 5. SSH access — what worked and what didn't

### Attempt 1: `gcloud compute ssh` (worked immediately)
```bash
gcloud compute ssh loadbalancetf-vm --zone=us-central1-a --command="echo it works"
```
`gcloud` auto-generated its own keypair at `~/.ssh/google_compute_engine` / `.pub` on first use, uploaded the public key to **project-level** metadata automatically, and the connection succeeded. Username was derived automatically as `shairj`.

### Attempt 2: manually generated key + plain `ssh` (did not work)
```bash
ssh-keygen -t rsa -f ~/.ssh/gcp-loadbalance -C "jshair"
gcloud compute instances add-metadata loadbalancetf-vm \
  --zone=us-central1-a \
  --metadata-from-file ssh-keys=/c/Users/shairj/.ssh/gcp-loadbalance.pub
ssh -i ~/.ssh/gcp-loadbalance jshair@104.155.153.172
```
Result: `Permission denied (publickey)`. Root cause not fully isolated — most likely the metadata format (`username:ssh-rsa ...`) wasn't exactly right, or the `add-metadata` write happened in a way that didn't take effect as expected. Not pursued further once the working path below was found.

### Working path: plain `ssh` using gcloud's own key
```bash
cat ~/.ssh/google_compute_engine.pub   # confirmed username at the end: shairj
ssh -i ~/.ssh/google_compute_engine shairj@104.155.153.172
```
This succeeded and satisfied the original goal — connecting with plain `ssh` from Git Bash, no PuTTY, no `gcloud compute ssh` wrapper for the actual working session.

**Takeaway:** any username works for a first-time GCE SSH connection — GCP auto-provisions a Linux user matching whatever username is embedded in the key you present.

---

## 6. Testing each component

### 6a. Order socket (`:9101`)

`nc` was not preinstalled on the Debian 12 image, so it needed installing first:
```bash
sudo apt-get update && sudo apt-get install -y netcat-openbsd
```

Then, from inside the SSH session:
```bash
nc 127.0.0.1 9101
```
Waited for the prompt to idle (connected, no output — that's expected), then typed:
```
{"order_id": 1, "symbol": "AAPL", "side": "buy", "qty": 10, "price": 200.5}
```
Result:
```
{"status": "accepted", "order_id": 1, "symbol": "AAPL", "side": "buy", "qty": 10, "price": 200.5, "filled_price": 200.5}
```
Confirmed working — validates required fields, checks `side`/`qty`/`price`, and acks back over the same persistent connection.

*(Note: an earlier attempt returned `{"status": "rejected", "reason": "invalid JSON"}` — caused by stale/buffered input left over from typing the JSON at a plain bash prompt before `nc` was installed. Reconnecting fresh resolved it.)*

### 6b. nginx (`/lb-demo/`) and HAProxy (`:8080`)

From the laptop (not inside SSH):
```bash
curl http://104.155.153.172/lb-demo/
curl http://104.155.153.172:8080/
```
Both alternate between:
```
Hello from BACKEND 1
Hello from BACKEND 2
```
confirming round-robin distribution across `backend1.py` (`:9001`) and `backend2.py` (`:9002`).

### 6c. Generating enough traffic to see HAProxy's stats counters move

A single request barely moves the counters, so a loop was used. **Important distinction: bash loop syntax only works in bash-family shells (Git Bash, or the SSH session itself), not PowerShell.**

From Git Bash (laptop) or inside the SSH session (using `localhost` instead of the external IP, since a GCP VM generally can't reach its own external IP from inside itself):
```bash
for i in {1..20}; do curl -s http://localhost:8080/; done
```

From PowerShell instead, the equivalent is:
```powershell
for ($i=1; $i -le 20; $i++) { curl.exe -s http://104.155.153.172:8080/ }
```
(`curl.exe` specifically — PowerShell's bare `curl` alias points to `Invoke-WebRequest`, which behaves differently.)

### 6d. Viewing the HAProxy stats dashboard

Opened directly in a regular browser (no SSH needed — port 8404 is open via the firewall rule):
```
http://104.155.153.172:8404/stats
```
After running the 20-request loop and refreshing the page, the `Sessions` and `Bytes` columns under the `lb_demo_back` row for `backend1` and `backend2` incremented by roughly 10 each, confirming even round-robin distribution and both backends reporting healthy (`UP`).

---

## 7. Cost notes

Running continuously, this setup costs roughly:

| Item | Approx. cost |
|---|---|
| `e2-small` VM, 24/7 | ~$13–14/month |
| 20GB boot disk | ~$0.80/month |
| Static IP (while attached to running VM) | Free |
| Firewall rules | Free |

**Total: roughly $14–15/month if left running continuously.**

To bring cost back to $0 between sessions:
```powershell
terraform destroy
```
(Reads the same `terraform.tfvars` automatically — tears down the VM, static IP, and both firewall rules. The original `stockapp-vm` is never part of Terraform's state and is unaffected by this command.)

---

## 8. Quick reference — commands used end-to-end

```powershell
# Deploy
cd C:\Temp\gcloud\source-code\load-balance
terraform init
terraform plan
terraform apply

# SSH in (working method)
ssh -i ~/.ssh/google_compute_engine shairj@104.155.153.172

# On the VM: install netcat, test the order socket
sudo apt-get update && sudo apt-get install -y netcat-openbsd
nc 127.0.0.1 9101
# then paste: {"order_id": 1, "symbol": "AAPL", "side": "buy", "qty": 10, "price": 200.5}

# On the VM: generate load for HAProxy
for i in {1..20}; do curl -s http://localhost:8080/; done

# From laptop browser: view live stats
http://104.155.153.172:8404/stats

# From laptop: spot-check nginx and HAProxy directly
curl http://104.155.153.172/lb-demo/
curl http://104.155.153.172:8080/

# Tear down when done (cost -> $0)
terraform destroy
```
