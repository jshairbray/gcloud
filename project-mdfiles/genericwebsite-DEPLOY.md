# Google Cloud Static Website Deployment & Styling Guide

This guide walks you through setting up, designing, and automating a public static website using modern Google Cloud CLI commands via Cloud Shell.

## 1. Environment and Project Initialization
Target your specific project configuration inside Cloud Shell to ensure resources deploy to the right workspace.

```bash
# Define your project variables
export PROJECT_ID=my-generic-project-504019
export BUCKET_NAME=my-generic-project-504019-site

# Force gcloud to use your project
gcloud config set project \$PROJECT_ID
```

## 2. Create Styled Website Files
Build your website source directories locally inside Cloud Shell. This setup builds an integrated, modern CSS card layout directly inside the `index.html` file to keep your infrastructure footprint light.

```bash
# Create and move into your project folder
mkdir -p ~/my-website && cd ~/my-website

# Generate a cleanly styled responsive landing page
cat << 'EOF' > index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GCP Live Site</title>
    <style>
        :root {
            --bg-color: #f4f7f6;
            --card-bg: #ffffff;
            --text-color: #333333;
            --primary-color: #34a853; /* GCP Green */
            --accent-color: #4285f4;  /* GCP Blue */
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .card {
            background: var(--card-bg);
            padding: 2.5rem;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
            text-align: center;
            max-width: 400px;
            width: 100%;
        }
        h1 {
            color: var(--accent-color);
            font-size: 1.8rem;
            margin-bottom: 0.5rem;
        }
        p {
            font-size: 1rem;
            line-height: 1.5;
            color: #666;
            margin-bottom: 1.5rem;
        }
        .status-badge {
            background-color: rgba(52, 168, 83, 0.15);
            color: var(--primary-color);
            padding: 0.35rem 0.75rem;
            border-radius: 50px;
            font-size: 0.85rem;
            font-weight: bold;
            display: inline-block;
        }
    </style>
</head>
<body>
    <div class="card">
        <div class="status-badge">● Live Deploy Success</div>
        <h1>Hello from Google Cloud!</h1>
        <p>Your cloud-hosted static application environment has built successfully using modern gcloud storage workflows.</p>
    </div>
</body>
</html>
EOF
```

## 3. Provision the Cloud Storage Infrastructure
Create a dedicated regional storage bucket using the modern `gcloud storage` provider command.

```bash
# Create the bucket in your preferred region
gcloud storage buckets create gs://\$BUCKET_NAME --location=us-central1
```

## 4. Deploy and Configure Web Routing
Upload the application files, manage their web access visibility, and configure the default entry point.

```bash
# Upload your index page directly into the bucket
gcloud storage cp index.html gs://\$BUCKET_NAME/

# Assign public read access permissions to all web users
gcloud storage objects update gs://\$BUCKET_NAME/index.html --add-acl-grant=entity=AllUsers,role=READER

# Tell the storage bucket to route root requests to index.html
gcloud storage buckets update gs://\$BUCKET_NAME --web-main-page-suffix=index.html
```

## 5. Automate Future Updates (Deployment Script)
Instead of typing multiple lines out every time you change your website code, write a swift wrapper script to execute it all in one blow.

```bash
# Generate the deploy shortcut helper script
cat << 'EOF' > deploy.sh
#!/bin/bash
export BUCKET_NAME="my-generic-project-504019-site"

echo "🚀 Syncing modern site files to cloud storage..."
gcloud storage cp index.html gs://\$BUCKET_NAME/

echo "🔒 Opening read permissions to the web público..."
gcloud storage objects update gs://\$BUCKET_NAME/index.html --add-acl-grant=entity=AllUsers,role=READER

echo "✅ Deployment finished successfully!"
EOF

# Make the helper script runnable
chmod +x deploy.sh
```
*Whenever you update `index.html` in the future, you can instantly upload your design changes by executing:* `./deploy.sh`

## 6. Verification
Verify that your beautifully styled content is live and reachable across the public web using terminal utilities or direct browsers.

* **Terminal check:**
  ```bash
  curl "https://googleapis.com\$BUCKET_NAME/index.html"
  ```
* **Browser check:**
  Open the following URL in any web browser to see your styled card component:
  `https://googleapis.commy-generic-project-504019-site/index.html`
