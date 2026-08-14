---
generated: '2026-08-14'
method: generated
name: Query the grants warehouse with SQL
description: Answer a cross-organization question the fixed endpoints cannot express, by writing read-only SQL against the 31-table Plinth warehouse — without walking into one of the documented traps.
api: openapi/plinth-us-grants-data-openapi.json
operations: [runSql, searchOrganizations]
source: >-
  Grounded in openapi/plinth-us-grants-data-openapi.json (operationId runSql verified verbatim) and
  https://data.useplinth.com/developers/schema, from which every table, column, ceiling and caveat
  below is quoted or paraphrased.
---

# Query the grants warehouse with SQL

`POST /api/sql` is the most powerful operation Plinth ships and the easiest to get quietly wrong. Most of what follows is caveats, and that is deliberate — as the provider puts it, "each one is a query that runs fine and returns the wrong number."

## Auth and access

- `X-API-Key` header, **paid key only** — "there's no free SQL tier."
- Some tables additionally require the **For consultants** plan; naming one on a lesser plan returns `403` rather than a partial answer.
- One query costs **one call** from the monthly allowance, same as any other endpoint.

See `plans/plinth-us-grants-data-plans-pricing.yml`.

## What is accepted

One statement, `SELECT` or `WITH … SELECT`. Rejected: multiple statements, any DDL/DML, and the file-reading functions (`read_parquet`, `read_csv`) — "you query named views, not files."

```
curl -X POST https://data.useplinth.com/api/sql \
  -H "X-API-Key: $PLINTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT funder_ein, sum(amount) AS total FROM grants WHERE tax_year = 2023 GROUP BY 1 ORDER BY total DESC LIMIT 10"}'
```

## Two ceilings, bounding different things

| Ceiling | Value | Signal |
|---|---|---|
| Result rows | 2,000 | `truncated: true` **inside a 200** |
| Execution time | 30 seconds | HTTP `400` (query cancelled) |

**Always check `truncated`.** It arrives on a success response, so an agent that ignores it will report an aggregate computed over a truncated set and never know. The fix is to aggregate in SQL, not to page.

The row cap limits what comes back; the timeout limits the scan behind it. A `GROUP BY` across every grant ever filed reads all of them before it truncates anything — narrow with `tax_year` or `funder_ein`.

## The traps

Each of these runs cleanly and returns a wrong answer. They are the reason to read `data-model/plinth-us-grants-data-data-model.yml` before writing a query.

1. **Never resolve a name with `LIKE`.** Org names are stored UPPERCASE and as-filed, so `LIKE '%Red Cross%'` returns zero rows. Use `find_org(name)`, or `ILIKE`.

2. **`orgs` is a snapshot, not a year panel.** Each row is that org's *most recent* filing, and `tax_year` differs across orgs. Writing `WHERE tax_year = (SELECT MAX(tax_year) FROM orgs)` returns ~70k of ~793k rows — it silently drops about 90% of the sector. For "all" or "current" nonprofits, apply **no** `tax_year` filter. For real year-over-year trends use `grants`, which is a true multi-year panel.

3. **`recipient_country='IN'` is India. `recipient_state='IN'` is Indiana.** Country codes are FIPS-10-4, not ISO-3166 (UK=`UK`, Switzerland=`SZ`). Never substring-match `'india'` on `recipient_raw` — it catches Indiana, American Indian, Indian River and Indian health boards.

4. **Group recipients by `coalesce(recipient_ein, recipient_raw)`,** never `recipient_raw` alone. One organization files under many spellings. Get the display name from `coalesce(ro.name, g.recipient_raw)` off a `LEFT JOIN orgs ro ON ro.ein = g.recipient_ein`.

5. **There is no cause column on `grants`.** A grant's cause is the *grantee's* NTEE major group — join `recipient_ein → org_bmf.ntee_cd` and map the first letter. Only matched recipients carry one, so **any by-cause dollar total covers ~67% of grant dollars. Say so in the answer.**

6. **Separate DAF money from institutional money.** Join `funder_kind` and filter `NOT is_daf` when the question is about who really funds an organization.

7. **`board_link.conf` below ~0.3 is probably a name collision,** not a shared trustee. Query both directions (`WHERE e1 = ? OR e2 = ?`) — the graph is undirected.

8. **UK: always filter `tie_kind = 'interlock' AND NOT structural`** on `uk_board_edge`. Unfiltered, the strongest edges are church administration and "the answer will be wrong in a way that sounds authoritative." `structural` has ~86% recall — say "mostly excluded," not "excluded." The UK graph does **not** join to US orgs.

9. **`gov_funding_federal.amount` is obligations, not outlays,** can be negative (deobligation), and the latest fiscal year is partial — a late-year drop is a floor, not a final figure.

10. **EINs are 9-character zero-padded strings.** Compare as strings; `lpad(x, 9, '0')` if a literal might be short.

## Steps

1. **Read the schema for the tables you intend to touch** — https://data.useplinth.com/developers/schema, mirrored in `data-model/plinth-us-grants-data-data-model.yml`.
2. **Resolve any names to EINs first** via the free `searchOrganizations`, rather than matching on text in SQL.
3. **Write one aggregating statement.** Aggregate in SQL; do not plan to page.
4. **POST it** to `/api/sql`.
5. **Check `truncated`** before using the result.
6. **State the ceilings** that apply to your answer — matched-recipient share, filing lag, state coverage, obligations-not-outlays.

## Errors

`400` query cancelled (narrow it) · `401` unknown key · `402` allowance spent or SQL not on your plan · `403` gated table named. Note that `400` and `403` are documented on the schema page but **not declared in the OpenAPI** — a generated client will not expect them. See `errors/plinth-us-grants-data-problem-types.yml`.

## Notes

- Read-only by construction: no statement you can send here writes anything.
- Same warehouse, conversationally: the "Ask the data" endpoint (`askQuestion`) writes its own SQL, and the MCP connector reaches the same corpus. See `mcp/plinth-us-grants-data-mcp.yml`.
