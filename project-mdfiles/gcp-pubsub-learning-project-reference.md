# GCP Pub/Sub Learning Project — Full Reference

**Project:** `my-generic-project-504019`
**Region used throughout:** `us-central1`
**Architecture:** Event-driven, serverless — no VMs, no public website
**Cost:** $0 (everything stays within Cloud Functions / Cloud Run / Pub/Sub / Firestore free tiers at this scale)

---

## 1. Core Concept: What Pub/Sub Actually Is

Pub/Sub is a **message delivery system**, not an app, database, or compute service. It has exactly one job: move a message from a **publisher** to one or more **subscribers**. It does not log anything, store data long-term for you to query, run code, or call APIs on its own.

```
Publisher (you, or any service)
      │
      ▼
   Topic  ──────► (mailbox with a label — anyone can drop a message in)
      │
      ▼
Subscription(s) ──────► one per subscriber, each gets its own full copy
      │
      ▼
Subscriber (a Cloud Function, a script, a person pulling manually, etc.)
```

Key properties:
- **Decoupled** — the publisher never knows who (if anyone) is listening, or what they do with the message.
- **Fan-out** — multiple independent subscribers can all receive their own copy of the same message and react completely independently.
- **Two delivery styles:**
  - **Push** — Pub/Sub actively wakes up the subscriber (e.g., triggers a Cloud Function) the instant a message arrives.
  - **Pull** — the message sits in a queue until something manually asks for it (`gcloud pubsub subscriptions pull` or the Console's **Pull** button).
- **Ack matters** — a subscriber must acknowledge ("ack") a message or Pub/Sub assumes it wasn't processed. With `RETRY_POLICY_DO_NOT_RETRY` (used throughout this project), an unacked/failed message is simply dropped, not retried.
- **Messages only flow to subscriptions that already exist.** Creating a subscription *after* publishing means that message is gone for that subscriber — there's no "replay from the past" without special seek/replay configuration.
- **Retention window** — unacked messages expire after a default of 7 days.

---

## 2. Project 1 — Pub/Sub Command Reference Lookup (`pubsub-learner`)

### What it does
A single event-driven Cloud Function that:
1. Receives a Pub/Sub message (plain text)
2. Optionally capitalizes it if prefixed with `-c`
3. Searches an existing "cmdref" API (a separate Cloud Run app with Firestore-backed command documentation) for matching commands
4. Writes the search + full results to a Firestore collection (`search-history`)
5. Publishes a summary reply to a second topic (`learning-app-replies`)

### Architecture

```
gcloud pubsub topics publish learning-app-topic --message="-c kubectl"
        │
        ▼
   learning-app-topic (topic)
        │
        ▼
Eventarc trigger → pubsub-learner (Cloud Function, Gen2)
        │
        ├──► calls cmdref API (GET /api/commands?q=...) with Bearer token from Secret Manager
        ├──► writes result to Firestore collection: search-history
        └──► publishes summary → learning-app-replies (topic)
                    │
                    ▼
             replies-viewer (pull subscription) — manually pulled to inspect
```

### Resources created
| Resource | Name | Purpose |
|---|---|---|
| Pub/Sub topic | `learning-app-topic` | Receives the raw search request |
| Pub/Sub topic | `learning-app-replies` | Receives the function's summary reply |
| Pub/Sub subscription (pull) | `replies-viewer` | Lets you manually inspect replies |
| Cloud Function (Gen2) | `pubsub-learner` | The actual logic — entry point `process_message` |
| Firestore collection | `search-history` | Persistent log of every search + result set |
| Secret Manager secret | `cmdref-access-token` (pre-existing, from cmdref app) | Bearer token used to call the cmdref API |

### Final `main.py`

```python
import base64
import json
import os
import functions_framework
import requests
from cloudevents.http import CloudEvent
from google.cloud import firestore
from google.cloud import pubsub_v1

BASE_URL = os.environ.get("CMDREF_BASE_URL", "")
API_TOKEN = os.environ.get("CMDREF_TOKEN", "")
PROJECT_ID = os.environ.get("GCP_PROJECT", "my-generic-project-504019")
REPLY_TOPIC = "learning-app-replies"

db = firestore.Client()
publisher = pubsub_v1.PublisherClient()
reply_topic_path = publisher.topic_path(PROJECT_ID, REPLY_TOPIC)


@functions_framework.cloud_event
def process_message(cloud_event: CloudEvent) -> None:
    message_data = cloud_event.data["message"]["data"]
    decoded = base64.b64decode(message_data).decode("utf-8")
    print(f"Received raw message: {decoded}")

    parts = decoded.split(maxsplit=1)
    if len(parts) == 2 and parts[0] == "-c":
        query = parts[1].upper()
    else:
        query = decoded

    try:
        response = requests.get(
            f"{BASE_URL}/api/commands",
            headers={"Authorization": f"Bearer {API_TOKEN}"},
            params={"q": query},
            timeout=10,
        )
        print(f"API status: {response.status_code}")

        results = []
        if response.status_code == 200:
            data = response.json()
            results = data if isinstance(data, list) else data.get("results", [])

        print(f"API results: {results}")

        db.collection("search-history").add({
            "query": query,
            "raw_message": decoded,
            "api_status": response.status_code,
            "result_count": len(results),
            "results": results,
            "searched_at": firestore.SERVER_TIMESTAMP,
        })
        print("Wrote search record to Firestore")

        reply_payload = {
            "query": query,
            "status": response.status_code,
            "result_count": len(results),
            "results": results,
        }
        publisher.publish(
            reply_topic_path,
            json.dumps(reply_payload).encode("utf-8"),
        )
        print(f"Published reply: {reply_payload}")

    except Exception as e:
        print(f"Error during processing: {e}")
```

### `requirements.txt`
```
functions-framework==3.*
cloudevents
requests
google-cloud-pubsub
google-cloud-firestore
```

### Deploy command
```bash
gcloud functions deploy pubsub-learner \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source="C:\Temp\gcloud\source-code\pubsub-learner" \
  --entry-point=process_message \
  --trigger-topic=learning-app-topic \
  --memory=256MB \
  --set-secrets=CMDREF_TOKEN=cmdref-access-token:latest \
  --set-env-vars=CMDREF_BASE_URL=https://cmdref-767471978058.us-central1.run.app
```

### Key lessons learned
- **`requirements.txt` is a build-time instruction**, not something your code calls. Cloud Build's Python buildpack reads it automatically and runs `pip install -r requirements.txt` before your container is ever deployed.
- **Eventarc** is the routing layer between Pub/Sub and Gen2 functions — it auto-enables on first Gen2 event-driven deploy and converts raw Pub/Sub messages into the standardized `CloudEvent` format your code receives.
- **The `-c` flag is just a manual string check**, not real argument parsing (`argparse` would be the "real" version for multiple flags).
- **The cmdref API search is a substring match**, not exact-match — searching `ls` returns 40+ results (`lsof`, `lsblk`, `docker network ls`, etc.) because all of them contain the letters "ls" somewhere in the command or description.
- **A subscriber only does what its code tells it to.** Pub/Sub itself never writes to Firestore or logs — the function's code makes those explicit calls after receiving the message.
- **Calling the cmdref API is a real HTTP request** to a separate, independently deployed Cloud Run service — authenticated via a Bearer token pulled from Secret Manager, not anything built into Pub/Sub.
- **Pub/Sub never executes commands.** Publishing the text `"ls"` never runs `ls` on any filesystem — it only ever gets treated as a search string against a documentation database.

---

## 3. Project 2 — Simple Ride-Share Fan-Out (`rideshare-*`)

### What it demonstrates
The **fan-out pattern**: one published event, multiple independent subscribers reacting simultaneously, none aware of the others — the same shape used by real ride-share, e-commerce, and IoT backends.

### Architecture

```
gcloud pubsub topics publish ride-requests \
  --message='{"rider_id": 123, "location": "Miami Beach"}'
        │
        ▼
   ride-requests (topic)
        │
   ┌────┴────────────┬────────────────┐
   ▼                 ▼                ▼
rideshare-matching  rideshare-billing  rideshare-notify
(finds a driver)    (pre-auths a       (sends rider a
                     charge)            confirmation)
```

Each function has its **own** Eventarc-managed subscription to the same topic — that's what causes the fan-out. Pub/Sub doesn't need any special "broadcast" configuration; every subscription to a topic independently receives every message published to it.

### Resources created
| Resource | Name | Entry point |
|---|---|---|
| Pub/Sub topic | `ride-requests` | — |
| Cloud Function (Gen2) | `rideshare-matching` | `match_driver` |
| Cloud Function (Gen2) | `rideshare-billing` | `preauth_charge` |
| Cloud Function (Gen2) | `rideshare-notify` | `send_confirmation` |

### `rideshare-matching/main.py`
```python
import base64
import json
import random
import functions_framework
from cloudevents.http import CloudEvent

DRIVERS = ["Alex (Toyota Camry)", "Jordan (Honda Civic)", "Sam (Tesla Model 3)"]

@functions_framework.cloud_event
def match_driver(cloud_event: CloudEvent) -> None:
    message_data = cloud_event.data["message"]["data"]
    decoded = base64.b64decode(message_data).decode("utf-8")
    ride = json.loads(decoded)

    driver = random.choice(DRIVERS)
    print(f"[MATCHING] Ride request from rider {ride['rider_id']} at {ride['location']}")
    print(f"[MATCHING] Matched with driver: {driver}, ETA 4 minutes")
```

### `rideshare-billing/main.py`
```python
import base64
import json
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def preauth_charge(cloud_event: CloudEvent) -> None:
    message_data = cloud_event.data["message"]["data"]
    decoded = base64.b64decode(message_data).decode("utf-8")
    ride = json.loads(decoded)

    estimated_fare = 12.50
    print(f"[BILLING] Pre-authorizing ${estimated_fare} hold for rider {ride['rider_id']}")
    print(f"[BILLING] Hold placed successfully")
```

### `rideshare-notify/main.py`
```python
import base64
import json
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def send_confirmation(cloud_event: CloudEvent) -> None:
    message_data = cloud_event.data["message"]["data"]
    decoded = base64.b64decode(message_data).decode("utf-8")
    ride = json.loads(decoded)

    print(f"[NOTIFY] Sending confirmation to rider {ride['rider_id']}")
    print(f"[NOTIFY] 'Your ride from {ride['location']} is confirmed. Driver arriving in 4 min.'")
```

### `requirements.txt` (identical for all three)
```
functions-framework==3.*
cloudevents
```

### Deploy commands (CLI)
```bash
gcloud functions deploy rideshare-matching \
  --gen2 --runtime=python312 --region=us-central1 \
  --source="C:\Temp\gcloud\source-code\rideshare-matching" \
  --entry-point=match_driver \
  --trigger-topic=ride-requests --memory=256MB

gcloud functions deploy rideshare-billing \
  --gen2 --runtime=python312 --region=us-central1 \
  --source="C:\Temp\gcloud\source-code\rideshare-billing" \
  --entry-point=preauth_charge \
  --trigger-topic=ride-requests --memory=256MB

gcloud functions deploy rideshare-notify \
  --gen2 --runtime=python312 --region=us-central1 \
  --source="C:\Temp\gcloud\source-code\rideshare-notify" \
  --entry-point=send_confirmation \
  --trigger-topic=ride-requests --memory=256MB
```

### Test
```bash
gcloud pubsub topics publish ride-requests --message="{\"rider_id\": 123, \"location\": \"Miami Beach\"}"

gcloud functions logs read rideshare-matching --gen2 --region=us-central1 --limit=5
gcloud functions logs read rideshare-billing --gen2 --region=us-central1 --limit=5
gcloud functions logs read rideshare-notify --gen2 --region=us-central1 --limit=5
```

---

## 4. Console Walkthrough Notes (Current UI, as encountered)

Google's Console has merged Cloud Functions into the Cloud Run experience. Steps that changed during this project:

1. **Entry point:** Cloud Run → **Write a function** (not a separate "Create Function" button under a dedicated Cloud Functions page).
2. **Basics:** set function name, region (`us-central1`), runtime (**Python 3.12**).
3. **Trigger setup — "Direct Pub/Sub subscription" dialog:**
   - **Subscription ID** — leave auto-generated
   - **Select a Cloud Pub/Sub topic** — pick your topic
   - **Service account** — **Compute Engine default service account**
   - **Service URL path** — set to `/` (not the function/entry-point name — `functions-framework` listens on root and routes internally; leaving this as `/entry_point_name` causes 404s on every push)
4. **Authentication:** **Require authentication** — Pub/Sub authenticates its push requests via the selected service account automatically; there is no reason to allow unauthenticated access for a function that's never meant to be called by the public.
5. **Ingress:** **Internal** — restricts calls to originate from within Google Cloud's network (where Pub/Sub's push requests come from), blocking direct public internet access regardless of authentication settings. This is what actually enforces "not a website."
6. **Identity-Aware Proxy (IAP):** leave **off** — IAP protects human-facing web logins; Pub/Sub authenticates via service account token, a different trust model entirely. Turning IAP on would block or complicate Pub/Sub's pushes for no benefit.
7. **Execution environment: Second generation** — a different setting than "Cloud Functions Gen2." This controls the underlying container sandbox (full Linux compatibility, faster CPU/network) versus the legacy first-gen sandbox. Everything created via "Write a function" is already built on Cloud Functions/Cloud Run's Gen2 architecture regardless of this toggle — there is no separate "pick Gen2 Cloud Functions" option to hunt for in this flow.
8. **Code entry:** Inline Editor tabs for `main.py` and `requirements.txt`, plus a separate **Entry point** field.
9. **Publishing a test message (Console):** Pub/Sub → Topics → select topic → **Messages** tab → **Publish Message** → paste JSON/text body → **Publish**.
10. **Viewing logs (Console):** Cloud Functions/Run → select function → **Logs** tab. Console shows logs per-function; there's no single combined view across multiple functions (same limitation as running `gcloud functions logs read` separately per function).
11. **Pulling from a reply/pull subscription (Console):** Pub/Sub → Subscriptions → select subscription → **Messages** tab → **Pull** → set message count → check **Enable ack messages** → **Pull**. Same "must exist before publish" rule applies as the CLI.

