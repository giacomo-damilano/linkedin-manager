# User Guide

This guide walks you through daily use of the LinkedIn Profile Sync Agent.

## 1. Prepare Your Data
Create a profile folder, e.g. `data/profiles/default` with modular files:

**headline.json**
```json
{"headline": "Senior Data Scientist | Causal ML | MLOps"}
```

**about.json**
```json
{"about": "I design robust ML systems that..."}
```

**experience/0001.json**
```json
{
  "id": "exp-0001",
  "title": "Senior Data Scientist",
  "company": "Acme Corp",
  "employment_type": "FULL_TIME",
  "location": "London, UK",
  "location_type": "HYBRID",
  "start": "2021-03-01",
  "end": null,
  "current": true,
  "description": "Built forecasting platform...",
  "skills": ["Python", "Forecasting", "MLOps"]
}
```

## 2. Validate
Via Flask UI (or CLI), run **Validate** to ensure your JSON matches schemas.

## 3. Dry‑Run
Simulate a sync: you’ll see the plan, selectors, and expected changes. Nothing is written to LinkedIn.

## 4. Apply
Run a real sync. The agent will:
- Open LinkedIn, login, and **pause for 2FA** if needed.
- Navigate to the profile editing interface.
- Perform edits per module, acknowledging **raised modals**.
- Verify and save.

## 5. Review & Commit
Check logs/screenshots. If good, the agent can commit and tag the version.

## 6. Rollback
Checkout a previous tag/branch and re-apply.

## Tips
- Start with a single module (e.g., headline) to validate selectors.
- Use small edits and verify.
- Prefer headless at first, then turn off headless to observe behavior if needed.
