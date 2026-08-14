---
generated: '2026-08-14'
method: generated
name: Find who funds a nonprofit
description: Traverse the funding graph backwards — from a nonprofit's name to every foundation whose own IRS filings report a grant to it.
api: openapi/plinth-us-grants-data-openapi.json
operations: [searchOrganizations, listFunders, getGrantsSummary, listGrants]
source: >-
  Grounded in openapi/plinth-us-grants-data-openapi.json (all operationIds verified verbatim) plus
  https://data.useplinth.com/developers and /developers/schema.
---

# Find who funds a nonprofit

The recipient-side workflow, and the one that distinguishes this API: **the same endpoints traverse the graph in both directions.** Swap `funder_id` for `recip_id` and the answer flips from "who did they fund" to "who funds them."

## Auth

`X-API-Key` header. See `authentication/plinth-us-grants-data-authentication.yml`.

## Cost

Two to three keyed calls, plus one free resolution.

## Steps

1. **Resolve the nonprofit** — `searchOrganizations` (`GET /api/search?q=<name>`). Free. Capture `ein`.

2. **Rank the funders** — `listFunders` (`GET /api/grants/funders?recip_id=<ein>&limit=1000`). Returns each funder with total amount, grant count, recipients reached, and the first and last fiscal years they were active.

   The `first_fiscal_year` / `last_fiscal_year` pair is the most useful field here and the easiest to miss: it separates a lapsed funder from a current one. A funder whose `last_fiscal_year` is 2019 is not a live relationship — though remember the 12–24 month filing lag before calling anyone lapsed.

3. **Size the picture** — `getGrantsSummary` (`GET /api/grants/summary?recip_id=<ein>`). Total dollars in, grant count, distinct funders, and the year-by-year breakdown that shows whether support is growing or thinning.

4. **Optional — read the individual grants** — `listGrants` (`GET /api/grants/transactions?recip_id=<ein>&limit=1000`). Row-level grants with the purpose text as filed. Use when the *why* matters, not just the *how much*.

## Narrowing

All four `/grants/*` operations share one filter set, so any of these compose with the above: `year`, `subject` (NTEE major group), `location` (recipient US state, two-letter), `min_amt` / `max_amt` (whole dollars), `page` / `limit`, `sort_by` (`amount` | `year`), `sort_order` (`asc` | `desc`).

## The DAF caveat — read this before presenting a funder list

A large share of what looks like institutional giving is donor-advised fund money. Plinth classifies this in the warehouse (`funder_kind.is_daf`), but **the REST endpoints do not expose the flag** — you only get it through `POST /api/sql` on a paid key.

This matters to the answer, not just to the data: a DAF grant is a donor you cannot re-approach as an institution. If you are answering "who should we cultivate," a top-funders list that silently leads with Fidelity Charitable and Schwab Charitable is a misleading answer. Either say so, or run the SQL:

```sql
SELECT g.funder_ein, o.name, sum(g.amount) AS total
FROM grants g
JOIN funder_kind fk ON fk.funder_ein = g.funder_ein
LEFT JOIN orgs o ON o.ein = g.funder_ein
WHERE g.recipient_ein = '530196605' AND NOT fk.is_daf
GROUP BY 1, 2 ORDER BY total DESC LIMIT 25
```

See `data-model/plinth-us-grants-data-data-model.yml` for the full warehouse.

## Coverage ceilings

- **~33% of grants carry no matched `recipient_ein`.** Foreign grantees have none at all. A "who funds them" list is built from the matched share.
- **A big recipient can be understated.** The same organization can also appear as a separate null-EIN raw-name block, apart from its EIN'd rows. Flag this when the number is load-bearing.
- **Cause totals cover ~67% of grant dollars**, because only matched recipients carry an NTEE code.

## Errors

`401` unknown key · `402` allowance spent or plan gate. See `errors/plinth-us-grants-data-problem-types.yml`.

## Notes

- Report as association, never causation. Date every figure to its fiscal year.
