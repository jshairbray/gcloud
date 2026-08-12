# Command Reference App — Cloud Run + Firestore (project: my-generic-project-504019)

Same pattern as the shopping list: Cloud Run + Firestore + link-based login.
Files needed: `app.py`, `requirements.txt`, `Dockerfile`, plus your two
source documents (`1000_linux_commands.md`, `gcloud_commands.md`) to upload
once the app is live.

## 1. Upload and enable APIs

```bash
mkdir -p ~/cmdref && cd ~/cmdref
gcloud config set project my-generic-project-504019
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
  cloudbuild.googleapis.com firestore.googleapis.com secretmanager.googleapis.com
```

If you haven't created a Firestore database in this project yet:
```bash
gcloud firestore databases create --location=us-central1
```
(If one already exists from the shopping list project, skip this — one Firestore database serves every app in the project, each using its own collections.)

## 2. Create secrets

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32), end='')" | gcloud secrets create cmdref-access-token --data-file=-
python3 -c "import secrets; print(secrets.token_hex(32), end='')" | gcloud secrets create cmdref-secret-key --data-file=-
```
Save your token:
```bash
gcloud secrets versions access latest --secret=cmdref-access-token
```

## 3. Build and push

```bash
gcloud artifacts repositories create cmdref-repo --repository-format=docker --location=us-central1
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/cmdref-repo/cmdref
```

## 4. Deploy

```bash
gcloud run deploy cmdref \
  --image=us-central1-docker.pkg.dev/my-generic-project-504019/cmdref-repo/cmdref \
  --region=us-central1 \
  --allow-unauthenticated \
  --set-secrets=ACCESS_TOKEN=cmdref-access-token:latest,FLASK_SECRET_KEY=cmdref-secret-key:latest
```

## 5. Grant permissions

```bash
PROJECT_NUMBER=$(gcloud projects describe my-generic-project-504019 --format="value(projectNumber)")

gcloud secrets add-iam-policy-binding cmdref-access-token \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding cmdref-secret-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding my-generic-project-504019 \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/datastore.user"
```

## 6. Load your two documents

Get your link:
```bash
gcloud run services describe cmdref --region=us-central1 --format="value(status.url)"
```
```
https://<service-url>/g/<access-token>
```

Open that link, click **Upload Document**, and upload each file with a
name for its source:
- `1000_linux_commands.md` → source name **"Linux Commands"**
- `gcloud_commands.md` → source name **"gcloud Commands"**

The parser reads `## Category` headers and `| # | Command | Description |`
tables generically — any future document in this same format (from you or
from me) can be uploaded the same way, no code changes needed.

## Using it

- **Browse page** — search box (matches command text or description),
  filter by category or source
- **Upload page** — add more documents anytime; delete a whole source's
  commands with one click if you want to replace/retire a document

## API (token-protected, same Bearer pattern as the stock tracker)

```bash
TOKEN="<your-access-token>"
BASE="<your-service-url>"
https://cmdref-767471978058.us-central1.run.app/g/_6GKxYfDCRCw23V25F68MpZwXCmP1Bv9i-JOOM649jU
# Search
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/commands?q=kubectl"

# Filter by category
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/commands?category=Networking"

# Filter by source
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/commands?source=gcloud%20Commands"

# Random command (nice for a "command of the day" habit)
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/random"

# List all categories / sources
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/categories"
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/sources"
```

## Updating the app

```bash
cd ~/cmdref
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/cmdref-repo/cmdref
gcloud run deploy cmdref --image=us-central1-docker.pkg.dev/my-generic-project-504019/cmdref-repo/cmdref --region=us-central1
```

## Cost

Same as the shopping list app: Firestore + Cloud Run free tiers comfortably
cover ~1,100 small records and personal-scale traffic. Expect $0.
