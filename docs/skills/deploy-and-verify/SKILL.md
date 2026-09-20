---
name: deploy-and-verify
description: Commit a change, push it to GitHub, watch the CI pipeline build and publish the Docker image to GHCR, verify the Dockhand auto-deploy webhook fired, and confirm the change is actually live on the server. Use when asked to "update the repo", "push and deploy", "check the deployment", "verify the change is live", or when a change was pushed but the server still shows the old version.
---

# Deploy and verify (BlazorDockerApp)

End-to-end procedure for shipping a change and **proving** it reached the server.
The last step matters most: a green CI run does not mean the change is live.

## Pipeline overview

```
local edit → git commit → git push → GitHub Actions → ghcr.io → webhook → Dockhand → Ubuntu
```

| Stage | Where | Typical time |
|-------|-------|--------------|
| Commit + push | local | ~3 s |
| Build + publish image | GitHub Actions | 50–70 s |
| Webhook + redeploy | Dockhand | 5 s (skipped) / 15–20 s (real) |

## 1. Make the change and verify locally

Always confirm the change works **before** pushing. For a CSS change:

```powershell
docker compose up -d --build web
Start-Sleep -Seconds 50
docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

Both containers must report `healthy`. Then confirm the asset is actually served:

```powershell
$html = (Invoke-WebRequest "http://localhost:9115/" -UseBasicParsing).Content
$link = [regex]::Match($html, 'href="(custom\.[^"]+\.css)"').Groups[1].Value
Write-Host "CSS: $link"
(Invoke-WebRequest "http://localhost:9115/$link" -UseBasicParsing).Content.Trim()
```

> **Blazor asset fingerprinting:** static files are renamed to `custom.<hash>.css`.
> The hash changes whenever the file content changes. A plain `-match 'custom\.css'`
> check will fail — always extract the real link from the HTML.

## 2. Commit and push

```powershell
git add <changed files>
git commit -q -m "<message>"
git push origin main
```

> **Never `git add -A` blindly.** The `deploy/` folder contains the user's personal
> notes (`Comment*.md`, `План дальнейших доработок*.txt`). Stage files explicitly.

## 3. Watch the CI run

```powershell
Start-Sleep -Seconds 12
$id = gh run list --limit 1 --json databaseId --jq '.[0].databaseId'
gh run watch $id --exit-status --compact
```

Expect **two** jobs:

| Job | Meaning |
|-----|---------|
| `build-and-push` | Image built and pushed to `ghcr.io/finellie/blazordockerapp` |
| `deploy` | Webhook POSTed to Dockhand |

Confirm both:

```powershell
gh run view $id --json status,conclusion,jobs --jq '"run: " + .status + " | " + .conclusion, (.jobs[] | "job: " + .name + " | " + .conclusion)'
```

## 4. Verify the webhook actually deployed

**This is the step that catches the silent failure.** Read the Dockhand response:

```powershell
gh run view $id --log 2>&1 | Select-String -Pattern "deploy\s+Trigger" |
  Select-String -Pattern "HTTP status|HMAC|bearer|success|Recreated|Started|Pulled|skipped"
```

| Dockhand output | Verdict |
|-----------------|---------|
| `No changes detected, skipping redeploy` | ❌ **Containers NOT updated** — old version still running |
| `Image ... Pulled` → `Container ... Recreated` → `Started` | ✅ Real deploy |
| `{"error":"Authentication required"}` | ❌ Missing `DOCKHAND_WEBHOOK_TOKEN` |
| `{"error":"Invalid webhook signature"}` | ❌ Missing/wrong `DOCKHAND_WEBHOOK_SECRET` |

Timing hint: a real deploy takes **15–20 s**; a skip takes **~5 s**.

### If Dockhand skipped the redeploy

The fix is a Dockhand setting, not a code change:

> **Always redeploy the stack on webhook or scheduled sync, even if no git changes are detected.**

Without it Dockhand compares the image digest, sees it already cached, and skips
recreating containers — while still returning HTTP 200. CI looks green, the server
keeps the old version.

## 5. Confirm the change is live on the server

Fetch the deployed app and check the fingerprint hash matches the local build:

```powershell
$html = (Invoke-WebRequest "http://92.63.100.145:9115/" -UseBasicParsing).Content
[regex]::Matches($html, '<link[^>]*custom[^>]*>') | ForEach-Object { $_.Value }
```

The hash in the deployed HTML must equal the one from step 1. If it differs, the
container is still running the old image.

## Required GitHub secrets

| Secret | Purpose |
|--------|---------|
| `DOCKHAND_WEBHOOK_URL` | Webhook endpoint (`http://` or `https://`) |
| `DOCKHAND_WEBHOOK_TOKEN` | Sent as `Authorization: Bearer <token>` |
| `DOCKHAND_WEBHOOK_SECRET` | HMAC-SHA256 of the body → `X-Hub-Signature-256` |

Dockhand validates **both** the token and the signature. Setting only one yields a 401.

## Pitfalls

- **Green CI ≠ deployed.** Always read the Dockhand response body.
- **`workflow_dispatch` skips the deploy job.** The `deploy` job is gated on
  `github.event_name == 'push' && github.ref == 'refs/heads/main'`. Trigger with a
  real push (or an empty commit) to test the webhook.
- **Windows reserved ports shift after reboot.** If the host port stops binding,
  re-check `netsh interface ipv4 show excludedportrange protocol=tcp` and pick a
  free port outside the excluded ranges.
- **`aspnet:10.0` has no `curl`/`wget`.** Healthchecks must use `bash` with
  `/dev/tcp`, not `wget`.
- **The server IP is not the Dockhand IP.** Dockhand runs on `157.22.188.159`
  (ports 80/443); the app is on `92.63.100.145:9115`. Do not confuse them.