---

## 5. IAM Checklist (What to Verify When Something "Does Nothing")

If a Pub/Sub-triggered function never fires and shows *zero* log entries (not even an error), IAM is the most common cause — the invocation never reached the function at all.

**Check the service account each function runs as:**
```bash
gcloud functions describe FUNCTION_NAME --gen2 --region=us-central1 \
  --format="value(serviceConfig.serviceAccountEmail)"
```
Expected: `767471978058-compute@developer.gserviceaccount.com`

**Check the function's invoker binding (most common failure point):**
```bash
gcloud functions get-iam-policy FUNCTION_NAME --gen2 --region=us-central1
```
Look for `roles/run.invoker` granted to the compute service account. Missing = Pub/Sub can't call it = silent failure, zero logs.

**Check everything the compute service account can do project-wide:**
```bash
gcloud projects get-iam-policy my-generic-project-504019 \
  --flatten="bindings[].members" \
  --filter="bindings.members:767471978058-compute@developer.gserviceaccount.com" \
  --format="table(bindings.role)"
```

**Console equivalent:** IAM & Admin → IAM → find the compute service account → view attached roles.

**The definitive test either way:** publish a message and check logs. If nothing shows up at all in the target function, it's an invocation/IAM problem, not a code problem (a code problem would still show at least the first `print()` line before failing).

