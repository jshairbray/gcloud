# New Cloud Run Project Checklist

The one rule that explains almost every "Permission denied" error you'll
hit: **creating a resource never grants access to it.** Every connection
between two Google Cloud things needs its own explicit
`add-iam-policy-binding` (or equivalent) — no exceptions, no shortcuts.

Use this checklist, in order, for any new Cloud Run project:

## 1. Create the resources first
- [ ] Secrets (`gcloud secrets create ...`)
- [ ] Bucket, if any (`gcloud storage buckets create ...`)
- [ ] Firestore database, if not already created in this project
- [ ] Cloud SQL instance, if any

## 2. Grant access — do this BEFORE your first deploy, not after
For every resource above, ask: "does my Cloud Run service need to touch
this?" If yes, grant it now:

```bash
PROJECT_NUMBER=$(gcloud projects describe PROJECT_ID --format="value(projectNumber)")
SA="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

# Secrets -- one binding per secret
gcloud secrets add-iam-policy-binding SECRET_NAME --member="$SA" --role="roles/secretmanager.secretAccessor"

# Storage bucket
gcloud storage buckets add-iam-policy-binding gs://BUCKET_NAME --member="$SA" --role="roles/storage.objectAdmin"

# Firestore
gcloud projects add-iam-policy-binding PROJECT_ID --member="$SA" --role="roles/datastore.user"

# Cloud SQL
gcloud projects add-iam-policy-binding PROJECT_ID --member="$SA" --role="roles/cloudsql.client"
```

Doing this **before** the first `gcloud run deploy` means you skip the
"deploy → fails → grant access → redeploy" loop entirely.

## 3. Deploy

```bash
gcloud run deploy SERVICE --image=IMAGE --region=REGION --allow-unauthenticated \
  --set-secrets=ENV_VAR=SECRET_NAME:latest
```

## 4. If it still fails on permissions
The error message tells you exactly what's missing — it names the
service account and the specific role required, right in the text. Copy
that role name directly into an `add-iam-policy-binding` command rather
than guessing.

## Why this can't just be automatic
It's deliberate (the "least privilege" principle) — automatically
granting broad access the moment something is created would mean any
container you ever deploy, including a buggy one, could silently read
every secret/bucket/database in the project by default. Explicit,
resource-by-resource grants are the tradeoff for that not being possible.
