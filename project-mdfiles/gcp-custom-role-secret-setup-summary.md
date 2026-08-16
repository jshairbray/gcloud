# GCP Custom Role & Secret Manager Setup — Summary

**Project:** `my-generic-project-504019`
**Date:** August 12–13, 2026

## Goal

Create a custom IAM role, apply it to a new principal, and use it to scope access to a resource — specifically, give the `gemini-api-app` Cloud Run service a dedicated identity with least-privilege access to its API key.

## What was created

### 1. Custom IAM role
- **Name:** `secretAccessorLite`
- **Scope:** Project-level (`projects/my-generic-project-504019/roles/secretAccessorLite`)
- **Permissions:** `secretmanager.versions.access`, `secretmanager.secrets.get`

```bash
gcloud iam roles create secretAccessorLite \
  --project=my-generic-project-504019 \
  --title="Secret Accessor Lite" \
  --description="Can read secret values but not manage secrets" \
  --permissions=secretmanager.versions.access,secretmanager.secrets.get \
  --stage=GA
```

### 2. Service account
- **Name:** `my-agent-sa@my-generic-project-504019.iam.gserviceaccount.com`

```bash
gcloud iam service-accounts create my-agent-sa \
  --project=my-generic-project-504019 \
  --display-name="My Agent Service Account"
```

### 3. Role binding (custom role → principal → resource)
Bound `secretAccessorLite` to `my-agent-sa`, scoped to the secret actually in use by the app — **`gemini-api-key`** (an existing secret, created Aug 6, that the app was already using; a separate `my-api-key` secret was created earlier in the process but turned out to be redundant).

```bash
gcloud secrets add-iam-policy-binding gemini-api-key \
  --project=my-generic-project-504019 \
  --member="serviceAccount:my-agent-sa@my-generic-project-504019.iam.gserviceaccount.com" \
  --role="projects/my-generic-project-504019/roles/secretAccessorLite"
```

### 4. Cloud Run service updated
`gemini-api-app` now runs as `my-agent-sa` and pulls its API key from Secret Manager at runtime instead of an env var / `.env` file.

```bash
gcloud run services update gemini-api-app \
  --region=us-central1 \
  --service-account=my-agent-sa@my-generic-project-504019.iam.gserviceaccount.com \
  --project=my-generic-project-504019

gcloud run services update gemini-api-app \
  --region=us-central1 \
  --update-secrets=GOOGLE_API_KEY=gemini-api-key:latest \
  --project=my-generic-project-504019
```

Confirmed via:
```bash
gcloud run services describe gemini-api-app --region=us-central1 --project=my-generic-project-504019 \
  --format="yaml(spec.template.spec.containers[0].env)"
```
```yaml
spec:
  template:
    spec:
      containers:
      - env:
        - name: GOOGLE_API_KEY
          valueFrom:
            secretKeyRef:
              key: latest
              name: gemini-api-key
```

Deployed successfully as revision `gemini-api-app-00008-wh5`.

## Issues hit along the way

- **Conditional IAM policy prompt** — the project already had a conditional binding (`cloudbuild-connection-setup`), so `add-iam-policy-binding` required explicitly specifying `--condition=None` for the new binding.
- **Service account didn't exist yet** — first attempt to bind the role failed until `my-agent-sa` was actually created.
- **Project ID typo** — `my-generic-project-50419` (missing a digit) caused a misleading permissions error instead of "not found."
- **Wrong secret name** — Cloud Run update initially failed because it referenced `gemini-api-key`, but the IAM binding had only been granted on the newly-created `my-api-key`. Resolved by granting `my-agent-sa` access to `gemini-api-key` (the secret actually in use) instead.

## Current state

| Component | Status |
|---|---|
| Custom role `secretAccessorLite` | ✅ Created, scoped to `secretmanager.versions.access` + `secretmanager.secrets.get` |
| Service account `my-agent-sa` | ✅ Created |
| Role binding | ✅ Bound to `my-agent-sa` on secret `gemini-api-key` only (not project-wide) |
| Cloud Run runtime identity | ✅ `gemini-api-app` runs as `my-agent-sa` |
| API key source | ✅ Pulled from Secret Manager (`gemini-api-key:latest`) at container start |
| Public access | ⬜ Still has `--allow-unauthenticated` enabled — not yet addressed |
| Cleanup | ⬜ Unused `my-api-key` secret can optionally be deleted |

## Suggested next step

Remove `--allow-unauthenticated` from `gemini-api-app` and set up proper caller authentication (e.g. `roles/run.invoker` scoped to specific callers).
