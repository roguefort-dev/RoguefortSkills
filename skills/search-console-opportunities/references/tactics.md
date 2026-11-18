# Tactic playbook

S is the fastest starting point for existing pages. A covers larger investments. B depends on site and audience. C covers maintenance. The tiers express work order and effort, and results depend on the site.

For the estimates below, divide by 3 when the export covers three months. For another verified window, divide by its length in months and state the conversion. Use `max(0, reference CTR − current CTR)` for incremental-click scenarios and round only for display.

## S tier

1. **Question headings and answers.** Find queries starting with how, what, why, when, can, is, does, should, or which (adapt for the query language). Keep those below the median CTR of the eligible query set. Return the top 15 by impressions with query, likely or confirmed page, impressions, average position, exact H2, and a 40–60 word answer starting with a direct sentence. Use supplied facts and mark any gap. Add an FAQ where multiple distinct questions help. Check current eligibility before suggesting FAQ rich-result markup, and match any markup to visible content. [Google FAQPage documentation](https://developers.google.com/search/docs/appearance/structured-data/faqpage)
2. **Titles, H1s, and descriptions.** Round average position to the nearest integer (half up) and use positions 1–5. Use the median eligible CTR in each position bucket. For buckets under five rows, fall back to the band 1–3 or 4–5 as applicable. If the band still has fewer than five, report insufficient data for that comparison. Flag queries with CTR below half their reference median. Return query, page, impressions, CTR, median and cohort size, two title options under 60 characters, a description under 155, and potential extra monthly clicks using the formula above and fixed impressions. When page content is available, flag duplicate metadata and unclear main headings, and suggest a specific H1 if needed.
3. **Add a section for positions 8–20.** Use unrounded average positions 8–20 inclusive. Exclude one-word searches using the query language's word boundaries. Return the top 15 by impressions with query, page, impressions, position, a heading, and two sentences for that page. Use the site's median eligible CTR at unrounded positions 3–5 for a conditional monthly click estimate at top-five CTR and fixed impressions. State the cohort size and flag sparse benchmarks.
4. **Review commercial landing pages.** Select up to ten sales or enquiry pages by page impressions. For each, check in order and report the first verified problem among overlapping intent with another of the site's pages, wrong page type for the intent, intent missing or unclear in the title, H1, or opening copy, and insufficient coverage. Give page, main query with match status, impressions and position with their dimension, finding, and a one-line fix. Name checks blocked by missing page content or query × page data.
5. **Refresh declining pages.** Require comparable current and prior period data. If missing, stop this tactic and obtain the comparison. Select pages with at least ten prior clicks and a relative drop of at least 30%. Prioritize refresh review if average position worsened by at least one point. If position held or improved and impressions fell, investigate demand or query mix before refreshing. Keep remaining cases open for diagnosis. Report both periods' clicks, impressions and position, the click change, and disposition. Detail the first three refreshes, ordered by clicks lost, using verified page content and queries where available.
6. **Add descriptive internal links.** Use the top ten pages by clicks and the user's commercial destination URLs. Exclude self-links and prioritize relevant source pages. Suggest up to three sources per destination, with a contextual sentence and descriptive anchor. Verify the placement against page content when available. For implementation, use a crawlable HTML link with an `href`. [Google's link guidance](https://developers.google.com/search/docs/crawling-indexing/links-crawlable)

## A tier

7. **Publish first-party data or a tool.** Find up to five topics seeking a number, score, price, average, or calculation. Give exported impressions and what data each topic requires. Ask which data exists, one topic at a time, before drafting results.
8. **Build useful comparison pages.** Find pair queries with terms such as *vs*, *or*, *alternative*, *compare*, and *difference*. Group equivalent pairs, count each query once per pair, and distinguish pair queries from generic alternatives intent. Give pair impressions, proposed title, five sections, and a verdict based on known evidence or marked as pending. Recommend a hub matrix and next step where they help. Report when no supported pairs appear.
9. **Prioritize translation.** Use Countries data to shortlist up to five plausible locale opportunities by impressions and show clicks and CTR. Check whether low clicks have a plausible language component. Confirm target languages with country-filtered queries, page data, and user context where available. Suggest up to ten proven page candidates per locale. If using global Pages data, label it as a provisional list and request a country-filtered export to verify. Check reciprocal hreflang links, local copy, and broken placeholders after release. [Google's localized versions guide](https://developers.google.com/search/docs/specialty/international/localized-versions)
10. **Write hub guides.** Identify up to five hub or category pages with high impressions and relatively few clicks within that cohort. Suggest four query-led H2s, five FAQ questions, and internal links for each. Cover essential terms and how to choose. Consider breadcrumb markup where it matches the hierarchy.
11. **Find missing pages.** Group eligible queries by intent and shortlist clusters exceeding 200 exported impressions. Check URL terms for candidate coverage and read the relevant pages or site inventory when available. An absent URL match is a gap hypothesis. Give up to ten topics, sample queries, impressions, proposed URL or title, and page type.
12. **Review potential overlap.** Find topics with two or more plausible existing pages that have impressions. List candidate URLs, page-level impressions (and their sum if relevant, labeled as a page total), and whether to keep both, retarget, or consider consolidation. Check exact queries in the Search Console Pages view and inspect each page's intent before recommending a redirect. Multiple ranking URLs alone are insufficient evidence.

## B tier

- Earn relevant links for pages with strong content and clear audience fit.
- Answer genuine community questions where the audience participates. Be transparent about affiliations.
- For local businesses, check the real name, address, and phone against listings.
- Pick a publishing cadence the site can sustain while keeping material current.
- Use programmatic pages when each page has distinct, useful evidence or functionality, such as its own data table.

## C tier

- Check rendering, speed, and user experience. Choose a rendering approach that supports the site's needs.
- Use supported structured data that matches visible content.
- Submit or verify the sitemap and inspect indexing for important URLs.
- Audit hreflang and check translated pages for broken placeholders and missing strings.

## Skip for Google Search

Skip publishing on third-party sites chiefly to exploit their ranking signals and mass-producing pages chiefly to manipulate rankings. Skip `llms.txt` as a Google Search tactic. [Google Search spam policies](https://developers.google.com/search/docs/essentials/spam-policies) · [Google's generative AI search guide](https://developers.google.com/search/docs/fundamentals/ai-optimization-guide)
