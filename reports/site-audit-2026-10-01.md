# User-testing audit of artificialspinice.com, 2026-10-01

## Summary

- **Method.** The live site (`https://artificialspinice.com`) was blocked by the sandbox egress proxy (HTTP 403 on CONNECT), so I served this checkout (commit `b3ba871`, "Last updated October 1, 2026", 925 works, 14,694 links) with `python3 -m http.server` and drove it with headless Chromium through Playwright. axe-core 4.x was used for accessibility. Content is identical to what GitHub Pages serves. Things that depend on the real host (the 404 page on a deep path, HTTP headers, external links, real clipboard/OS behaviour) were not tested. No external link was followed, and no form was submitted.
- **Result.** No high-severity problem was found. There were no console errors or warnings, no failed requests and no HTTP errors in any view, in the sweep of all 925 paper pages, or at 375, 320 and 768 px widths. The site is solid and the data is internally consistent. The findings are mostly accessibility and polish, plus a few navigation and map-usability gaps.
- **Counts.** 0 high, 4 medium, 11 low.
- **Method note.** For the map I used a private copy of `index.html` with a one-line debug hook (`window.__t`), kept in the scratchpad and not in the repo, so I could locate nodes on the canvas and click them with the real mouse. No site file in the repo was changed.

## Findings

### Medium

**M1. The "faint" text colour fails WCAG AA contrast on every view**
- Where: all views. `--faint:#6B7488` in the `:root` block of `index.html` (line 19), used by `.faint`, `.brand .tld`, `.src`, the facet counts `.fopt .c`, ranking meta, footer text and more.
- Steps: open any view (for example `#home`, `#browse`, `#rankings`, `#paper-W3000049529`) and run axe-core `color-contrast`.
- Expected: at least 4.5:1 for small text.
- Actual: `#6B7488` is 4.06:1 on the page background `#0D1016`, 3.75:1 on cards and tables (`--surface #151922`) and 3.44:1 on `--surface-2`. axe flagged 170 elements on the `#0D1016` background and 58 on `#151922` across 9 sampled routes. Examples: the ".com" in the wordmark, "(4 authors)" in the home lists, the facet counts in Browse (90 nodes on `#browse`), the abstract source notes on paper pages, and the table meta in Rankings. A smaller group (9 nodes, `#737A88` on `#0D1016`, 4.41:1) is the timeline `.kick` labels.
- Fix: lighten `--faint`. `#8089A0` gives 5.44:1 on `--bg`, 5.03:1 on `--surface` and 4.61:1 on `--surface-2`. Also check the timeline `.kick` colour (`#737A88`).

**M2. The map cannot be used from the keyboard, and Enter does nothing in the map search**
- Where: `#map`, `#mapCanvas` (the `cv` handlers in `index.html`, around lines 1021 to 1070), and `#mapSearch` (around line 1150).
- Steps: (a) Tab through `#map`. Focus goes through the controls, then `+`, `−`, `Fit`, and then leaves the map. (b) Type "Heyderman" in the map search and press Enter.
- Expected: some keyboard way to pick a paper, and Enter on a search box selecting the first hit, as the header search does.
- Actual: (a) the canvas has an `aria-label` but no `tabindex`, no role and no key handlers, so hover, select, Local graph and details are mouse- or touch-only unless the user searches. (b) After Enter nothing was selected and `#mapInfo` stayed hidden. Only a click, or Tab onto a result button and Enter, selects. Escape and clicking outside also do not close `#mapSearchRes` (read from the code: only a click handler on the results exists).
- Fix: add a `keydown` handler on `#mapSearch` (Enter → `focusNode` of the first hit, Escape → hide, plus a click-outside close, like `gq`). Give the canvas `tabindex="0"` with arrow or `n`/`p` keys that move through the visible nodes (or point to the search as the keyboard route in the page text). Also add `role="img"` or `role="application"` with a short description.

**M3. The browser tab title is the same on every view and every paper**
- Where: all routes. `document.title` is always `artificialspinice.com`. `show()` and `renderPaper()` never update it.
- Steps: open `#paper-W1981179226`, then `#browse`, then check the tab title, history list or a bookmark.
- Expected: for example "Wang 2006, Artificial 'spin ice' in a … · artificialspinice.com", "Rankings · …". WCAG 2.4.2 also asks for a descriptive page title.
- Actual: always the bare site name. Researchers who bookmark papers or look through history see a long list of identical entries. Screen-reader users get no indication of a view change.
- Fix: in `show(v)` set `document.title`: the paper's title/label for `paper`, else the view name. Optionally move focus to the view's `<h1>`.

