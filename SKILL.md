---
name: trackiq-amazon-category-priority-keywords
description: Groups an Amazon brand's ASINs into product categories and, from the Search Query Performance report, picks the 25 priority search terms per category worth tracking organic rank on — with each term's weekly search volume, estimated monthly revenue, impression share, and whether it is already in the rank tracker. Use when the user asks for priority keywords, priority search terms, a keyword tracking list, which keywords to track organic rank for, SQP analysis, search terms by product category, revenue by keyword, keyword revenue, or a rank-tracking gap check.
---

# TrackIQ: Amazon Category Priority Keywords

Answers one question per category: **which search terms should we be
tracking organic rank on, and what is each one worth?** Output is a branded
HTML report — a category summary, then 25 ranked terms per category with
revenue, and a list of the ones the rank tracker is missing.

The last column is usually the finding. Rank trackers drift toward brand
terms while the revenue sits in generic head terms nobody added.

## Requires

- The TrackIQ MCP, for `list_marketplaces`,
  `get_search_query_performance`, `get_product_performance`,
  `get_product_categories_performance` and `get_keyword_rank`. Ask which
  brand and marketplace before pulling anything.
- The account must have **SQP ingestion running**. It is a separate weekly
  feed and is often sparse or absent on newer brands — check before
  promising a report.
- Nothing else. No filesystem, no shell, no internet.
- **Without the MCP:** ask for the Brand Analytics Search Query Performance
  export (ASIN view, one file per week), plus revenue and orders by ASIN for
  the month. Everything except the rank-tracker column works from those.

## First run

Fill in a copy of `assets/account.example.md` saved as account.md beside the
skill. Every TrackIQ skill reads the same file, so an account already set up
for the daily or weekly send needs nothing added here.

1. **Brand and marketplace** — which account and catalogue to pull
2. **Product lines** — the account's own category names, if they differ from
   what `get_product_categories_performance` returns
3. **Delivery** — in-chat, file, Slack, n8n or email

If the runtime has no filesystem, print the same block and ask the user to
paste it into their project instructions once.

## Read first

- `assets/pulls.md` — the call sequence, the week-discovery step, and the
  six ways these rows mislead. Read before the first tool call.
- `assets/method.md` — categorisation, the three gates, the labels, and the
  revenue derivation
- `assets/checks.md` — the data, judgement, honesty and render checks
- `assets/account.example.md` — the first-run answers, filled in once

Copy `assets/report-template.html` and replace every `{{TOKEN}}`, repeating
the category section per category. Do not restyle it.

## Delivery

The report is produced in the chat first. Delivery is the last step and the
method comes from the Delivery block in account.md — never ask per run.

| Method | What to do | Needs |
|---|---|---|
| `in-chat` | Return the HTML. The default, and the fallback for every other method. | nothing |
| `file` | Write it beside the skill as `priority-keywords-<YYYY-MM-DD>.html`. | a filesystem |
| `slack` | Post the per-category untracked counts as text, then upload the HTML. Slack will not render the report inline. | a connected Slack tool |
| `n8n` | POST the HTML to the configured webhook, `Content-Type: text/html`. | network access |
| `email` | Hand it to the connected mail tool. | a connected mail tool |

Confirm before the first outward send of a session, fall back to in-chat
loudly when a method is unavailable, and never substitute a different
outward channel. Naming Slack or n8n here is configuration, not report
content — the rendered report names no platform but TrackIQ and Amazon.

## Non-negotiables

1. **Discover which SQP weeks exist before using any of them.** The feed is
   intermittent and a missing week returns `{"rows": []}`, not an error. One
   account had two of August's four weeks missing. Probe each week, report
   which were present and which were absent, and scale by the weeks actually
   observed. Assuming four weeks silently halves every revenue figure.
2. **Revenue per keyword is derived and must be labelled as an estimate.**
   SQP reports purchases, never revenue. It is `purchases × scale ×
   category ASP`, and the report states all three assumptions behind it.
3. **Never allocate a category's whole revenue across its queries by
   purchase share.** It ties out to a clean total and overstates every
   keyword by roughly 4x, because search accounts for only 10–35% of orders.
   Show that coverage percentage per category.
4. **Relevance is a judgement the model makes, and it is recorded.** No
   metric separates a category's head term from a high-volume coincidence —
   "outdoor wedding decor" can carry 107,000 searches and nine purchases. Read every
   shortlisted query. Keep competitor brand terms; drop other-category and
   marketplace-event queries; write down what was dropped and why.
5. **Deduplicate on `(week, asin, query)`.** One ASIN appears under several
   SKU rows — FBA and FBM — and summing them inflates purchases and
   therefore revenue.
6. **Verify the rank-tracker list is complete before reporting a gap.** The
   tracker pull returns one row per keyword × ASIN × day and truncates
   silently. An incomplete list overstates the gap in the brand's favour,
   which is the one direction a client will notice.
7. **Never rank on the `score` field.** It is undocumented and does not
   track relevance — an irrelevant query scored 7 while the best converting
   term scored 20.
8. **Categories are derived, and the report says so.** No tool exposes which
   ASIN belongs to which category. Map titles onto the account's own
   category names, put unmatched ASINs in `Uncategorised` rather than
   dropping them, and reconcile category revenue to account revenue.
9. **Flag duplicate and empty category definitions** found in the account
   rather than quietly picking one. Two categories reporting identical
   figures is a data-hygiene finding the client should act on.
10. **Fewer than 25 qualifying terms means fewer than 25 rows.** Never pad a
    category to reach the number.
11. **Never print `account_id`.** Refer to the account by name or as "your
    US Seller account".

## The actionable output

The report is the artifact; the keyword list is the action. End by offering
the untracked priority terms as a plain list grouped by category, ready to
paste into a rank project. That is what the client does next, and it is what
makes the next run's tracking column improve.

## Relationship to the other TrackIQ skills

`rank-readiness` and the rank sections of the daily and weekly reports tell
you how you rank on the keywords already tracked. This one decides **which
keywords should be tracked in the first place**, and prices them. Run it
when a brand is onboarded, when the catalogue changes, and quarterly after
that — not weekly. The list should be stable between runs; if it is not,
something changed in the catalogue or the category rules.

## Version

`trackiq-amazon-category-priority-keywords` v1.0.0 (2026-09-17).

If the user asks whether this skill is current, fetch
`https://trackiq.com/skills/registry.json`, compare the `version` field for
`trackiq-amazon-category-priority-keywords`, and if it is newer, give them the
download link and the one-line changelog. Do not fetch at any other time.
