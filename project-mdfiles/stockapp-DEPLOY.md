# Stock Tracker — Compute Engine VM (project: my-generic-project-504019)

Files needed: `app.py`, `update_quotes.py`, `generate_docs.py`, `requirements.txt`, `stockapp.service`

## 1. One-time setup

```bash
gcloud config set project my-generic-project-504019
gcloud services enable compute.googleapis.com
```

## 2. Firewall (port 80 only — see nginx section below for why not 8501)

```bash
gcloud compute firewall-rules create allow-stockapp-80 \
  --allow=tcp:80 \
  --target-tags=stockapp \
  --direction=INGRESS \
  --source-ranges=0.0.0.0/0
```

## 3. Create the VM

```bash
gcloud compute instances create stockapp-vm \
  --project=my-generic-project-504019 \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=30GB \
  --boot-disk-type=pd-standard \
  --tags=stockapp
```

## 4. Copy files to the VM

```bash
gcloud compute scp app.py update_quotes.py generate_docs.py requirements.txt stockapp.service \
  stockapp-vm:~/stockapp-src --zone=us-central1-a
```

## 5. SSH in and install

```bash
gcloud compute ssh stockapp-vm --zone=us-central1-a
```

```bash
sudo apt-get update && sudo apt-get install -y python3-venv python3-pip nginx

sudo useradd -r -m -d /opt/stockapp -s /usr/sbin/nologin stockapp || true
sudo mkdir -p /opt/stockapp
sudo cp ~/stockapp-src/app.py ~/stockapp-src/update_quotes.py ~/stockapp-src/generate_docs.py ~/stockapp-src/requirements.txt /opt/stockapp/
sudo python3 -m venv /opt/stockapp/venv
sudo /opt/stockapp/venv/bin/pip install -r /opt/stockapp/requirements.txt
sudo mkdir -p /opt/stockapp/data
sudo chown -R stockapp:stockapp /opt/stockapp
```

`venv` = an isolated, private copy of Python just for this app, so its packages never collide with anything else on the VM. `chown -R` hands ownership of everything to the low-privilege `stockapp` user the app actually runs as (not root) — a security boundary, since a bug in the app then can't do root-level damage.

## 6. systemd service (keeps it running, Streamlit bound to localhost only)

Note: Streamlit listens on `127.0.0.1:8501` (not `0.0.0.0`) — it's deliberately unreachable from outside the VM. Only nginx (next step) is exposed publicly, and relays to Streamlit locally.

```bash
sudo cp ~/stockapp-src/stockapp.service /etc/systemd/system/stockapp.service
sudo systemctl daemon-reload
sudo systemctl enable --now stockapp
sudo systemctl status stockapp   # confirm "active (running)"
```

## 7. nginx reverse proxy (fixes WebSocket issues some networks have with port 8501)

```bash
sudo tee /etc/nginx/sites-available/stockapp > /dev/null << 'EOF'
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
}
EOF

sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/stockapp /etc/nginx/sites-enabled/stockapp
sudo nginx -t
sudo systemctl restart nginx
```

`proxy_pass http://127.0.0.1:8501` is the key line — it tells nginx "forward everything to Streamlit, running locally on this same machine, port 8501." The `Upgrade`/`Connection` headers specifically preserve WebSocket connections through the proxy, which plain HTTP forwarding would otherwise break.

## 8. Cloudflare Tunnel (real HTTPS, fixes networks that block plain-HTTP WebSockets)

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

sudo tee /etc/systemd/system/cloudflared.service > /dev/null << 'EOF'
[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/cloudflared tunnel --url http://localhost:80
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared
```

Get your URL (changes each time the tunnel restarts, since this is the free/anonymous "quick tunnel" — no fixed domain registered):

```bash
sudo journalctl -u cloudflared --no-pager | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | tail -1
```

## Updating the app later

```bash
gcloud compute scp app.py stockapp-vm:~/app.py --zone=us-central1-a
gcloud compute ssh stockapp-vm --zone=us-central1-a
```
```bash
sudo cp ~/app.py /opt/stockapp/app.py
sudo chown stockapp:stockapp /opt/stockapp/app.py
grep -c "<a known unique string from the new file>" /opt/stockapp/app.py   # confirm the right version landed before restarting
sudo systemctl restart stockapp
```

## Common pitfalls we actually hit

- **Duplicate uploads** (`app (1).py`, `app_(1).py`) — Cloud Shell renames instead of overwriting same-named files. Always `grep` a known string to confirm which copy is current before deploying.
- **"Network Issue: Cannot load Streamlit frontend"** — almost always a stale browser tab after a restart. Hard refresh (`Ctrl+Shift+R`) or open a fresh tab.
- **Works on laptop, not on phone** — check Android's **Private DNS** setting (Settings → Network & Internet → Private DNS) — filtering DNS providers commonly block `trycloudflare.com` subdomains as a phishing precaution.

## Cost

This is the one project that's genuinely free forever: `e2-micro` + 30GB standard disk is Google's permanent Always Free allowance (one instance, one billing account, specific regions including `us-central1`). Leave it running continuously — that's the intended use, not something that costs more the longer it's on.
