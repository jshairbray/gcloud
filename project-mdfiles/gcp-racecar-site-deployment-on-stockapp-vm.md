# Deploying the Race Car Site on stockapp-vm (GCP)

**Project:** My-Generic-Project (`my-generic-project-504019`, project number `767471978058`)
**Target VM:** `stockapp-vm` (zone `us-central1-a`, internal IP `10.10.0.2`, external IP `136.65.201.133`)
**VPC:** `stockapp-vpc` (custom, non-default), peered with a second custom VPC containing `peer-vm` (`10.20.0.2`, internal-only)
**Web server on VM:** Nginx (already running, reverse-proxying an existing Streamlit-based stock app)
**Site source:** originally intended from `C:\Temp\gcloud\source-code\racecar-site` on Windows; ultimately built by Claude as a single-file static site and uploaded directly to Cloud Storage
**URL path chosen:** `http://136.65.201.133/racecars/`

---

## 1. Background / Decisions Made

- Originally planned a dedicated VM for the race car site. Decided against it once we confirmed:
  - The Compute Engine Always Free tier only covers **one** e2-micro instance per billing account per month (in eligible US regions, including `us-central1`). Two e2-micro VMs (`peer-vm`, `stockapp-vm`) were already running, so a third would incur charges.
  - `stockapp-vm` was already running Nginx and had an external IP, making it a suitable host for a second, low-traffic static site.
- Decided to host the race car site as a **path-based route** (`/racecars/`) on the existing Nginx instance on `stockapp-vm`, rather than a new port or new VM.
- Network topology confirmed: `peer-vm` and `stockapp-vm` live in **two separate custom VPCs connected via VPC Peering**, not one shared VPC.

---

## 2. Pre-Checks Performed

### 2.1 Firewall rule check
Confirmed an existing ingress rule on `stockapp-vpc`:

| Name | Network | Direction | Priority | Ports |
|---|---|---|---|---|
| `stockapp-vpc-allow-80` | `stockapp-vpc` | INGRESS | 1000 | `tcp:80` |

No new firewall rule was required — port 80 was already open to `0.0.0.0/0`.

### 2.2 Checked what was already listening on the VM
```bash
sudo ss -tulpn | grep LISTEN
```
Result: Nginx was bound to `0.0.0.0:80`.

### 2.3 Inspected the existing Nginx config
Config file: `/etc/nginx/sites-enabled/stockapp`

Existing `server {}` block contained:
```nginx
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
```
This confirmed the stock app (Streamlit, port 8501) owns `/`, so the race car site needed its own `location` block on a distinct path rather than conflicting with the root.

---

## 3. Site Source

The original plan was to upload site files from `C:\Temp\gcloud\source-code\racecar-site` on the local Windows machine. That folder turned out to be empty, so Claude generated a complete single-file static site (`index.html`, inline CSS/JS, no external image dependencies) featuring an original race car garage design — fictional cars, SVG illustrations, no copyrighted photos or brand names.

The file was uploaded directly to the bucket root rather than inside a subfolder:
```
gs://racecar-site-my-generic-project-504019/index.html
```

---

## 4. Deployment Steps

### 4.1 Create the Cloud Storage bucket (Console)
1. Console → **Cloud Storage → Buckets → Create**.
2. Name: `racecar-site-my-generic-project-504019` (globally unique name required — plain `racecar-site` was already taken).
3. Region: same region as the VM (`us-central1`) for convenience, not strictly required for a bucket.
4. Leave remaining defaults, click **Create**.

### 4.2 Upload the site file (Console)
1. Open the bucket.
2. **Upload → Upload file** → select `index.html`.
3. Confirm it appears at the bucket root: `gs://racecar-site-my-generic-project-504019/index.html`.

### 4.3 Pull the file onto the VM (SSH — Console browser terminal)
```bash
mkdir -p ~/racecars
gcloud storage cp gs://racecar-site-my-generic-project-504019/index.html ~/racecars/index.html
ls ~/racecars
```

### 4.4 Move into a web-servable directory
```bash
sudo mkdir -p /var/www/racecars
sudo cp ~/racecars/index.html /var/www/racecars/index.html
sudo chown -R www-data:www-data /var/www/racecars
```

### 4.5 Add the Nginx location block
```bash
sudo nano /etc/nginx/sites-enabled/stockapp
```
Added inside the existing `server {}` block:
```nginx
location /racecars/ {
    alias /var/www/racecars/;
    index index.html;
}
```

> **Why `alias` and not `root`:** the URL path (`/racecars/`) doesn't match the real folder path under `/var/www/`. `alias` strips the `/racecars/` prefix before looking up files on disk; `root` would incorrectly look for `/var/www/racecars/racecars/index.html`.

### 4.6 Test and reload Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 4.7 Verify
Visit: `http://136.65.201.133/racecars/`

---

## 5. Troubleshooting Notes Encountered Along the Way

| Symptom | Cause | Fix |
|---|---|---|
| `gcloud storage cp` returned `404` on `gs://racecar-site` | Bucket name `racecar-site` was already taken globally; Console had actually created it as `racecar-site-my-generic-project-504019` | Used the full, correct bucket name in all subsequent commands |
| `gcloud storage cp` ran with no visible effect / "no files in local" | Command was initially run in a **local Windows Git Bash (MINGW64)** terminal instead of the **VM's SSH session** | Re-ran the same command inside the Console SSH window for `stockapp-vm` |
| `ERROR: ... matched no objects or files` on `gs://.../racecar-site` | Source folder `C:\Temp\gcloud\source-code\racecar-site` was empty on Windows — nothing had actually been uploaded | Generated a new `index.html` and uploaded it directly to the bucket root instead of a subfolder |

---

## 6. Current End State

- `stockapp-vm` serves two apps behind one Nginx instance on port 80:
  - `/` → Streamlit stock app (port 8501)
  - `/api/` → API backend (port 8503)
  - `/lb-demo/` → load-balancer demo backend
  - `/racecars/` → static race car garage site (new)
- No new VM, no new firewall rule, no new billable compute resource was needed.
- Site source of truth lives in Cloud Storage bucket `racecar-site-my-generic-project-504019`, with a working copy deployed to `/var/www/racecars/` on `stockapp-vm`.

## 7. Possible Next Steps (Not Yet Done)

- Add a custom domain via Cloud DNS pointing at `136.65.201.133`.
- Add HTTPS (Let's Encrypt via `certbot`, or a Google-managed load balancer with a managed SSL cert).
- Move the site into a versioned source location (e.g., a Git repo) instead of manual bucket uploads, so future edits don't require re-running the manual copy steps.
- If the site grows beyond a single file (images, multiple pages), re-establish the local folder structure at `C:\Temp\gcloud\source-code\racecar-site\` and re-upload as a folder rather than a single file.