**M4. On a phone the map is about 1,500 px down the page, behind all the controls; swiping on the map does not scroll the page**
- Where: `#map` at 375×667 (and, less so, 768×1024).
- Steps: open `#map` on a 375 px viewport.
- Expected: the map is visible quickly, or the controls are collapsed.
- Actual: the whole 1,165 px controls panel comes first. The map starts at about y=1646 (768 px wide: y=1484), so the first screen shows only the intro and the first controls. `#mapCanvas` has `touch-action:none` (computed style checked), so a swipe that starts on the 426 px tall map pans or zooms the graph and cannot scroll the page. A tap on a node does work: it opened the info panel, which appears below the map, partly off-screen. I did not test pinch zoom or drag on a real device.
- Fix: in the narrow layout put the map first and the controls in a collapsible `<details>`, or add a "Jump to map" link. Show the info panel as a bottom sheet. Consider `touch-action:pan-y` plus two-finger pan, or an obvious exit from the map.

### Low

**L1. Unknown or malformed hashes silently show Home, with no message**
- Where: `route()` and `show()` in `index.html` (around lines 798 to 815).
- Steps: open `#paper-W999`, `#paper-W1981179226x`, `#paper-w1981179226` (lower case), `#PAPER-W1981179226`, `#bogus` or `#map?x=1`.
- Expected: either a "Paper not found / this record may have been merged or removed" message, or normalising the hash.
- Actual: the Home view is shown, the address bar keeps the bad hash. A shared link to a record that is later dropped will look like a working but wrong page. Old merged ids do redirect correctly (checked `W2729679054` and `W2967459249` → `#paper-W1981179226`, and all 143 aliases in the sweep).
- Fix: in `route()` return a "not found" view (or a banner on Home) for any `paper-…` id that is neither a node nor an alias. Match the hash case-insensitively.

