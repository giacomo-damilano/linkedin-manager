# LinkedIn Profile Sync Agent (Blueprint)

Sync a **local, versioned JSON profile** to **LinkedIn** via **Pyppeteer**, with a **Flask** admin UI for versions, rapid editing, and live run logs.

> ⚠️ **Terms & Risk**: Automating LinkedIn can violate ToS and trigger bot detection. Use only on accounts you own and accept responsibility. Implement rate limiting, human-in-the-loop pauses, and respect legal constraints.

---

## Why this exists
- Keep your LinkedIn profile in **Git-style version control**.
- Edit discrete modules (headline, about, experience, etc.) via **modular JSON**.
- Apply changes reliably with **robust selectors** and **acknowledged dynamic modals**.

## Features (planned)
- Modular JSON (strictly validated by JSON Schema).
- Selector registry with fallbacks; UI flow acknowledges raised forms.
- CLI + Flask UI: validate, dry-run, apply, view logs/screenshots.
- GitOps: branches, tags, rollbacks.
- Optional “scrape live → diff → plan” later.

---

## Project Layout (blueprint)
```
agent/
  adapters/{browser.py, selectors.py, linkedin.py}
  domain/{models.py, schemas/, diffs.py}
  plan/{planner.py, steps.py}
  exec/{runner.py, verify.py}
  io/{repo.py, store.py, logs.py}
app/
  flaskapp/{app.py, templates/, static/}
data/
  profiles/default/{headline.json, about.json, skills.json, experience/, education/, certifications/}
selectors.json  # single source of truth for locators
```

---

## Quick Start (Docker Compose)

1) Create `.env`:
```
LINKEDIN_EMAIL=your_email@example.com
LINKEDIN_PASSWORD=your_password
HEADLESS=true
BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
DATA_DIR=/workspace/data/profiles/default
GIT_REMOTE=origin
```

2) Build & run:
```
docker compose up --build
```

3) Open Flask UI: `http://localhost:8000`  
   - Validate JSON, edit modules, dry-run, then apply.

> The initial image only provides the **blueprint**. Populate `agent/*` and `app/*` per **DEVELOPER_GUIDE.md**. All interfaces and flows are fully specified to prevent ambiguity.

---

## Environment Variables
- `LINKEDIN_EMAIL`, `LINKEDIN_PASSWORD`: credentials (consider secret manager).
- `HEADLESS`: `true|false`.
- `BROWSER_EXECUTABLE_PATH`: default `/usr/bin/chromium`.
- `DATA_DIR`: profile data root.
- `GIT_REMOTE`: remote name for pushes.
- `SLOWMO_MS`: optional slow motion per action.
- `ACTION_TIMEOUT_MS`: per-action timeout (default 8000).
- `NAV_TIMEOUT_MS`: navigation timeout (default 15000).

---

## Selector Strategy (until snapshot is provided)
- Prefer ARIA roles/labels and data-* attributes.
- Avoid brittle XPaths.
- Keep all locators in `selectors.json` with notes and fallbacks.
- Many selectors are placeholders until you provide static HTML snapshots.

---

## Legal & Ethical
- Respect site terms.
- Use human-in-the-loop confirmation for destructive edits.
- Redact PII in logs.
