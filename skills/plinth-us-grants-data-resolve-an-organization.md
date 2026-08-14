---
generated: '2026-08-14'
method: generated
name: Resolve an organization name to an EIN
description: Turn a US foundation or nonprofit name into its EIN, its type, its state and the canonical URL of its Plinth page — with no API key and no allowance cost.
api: openapi/plinth-us-grants-data-openapi.json
operations: [searchOrganizations]
source: >-
  Grounded in openapi/plinth-us-grants-data-openapi.json (operationId verified verbatim) plus
  https://data.useplinth.com/developers. Response fields observed on a live 200 for q=barancik on
  2026-08-14.
---

# Resolve an organization name to an EIN

Every other Plinth call is keyed by EIN. This is the step that comes first, and it is the only one that costs nothing.

## Auth

**None.** `searchOrganizations` is the one operation in the API with `security: []` — no key, no allowance. The provider's own onboarding descriptor calls it "the intended first call."

Do not spend a keyed call resolving a name. See `authentication/plinth-us-grants-data-authentication.yml`.

## Steps

1. **Resolve the name** — `searchOrganizations` (`GET /api/search?q=<name>`). Optional: `state=<XX>` to narrow to a US state, `mode=semantic|hybrid` to add vector matching.

   ```
   curl "https://data.useplinth.com/api/search?q=barancik"
   ```

2. **Read the result.** Each row carries `ein`, `name`, `kind` (`foundation` / nonprofit), `state`, `cause`, `revenue`, `score` (relevance) and `url` — the canonical, citable Plinth page for that organization. Results come back best-first.

3. **Pick deliberately.** The API returns candidates, not a single answer. Disambiguate on `state`, `kind` and `revenue` before committing an EIN to downstream calls — a wrong EIN produces a confident, wrong dossier.

4. **Carry the `ein` forward** into `getEssentials`, `getPremier`, `getCompliance`, or the `funder_id` / `recip_id` filters on the `/grants/*` endpoints.

## Gotchas

- **Under two characters returns an empty 200**, not an error. `"Fewer than 2 characters returns an empty result set."`
- **`mode` silently downgrades.** `semantic` and `hybrid` "fall back to text if unavailable." The response echoes the `mode` actually used — check it if semantic matching matters to your answer.
- **Never substring-match names yourself.** Org names are stored uppercase and as-filed, and one organization files under many spellings (the American Red Cross, EIN 530196605, appears a dozen ways). This endpoint exists precisely so you don't have to guess.
- **EIN format is forgiving on input** (`"any format; we normalize"`) but is a 9-character zero-padded string everywhere else, including in SQL.

## Errors

No 401/402 apply — this operation is unauthenticated and unmetered. See `errors/plinth-us-grants-data-problem-types.yml`.

## Notes

- `url` is a stable public page you can cite. If your task is to point a human at a source rather than compute a figure, you may be done after this one free call.
