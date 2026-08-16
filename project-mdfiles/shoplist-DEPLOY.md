# Shopping List — Cloud Run + Firestore (project: my-generic-project-504019)

Files needed: `app.py`, `requirements.txt`, `Dockerfile`

## 1. Upload and enable APIs

```bash
mkdir -p ~/shoplist && cd ~/shoplist
gcloud config set project my-generic-project-504019
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
  cloudbuild.googleapis.com firestore.googleapis.com secretmanager.googleapis.com

gcloud firestore databases create --location=us-central1
```

That last command is the one new piece versus the photo site — it provisions your actual Firestore database (one-time per project).

## 2. Create secrets

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32), end='')" | gcloud secrets create shoplist-access-token --data-file=-
python3 -c "import secrets; print(secrets.token_hex(32), end='')" | gcloud secrets create shoplist-secret-key --data-file=-
```
(Note the `end=''` — avoids a trailing-newline mismatch that silently breaks the login link later.)

Save your token:
```bash
gcloud secrets versions access latest --secret=shoplist-access-token
```

## 3. Build and push

```bash
gcloud artifacts repositories create shoplist-repo --repository-format=docker --location=us-central1
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/shoplist-repo/shoplist
```

## 4. Deploy

```bash
gcloud run deploy shoplist \
  --image=us-central1-docker.pkg.dev/my-generic-project-504019/shoplist-repo/shoplist \
  --region=us-central1 \
  --allow-unauthenticated \
  --set-secrets=ACCESS_TOKEN=shoplist-access-token:latest,FLASK_SECRET_KEY=shoplist-secret-key:latest
```

## 5. Grant permissions (required — nothing has access by default)

```bash
PROJECT_NUMBER=$(gcloud projects describe my-generic-project-504019 --format="value(projectNumber)")

gcloud secrets add-iam-policy-binding shoplist-access-token \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding shoplist-secret-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding my-generic-project-504019 \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/datastore.user"
```

## 6. Get the link

```bash
gcloud run services describe shoplist --region=us-central1 --format="value(status.url)"
```
```
https://<service-url>/g/<access-token>
```

#get current token from secret manager
gcloud secrets versions access latest --secret="shoplist-access-token" --project=my-generic-project-504019

## Features included

- **Categories** — fully editable from the Repository page (add/remove, each with its own tax-exempt flag); no longer hardcoded, seeds 11 sensible defaults on first load
- **Stores** — dedicated page, add/rename/delete
- **Active shopping list** — associate with a store, check items off, edit prices live, manual add/remove, running subtotal/tax/total
- **Tax logic** — 6% applied only to non-exempt categories (produce, meat/poultry, fish/seafood, dairy/eggs, bread, canned/baking goods, and bottled water are exempt by default)
- **"Save to My Lists"** — snapshots the current list (store + items + prices) without clearing the active list
- **"My Lists" page** — every saved list, grouped by store name (alphabetical), newest-first within each group; **Open** to edit a saved list in place (check off, edit prices, add/remove, reassign store) or **Delete** to remove it
- **"Start new trip"** vs **"Delete entire list"** — the first just unchecks everything (reuse items/prices next time), the second wipes the active list completely

## Updating the app

```bash
cd ~/shoplist
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/shoplist-repo/shoplist
gcloud run deploy shoplist --image=us-central1-docker.pkg.dev/my-generic-project-504019/shoplist-repo/shoplist --region=us-central1
```

## Cost

Firestore's free tier (1GB storage, 50K reads/20K writes per day) plus Cloud Run's free tier comfortably cover personal use — expect $0.
