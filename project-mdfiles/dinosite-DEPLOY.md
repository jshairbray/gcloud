# Dinosaur Roar Guide — GKE / Kubernetes (project: my-generic-project-504019)

Files needed: `index.html`, `Dockerfile`, `deployment.yaml`, `service.yaml`, `dino-on.sh`, `dino-off.sh`

⚠️ **Unlike the other two projects, GKE nodes are NOT in the Always Free tier.** Costs roughly $1-2/day while running. Use the on/off scripts (bottom of this guide) rather than leaving it up continuously.

## 1. Upload and enable APIs

```bash
mkdir -p ~/dinosite && cd ~/dinosite
gcloud config set project my-generic-project-504019
gcloud services enable container.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
```

## 2. Build and push the image

```bash
gcloud artifacts repositories create dinosite-repo --repository-format=docker --location=us-central1
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/dinosite-repo/dinosite
```

## 3. Create the cluster

```bash
gcloud container clusters create dinosite-cluster \
  --zone=us-central1-a \
  --num-nodes=1 \
  --machine-type=e2-small \
  --disk-size=32
```

`e2-small` (not `e2-micro`) — Kubernetes' own system components need more headroom than the tiny free VM tier provides.

## 4. Connect kubectl and deploy

```bash
gcloud container clusters get-credentials dinosite-cluster --zone=us-central1-a
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

```bash
kubectl get pods                              # watch 2 pods reach "Running"
kubectl get service dinosite-service --watch  # watch for an EXTERNAL-IP (takes 1-3 min)
```

Visit `http://<external-ip>` once assigned.

## Kubernetes concepts this taught

- **Deployment** (`replicas: 2`) — declares "always keep 2 pods running." Test it: `kubectl delete pod <name>` — the site stays up on the surviving pod, and a replacement appears automatically within seconds.
- **Service (LoadBalancer)** — gives a stable external IP that balances traffic across pods; it's what actually costs money while running (separate hourly charge from the node itself).
- **kubectl** — talks to the Kubernetes API for every action (`apply`, `get`, `delete`, `describe`, `logs`).

## Turning it on/off sporadically (recommended day-to-day)

Keep the cluster (its management is free) but scale nodes to zero between viewings — much faster than deleting/recreating the whole cluster.

```bash
chmod +x ~/dinosite/dino-on.sh ~/dinosite/dino-off.sh
```

**Before showing your kid:**
```bash
~/dinosite/dino-on.sh
```
~1-2 minutes, prints the URL when ready. **Note: the external IP changes every time** (the load balancer is recreated) — always use whatever it prints that time, don't bookmark an old one.

**When done:**
```bash
~/dinosite/dino-off.sh
```
Removes the load balancer and scales nodes to 0 — nothing billing until next `dino-on.sh`.

## Updating the site

```bash
cd ~/dinosite
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/dinosite-repo/dinosite
kubectl rollout restart deployment dinosite   # or just run dino-on.sh if currently scaled down
```

## Full cluster teardown (if you want to stop paying the small baseline entirely for a long stretch)

```bash
gcloud container clusters delete dinosite-cluster --zone=us-central1-a
```

Your container image stays safe in Artifact Registry — recreating the cluster later is just Steps 3-4 again, no rebuild needed.
