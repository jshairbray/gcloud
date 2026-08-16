# Text-to-JSON Converter — GCP Cloud Function

Converts a "labeled record" text layout like:

```
ID: 101
Name: Alice Smith
Department: Engineering
Skills: Python, SQL, Docker
```

into JSON:

```json
[
  {
    "id": 101,
    "name": "Alice Smith",
    "department": "Engineering",
    "skills": ["Python", "SQL", "Docker"]
  }
]
```

Handles records separated by blank lines, single newlines, or no
line breaks at all — it splits on each `ID:` marker.

**Status: deployed and confirmed working** as an authenticated
(non-public) Cloud Function.

## Source location

```
C:\Temp\gcloud\source-code\txt-converter
```

Files in this folder:
- `main.py` — Cloud Function HTTP entrypoint
- `parser.py` — core parsing logic
- `requirements.txt` — dependencies
- `sample_input.txt` — example input file
- `local_convert.py` — optional, for converting files without GCP at all

## Deployment

Run from inside `C:\Temp\gcloud\source-code\txt-converter`:

```powershell
gcloud functions deploy text-to-json-converter --gen2 --project=YOUR_PROJECT_ID --runtime=python312 --region=us-central1 --source=. --entry-point=convert_text_to_json --trigger-http
```

Notes:
- No `--allow-unauthenticated` flag — the function requires a valid
  GCP identity token on every call. This was a deliberate choice to
  keep the endpoint private.
- Replace `YOUR_PROJECT_ID` with your real project ID (find it with
  `gcloud projects list` if you only know the display name).
- Deploy takes a couple of minutes; wait for it to finish and note the
  `url:` printed at the end of the output — you'll need it below.
- To let someone else call the function, grant them access:
  ```powershell
  gcloud functions add-invoker-policy-binding text-to-json-converter --region=us-central1 --member="user:someone@example.com" --project=YOUR_PROJECT_ID
  ```

## Calling the function from Windows PowerShell

**`curl.exe` is intentionally avoided** (flagged by infosec tooling in
this environment). `Invoke-RestMethod -Form` is also not usable here
because it requires PowerShell 7+, and this machine is on the
built-in Windows PowerShell 5.1 (`winget` was unavailable to install
PS7, so this approach needs zero new installs).

The confirmed working method uses only built-in .NET classes
(`System.Net.Http`), already part of Windows — nothing new to install:

```powershell
cd C:\Temp\gcloud\source-code\txt-converter

$TOKEN = gcloud auth print-identity-token
$uri = "https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/text-to-json-converter"
$filePath = "sample_input.txt"

Add-Type -AssemblyName System.Net.Http

$httpClient = New-Object System.Net.Http.HttpClient
$httpClient.DefaultRequestHeaders.Authorization = New-Object System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", $TOKEN)

$multipartContent = New-Object System.Net.Http.MultipartFormDataContent
$fileStream = [System.IO.File]::OpenRead((Resolve-Path $filePath))
$fileContent = New-Object System.Net.Http.StreamContent($fileStream)
$multipartContent.Add($fileContent, "file", (Split-Path $filePath -Leaf))

$response = $httpClient.PostAsync($uri, $multipartContent).Result
$responseBody = $response.Content.ReadAsStringAsync().Result

Write-Output $responseBody

$fileStream.Close()
$httpClient.Dispose()
```

Replace `YOUR_PROJECT_ID` in the `$uri` line, and change `$filePath`
to point at whatever file you want to convert.

**Token expiry:** `$TOKEN` is valid for about an hour. If a later
call returns a `401`/`403` in `$responseBody`, just re-run the
`$TOKEN = gcloud auth print-identity-token` line and retry.

### Saving the output to a file

Append this after the `Write-Output` line to save instead of
(or in addition to) printing:

```powershell
$responseBody | Out-File -FilePath "output.json" -Encoding utf8
```

## Alternative: local conversion (no GCP, no network call)

If you just want the JSON without going through the deployed
function at all:

```powershell
python3 local_convert.py sample_input.txt output.json
```

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `401` or `403` in `$responseBody` | Token expired — re-run `$TOKEN = gcloud auth print-identity-token`. Or your account lacks the Cloud Functions Invoker role — see the `add-invoker-policy-binding` command above. |
| Nothing prints at all | Add `Write-Output $response.StatusCode` after the `PostAsync` line to see the raw HTTP status. |
| `.NET`/SSL/TLS exception | Corporate proxy intercepting HTTPS — let me know if this comes up, there's a `ServicePointManager` fix for it. |
| `winget` fails with "process has no package identity" | Known issue on some corporate images; don't rely on `winget` — the HttpClient script above needs no new installs. |
| `Invoke-RestMethod : A parameter cannot be found that matches parameter name 'Form'` | You're on PowerShell 5.1, not 7 — use the HttpClient script instead of `-Form`. |

## Adjusting the parser for other file layouts

`parse_records()` in `parser.py` currently expects the fields `ID`,
`Name`, `Department`, `Skills` (comma-separated list). Send a sample
of any different layout and the parser can be extended to match it.