---

## 6. Real-World Pub/Sub Patterns (For Context)

| Scenario | Publisher event | Independent subscribers |
|---|---|---|
| Ride-share request | "Ride requested" | Matching, billing pre-auth, fraud check, analytics |
| E-commerce order | "Order confirmed" | Inventory decrement, shipping label, email confirmation, recommendation engine |
| Photo/video upload | "File uploaded" | Thumbnail generator, content moderation scan, metadata extractor, search indexer |
| IoT sensor reading | "Reading recorded" | Time-series DB write, threshold alerting, live dashboard feed |
| Microservice logging | "Log/error emitted" | Long-term storage, real-time alerting, debugging search tool |

Common thread: **one event, several independent reactions, publisher never needs to know who's listening or how many there are.**

---

## 7. Cleanup Commands (To Avoid Any Lingering Resources)

```bash
# Project 1
gcloud functions delete pubsub-learner --gen2 --region=us-central1
gcloud pubsub topics delete learning-app-topic
gcloud pubsub topics delete learning-app-replies
gcloud pubsub subscriptions delete replies-viewer

# Project 2
gcloud functions delete rideshare-matching --gen2 --region=us-central1
gcloud functions delete rideshare-billing --gen2 --region=us-central1
gcloud functions delete rideshare-notify --gen2 --region=us-central1
gcloud pubsub topics delete ride-requests
```

---

## 8. Security Note

A Secret Manager token (`cmdref-access-token`) appeared in plaintext in an earlier uploaded file (`CMDDEPLOY.md`) referencing a live service URL. If that document has ever left your direct control, rotate the token:
```bash
gcloud secrets versions add cmdref-access-token --data-file=-
```
(pipe in a freshly generated value, same pattern used to create it originally).
