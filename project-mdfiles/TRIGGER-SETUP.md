# Automatic Build Trigger for cmdref

Right now, every update means running `gcloud builds submit` and
`gcloud run deploy` yourself. A **trigger** automates that: push code to
GitHub, and Cloud Build builds + deploys it automatically, no commands
needed.

**Two things make this setup different from a simple single-project repo:**
1. `cmdref` lives as a *subfolder* inside your existing `source-code` repo
   (`github.com/jshairbray/source-code/tree/main/cmdref`), not as its own
   standalone repo.
2. Google Cloud has moved GitHub integration to a newer **2nd-gen
   connection** system (via "Developer Connect"), which uses a different
   set of commands than older tutorials show. The old
   `--repo-name`/`--repo-owner` flags only work with a legacy connection
   type — if you connected through the Console recently, you almost
   certainly have a 2nd-gen connection instead, which is why that error
   happened.

## 1. Add the pipeline definition

```powershell
cd C:\Temp\gcloud\source-code\cmdref
git add cloudbuild.yaml
git commit -m "Add Cloud Build pipeline"
git push
```

## 2. Create a 2nd-gen connection to GitHub

```powershell
gcloud builds connections create github cmdref-connection --region=us-central1
```

This prints a URL — open it in your browser and authorize Cloud Build to
access your GitHub account. This is the one step that can't be scripted,
since it's a real OAuth permission grant.

Confirm it worked:
```powershell
gcloud builds connections describe cmdref-connection --region=us-central1
```

## 3. Link your specific repo to that connection

A connection can host multiple repos — this step tells it which one you actually want:

```powershell
gcloud builds repositories create source-code --remote-uri="https://github.com/jshairbray/source-code.git" --connection=cmdref-connection --region=us-central1
```

Get its full resource path (you'll need this exact string in Step 5):
```powershell
gcloud builds repositories describe source-code --connection=cmdref-connection --region=us-central1 --format="value(name)"
```
This prints something like:
```
projects/my-generic-project-504019/locations/us-central1/connections/cmdref-connection/repositories/source-code
```
**Copy that whole string.**

## 4. Grant Cloud Build's service account permission to deploy

```powershell
$PROJECT_NUMBER = (gcloud projects describe my-generic-project-504019 --format="value(projectNumber)")
$CB_SA = "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/run.admin"
gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/iam.serviceAccountUser"
gcloud projects add-iam-policy-binding my-generic-project-504019 --member=$CB_SA --role="roles/artifactregistry.writer"
```

## 5. Create the trigger, using the 2nd-gen `--repository` flag

```powershell
gcloud builds triggers create github --name="cmdref-auto-deploy" --region=us-central1 --repository="projects/my-generic-project-504019/locations/us-central1/connections/cmdref-connection/repositories/source-code" --branch-pattern="^main$" --build-config="cmdref/cloudbuild.yaml" --included-files="cmdref/**"
```
(Replace the `--repository` value with whatever Step 3 actually printed, if it differs.)

Note: `--region` is required alongside `--repository` for 2nd-gen triggers — that's likely the other piece the original command was missing.

## 6. Test it

```powershell
git add app.py
git commit -m "test trigger"
git push
```

```powershell
gcloud builds list --limit=5
```

Or watch live in the Console: **Cloud Build → History**.

## Manually firing a build (without pushing code)

```powershell
gcloud builds triggers run cmdref-auto-deploy --branch=main --region=us-central1
```

## Doing this for your other projects too

The connection and repository link (Steps 2-3) only need to happen **once
per repo** — reuse `cmdref-connection` and the `source-code` repository
link for every other project trigger too. You only repeat Steps 1 and 5
per project:

```powershell
gcloud builds triggers create github --name="photosite-auto-deploy" --region=us-central1 --repository="projects/my-generic-project-504019/locations/us-central1/connections/cmdref-connection/repositories/source-code" --branch-pattern="^main$" --build-config="photosite/cloudbuild.yaml" --included-files="photosite/**"
```

Each project needs its own `cloudbuild.yaml` (with a build context pointing at its own subfolder, same fix as cmdref's) and its own `--included-files` scope, so pushes to one project never rebuild another.

## From here on

Edit code locally → `git add` → `git commit` → `git push`. Build and
deploy happen automatically for whichever project's folder changed.
