# Deploying `linux-commands-dashboard` to Cloud Run — Troubleshooting Summary

**Project:** `my-generic-project-504019`
**Service:** `linux-commands-dashboard` (Cloud Run, `us-central1`)
**Stack:** Static HTML/CSS site served via `nginx:alpine`, built with Cloud Build, deployed with Terraform

---

## Final Working Setup

### `Dockerfile`
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY style.css /usr/share/nginx/html/style.css
EXPOSE 80
```

### `main.tf`
```hcl
provider "google" {
  project = "my-generic-project-504019"
  region  = "us-central1"
}

resource "google_cloud_run_v2_service" "dashboard" {
  name                = "linux-commands-dashboard"
  location            = "us-central1"
  ingress             = "INGRESS_TRAFFIC_ALL"
  deletion_protection = false

  template {
    containers {
      image = "us-central1-docker.pkg.dev/my-generic-project-504019/linux-dashboard-repo/linux-dashboard:latest"
      ports {
        container_port = 80
      }
    }
  }
}

resource "google_cloud_run_v2_service_iam_member" "public_access" {
  project  = google_cloud_run_v2_service.dashboard.project
  location = google_cloud_run_v2_service.dashboard.location
  name     = google_cloud_run_v2_service.dashboard.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

output "url" {
  value = google_cloud_run_v2_service.dashboard.uri
}
```

### Deploy commands (from project root, e.g. `C:\Temp\gcloud\source-code\linux-dashboard`)
```powershell
# One-time setup
gcloud config set project my-generic-project-504019
gcloud auth application-default set-quota-project my-generic-project-504019
gcloud services enable artifactregistry.googleapis.com
gcloud artifacts repositories create linux-dashboard-repo `
  --repository-format=docker `
  --location=us-central1
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build (no local Docker install needed — builds in the cloud)
gcloud builds submit --tag us-central1-docker.pkg.dev/my-generic-project-504019/linux-dashboard-repo/linux-dashboard:latest .

# Deploy
terraform init
terraform apply
```

---

## Issues Hit & Fixes

| # | Problem | Root Cause | Fix |
|---|---------|-----------|-----|
| 1 | `gcloud builds submit` failed with "repository gcr.io may not exist" | Local `gcloud` active project (`random-linux-cmd`) didn't match the target project (`my-generic-project-504019`) in the image tag | `gcloud config set project my-generic-project-504019` |
| 2 | Warning: active project ≠ quota project in ADC | Application Default Credentials pointed at a different project than the CLI config | `gcloud auth application-default set-quota-project my-generic-project-504019` |
| 3 | Wanted to move off deprecated `gcr.io` | Container Registry (`gcr.io`) is being phased out in favor of Artifact Registry | Created an Artifact Registry Docker repo (`linux-dashboard-repo`), rebuilt/pushed to `us-central1-docker.pkg.dev/...`, updated `image` in `main.tf` |
| 4 | `terraform apply` → `Error: No configuration files` | The Terraform config file was still named `main.hc` instead of `main.tf` — Terraform only reads `.tf` files | `ren main.hc main.tf` |
| 5 | Cloud Run revision failed to start ("container failed to start and listen on the port... PORT=8080") | `nginx:alpine` listens on port **80** by default, but Cloud Run's default expected port is **8080** | Added `ports { container_port = 80 }` to the `containers` block in `main.tf` |
| 6 | `Error: cannot destroy service without setting deletion_protection=false` | Cloud Run v2's Terraform resource defaults to `deletion_protection = true`; changing `ports` forces a resource replace, which is blocked while protection is on | Added `deletion_protection = false` to the resource |
| 7 | Same `deletion_protection` error persisted after setting it to `false` | The failed revision had left the resource **tainted**; a tainted resource is always destroyed and recreated regardless of other changes, and Terraform re-checked the *old* state value (`true`) before allowing the destroy | `terraform untaint google_cloud_run_v2_service.dashboard`, then `terraform apply` to update `deletion_protection` in place first |
| 8 | Container still failed with the same PORT=8080 error after "fixing" it | The `ports` block had been commented out earlier while debugging #6/#7 and was never re-enabled before applying | Uncommented the `ports { container_port = 80 }` block, confirmed via `terraform plan` that it showed `1 to change`, then applied |
| 9 | `terraform output url` returned `""` after a successful apply | Known quirk — the `uri` attribute doesn't always refresh in output after an in-place update | Fetched URL directly: `gcloud run services describe linux-commands-dashboard --region=us-central1 --format="value(status.url)"`, or ran `terraform apply -refresh-only` |
| 10 | Site loaded infra but returned `403 Forbidden` in browser | Cloud Run requires explicit IAM permission for public/unauthenticated access; `ingress = "INGRESS_TRAFFIC_ALL"` only controls traffic *source*, not *who* can invoke the service | Added `google_cloud_run_v2_service_iam_member` resource granting `roles/run.invoker` to `allUsers`, then `terraform apply` |

---

## Key Lessons

- **`gcloud builds submit`** builds and pushes images entirely in Google Cloud — no local Docker daemon or admin rights required.
- **Terraform only reads `.tf` files** — a `.hc` extension (or any other) will silently be ignored, producing a "no configuration files" error.
- **Cloud Run's default expected port is 8080.** Any container listening on a different port (like nginx's default of 80) needs an explicit `ports { container_port = ... }` block.
- **`deletion_protection = true` is the Cloud Run v2 Terraform default.** Any change that forces a resource replacement (like a port change) will be blocked until you explicitly set it to `false`.
- **Tainted resources always get destroyed and recreated** on the next apply, regardless of what other changes are pending — `terraform untaint` lets other changes (like `deletion_protection`) apply first as in-place updates instead.
- **Public access on Cloud Run requires an explicit IAM grant** (`roles/run.invoker` for `allUsers`) — ingress settings alone don't make a service publicly invokable.

⚠️ **Security note:** Granting `roles/run.invoker` to `allUsers` makes the dashboard accessible to anyone with the URL, with no authentication. Fine for a public demo; revisit if this should be restricted later.
