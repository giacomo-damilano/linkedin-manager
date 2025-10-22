# AGENT.md — LinkedIn Profile Sync Agent

## Purpose
This agent orchestrates **end‑to‑end profile synchronization** between a **modular local JSON repository** and the **LinkedIn profile editing UI** using **Pyppeteer** (headless Chromium) for browser automation. It also coordinates with a **Flask app** that provides a GUI for version management, rapid edits, previews, and sync/run controls. The agent is designed to be **tool-driven** so that further development can be AI-assisted without ambiguity.

> ⚠️ **Important**: Interacting with LinkedIn via automation may violate LinkedIn’s Terms of Service and trigger bot-detection. Use only on accounts you control, for personal/enterprise compliance-reviewed purposes. Ensure you have permission, accept risk, and throttle activity.

---

## Top-Level Capabilities
1. **Versioned Local Profile Store**  
   - Read/write modular JSON files (e.g., `headline.json`, `about.json`, `experience/*.json`, etc.).  
   - Validate each JSON file against strict JSON Schemas.  
   - Compute diffs between local JSON and the **retrieved live profile state** (when retrieval is implemented).

2. **Sync Planning & Execution**  
   - Plan atomic UI steps per module (intro/headline, about, experience, education, skills, etc.).  
   - Execute robust UI interactions via Pyppeteer with **stable selectors** from a **selector registry** (no brittle XPaths).  
   - **Acknowledge dynamic “raised forms”** (modals/popovers) by staging actions that intentionally **await and refine** after the DOM raises new elements.

3. **Verification & Logging**  
   - Post-step checks (DOM assertions, screenshots, HTML snapshots).  
   - Structured logs (JSONL) for every action, timing, and retry.  
   - Rollback strategy: if a step fails, stop and persist state for triage.

4. **Flask GUI**  
   - Manage versions/branches/tags (via GitPython).  
   - Rapid edit forms per JSON module with live validation.  
   - “Dry‑run” simulation vs “Apply” real sync, with live logs and screenshots.

5. **Human‑in‑the‑Loop**  
   - Pauses for 2FA and manual approval.  
   - Optional “confirm before apply” checkpoints per section.  
   - Safe rate limits, randomized delays to reduce bot signals.

---

## Agent Architecture

```
/agent
  /adapters
    selectors.py        # Selector registry access (stable CSS/ARIA/data-* locators)
    browser.py          # Pyppeteer wrapper (launch, contexts, tracing, screenshots)
    linkedin.py         # High-level navigation primitives (login, open edit UI, etc.)
  /domain
    models.py           # Pydantic models for profile modules
    schemas/            # JSON Schemas used for strict validation
    diffs.py            # Diff engine mapping local vs current (future: scrape current)
  /plan
    planner.py          # Convert diffs into ordered UI edit plans (Steps/Actions)
    steps.py            # Step/Action dataclasses; idempotency keys; retry policy
  /exec
    runner.py           # Execute plans; apply retry/backoff; emit logs/artifacts
    verify.py           # Postcondition checks; DOM asserts; visual checks
  /io
    repo.py             # GitOps—commits, branches, tags, push/pull (GitPython)
    store.py            # Read/write modular JSON; profile sets; versions
    logs.py             # JSONL + rotating file handler + HTML report
  /flaskapp
    app.py              # Flask server, blueprints, API, web UI
    templates/          # Jinja2 pages (version mgmt, editors, run console)
    static/
  cli.py                # Typer CLI entrypoint (sync, dry-run, validate, init, etc.)
```

### Key Abstractions

- **SelectorRegistry**: A versioned dictionary that maps **semantic keys** to **stable selectors** and **fallback strategies**.  
  Example entry:
  ```json
  {
    "profile.edit.open_intro_button": {
      "by": "aria",
      "value": "Edit intro",
      "fallback": [{"by":"text","value":"Edit"}],
      "notes": "Stable by aria-label; refined when snapshot provided"
    }
  }
  ```
  The registry is updated when you provide static snapshots. The agent prefers **ARIA roles/labels** and **data-* attributes** (e.g., `data-test-id`, `data-control-name`) to avoid brittle CSS paths.

- **Plan/Step/Action**:  
  - **Plan** = ordered list of **Steps** to update modules.  
  - **Step** = one UI unit (e.g., open intro modal → set headline → save).  
  - **Action** = a primitive (click, type, wait, assert). Steps contain multiple actions.

- **Verifiers**: Assertions that confirm state after a step (e.g., headline text matches expected).

- **Backoff Policy**: All Actions use a retry strategy with jitter (e.g., 3 attempts, exponential backoff, `tenacity`).

---

## High-Level Flow

1. **Load Target Version**  
   - Select version/branch (`main`, `feature/new-headline`) from Flask UI or CLI.  
   - Validate all JSON files.

2. **(Optional) Retrieve Live State**  
   - When scraping is enabled, retrieve current profile to build diffs; for now, the plan may trust the local version as source of truth.

3. **Plan Generation**  
   - Compute needed Steps per module (e.g., **Intro/Headline** → open modal → fill → save).  
   - Insert **Refinement Waits** for “raised” modals/forms (acknowledged dynamic behavior).

4. **Execute (Dry‑run or Apply)**  
   - **Dry‑run** logs intended actions, selectors, and expected results.  
   - **Apply** runs Pyppeteer, navigates, edits, verifies, and logs screenshots.

5. **Commit & Tag**  
   - On success, commit local changes (and optionally a sanitized run log) and tag the version (`v1.2.3-profile-sync`).

---

## Selector Strategy (until snapshots are provided)
- Prefer **role/ARIA** selectors (e.g., `button[aria-label="Edit intro"]`).  
- Prefer **data attributes** (e.g., `[data-test-id="profile-about-edit"]`) if present.  
- Use **label proximity**: find input by label text within the same form section (query label, then closest input).  
- Avoid brittle XPaths and deep CSS chains.  
- Encapsulate in `SelectorRegistry` with notes and fallbacks.  
- **Acknowledge refinement**: many selectors are placeholders pending your static snapshots.

---

## Security, Ethics, Compliance
- Store credentials only via env vars or secret managers.  
- Support 2FA pauses and manual verification.  
- Respect site ToS; do not scrape beyond personal data; throttle requests; avoid abusive automation.

---

## Minimum Viable Modules
- Intro/Headline, About/Summary, Experience, Education, Skills.  
- Each module gets a JSON file or directory of JSON files and a Step Plan.

---

## Agent Interfaces

### CLI (Typer)
- `sync apply --module headline --version feature/new-headline`
- `sync dry-run --all`
- `validate json --all`
- `versions list|create|checkout`
- `selectors audit --module headline`

### Flask API (selected endpoints)
- `GET /` dashboard  
- `GET /versions` / `POST /versions`  
- `GET /edit/<module>` (form view)  
- `POST /sync/dry-run` / `POST /sync/apply`  
- `GET /logs/run/<id>` (live tail, screenshots)

---

## Non-Goals (initial)
- Bypassing LinkedIn anti-bot; advanced stealth is out-of-scope.  
- Editing proprietary features beyond standard profile sections.
