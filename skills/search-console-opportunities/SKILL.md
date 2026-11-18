---
name: search-console-opportunities
description: Turn Google Search Console Search results exports into prioritized SEO actions. Use for CTR and ranking opportunities, page refreshes, internal links, content gaps, comparisons, or localization when the user supplies exports or wants them downloaded through a connected browser extension.
---

# Search Console Opportunities

Analyze one tactic or tier at a time. Start with a feasible S-tier tactic when the user hasn't picked one, and suggest a week of focused implementation. Use the [tactic playbook](references/tactics.md) for selection, inputs, and deliverables.

## Get the data

- Identify the site URL or Search Console property, brand name and aliases, and three to five facts about what the site offers. Ask for missing details only when they affect the chosen tactic.
- Use supplied exports when available. Otherwise use a connected browser extension to export from the user's signed-in Search Console and read the downloaded files. Follow the [export workflow](references/exports.md). Use a three-month window by default and collect only the dimensions needed.
- Record the property, date range, search type, filters, and source files. Search Console UI exports can be capped and omit anonymized queries. Label totals as sums of exported rows when applicable. [Search Console export documentation](https://support.google.com/webmasters/answer/12919797)

## Analyze

1. Use code or a spreadsheet calculation engine for all arithmetic, filters, sorting, and aggregation. Use metrics from the user's exports or equivalent Search Console data. If the environment cannot calculate reliably, ask for a computed extract before reporting numbers.
2. For query analysis, flag brand terms and supplied aliases using case-insensitive substring matches. Confirm the intended matching rule for ambiguous names. Count unique excluded queries and compute their share of exported query impressions before applying any impression floor. Pause query analysis and ask how to proceed if the share exceeds 20%. State the size of the denominator and flag a zero or unavailable denominator.
3. After removing branded queries, retain queries with at least 100 impressions. If fewer than 20 clear that floor, use 20 impressions instead and disclose the fallback. Respect user-specified settings.
4. Use clicks divided by impressions for CTR when both exist and impressions are positive. Treat CTR as a fraction in calculations and display it as a percentage. Use the median of eligible row CTRs where requested. Treat blank or unavailable metrics as missing and report omissions. Avoid unweighted averages of page or country metrics across rows when presenting an aggregate.
5. Keep query, page, and country metrics attached to their own dimensions. Separate export tabs lack a join key. Label page matches inferred from content or URL as **likely page**, with an evidence cue. Use **confirmed** for query-filtered Pages data or equivalent query × page evidence from the same scope. If no plausible match is available, leave it unresolved. [Search Console aggregation](https://support.google.com/webmasters/answer/17011364)

## Report and measure

State the chosen tactic, input scope, filters, and evidence gaps. Give a ranked action table with the requested metrics, candidate URL, copy or next step, and verification. Cite supplied product facts or flag missing inputs for claims and drafts. Recommendations are the default deliverable.

For click estimates, show the reference cohort and formula, identify the estimate as a conditional scenario, and withhold it if the data cannot support the comparison. For follow-up, record the change date and compare equivalent windows and filters for the affected queries or pages. For visibility in Google's generative AI features, check the current Search Console report and its coverage. Use separate evidence for other platforms. [Google's generative AI search guide](https://developers.google.com/search/docs/fundamentals/ai-optimization-guide)

[MIT license](licenses/roguefortskills-MIT.txt)
