# Git → GitHub → Google Cloud Workflow

A quick reference for checking code into Git, pushing to GitHub, and pulling it down onto a Google Cloud machine (VM, Cloud Shell, etc.) — both for a whole repo and for just a single file.

---

## 1. Check code into Git (local machine)

```bash
# One-time setup (if the folder isn't a repo yet)
cd /path/to/your/project
git init

# Check status of your changes
git status

# Stage files
git add .                  # everything
git add path/to/file.py    # just one file

# Commit
git commit -m "New Updates"
```

---

## 2. Push to GitHub

If you haven't connected this repo to GitHub yet:

```bash
# Create the repo on GitHub first (via web UI or gh CLI):
gh repo create my-project --public --source=. --remote=origin

# Or if the GitHub repo already exists, just add it as a remote:
git remote add origin https://github.com/<username>/<repo>.git
```

Push your commits:

```bash
git branch -M main          # rename branch to main if needed
git push -u origin main     # first push sets the upstream
```

After that, future pushes are just:

```bash
git push
```

---

## 3. Pull the whole repo into Google Cloud

This works the same whether you're on a Compute Engine VM, Cloud Shell, or Cloud Workstation — anywhere you have a terminal and `git` installed.

```bash
# SSH into your VM (example)
gcloud compute ssh <instance-name> --zone=<zone>

# Clone the repo for the first time
git clone https://github.com/<username>/<repo>.git
cd <repo>

# Later, to get the latest changes
git pull origin main
```

If the repo is private, you'll need to authenticate — either:
- Use a [GitHub Personal Access Token](https://github.com/settings/tokens) as the password when prompted, or
- Set up SSH keys on the GCP instance and add the public key to your GitHub account.

---

## 4. Pull down just ONE file (not the whole repo)

Sometimes you don't want to clone the entire repository — just grab a single file. A few options:

### Option A: `curl` / `wget` the raw file
Works for public repos (or private repos using a token in the URL).

```bash
curl -O https://raw.githubusercontent.com/<username>/<repo>/main/path/to/file.py
```

For a private repo:

```bash
curl -H "Authorization: token YOUR_GITHUB_TOKEN" \
  -O https://raw.githubusercontent.com/<username>/<repo>/main/path/to/file.py
```

### Option B: Sparse checkout (if you also want Git tracking)
Useful if you plan to keep pulling updates to just that file later.

```bash
git clone --no-checkout https://github.com/<username>/<repo>.git
cd <repo>
git sparse-checkout init --cone
git sparse-checkout set path/to/file.py
git checkout main
```

### Option C: GitHub CLI
```bash
gh api repos/<username>/<repo>/contents/path/to/file.py \
  --jq '.content' | base64 --decode > file.py
```

---

## Quick Command Cheat Sheet

| Task | Command |
|---|---|
| Stage all changes | `git add .` |
| Stage one file | `git add file.py` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| Clone whole repo | `git clone <url>` |
| Pull latest changes | `git pull` |
| Download one file (public) | `curl -O <raw-url>` |
| Get one file via Git (tracked) | `git sparse-checkout set <file>` |

---

### Notes
- Replace `<username>`, `<repo>`, `<zone>`, `<instance-name>`, and file paths with your actual values.
- `main` is the default branch name in most new repos; older repos may use `master`.
- For SSH-based cloning instead of HTTPS, use: `git clone git@github.com:<username>/<repo>.git`
