---
name: anatoli-google-infra
description: Full Google Workspace setup. Service account for Drive/Sheets/Docs/Calendar + OAuth for Gmail. Auto-refresh enabled. Recovery instructions.
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [google, workspace, gmail, drive, oauth, service-account]
---

# Google Workspace Infrastructure

## Current State (2026-04-18)

| Service | Method | Status | Expiration |
|---------|--------|--------|------------|
| Drive | Service account | Active | Never |
| Sheets | Service account | Active | Never |
| Docs | Service account | Active | Never |
| Calendar | Service account | Active | Never |
| Gmail | OAuth (refresh token) | Active | Auto-refresh |

## Files

- Service account key: `~/.hermes/google_service_account.json`
- OAuth token: `~/.hermes/google_token.json` (auto-refreshes via `get_credentials()`)
- API script: `~/.hermes/skills/productivity/google-workspace/scripts/google_api.py`

## Recovery Scenarios

### 1. Service account stops working

**Symptom:** Drive/Sheets/Docs return 403 or empty results.

**Fix:**
```bash
# Check file exists
ls ~/.hermes/google_service_account.json

# If missing — restore from backup or recreate in Google Cloud Console:
# https://console.cloud.google.com/iam-admin/serviceaccounts
# → Select project → Create key → JSON → save to ~/.hermes/google_service_account.json
```

### 2. Gmail OAuth fails (400/401/invalid_grant)

**Symptom:** `google_api.py gmail search` returns auth error.

**Fix:**
```bash
# Check token
python ~/.hermes/skills/productivity/google-workspace/scripts/setup.py --check

# If token expired/invalid — re-authorize manually:
python ~/.hermes/skills/productivity/google-workspace/scripts/setup.py --auth-url
# Open URL in browser → authorize → paste redirect URL back:
python ~/.hermes/skills/productivity/google-workspace/scripts/setup.py --auth-code "PASTE_URL_HERE"
```

**Note:** Do NOT use browser automation for OAuth — Google blocks headless browsers.

### 3. Gmail API "Precondition check failed"

**Cause:** `google_api.py` was using service account for Gmail (not supported for personal Gmail).

**Fix:** Ensure `build_service("gmail", "v1", force_oauth=True)` is used for all Gmail calls.
This is already patched in the current `google_api.py`.

### 4. IMAP/App Password fallback (if OAuth breaks permanently)

If OAuth is revoked and cannot be restored:
1. Enable 2-Step Verification: https://myaccount.google.com/signinoptions/two-step-verification
2. Create App Password: https://myaccount.google.com/apppasswords (select Mail)
3. Update `~/.hermes/.env`:
   ```
   EMAIL_ADDRESS=your-email@gmail.com
   EMAIL_PASSWORD=YOUR_APP_PASSWORD
   EMAIL_IMAP_HOST=imap.gmail.com
   EMAIL_IMAP_PORT=993
   EMAIL_SMTP_HOST=smtp.gmail.com
   EMAIL_SMTP_PORT=587
   ```
4. Use `~/.hermes/skills/productivity/google-workspace/scripts/gmail_imap.py`

**Note:** App Password may require unlocking CAPTCHA first:
https://accounts.google.com/b/0/displayunlockcaptcha

## Quick Commands

```bash
GAPI="python ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py"

# Drive
$GAPI drive search "report" --max 10

# Gmail
$GAPI gmail search "is:unread" --max 20
$GAPI gmail get MESSAGE_ID
$GAPI gmail send --to user@example.com --subject "Hello" --body "Text"

# Sheets
$GAPI sheets get SHEET_ID "Sheet1!A1:D10"
$GAPI sheets update SHEET_ID "Sheet1!A1" --values '[["new"]]'

# Docs
$GAPI docs get DOC_ID

# Calendar
$GAPI calendar list
$GAPI calendar create --summary "Meeting" --start "2026-04-20T10:00:00Z" --end "2026-04-20T11:00:00Z"
```

## Google Cloud Project

- Project: `your-project-id`
- Service account: `hermes-drive@your-project.iam.gserviceaccount.com`
- Enabled APIs: Gmail API, Drive API, Sheets API, Docs API, Calendar API, People API

## Shell Aliases (Optional Convenience)

