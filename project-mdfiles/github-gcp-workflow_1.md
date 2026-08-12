# Syncing Code Between GitHub and GCP (Cloud Shell)

A quick note on terminology first: code doesn't usually get "pulled from GCP to GitHub" — it's the other way around. GitHub is the source of truth (your remote repo), and GCP (Cloud Shell, a VM, etc.) is a *client* that pulls from it, the same way your laptop is. This doc covers both directions since you'll use both.

## The Big Picture

```
Laptop  →  Local Git  →  GitHub  →  GCP (Cloud Shell / VM / etc.)
```

- **Push** = sending your local commits up to GitHub
- **Pull** = bringing GitHub's latest commits down to a local copy (laptop, Cloud Shell, VM — whichever machine you're on)

Both your laptop and Cloud Shell are just two separate local copies of the same GitHub repo. Neither one talks to the other directly — everything flows through GitHub in the middle.

## Pushing Changes: Laptop → GitHub

Run this from your local project folder after editing files:

```bash
git add .
git commit -m "Describe your change"
git push origin main
```

- `git add .` stages all changed/new/deleted files in the current directory (respects `.gitignore`)
- `git commit` saves a snapshot locally with a message
- `git push origin main` uploads those commits to GitHub

## Pulling Changes: GitHub → GCP (Cloud Shell)

**First time only** — you don't have the repo in Cloud Shell yet:

```bash
git clone https://github.com/username/repo.git
```

**Every time after that** — you already have the folder, you just want the latest:

```bash
cd ~/repo-name
git pull origin main
```

⚠️ Don't run `git clone` again on a folder that already exists — you'll get:
```
fatal: destination path 'repo' already exists and is not an empty directory.
```
Use `git pull` instead once the folder is already there.

## Handling "local changes would be overwritten" Errors

If Cloud Shell has its own uncommitted edits sitting in a file, `git pull` will refuse to overwrite them:

```
error: Your local changes to the following files would be overwritten by merge:
        Main.txt
```

Pick one:

**Keep the Cloud Shell edit** — commit it, then pull (may require resolving a merge conflict):
```bash
git add .
git commit -m "Cloud Shell edit"
git pull origin main
```

**Discard the Cloud Shell edit** — take GitHub's version instead:
```bash
git restore Main.txt
git pull origin main
```

## Getting Files Further Into GCP

Once your Cloud Shell copy is synced with GitHub, you can move the files to wherever they actually need to live in GCP:

**Copy to Cloud Storage:**
```bash
gsutil -m cp -r ~/repo-name gs://your-bucket-name/
```

**Deploy to Cloud Run:**
```bash
gcloud run deploy --source ~/repo-name
```

**Copy to a Compute Engine VM:**
```bash
gcloud compute scp --recurse ~/repo-name your-vm-name:~/ --zone=your-zone
```

## Automating It (Optional, for ongoing projects)

Instead of manually pulling and copying files every time, most real projects automate this with:

- **GitHub Actions** — a workflow file in the repo (`.github/workflows/deploy.yml`) that runs automatically on every push to GitHub, authenticates to GCP, and deploys/copies files
- **Google Cloud Build** — GCP's own CI/CD tool, triggered by a connected GitHub repo, running a `cloudbuild.yaml` pipeline on every push

Either removes the need to manually SSH in and run `git pull` + `gsutil`/`gcloud` commands each time.

## Checking What Changed (git log and friends)

After a pull (or anytime you want to confirm what's actually in the repo), these commands help you see what happened without guessing:

**See recent commit history:**
```bash
git log
```
Press `q` to exit. Add `--oneline` for a condensed one-line-per-commit view:
```bash
git log --oneline
```

**See the actual content changes in the last commit:**
```bash
git log -p -1
```
`-p` shows the diff, `-1` limits it to just the most recent commit. Increase the number for more history (`-3` for the last 3 commits, etc.).

**See which files changed in the last commit, without the full diff text:**
```bash
git show --stat HEAD
```

**Check if your local branch is behind GitHub (without pulling yet):**
```bash
git fetch origin
git log HEAD..origin/main --oneline
```
If this prints commits, you're behind and need to `git pull`. If it's empty, you're already caught up.

**Confirm your local branch, GitHub's main, and GitHub's default pointer all match:**
Look at the first line of `git log -p -1` — it lists all the branch/remote labels pointing at that commit:
```
commit ddb9e9d... (HEAD -> main, origin/main, origin/HEAD)
```
If `HEAD`, `origin/main`, and `origin/HEAD` are all listed together like this, your local copy is fully in sync — nothing missing.

**See what's staged/unstaged/untracked right now:**
```bash
git status
```

**See the line-by-line diff of unstaged changes:**
```bash
git diff
```

## Quick Reference

| I want to...                          | Command |
|----------------------------------------|---------|
| Get the repo for the first time         | `git clone https://github.com/user/repo.git` |
| Update my local copy with GitHub's latest | `git pull origin main` |
| Send my local edits to GitHub           | `git add .` → `git commit -m "..."` → `git push origin main` |
| See what's changed but not committed    | `git status` |
| See the actual line changes             | `git diff` |
| See commit history                      | `git log` (or `git log --oneline`) |
| See what the last commit actually changed | `git log -p -1` |
| Check if you're behind GitHub before pulling | `git fetch origin` → `git log HEAD..origin/main --oneline` |
| Throw away my uncommitted local edits   | `git restore <file>` |
