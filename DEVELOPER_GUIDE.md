# Developer Guide

This guide defines the **foundational architecture** to build the LinkedIn Profile Sync Agent without ambiguity.

> Note: Most UI interactions are **two-stage** due to LinkedIn’s dynamic UI: a click raises a modal/popover, then fields become interactive. **Your code must acknowledge this and wait/refine selectors** between stages.

---

## 1. Module & Data Model

### 1.1 Modules
- `intro` (headline, current position, location, open-to-work flags)
- `about`
- `experience` (multiple items)
- `education` (multiple items)
- `skills` (ordered list)
- `certifications` (multiple items)

### 1.2 JSON Layout
```
/data
  /profiles
    /default
      headline.json
      about.json
      skills.json
      /experience
        0001.json
        0002.json
      /education
        0001.json
      /certifications
        0001.json
```

### 1.3 JSON Schemas (extracts)

**headline.schema.json**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Headline",
  "type": "object",
  "required": ["headline"],
  "additionalProperties": false,
  "properties": {
    "headline": {"type": "string", "minLength": 1, "maxLength": 220}
  }
}
```

**about.schema.json**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "About",
  "type": "object",
  "required": ["about"],
  "additionalProperties": false,
  "properties": {
    "about": {"type": "string", "minLength": 1, "maxLength": 2600}
  }
}
```

**experience-item.schema.json**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Experience Item",
  "type": "object",
  "required": ["id", "title", "company", "start"],
  "additionalProperties": false,
  "properties": {
    "id": {"type": "string", "pattern": "^[0-9A-Za-z_-]{3,}$"},
    "title": {"type": "string", "minLength": 2, "maxLength": 150},
    "company": {"type": "string", "minLength": 1, "maxLength": 200},
    "employment_type": {"type": "string", "enum": ["FULL_TIME","PART_TIME","CONTRACT","SELF_EMPLOYED","INTERNSHIP","APPRENTICESHIP","FREELANCE","SEASONAL","OTHER"]},
    "location": {"type": "string"},
    "location_type": {"type": "string", "enum": ["ON_SITE","HYBRID","REMOTE","UNSPECIFIED"]},
    "start": {"type": "string", "format": "date"},
    "end": {"type": ["string","null"], "format": "date"},
    "current": {"type": "boolean", "default": false},
    "description": {"type": "string", "maxLength": 2000},
    "skills": {"type": "array", "items": {"type": "string"}, "maxItems": 50}
  }
}
```

(Define similar schemas for `education`, `skills`, `certifications`.)

---

## 2. Code Architecture

### 2.1 Packages
```
agent/
  adapters/
    browser.py
    selectors.py
    linkedin.py
  domain/
    models.py
    schemas/
    diffs.py
  plan/
    planner.py
    steps.py
  exec/
    runner.py
    verify.py
  io/
    repo.py
    store.py
    logs.py
app/
  flaskapp/
    app.py
    templates/
    static/
cli.py
```

### 2.2 Core Components

**Browser (Pyppeteer Wrapper)**
- `launch(headless: bool, executable_path: str | None) -> Browser`
- `new_context(user_agent: str | None, viewport: dict | None) -> Context`
- `new_page(context) -> Page`
- `wait_for_network_idle(page, timeout=5000)`
- `screenshot(page, name)`
- `trace_start/trace_stop` (optional)

**SelectorRegistry**
- Backed by `selectors.json` (versioned).
- `get(key) -> Selector` where `Selector = {by, value, fallback[], notes}`.
- Supports `aria`, `role`, `data`, `css`, `text` lookup.

**LinkedIn Adapter**
- `login(page, email, password)`: navigates to login, handles 2FA pause.
- `open_profile_edit(page)`: navigates to the profile edit interface.
- `open_intro_modal(page)`, `open_about_editor(page)`, etc.
- Each method waits for **raised elements** and refines handles.

**Planner**
- Converts **desired module state** → **Steps**.
- Step example (Headline):
  1) `open_intro_modal`
  2) `fill_headline`
  3) `save_intro`

**Runner**
- Executes Steps → Actions with `tenacity` retry, randomized delays (jitter), network-idle waits.
- Emits JSONL logs + screenshots per Step.

**Verify**
- Postconditions: check visible text equals expected; confirm save toast.

**GitOps (repo.py)**
- Branch create/checkout, commit, tag, push.
- Store run artifacts under `/runs/<timestamp>` (logs, screenshots).

---

## 3. Selector Strategy (Static Snapshot → Stable Locators)

When you provide a static snapshot, **map it** to stable selectors:
- Prefer **ARIA roles/labels**: `button[aria-label="Edit intro"]`.
- Prefer **data-* invariants**: `[data-control-name="edit_profile"]`, `[data-test-id="profile-edit-intro"]` (if present).
- Prefer **label proximity**: locate `<label>` by text ("Headline") then traverse to nearest input/textarea within the same form container.
- Avoid brittle full XPaths or nth-child CSS.
- Maintain a **single source of truth** (`selectors.json`) and expose it in the Flask UI for audits.

**Example Selector Keys** (pending actual snapshot mapping):
- `profile.nav.open_edit`
- `profile.intro.open_modal`
- `profile.intro.input_headline`
- `profile.intro.save`
- `profile.about.open_editor`
- `profile.about.textarea`
- `profile.about.save`
- `profile.experience.add_button`
- `profile.experience.item_container`
- `profile.experience.item_title_input`
- `profile.experience.save`

Each entry carries: `{ "by": "aria|data|css|role|text", "value": "...", "fallback": [...], "notes": "..." }`.

---

## 4. Flask App

- **Views**: dashboard, versions, editors (per module), run console.
- **API** (selected): `/api/validate`, `/api/dry-run`, `/api/apply`, `/api/versions`, `/api/logs/<id>`.
- **Security**: local only by default; behind reverse proxy when exposed; `.env` driven secrets.

---

## 5. CLI

```
$ python -m agent.cli init --profile default
$ python -m agent.cli validate --all
$ python -m agent.cli sync dry-run --module headline
$ python -m agent.cli sync apply --module headline --confirm
$ python -m agent.cli versions create feature/new-headline
$ python -m agent.cli selectors audit --module about
```

---

## 6. Testing Strategy

- **Unit tests** for planner, store, schemas, selector registry.
- **Integration (mocked)** browser tests using lightweight HTML fixtures.
- **(Optional) Live smoke** behind an opt-in flag, with rate limits and manual approval pauses.
- Screenshot-based verification when practical.

---

## 7. Error Handling & Observability

- `tenacity` retries with exponential backoff & jitter.
- Detailed JSONL logs with trace IDs.
- Structured failures with context: selector key, page URL, HTML snippet, last screenshot.
- Flask run console shows live tail & thumbnails.

---

## 8. Security/Privacy

- Credentials only via environment variables or secret store.
- Redact secrets in logs.
- Avoid storing retrieved PII beyond what is needed.
- Respect ToS; consider legal review before usage.

---

## 9. Extensibility

- Abstract automation backend (Pyppeteer today; Playwright tomorrow).  
- Add new modules via Schema → Planner Steps → Selectors → Verify rules.  
- `selectors.json` can hold per-locale variants if UI language differs.

