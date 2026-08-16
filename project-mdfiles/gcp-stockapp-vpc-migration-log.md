# Stock Tracker VM — Migration from Default VPC to Custom VPC

**Project:** `my-generic-project-504019`
**Date:** 2026-08-14
**VM:** `stockapp-vm` (zone: `us-central1-a`, machine type: `e2-micro`, free tier)
**Result:** ✅ Successful, zero data loss, same external IP retained

---

## Why

Moved `stockapp-vm` off the auto-created `default` VPC and into a purpose-built custom VPC (`stockapp-vpc`), as a hands-on exercise in VPC design, firewall scoping, and safe VM lifecycle management. GCP does not support live-migrating a VM's network interface between VPCs — the VM must be recreated, with its boot disk preserved and reattached.

## What Changed

| Item | Before | After |
|---|---|---|
| VPC | `default` | `stockapp-vpc` |
| Subnet | `default` (auto, `10.128.0.0/20`) | `stockapp-subnet` (`10.10.0.0/24`) |
| Internal IP | `10.128.0.2` | `10.10.0.2` |
| External IP | `136.65.201.133` (ephemeral) | `136.65.201.133` (now static, reserved) |
| SSH access | Open to `0.0.0.0/0` | Restricted to a single `/32` (your IP) |
| RDP rule | Present network-wide (`default-allow-rdp`) | Not recreated — not needed for this Linux VM |
| Boot disk | `stockapp-vm` disk, `autoDelete=False` | Same disk, reattached, `autoDelete=False` preserved |
| App data (SQLite DB, PDFs) | On boot disk | Untouched — carried over on the same disk |

## Steps Performed

1. **Reserved the existing external IP as static**
   `gcloud compute addresses create stockapp-static-ip --addresses=136.65.201.133 --region=us-central1`
2. **Created custom VPC + subnet**
   `stockapp-vpc` (custom mode), `stockapp-subnet` (`10.10.0.0/24`, `us-central1`)
3. **Created firewall rules on `stockapp-vpc`**, matching prior exposure (SSH tightened):
   - `stockapp-vpc-allow-80` — TCP 80, `0.0.0.0/0`, target tag `stockapp`
   - `stockapp-allow-ssh` — TCP 22, restricted to home IP `/32`, target tag `stockapp`
   - `stockapp-allow-icmp` — ICMP, target tag `stockapp`
   - `stockapp-allow-internal` — all TCP/UDP/ICMP within `10.10.0.0/24`
4. **Verified boot disk `autoDelete` was `False`** before touching anything (it was)
5. **Stopped and deleted the old VM**, disk survived (confirmed via `gcloud compute disks list`)
6. **Recreated the VM** in `stockapp-vpc` / `stockapp-subnet`, reattaching the original disk as boot disk and the reserved static IP:
   ```
   gcloud compute instances create stockapp-vm \
     --zone=us-central1-a --machine-type=e2-micro \
     --network=stockapp-vpc --subnet=stockapp-subnet \
     --tags=stockapp \
     --disk=name=stockapp-vm,boot=yes,auto-delete=no \
     --address=stockapp-static-ip
   ```
7. **Verified**: app loads at `http://136.65.201.133`, SSH connects from home IP, network field confirmed as `stockapp-vpc`.

## Cleanup (optional, not yet done)

Old orphaned rule on `default` VPC, no longer used since the VM moved:
```
gcloud compute firewall-rules delete allow-stockapp-80 --project=my-generic-project-504019
```
Left in place (network-wide, may serve other resources like the `gke-dinosite-cluster` workload in this project):
`default-allow-ssh`, `default-allow-rdp`, `default-allow-icmp`, `default-allow-internal`

## Notes for Next Time

- Firewall rule names must be unique **per project**, not per VPC — collided once with `allow-stockapp-80`, renamed to `stockapp-vpc-allow-80`.
- Always check `disks[0].autoDelete` before deleting a VM you want to keep the disk from.
- Reserving the external IP *before* deleting the old VM is what let the same IP carry over.