Add to `~/.bashrc`:
```bash
alias gdrive='python ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py drive'
alias gmail='python ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py gmail'
alias gcal='python ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py calendar'
alias gsheet='python ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py sheets'
```

Then: `gdrive search "report" --max 5`

## Technical Implementation Detail: force_oauth Patch

The `google_api.py` script originally used service account for ALL APIs. This caused `HttpError 400: Precondition check failed` for Gmail because service accounts cannot access personal Gmail.

**Patch applied to `google_api.py`:**

```python
def get_credentials(force_oauth=False):
    # Gmail requires OAuth even when service account exists
    if not force_oauth and SERVICE_ACCOUNT_PATH.exists():
        from google.oauth2 import service_account
        return service_account.Credentials.from_service_account_file(...)
    # ... OAuth path ...

def build_service(api, version, force_oauth=False):
    from googleapiclient.discovery import build
    return build(api, version, credentials=get_credentials(force_oauth=force_oauth))
```

All Gmail functions now call: `build_service("gmail", "v1", force_oauth=True)`
Drive/Sheets/Docs/Calendar use: `build_service("drive", "v3")` (no force_oauth)

## What We Tried That Did NOT Work

**App Password + IMAP**: Google blocks App Password authentication from server/cloud IPs with `AUTHENTICATIONFAILED`. This is not a reliable path for headless/server-based agents. The unlock CAPTCHA page (`displayunlockcaptcha`) sometimes helps for desktop IPs, but not consistently for cloud servers.

**Himalaya CLI**: Installation from GitHub releases timed out due to network issues. Python `imaplib` + `smtplib` was used instead as fallback, but also failed due to the App Password block above.

## Batch Gmail Operations

For bulk actions (e.g., moving 40 emails to trash):
```python
import subprocess, json

# Search
result = subprocess.run(["python", "google_api.py", "gmail", "search",
                        "from:pinterest.com in:inbox", "--max", "50"],
                       capture_output=True, text=True)
emails = json.loads(result.stdout)

# Move each to trash
for msg in emails:
    subprocess.run(["python", "google_api.py", "gmail", "modify",
                   msg["id"], "--add-labels", "TRASH", "--remove-labels", "INBOX"])
```

## Lessons Learned from Trial and Error

### Lesson 1: Gmail API must be explicitly enabled

Even with a valid OAuth token, Gmail calls fail with `Precondition check failed` if the Gmail API is not enabled in the Google Cloud project. Always verify: https://console.cloud.google.com/apis/library/gmail.googleapis.com

### Lesson 2: `google_api.py` defaults to service account for ALL APIs

The original `google_api.py` checked `SERVICE_ACCOUNT_PATH.exists()` first and used it for everything. This caused `HttpError 400` for Gmail because service accounts cannot access personal Gmail. The fix is a `force_oauth` parameter that Gmail functions must pass.

### Lesson 3: App Password is blocked from cloud servers

Multiple App Passwords were generated and all returned `AUTHENTICATIONFAILED` from the server IP. Google's security systems block App Password logins from cloud/server IPs. The unlock CAPTCHA (`displayunlockcaptcha`) does not help for non-residential IPs. **Do not rely on App Password for headless/server deployments.**

### Lesson 4: OAuth refresh token is the reliable path for Gmail

Once obtained, the refresh token in `google_token.json` auto-refreshes the access token indefinitely. The only way it breaks is if the user manually revokes access in their Google Account. This is the most stable known method for server-based Gmail access on personal accounts.

### Lesson 5: Service account files must be explicitly shared

Service accounts cannot see the user's personal Drive files by default. You must share folders/files with the service account email (`hermes-drive@...`) inside Google Drive. This is easy to forget and causes "empty Drive" confusion.

## Important Rules

1. **Never delete `google_service_account.json`** — it's the permanent key for Drive/Sheets/Docs/Calendar.
2. **Never delete `google_token.json`** — it's the OAuth refresh token for Gmail. Back it up.
3. If both files exist, `google_api.py` uses service account for Drive/Sheets/Docs/Calendar and OAuth for Gmail automatically.
4. Service account CANNOT access Gmail on personal accounts — always use OAuth for Gmail.
5. App Password is NOT a reliable fallback for server/cloud deployments — Google actively blocks it.
