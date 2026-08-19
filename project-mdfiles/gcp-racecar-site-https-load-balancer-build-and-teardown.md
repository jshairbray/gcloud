# HTTPS Load Balancer for the Race Car Site — Build, Verify, Teardown

**Project:** `my-generic-project-504019` (`767471978058`)
**Target VM:** `stockapp-vm` (zone `us-central1-a`, internal IP `10.10.0.2`, external IP `136.65.201.133`)
**Goal:** put a real GCP HTTPS Load Balancer (with a Google-managed SSL cert) in front of the existing `/racecars/` path on `stockapp-vm`, verify it end-to-end, then tear it down completely to avoid ongoing charges.
**Domain used:** none owned — used **sslip.io** (`<ip-with-dashes>.sslip.io`) as a free, instantly-resolving hostname so a Google-managed cert could still be issued.
**Outcome:** fully built, verified working over HTTPS, fully torn down. Confirmed no residual billing resources afterward.

This session followed on from `gcp-racecar-site-deployment-on-stockapp-vm.md` (original site deployment) and `gcp-racecar-site-learning-extensions.md` (the four-part learning plan: Monitoring, Serverless, CI/CD, Networking/Security). This document covers the **Networking/Security** section in full, as actually executed.

---

## 1. Decisions made before building

- Confirmed the user's existing "load balancer" (`loadbalancetf-vm`, from a separate Terraform-managed demo) is **not** a GCP Cloud Load Balancer resource — it's an nginx + HAProxy round-robin setup running as software on a single VM. It has no forwarding rule, backend service, or managed cert, so it could not be reused for this goal.
- Decided to build a real GCP HTTPS Load Balancer (global external Application Load Balancer components: static IP, instance group, health check, backend service, URL map, managed SSL cert, target HTTPS proxy, forwarding rule) pointed at the existing `stockapp-vm`.
- Decided against registering a domain. Used **sslip.io** instead — a free third-party service where `<ip-with-dashes>.sslip.io` automatically resolves to that IP with no registration or DNS record needed. Google-managed certs can issue against it because it's a real, correctly-resolving hostname.
- Decided upfront to tear everything down immediately after verifying it worked, since the forwarding rule is the piece that bills continuously (~$0.025/hour) while it exists.

---

## 2. Resources created

| Resource | Name | Scope | Purpose |
|---|---|---|---|
| Static external IP | `racecars-lb-ip` | global | Frontend IP for the load balancer; also became the sslip.io hostname |
| Unmanaged instance group | `racecars-ig` | zonal (`us-central1-a`) | Wraps `stockapp-vm` as a backend target, named port `http:80` |
| Health check | `racecars-health-check` | global | HTTP check on port 80, path `/racecars/` |
| Backend service | `racecars-backend` | global | HTTP, references the health check and instance group |
| URL map | `racecars-url-map` | global | Routes all traffic to `racecars-backend` |
| Managed SSL certificate | `racecars-cert` | global | Issued for `34-49-119-111.sslip.io` |
| Target HTTPS proxy | `racecars-https-proxy` | global | Binds the URL map and cert |
| Forwarding rule | `racecars-https-rule` | global | Port 443, bound to `racecars-lb-ip` — **this is what starts billing** |

The existing firewall rule `stockapp-vpc-allow-80` (ingress, `tcp:80`, source `0.0.0.0/0`) was reused as-is — it already permits traffic from Google's health-check/LB IP ranges since it's open to `0.0.0.0/0`. No new firewall rule was needed.

---

## 3. Build steps actually run

```bash
# 1. Static IP
gcloud compute addresses create racecars-lb-ip --global
gcloud compute addresses describe racecars-lb-ip --global --format="get(address)"
# -> 34.49.119.111

# 2. Instance group wrapping stockapp-vm
gcloud compute instance-groups unmanaged create racecars-ig --zone=us-central1-a
gcloud compute instance-groups unmanaged add-instances racecars-ig \
  --zone=us-central1-a --instances=stockapp-vm
gcloud compute instance-groups set-named-ports racecars-ig \
  --zone=us-central1-a --named-ports=http:80

# 3. Health check
gcloud compute health-checks create http racecars-health-check \
  --port=80 --request-path=//racecars/

# 4. Backend service + URL map
gcloud compute backend-services create racecars-backend \
  --global --protocol=HTTP --port-name=http --health-checks=racecars-health-check
gcloud compute backend-services add-backend racecars-backend \
  --global --instance-group=racecars-ig --instance-group-zone=us-central1-a
gcloud compute url-maps create racecars-url-map --default-service=racecars-backend

# 5. Managed SSL cert (sslip.io hostname derived from the static IP)
gcloud compute ssl-certificates create racecars-cert \
  --domains=34-49-119-111.sslip.io --global

# 6. Target HTTPS proxy + forwarding rule (billing starts here)
gcloud compute target-https-proxies create racecars-https-proxy \
  --url-map=racecars-url-map --ssl-certificates=racecars-cert
gcloud compute forwarding-rules create racecars-https-rule \
  --global --target-https-proxy=racecars-https-proxy \
  --address=racecars-lb-ip --ports=443
```

---

## 4. Verification

```bash
gcloud compute ssl-certificates describe racecars-cert --global --format="get(managed.status)"
# PROVISIONING -> ACTIVE (took roughly 10-20 minutes)

curl -v https://34-49-119-111.sslip.io/racecars/
# TLS handshake completed, connection left intact, full page HTML returned
# including the closing </body></html> and inline JS (IntersectionObserver
# card reveal + animated banner numbers)
```

