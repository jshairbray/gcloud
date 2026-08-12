# GCloud Learning Projects — Overview

Project: `my-generic-project-504019`
Region: `us-central1`

---

## 1. ADK Agent — `gcloud-agent-learning`

**Location:** `C:\Temp\gcloud\source-code\gcloud-agent-learning\my_agent`

A local AI agent built with Google's **Agent Development Kit (ADK)**, running against the free Gemini API (not Vertex AI).

### Structure
```
gcloud-agent-learning/
├── adk-env/              # Python virtual environment
└── my_agent/
    ├── agent.py           # Agent definition + tool
    ├── .env                # GOOGLE_API_KEY, GOOGLE_GENAI_USE_ENTERPRISE
    ├── .gitignore
    └── __init__.py
```

### What it does
- Defined with `google.adk.agents.llm_agent.Agent`
- Model: `gemini-3.5-flash` (via free Gemini API, not Vertex)
- Has one custom **tool**: `get_stock_price(ticker)` — returns a fake price from a hardcoded dict (`GOOG`, `AAPL`, `MSFT`), or an "Unknown ticker" error otherwise
- The model decides when to call the tool based on its docstring and the `instruction` field

### Running it
```powershell
cd C:\Temp\gcloud\source-code\gcloud-agent-learning
.\adk-env\Scripts\Activate.ps1
adk web
```
Opens a local dev chat UI at `http://127.0.0.1:8000`.

### Status
Runs locally only — not deployed. A natural next step would be `adk deploy cloud_run` to host the agent itself as a service (this was identified but not yet done).

---

## 2. Gemini API Service — `gemini-api-app`

**Location:** `C:\Temp\gcloud\source-code\gemini-api-app`
**Terraform:** `C:\Temp\gcloud\source-code\gemini-api-terraform`

A standalone **FastAPI** service that calls the Gemini API directly (no ADK), deployed as a real internet-reachable service on **Cloud Run**.

### Structure
```
gemini-api-app/
├── venv/
├── main.py            # FastAPI app, single POST /chat endpoint
├── requirements.txt
└── Procfile           # web: uvicorn main:app --host 0.0.0.0 --port $PORT

gemini-api-terraform/
└── main.tf            # Terraform-managed Cloud Run service definition
```

### What it does
- Single endpoint: `POST /chat` — takes `{"message": "..."}`, calls `gemini-3.5-flash`, returns `{"reply": "..."}`
- No tools/function-calling yet — plain model responses only

### Live service
- **URL:** `https://gemini-api-app-767471978058.us-central1.run.app`
- **Auth:** Locked down — `allUsers` invoker binding removed. Requires a Google identity token to call.
- **Secret:** `GOOGLE_API_KEY` is stored in **Secret Manager** (`gemini-api-key` secret) and mounted into the container — not a plaintext env var.
- **Image:** Pinned to an exact digest in Terraform (not a floating `:latest` tag), to avoid drift between what Terraform describes and what's actually running.

### Calling it
```powershell
$token = gcloud auth print-identity-token
Invoke-RestMethod -Uri "https://gemini-api-app-767471978058.us-central1.run.app/chat" `
  -Method Post -ContentType "application/json" `
  -Headers @{Authorization="Bearer $token"} `
  -Body '{"message":"say hi"}'
```

### Managing infrastructure
Infra changes should go through Terraform now, not `gcloud run deploy --source .` directly (mixing the two causes drift):
```powershell
cd C:\Temp\gcloud\source-code\gemini-api-terraform
terraform plan
terraform apply
```

### Status
Deployed, working, locked down, and Terraform-managed. Not yet connected to the `get_stock_price` tool from the ADK agent project — that integration is still open.

---

## Open next steps
- Decide how to merge the two projects: either deploy the ADK agent itself to Cloud Run (`adk deploy cloud_run`), or add `get_stock_price` as a Gemini function-calling tool inside `main.py`
- Bring `scaling { manual_instance_count = 0; min_instance_count = 0 }` into `main.tf` to eliminate a harmless but recurring cosmetic Terraform diff
- Optionally bring the `gemini-api-key` secret itself under Terraform management (currently only referenced, not created by Terraform)
