# The pull sequence

## 0. Identify the account

`list_marketplaces` first. More than one TrackIQ MCP can be connected at
once with identical tool names and different brands behind them — call it on
each connected server and match on `name`. Never print `account_id` back to
the user.

## 1. Find the SQP weeks that actually exist

**This is the step everyone skips and it invalidates the whole report.**

SQP is a weekly Sun–Sat cohort and the ingestion is intermittent. Weeks go
missing with no error: the tool returns `{"rows": []}`. On the account this
skill was built against, August 2026 had data for the weeks of 2 Aug and
23 Aug and **nothing at all** for 9 Aug and 16 Aug.

`get_search_query_performance` pins to the *latest* reporting week that
overlaps the date range and has data. So:

1. For each Sun–Sat week overlapping the target month, call it with
   `start_date` and `end_date` set to that week and `limit=2`.
2. A week that returns rows exists. A week that returns `[]` does not.
3. Record the list of weeks present and absent. Both go on the report.

Do not assume four weeks. Do not infer a missing week from its neighbours.

## 2. The catalogue and the categories

| Call | Why |
|---|---|
| `get_product_performance` (`group_by='product'`, whole month, `limit=200`) | ASIN, title, revenue, orders. This is the revenue the report allocates. |
| `get_product_categories_performance` (same month) | the account's **own** category names — the taxonomy to map onto |

`get_product_categories_performance` returns category-level ad totals but
**not which ASIN belongs to which category**. No tool exposes that mapping.
Categories therefore have to be derived from product titles and matched to
the names this call returns. See `assets/method.md`.

Watch for two things in its output:

- **Duplicate categories.** Accounts often carry two overlapping schemes —
  a legacy set and a newer prefixed set — reporting identical figures.
  Never sum them; pick one scheme and say which.
- **Empty categories.** A category with zero spend and zero sales is a
  definition nobody maintains. Leave it out and note it.

## 3. Search Query Performance, per week

For each week that exists, one call:

```
get_search_query_performance(
    account_id, start_date=<week_start>, end_date=<week_end>,
    by_product=True, sort='asin_purchases DESC', limit=1000)
```

`by_product=True` is what makes this skill possible — it returns one row per
(product, query), which is how queries get attached to ASINs and therefore
to categories. The account-level grain cannot be split by category.

**The completeness check:** with `sort='asin_purchases DESC'`, the last row
returned must have `asin_purchases = 0`. If it does, every purchase-bearing
query is in hand and the tail that got cut carries no purchases, so nothing
the report ranks on is missing. If the last row is non-zero, raise `limit`
and pull again.

## 4. What is currently rank-tracked

```
get_keyword_rank(account_id, start_date=<one day>, end_date=<same day>, limit=1000)
```

Use a **single day** — the tool returns one row per (keyword, ASIN, day), so
a week across 50 keywords and 60 ASINs is tens of thousands of rows and the
limit truncates the keyword list silently. Even one day may truncate: if
rows == limit, paginate with `offset` until you have every distinct keyword,
or the "not tracked" column will be wrong in the brand's favour.

Only keywords configured in a rank project come back. `ownership` is `OWN`
or `EXTERNAL` (a competitor ASIN tracked against the same keyword).

## 5. Gotchas in the SQP rows

1. **The same ASIN appears under several `product_id`/`sku` rows** — FBA and
   FBM SKUs of one ASIN. Deduplicate on `(week_start, asin, query)` and keep
   the larger row. Summing them double-counts purchases.
2. **`title` is often null on SQP rows.** Join titles from
   `get_product_performance` by ASIN; never rely on the SQP title.
3. **`volume` is market-level weekly search volume**, not the brand's. It is
   the same number on every ASIN row for that query — take the max, never
   the sum.
4. **`score` is undocumented and does not track relevance.** On the account
   this was built against, an irrelevant query scored 7 while the brand's
   single best converting term scored 20. Do not rank on it.
5. **Shares are percentages of a market total.** Recompute
   `asin_impressions / market_impressions` when aggregating; never average
   the share columns.
6. **Purchase counts may be summed across weeks.** They are transaction
   counts, not unique users. Volume and impressions may be summed too;
   shares may not.

## 6. Large results

The per-week `by_product` pull is around 1,000 rows and will usually exceed
what one tool result carries. When the runtime spills it to a file,
aggregate with a short script. With no filesystem, drop to the single most
recent week and say on the report that one week was used.
