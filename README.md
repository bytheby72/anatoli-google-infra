# 🔥 Google Workspace Infrastructure for Hermes Agent

**[🇷🇺 Русский](README.ru.md) · [🇨🇳 中文](README.zh.md)**

> Configure once — forget forever.

---

## 😤 The Pain: Why Google Keeps Breaking

If you've ever set up Google Drive or Gmail access for a bot or script, you know this pain:

- **OAuth token expires** in an hour, refresh token gets revoked by Google "due to suspicious activity"
- **"Precondition check failed"** — API is enabled, token is fresh, but Gmail still doesn't work
- **Service account** works for Drive, but **cannot access personal Gmail**
- Every two weeks you're back in the browser, clicking "allow", copying URLs, pasting into terminal...

That's not automation. That's a new job.

---

## ✨ The Solution: Two-Layer Architecture

This skill solves the problem once and for all. No manual re-auth. No dancing with tambourines.

| Service | Method | Why It Lasts Forever |
|---------|--------|---------------------|
| **Drive** | Service Account | JSON key. Never expires. Never. |
| **Sheets** | Service Account | Same key. |
| **Docs** | Service Account | Same key. |
| **Calendar** | Service Account | Same key. |
| **Gmail** | OAuth + Auto-Refresh | Refresh token lives for years. Auto-updates. |

### The Key Insight

Google doesn't give one method for everything. Service account is bulletproof but blind to Gmail. OAuth is flexible but capricious.

**We use both:**
- **Service account** → Drive, Sheets, Docs, Calendar (everything except Gmail)
- **OAuth** → Gmail (with automatic refresh)

Hermes picks the right method for each API automatically. You just write:

```bash
check drive
check mail
```

---

## 🚀 Quick Start

### 1. Service Account (Drive/Sheets/Docs/Calendar)

```bash
# Create in Google Cloud Console:
# https://console.cloud.google.com/iam-admin/serviceaccounts
# → Create key → JSON → save as:
~/.hermes/google_service_account.json

# Enable APIs:
# Drive API, Sheets API, Docs API, Calendar API
```

### 2. OAuth (Gmail)

```bash
# Enable Gmail API in Cloud Console
# https://console.cloud.google.com/apis/library/gmail.googleapis.com

# Run authorization:
python setup.py --client-secret /path/to/client_secret.json
python setup.py --auth-url
# Open link in browser → authorize → paste URL back
python setup.py --auth-code "URL_FROM_BROWSER"
```

### 3. Verify

```bash
python google_api.py gmail search "is:unread" --max 5
python google_api.py drive search "report" --max 5
```

Works? Works. **You'll never have to touch it again.**

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Hermes Agent                          │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
   ┌────▼────┐              ┌─────▼─────┐
   │  OAuth  │              │  Service  │
   │  Token  │              │  Account  │
   │(Gmail)  │              │(Drive/    │
   │         │              │ Sheets/   │
   └────┬────┘              │ Docs/Cal) │
        │                   └─────┬─────┘
        │                         │
   ┌────▼────┐              ┌─────▼─────┐
   │ Gmail   │              │ Google    │
   │ API     │              │ Drive API │
   └─────────┘              │ Sheets API│
                            │ Docs API  │
                            │ Calendar  │
                            └───────────┘
```

---

## 🛡 Recovery (If Something Breaks)

| Symptom | Cause | Fix |
|---------|-------|-----|
| Drive empty | Service account not shared on folder | Share folder with `hermes-drive@...` |
| Gmail 400/401 | OAuth revoked | Re-authorize via `setup.py --auth-url` |
| "Precondition check failed" | Service account trying Gmail | Already fixed — uses `force_oauth=True` for Gmail |

Full instructions in `SKILL.md`.

---

## 💡 The "Wow" Effect

> You set this up **once in 10 minutes**. Then you write `check mail` — and six months later it still works. No cron jobs. No reminders. No `invalid_grant`.

This isn't a script. It's infrastructure that just lives.

---

## 📦 Install as Hermes Skill

```bash
hermes skills install https://github.com/bytheby72/anatoli-google-infra
```

Or add to `config.yaml`:
```yaml
skills:
  external_dirs:
    - https://github.com/bytheby72/anatoli-google-infra
```

---

*Built with 🔥 and zero patience for broken OAuth.*
