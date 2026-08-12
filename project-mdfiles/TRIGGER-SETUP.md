# Automatic Build Trigger for cmdref

Right now, every update means running `gcloud builds submit` and
`gcloud run deploy` yourself. A **trigger** automates that: push code to
GitHub, and Cloud Build builds + deploys it automatically, no commands
needed.

**Your repo structure changes this slightly:** `cmdref` lives as a
*subfolder* inside your existing `source-code` repo
(`github.com/jshairbray/source-code/tree/main/cmdref`), alongside your
other projects — not as its own standalone repo. That means:
- The trigger connects to `source-code`, not `cmdref`
- The build steps need to look *inside* the `cmdref` subfolder specifically (already fixed in `cloudbuild.yaml` — the Docker build context is `cmdref`, not the repo root)
- We scope the trigger to only fire when files inside `cmdref/` actually change, so pushing an update to `photosite` or `shoplist` doesn't needlessly rebuild `cmdref` too

## 1. Add the pipeline definition

Since `cmdref` is already on GitHub, just add `cloudbuild.yaml` (updated version attached) into that same local folder and push it:

```powershell
cd C:\Temp\gcloud\source-code\cmdref
git add cloudbuild.yaml
git commit -m "Add Cloud Build pipeline"
git push
```

## 2. Connect Cloud Build to the source-code repo (one-time, via Console)

This step needs a browser click to authorize — can't be fully scripted, since it's a GitHub OAuth permission grant. If you already connected this repo for another project's trigger, skip this step — one connection covers the whole repo.

1. Console → search **Cloud Build** → **Triggers** (left sidebar)
2. Click **Connect Repository**
3. Choose **GitHub**, sign in / authorize if prompted
4. Select your **`source-code`** repo (not `cmdref`), click **Connect**

## 3. Grant Cloud Build's service account permission to deploy

Same "nothing has access by default" pattern as everything else. **Skip
this step if you've already set up a trigger for another project in this
repo** — it's a project-wide grant, not per-trigger.

```powershell
$PROJECT_NUMBER = (gcloud projects describe my-generic-project-504019 --format="value(projectNumber)")
$CB_SA = "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/run.admin"
gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/iam.serviceAccountUser"
gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/artifactregistry.writer"
```

## 4. Create the trigger, scoped to the cmdref subfolder

```powershell
gcloud builds triggers create github --name="cmdref-auto-deploy" --repo-name="source-code" --repo-owner="jshairbray" --branch-pattern="^main$" --build-config="cmdref/cloudbuild.yaml" --included-files="cmdref/**"
```

Two things different from a single-project repo:
- `--repo-name="source-code"` — the actual repo, not the subfolder
- `--build-config="cmdref/cloudbuild.yaml"` — path to the pipeline file *within* the repo
- `--included-files="cmdref/**"` — **this is the important one**: only triggers a build when something under `cmdref/` actually changed. Without it, pushing any change anywhere in `source-code` (even to an unrelated project) would trigger a `cmdref` rebuild every time.

## 5. Test it

```powershell
git add app.py
git commit -m "test trigger"
git push
```

Watch it build automatically:
```powershell
gcloud builds list --limit=5
```

Or watch live in the Console: **Cloud Build → History**. Once it finishes, Cloud Run is already updated — no manual deploy needed.

## Manually firing a build (without pushing code)

```powershell
gcloud builds triggers run cmdref-auto-deploy --branch=main
```

## Doing this for your other projects too

Since `photosite`, `dinosite`, `shoplist`, and the stock tracker's files
likely live as sibling folders in this same `source-code` repo, the exact
same pattern applies to each — just repeat Steps 1 and 4 with the
matching folder/service names, e.g.:

```powershell
gcloud builds triggers create github --name="photosite-auto-deploy" --repo-name="source-code" --repo-owner="jshairbray" --branch-pattern="^main$" --build-config="photosite/cloudbuild.yaml" --included-files="photosite/**"
```

Each project gets its own `cloudbuild.yaml` (living in its own subfolder,
same fix as we just made for cmdref's build context) and its own trigger,
independently scoped so they never rebuild each other.

## From here on

Your workflow becomes: edit `app.py` locally → `git add` → `git commit` →
`git push`. Build and deploy happen automatically for whichever project's
folder you touched. The manual `gcloud builds submit` / `gcloud run
deploy` commands still work fine for a quick one-off test if you ever
want to bypass the trigger.