Also opened `https://34-49-119-111.sslip.io/racecars/` directly in a browser: padlock showed as trusted (Google-managed cert working as expected), page rendered and animated correctly.

Confirmed `http://` (port 80, no `s`) on the same sslip.io hostname does **not** work through the load balancer — expected, since the forwarding rule was only created for port 443.

---

## 5. Troubleshooting encountered

| Symptom | Cause | Fix |
|---|---|---|
| `health-checks create` set `--request-path` to a literal Windows path (`C:/Users/shairj/.../racecars/`) | Git Bash / MINGW64 path conversion rewrites any argument starting with `/` into a Windows path before `gcloud` sees it — same root cause as the `gcloud storage cp` issue in the original deployment doc | Used a doubled leading slash: `--request-path=//racecars/` (functionally equivalent to `/racecars/` for HTTP path matching) |
| `MSYS_NO_PATHCONV=1 gcloud ...` failed with `python.exe: can't open file 'C:\c\Users\...\gcloud.py'` | Disabling MSYS path conversion entirely also broke gcloud's own internal path resolution for its launcher script — not a viable fix for this environment | Reverted to the doubled-slash approach instead of disabling path conversion globally |
| `backend-services create` / `add-backend` / `url-map create` all failed with "resource ... was not found" | These commands ran in the same paste as an earlier failed `health-checks create`, so the health check didn't exist yet when they ran | Re-ran the health check creation on its own line, confirmed it succeeded, then re-ran the dependent commands |
| Browser DevTools Network tab was empty after opening the working HTTPS page | DevTools only records requests made while the Network panel is open; the page had already finished loading before it was opened | Checked **Preserve log**, then reloaded the page to capture the request list |

---

## 6. Teardown

Run in reverse dependency order:

```bash
gcloud compute forwarding-rules delete racecars-https-rule --global --quiet
gcloud compute target-https-proxies delete racecars-https-proxy --quiet
gcloud compute ssl-certificates delete racecars-cert --global --quiet
gcloud compute url-maps delete racecars-url-map --quiet
gcloud compute backend-services delete racecars-backend --global --quiet
gcloud compute health-checks delete racecars-health-check --quiet
gcloud compute instance-groups unmanaged delete racecars-ig --zone=us-central1-a --quiet
gcloud compute addresses delete racecars-lb-ip --global --quiet
```

### Post-teardown verification actually run

```bash
gcloud compute addresses list
```
Result: only `stockapp-static-ip` (`136.65.201.133`, `IN_USE`, attached to `stockapp-vm`) remained. This one is **free** — static IPs attached to a running instance don't bill; only reserved-but-unattached static IPs do. `racecars-lb-ip` was confirmed gone.

```bash
gcloud compute instances list
```
Result: only `peer-vm` and `stockapp-vm` (both `e2-micro`, `RUNNING`) remained — confirming the separate `loadbalancetf-vm` demo (from the earlier Terraform walkthrough) had already been destroyed via `terraform destroy`, along with its associated static IP and two firewall rules.

`forwarding-rules list`, `target-https-proxies list`, and `ssl-certificates list` were also expected to come back empty (not independently pasted back in this session, but no errors were reported and no charges were expected to remain based on the addresses/instances checks above).

---

## 7. Cost summary for this exercise

| Phase | Approx. cost |
|---|---|
| Steps 1–5 (static IP reserved, instance group, health check, backend service, URL map, managed cert) | $0 — no forwarding rule yet |
| Step 6 onward, while `racecars-https-rule` existed | ~$0.025/hour (first 5 global forwarding rules) |
| After teardown | $0 — confirmed no orphaned static IP, forwarding rule, or proxy |

Total actual spend for this session: a few cents to roughly a dollar, depending on how long the forwarding rule existed between creation and teardown.

---

## 8. Current end state

- `stockapp-vm` is unchanged from the original deployment: still serving `/`, `/api/`, `/lb-demo/`, and `/racecars/` over plain HTTP on `136.65.201.133`.
- No load balancer, managed cert, or extra static IP remain.
- `loadbalancetf-vm` and its associated Terraform-managed resources are destroyed.
- The full HTTPS Load Balancer build is documented above and can be re-run from Section 3 at any time by re-running the same commands (the sslip.io hostname will change if the reserved IP changes, since a new `gcloud compute addresses create` call generally returns a different address).

## 9. Possible next steps (not yet done)

- Revisit the other three sections of the learning plan: Cloud Monitoring/Alerting, Cloud Function + Firestore visitor counter, Cloud Build CI/CD from GitHub.
- If HTTPS is wanted as a persistent (not just learning-exercise) feature, consider Option A from the learning-extensions doc instead: Certbot directly on `stockapp-vm`'s existing Nginx, which has no forwarding-rule billing at all — trading the "real load balancer" architecture for zero ongoing cost.
- If a real GCP Load Balancer is wanted long-term, a registered domain (Cloud DNS or external registrar) would replace the sslip.io hostname, avoiding reliance on a third-party free service for anything beyond learning purposes.



User Browser → Forwarding Rule (Port 443) → HTTPS Proxy (SSL Decryption) → URL Map (Routing) → Backend Service (Health Checking) → Instance Group → stockapp-vm (Port 80).
The URL Map sends users to the Backend Service. The Backend Service checks its Health Check notes to see if the VM is alive. If the notes say "Healthy," it passes the users to the VM.