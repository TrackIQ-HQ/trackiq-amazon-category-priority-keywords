# Before you send it

## 1. The data checks

- **Weeks.** The report names which SQP weeks exist and which are missing.
  If every week of the month returned data, say that explicitly — a reader
  cannot tell "all four weeks" from "nobody checked".
- **Completeness.** The last row of each per-week pull has
  `asin_purchases = 0`. If not, the limit truncated purchase-bearing rows.
- **Dedupe.** Distinct `(week, asin, query)` keys equal the row count kept.
- **Reconciliation.** Category revenues sum to the account's monthly
  revenue, to the dollar.
- **Purchases ≤ orders.** For every ASIN, scaled SQP purchases do not exceed
  that ASIN's monthly orders.
- **Tracker list is complete.** The rank-tracker pull did not hit its limit.
  An incomplete list overstates the tracking gap, which is the report's
  headline — get this one right or drop the column.

## 2. The judgement checks

- Every term in every table would be recognised by the brand as a search for
  that product. Read the lists. One "outdoor wedding decor" in a client-facing table
  costs more credibility than it saves in effort.
- Competitor brand terms are present, not filtered out.
- Terms dropped at the relevance gate are recorded with a reason.
- No category has padded rows to reach 25.

## 3. The honesty checks

- Revenue per keyword is labelled an estimate wherever it appears, and the
  method section states the ASP assumption, the scaling assumption and the
  search-coverage assumption.
- Coverage percentage is shown per category, so nobody reads keyword revenue
  as the ASIN's total revenue.
- If a category's ASINs span more than roughly 2x in price, either it was
  split or the report says the ASP is unreliable for it.
- The category caveat names any duplicate or empty category definitions found
  in the account.

## 4. The render check

Open the HTML and confirm:

```js
({ overflows: document.documentElement.scrollWidth > window.innerWidth,
   tables: document.querySelectorAll('table').length,
   rows: [...document.querySelectorAll('table')].map(t => t.querySelectorAll('tbody tr').length),
   chips: document.querySelectorAll('.chip').length,
   untracked: document.querySelectorAll('td.no').length,
   logos: [...document.images].map(i => i.naturalWidth > 0) })
```

- `overflows` is false.
- `tables` is categories + 1 (the summary).
- `rows` matches the term counts you computed.
- `chips` equals the total number of priority terms.
- `untracked` equals the untracked count in the headline and the KPI tile.
- `logos` is all true — the images are base64 data URIs, so a false here
  means the template was edited badly.

Then look at it. If the page will not paint (a hidden or backgrounded
window), say the verification was structural rather than visual — do not
claim to have inspected something you did not see.

## 5. Ship

Save as `<client>-category-priority-keywords-<month>-<year>.html`. Logos are
embedded, so the file travels alone.

The rank-tracker keyword list is the actionable output. Offer it as a plain
list of the untracked terms, grouped by category, ready to paste into a rank
project — that is the thing the client does next.
