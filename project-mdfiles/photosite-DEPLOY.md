# Family Photo Site — Cloud Run + Cloud Storage (project: my-generic-project-504019)

Files needed: `app.py`, `requirements.txt`, `Dockerfile`

## 1. Upload and enable APIs

```bash
mkdir -p ~/photosite && cd ~/photosite
gcloud config set project my-generic-project-504019
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
  cloudbuild.googleapis.com storage.googleapis.com secretmanager.googleapis.com
```

## 2. Create the private bucket

```bash
gcloud storage buckets create gs://my-generic-project-504019-family-photos \
  --location=us-central1 \
  --uniform-bucket-level-access
```

`--uniform-bucket-level-access` keeps it private by default — nothing here is publicly reachable except through the app's own login.

## 3. Create secrets

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32), end='')" | gcloud secrets create access-token --data-file=-
python3 -c "import secrets; print(secrets.token_hex(32), end='')" | gcloud secrets create flask-secret-key --data-file=-
```

Note `end=''` in the Python one-liner — without it, `print()` adds an invisible trailing newline that gets stored as part of the secret, which then silently fails to match the token in your URL later. Save your access token:

```bash
gcloud secrets versions access latest --secret=access-token
```

## 4. Build and push

```bash
gcloud artifacts repositories create photosite-repo --repository-format=docker --location=us-central1
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/photosite-repo/photosite
```

## 5. Deploy

```bash
gcloud run deploy photosite \
  --image=us-central1-docker.pkg.dev/my-generic-project-504019/photosite-repo/photosite \
  --region=us-central1 \
  --allow-unauthenticated \
  --set-env-vars=BUCKET_NAME=my-generic-project-504019-family-photos \
  --set-secrets=ACCESS_TOKEN=access-token:latest,FLASK_SECRET_KEY=flask-secret-key:latest
```

If videos are enabled in `app.py` (see note at the bottom), add `--memory=1Gi --timeout=300` to this command.

## 6. Grant permissions (required — nothing has access by default)

```bash
PROJECT_NUMBER=$(gcloud projects describe my-generic-project-504019 --format="value(projectNumber)")

gcloud storage buckets add-iam-policy-binding gs://my-generic-project-504019-family-photos \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

gcloud secrets add-iam-policy-binding access-token \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding flask-secret-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

(If you deploy before running this step, you'll see "Permission denied on secret" — just run this and redeploy.)

## 7. Get the link and share it

```bash
gcloud run services describe photosite --region=us-central1 --format="value(status.url)"
```

Combine with your token:
```
https://<service-url>/g/<access-token>
```

One click logs in and sets a 30-day secure cookie — no password typing. Text this link (not email) to whoever you're sharing with.

## Features included

- **Multi-file upload** — select several photos at once (Ctrl/Cmd-click)
- **Video support** — mp4/mov/webm accepted, shown with a player in the gallery (requires the `--memory=1Gi --timeout=300` flags above)
- **📷 Take Photo button** — opens the phone camera directly, uploads immediately
- **Add to Home Screen** — visit once, then Chrome ⋮ menu → Add to Home Screen for a one-tap app icon
- **Upload progress bar** — live percentage during upload
- **Security**: `noindex`/`robots.txt` (never shows up in search), secure/HttpOnly cookies, basic rate-limiting, no password to guess or leak

## Revoking access later

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32), end='')" | gcloud secrets versions add access-token --data-file=-
gcloud run services update photosite --region=us-central1 --update-secrets=ACCESS_TOKEN=access-token:latest
```
Instantly kills the old link and logs everyone out.

## Updating the app

```bash
cd ~/photosite
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/photosite-repo/photosite
gcloud run deploy photosite --image=us-central1-docker.pkg.dev/my-generic-project-504019/photosite-repo/photosite --region=us-central1
```

## Cost

Free at this scale: Cloud Run's free tier (2M requests/month) and Cloud Storage's 5GB free tier comfortably cover personal photo sharing. Video storage is the one thing that could eventually push past free storage if used heavily.
