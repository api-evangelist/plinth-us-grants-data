---
generated: '2026-08-14'
method: generated
name: Profile a grantmaking foundation
description: Build a sourced profile of one US foundation — IRS status, financials, total giving, and who it funds — from a name, in four keyed calls.
api: openapi/plinth-us-grants-data-openapi.json
operations: [searchOrganizations, getCompliance, getPremier, getGrantsSummary, listRecipients]
source: >-
  Grounded in openapi/plinth-us-grants-data-openapi.json (all operationIds verified verbatim) plus
  https://data.useplinth.com/developers. Allowance arithmetic from /developers#access.
---

# Profile a grantmaking foundation

The funder-side workflow: name in, sourced profile out.

## Auth

`X-API-Key` header (or `Authorization: Bearer`). Keys start `plinth_sk_` and are minted by a human at https://data.useplinth.com/account — there is no programmatic issue endpoint, so an agent cannot recover from a 401 on its own. See `authentication/plinth-us-grants-data-authentication.yml`.

## Cost

**Four keyed calls.** On a free key that is 8% of the monthly allowance of 50. Step 1 is free. Budget before you start, and read `X-Calls-Remaining` on every response.

## Steps

1. **Resolve the name** — `searchOrganizations` (`GET /api/search?q=<name>`). Free and unmetered. Capture `ein`. See `skills/plinth-us-grants-data-resolve-an-organization.md`.

2. **Check IRS standing first** — `getCompliance` (`GET /api/compliance/{ein}`). Returns exemption subsection, deductibility, foundation type, NTEE, IRS auto-revocation status and an OFAC (Treasury SDN) name screen. Do this before the financials: if the organization is auto-revoked, that fact leads the profile.

   The OFAC field is **a name screen, not a sanctions determination** — the spec says so explicitly. Report it as a triage signal and never as a finding.

3. **Pull the profile** — `getPremier` (`GET /api/premier/{ein}`) for full financials plus the resolved geographic service footprint. Use `getEssentials` instead if you only need name, location, NTEE and a financials summary — same cost, less data, so prefer `premier` unless you have a reason.

4. **Total the giving** — `getGrantsSummary` (`GET /api/grants/summary?funder_id=<ein>`). Returns total dollars, grant count, distinct funders and recipients, plus a year-by-year breakdown. This is one call instead of paging every grant to count them — the spec calls it out as "cheaper than paging the rows to count them."

5. **Name the grantees** — `listRecipients` (`GET /api/grants/recipients?funder_id=<ein>&limit=1000`). Ranked by dollars received, with location and how many distinct funders backed each one.

   Set `limit` high. One call is billed per request **regardless of page size**, so a 1,000-row page costs the same as a 10-row page.

## Reporting rules

These are Plinth's own terms of use on the data, published on `/methodology`, in `llms.txt` and in the OpenAPI description. Carry them into the answer, not just into a footnote.

- **Date every figure to its fiscal year, never to today.** IRS e-file data lags 12–24 months.
- **Association, never causation.** A funding relationship is not evidence a funder caused an outcome.
- **Cite the filing.** Every figure traces to a named IRS filing; link the Plinth page you took it from.
- **Absence is not zero.** Organizations that do not e-file may be missing entirely.

## Errors

- `401` — no key or unknown key. Not self-recoverable; escalate to your principal.
- `402` — allowance spent, plan gate, or inactive account. **Not transient** — retrying the same call this month fails identically.

See `errors/plinth-us-grants-data-problem-types.yml`.

## Notes

- Every operation here is read-only; a retry cannot corrupt anything, but it does spend allowance again — cache hits are billed. See `conventions/plinth-us-grants-data-conventions.yml`.
- For the same profile conversationally rather than as HTTP, the MCP connector's dossier tool covers steps 2–5 in one turn, but requires the For consultants plan. See `mcp/plinth-us-grants-data-mcp.yml`.