**L2. Header search: Enter jumps to the top paper, no DOI search, short and symbol queries**
- Where: `gsearch()` and the `gq` keydown handler (around lines 762 to 790), `fold()`/`matchQ()`.
- Steps and observations (typed into `#gq`):
  - "Schiffer" then Enter opens the first paper (Wang 2006), not the list of the author's papers. The list is only reached by clicking "All N matches in Browse →".
  - A DOI (`10.1038/nature04447`) returns "Nothing matches". The DOI is not part of the search text (`n.sx`).
  - "ASI" returns only 3 papers, because the match is by word prefix on title/author/venue text. The abbreviation is the site's own term for the field.
  - Single-letter tokens match widely: "a&b" returns 11 results (any word starting with "a" and "b").
  - Queries made only of symbols or Greek/µ/³ characters (for example `γ`, `µ`, `³`) fold to nothing, the dropdown is hidden and no "nothing matches" message appears (for `γ` the hidden list still held the previous query's results).
- Working correctly: single and multi-word queries, extra spaces, quotes and punctuation, diacritic folding (Néel/Neel, Skjærvø/Skjaervo, Müller/Muller), regex characters (`( [ \ * + ? .*`), `<script>` (no injection, shows "Nothing matches"), a 200-character string, years and journal names, `/` to focus, Escape to close.
- Fix: Enter with several matches could go to Browse with the query (the single-result case can still open the paper). Add the DOI to `n.sx`. Add "asi" as an alias of "artificial spin ice" (or document it). Consider a minimum token length of 2 for non-final tokens.

**L3. Two records have no author, which leaves a blank authors line and a dangling separator**
- Where: `W7202126207` (PRB 2026) and `W4211160221` (Elsevier book chapter "Synthesis and processing"). `metaLine()` and `renderPaper()`.
- Steps: open the paper pages, or search "Current-induced magnetization control" in Browse.
- Actual: the author line on the paper page is empty. In Browse and Rankings the meta line starts with " · Physical Review B · 2026".
- Fix: skip the separator when `fa` is empty and show "Authors not listed" (or similar). The data itself (`na` = 0, no `fa`) is probably an upstream gap.

**L4. The "Free full text" button shows a garbled host name for 35 papers**
- Where: `accessButtons()` and `access.js` (`f.host`). Example: `#paper-W3000049529`.
- Actual: the button reads "Free full text · Physical review. B./Physical review. B ↗" (the OpenAlex host display name for the publisher repository). 35 entries have a host of this `X./X` form.
- Fix: normalise `host` (keep the part before `/`, or map it to "Physical Review B") in the generator, or in `accessButtons()`.

**L5. A few records look off-topic (please check)**
- Where: Browse / Map. These have no ASI-style words in the title and few or no links, so they may be search false positives: `W2614060977` "Newly launched activity in JMMM – Outreach to the General Public" (editorial, 0 citations, no references); `W4414874592` "Orthogonal projections of hypercubes"; `W2185095381` "Effective relativistic quantum mechanics on a causal net"; `W3186462942` "Studies of Dielectric Originated Matter Waves"; `W4318998224` "The 2022 applied physics by pioneering women: a roadmap"; `W3202958030` "Numerical Studies of Vortex Dynamics in Superconductivity"; `W4211160221` "Synthesis and processing"; `W4200349153` "Scientific Background" (a Springer Theses chapter).
- I cannot tell from the page whether they truly belong (some may cite ASI work). If not, they inflate the "works" count and the unlinked-papers count (108).
- Fix: review them in the generator's inclusion rule, or add a "Not about ASI" flag.

**L6. Keyboard and screen-reader gaps (axe-core)**
- Where and what:
  - No "skip to main content" link: 8 header links plus the search box come before the content on every view.
  - The segmented controls (`.seg` buttons: Layout, Color by, Rankings metric, Stats period, Depth 1/2) show their state only through the `.on` class. There is no `aria-pressed`, so a screen reader cannot tell which one is active.
  - `landmark-unique`: the header search and the home search form (and Browse's `.fsearch`) are all `role="search"` with no distinct labels.
  - `region`: the `.mockbar` strip is outside any landmark.
  - `heading-order` on Browse: the facet group titles are `<h5>` directly after the page `<h1>`.
  - `scrollable-region-focusable` on `#timeline` at 375 px: `.chartbox` (`overflow-x:auto`) scrolls horizontally but is not reachable by keyboard (add `tabindex="0"` and a label).
  - The header search results `#gres` have no listbox roles or arrow-key navigation (Tab does reach the links).
- Working correctly: every control I tabbed through had a visible 2 px focus ring (`:focus-visible`), all inputs have labels, the lightbox is a `role="dialog"` with `aria-modal` and closes with Escape, the card toggle uses `aria-expanded`, and the timeline chart has `role="img"` with an `aria-label`.
- Fix: add the skip link and the `aria-pressed` toggling in `segPick()` and the rank/stats `seg` handlers, give each `role="search"` an `aria-label`, wrap the mock bar in a landmark, and change `<h5>` to `<h2>`/`<h3>` styled the same.

**L7. At common laptop heights the bottom of the map panel is below the fold**
- Where: `#map`. At 1280×900 `.map-wrap` ends at y≈972, so the bottom 70 px of the map (including the lowest map rows) is hidden until the page is scrolled. At 1440×800 it is 94 px over. The map and controls are sized by content, not by the viewport.
- Fix: `height:calc(100vh - header - page-head)` with a minimum, or a shorter intro on the map view.

**L8. Local graph and dense areas have overlapping labels**
- Where: `#map`, Local graph of `W1981179226` (Wang 2006), depth 1 (565 papers) and depth 2 (786). Labels near the centre pile up ("Farhan 2013", "Gilbert 2016", "Mengotti 2011", "Kapaklis 2014" overlap and are unreadable).
- Fix: a lower label cap in local mode, or a collision check in `drawGraph()`.

**L9. Small wording points**
- The info panel and local view print "1 ASI papers it cites" (singular/plural, `renderInfo()`).
- Thesis author names sometimes appear surname-first: "Chaurasiya Avinash" on the thesis, while the same author is "Avinash Chaurasiya" on the papers.

**L10. Papers with no links still draw an empty citation neighbourhood**
- Where: paper pages with 0 references and 0 citations, for example `#paper-W2767623558`, `#paper-W2614060977`, `#paper-W2934105858`: a 420 px tall mini-map with a single dot and the zoom buttons, followed by two "0" boxes. Many of the 108 unlinked records are like this.
- Fix: hide `#miniMap` when `OUT` and `IN` are both empty, and show only the explanatory text.

**L11. Mobile tap targets**
- Where: 375 px. Between 2 and 11 interactive elements per view are smaller than 32 px in one dimension (for example Browse facet checkboxes and "Show all N" buttons, the map's `+`/`−` buttons). Not a blocker; I did not measure against the 44 px guideline view by view.

## What I tested and found working

- **Views.** `#home`, `#map`, `#browse`, `#rankings`, `#timeline`, `#stats`, `#disclaimer`, the 404 page source and `robots.txt`. Footer, "Live" bar and last-updated line are consistent with `README.md` (925 works, 14,694 links, 61 theses, 46 reviews, 498 cards).
- **Console and network.** No console errors or warnings, no failed or 4xx requests, in every view, in the sweep, and at all viewport sizes.
- **All 925 paper pages**, plus all 143 aliases in `ALIASES`: each page renders the right title, has no `undefined`/`NaN`/`null` text, every internal `#paper-…` link resolves, and every old id redirects to the kept paper with the address bar corrected. Data checks on the files: no duplicate DOIs or titles, no out-of-range or self-referencing edges, every `FIGMETA` entry has a matching `figs/*.json` (130 each), every alias target exists, every `THESIS_LINKS` pair matches the thesis author's surname (108 links, 41 theses).
- **38 hand-picked paper pages** read in full or in structure: theses with and without paper links, papers with the "Thesis of the first author" box, articles, reviews, review book chapters, book chapters, conference papers and abstracts, preprints, a report, merged records ("Also recorded as"), papers with and without a card, with figures (lightbox, arrows, Escape, focus handling), with no abstract, with no free version, with a corresponding author's "Show email" button, zero-link records, and an old alias id. The Copy DOI button and the footer "Copy address" button put the right text on the clipboard and reset their label.
- **Header search**: see L2 for what worked. Results link to the right pages.
- **Browse**: all 111 facet checkboxes tested one by one, each result count equals its facet count; text plus facet combinations; "Show all", "Show more" (25 → 50), the five sort orders (order checked for the top results), year range (an inverted range is corrected to 2010–2020), "Clear all filters", empty result message, Back restoring the query, jump from the Stats journal bars with the journal chip and its × button, and the home-page search form.
- **Rankings**: 48 combinations (4 types × 3 year filters × 4 metrics): every list is correctly sorted and respects the year filter, and no list is empty (smallest 18 rows).
- **Timeline**: chart, milestone diamonds, 18 timeline entries and 15 paper links. **Statistics**: 3 period buttons, the journal chart, the table (117 rows), and the totals (668 journal papers + other groups = 925).
- **Map**: Network and By year layouts, "Rows by type" on and off, the four colour modes and their legends, the three size options, the year slider (2006 → 8 papers, 2010 → 50), Play/Pause, the minimum-citations slider (10 → 280 papers, 40 → 106), citation and thesis link toggles (counts updated), "Show unlinked papers" (817 → 925), hover tooltip, click for the info panel, the "Most-cited papers citing it" list, Local graph depth 1 and 2 (565 and 786 papers) and Exit, double-click opening the paper page, zoom +, − and Fit (scale 0.55 → 0.82 → 0.36 → 0.54), the map search by clicking a result, and the "No paper matches" message. Tap on a node on a touch device.
- **Paper-page mini-map**: hover tooltip, click navigation, +/−/Fit buttons.
- **Responsive**: 320, 375 (touch emulation), 768 and the desktop sizes 1024, 1280, 1440 and 1920: no horizontal page scroll on any view.
- **Keyboard**: full Tab order on Home, Map, Browse and Statistics; every stop had a visible focus ring; `/` focuses the search; Escape closes the dropdown.
- **History**: Back/Forward between papers and views; scroll resets to top on a view change.

## Not tested

- The live domain, HTTP headers/caching and the GitHub Pages 404 behaviour.
- External links (DOI, arXiv, repository URLs), and whether the "free" copies still resolve.
- Real-device touch gestures (pinch, drag), real screen readers, and Firefox/Safari.
- The scientific correctness of paper cards and quotes.
