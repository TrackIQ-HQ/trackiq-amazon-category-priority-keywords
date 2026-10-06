# Method

Three jobs: put ASINs in categories, pick the 25 terms per category worth
tracking, and price each term.

---

## 1. ASINs into categories

The account's own category names are the target taxonomy — take them from
`get_product_categories_performance`. The mapping from ASIN to category is
not exposed by any tool, so derive it from product titles.

1. **Scope to revenue-bearing ASINs.** Drop anything with zero revenue in
   the month. A catalogue of 60 rows is typically 25 real products plus FBM
   duplicates, null-ASIN rows and dead listings. They cannot carry a revenue
   allocation and they pollute the category counts.
2. **Write ordered title rules,** most specific first. Order matters:
   a rule for `48ft` must run before a rule for `4ft`, and a form rule
   (`cover`, `cushion`, `replacement bulb`) must run before a size rule, or
   a replacement bulb lands in a size category.
3. **Map each rule to one of the account's category names.** Where the
   account has no matching category, create one and flag it as derived.
4. **Every ASIN lands somewhere.** Anything unmatched goes to
   `Uncategorised` and appears in the report. Silently dropping an ASIN
   loses its revenue from the denominator.
5. **Reconcile.** Category revenues must sum to the account's total monthly
   revenue. If they do not, an ASIN was dropped or double-counted.

Print the ASIN-to-category table and the reconciliation before going on.

## 2. Priority search terms

For each category, aggregate the per-week SQP rows of its ASINs by query:
sum `asin_purchases`, `asin_impressions`, `asin_clicks`; take the max of
`volume`; recompute impression share as
`sum(asin_impressions) / sum(market_impressions)`.

### Gates — a term must pass all three

1. **Volume floor.** `volume >= 250` a week. Below that the list fills with
   typos and one-off long tail: "patio ligths", volume 1, is not a term to
   track organic rank on.
2. **Presence.** At least one purchase in the observed weeks, **or**
   impression share ≥ 1%. Either the brand converts on it or it is visibly
   in the results.
3. **Relevance — read the query and judge it.** The quantitative gates
   cannot do this. A query can carry 107,000 weekly searches, nine purchases
   and still be "outdoor wedding decor". A marketplace-mechanic query like "prime
   deals sale", "labor day sale" or "lightning deal today" is demand for
   a promotion, not for the product.

   **Keep** competitor and substitute brand terms — "northpeak grill cover",
   "lumengarden string lights". Those are exactly the searches a category
   needs to rank on. **Drop** queries for a different product category, generic
   marketplace events, and anything whose match is coincidental.

   Record every term dropped at this gate and why. The exclusion list goes
   in the working notes so the next run does not re-litigate it.

### Ranking and labelling

Rank the survivors by estimated monthly revenue, descending, then by volume.
Take the top 25. If fewer than 25 pass, take what passes and say so — a
category with nine real terms gets nine rows, not nine plus padding.

Label each against that category's medians:

| Label | Rule | What it means |
|---|---|---|
| **Defend** | converts, and impression share ≥ category median | already won; losing rank costs money now |
| **Grow** | volume ≥ category median, impression share < median | the opportunity — rank movement is worth buying |
| **Watch** | everything else that passed | small but real; on the list so a change is visible |

Then mark each term tracked or not tracked against the rank-tracker keyword
list, matching on exact lowercase text. **The untracked count is the point of
the report** — near-misses matter, so "waterproof grill cover xl" is not
covered by "grill cover xl" being tracked.

## 3. Revenue per keyword

SQP reports purchases per query. It does not report revenue. Revenue is
derived, and the report says so.

```
asp(category)          = category revenue / category orders
observed_purchases(q)  = sum of asin_purchases across the weeks that exist
scale                  = days_in_month / (7 x weeks_observed)
revenue_per_month(q)   = observed_purchases(q) x scale x asp(category)
```

Three things this assumes, all of which belong on the report:

- **Every order in a category is worth the category average.** False at ASIN
  level — a category holding a 4ft and a 48ft string light overstates the small
  one. Keep categories tight enough that the ASP means something; if a
  category's ASINs differ in price by more than roughly 2x, split it.
- **The observed weeks represent the month.** A promotion or a stockout
  inside a missing week is invisible.
- **Search purchases are a subset of orders.** Summed across every query,
  SQP purchases typically account for only 10–35% of a category's orders;
  the rest is repeat, Subscribe & Save, browse and off-search. So the
  revenue against a keyword is revenue *from that search*, not the ASIN's
  total. State the coverage percentage per category.

**Do not allocate the category's whole revenue across its queries by
purchase share.** It ties out to a satisfying total and is wrong by roughly
4x, because it attributes non-search revenue to search terms.

## 4. Sanity checks

- Category revenues sum to the account's monthly revenue.
- Per ASIN, summed SQP purchases (scaled) ≤ that ASIN's monthly orders. If
  it exceeds, the dedupe in `assets/pulls.md` step 5.1 did not happen.
- Every priority term's revenue is less than its category's revenue.
- The top-25 revenue for a category is a plausible fraction of that
  category — typically 5–25%. Near 100% means the allocation method drifted
  into the share-based version this file warns against.
