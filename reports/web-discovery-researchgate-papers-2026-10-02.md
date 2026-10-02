# Web discovery: ResearchGate papers (2026-10-02)

- Queries run: 47. The first 8 used `site:researchgate.net` and returned no ResearchGate pages. The other 39 used a domain filter on researchgate.net.
- RG publication records seen: about 116. 96 were already on the site (matched by title, or by OpenAlex id or DOI when the RG title differed). That is the coverage measure.
- Records not on the site: 19.
  - By type: 16 articles, 2 preprints, 1 other (a commentary).
  - By relevance: 6 yes, 7 maybe, 6 no. The "no" rows are off-topic items that the searches returned.

## Most notable finds (yes)
- Pisanty et al. 2021, "Putting a spin on metamaterials: Mechanical incompatibility as magnetic frustration" (W3120521210).
- Chern, Reichhardt and Reichhardt 2013, "Frustrated colloidal ordering and fully packed loops in arrays of optical traps" (W2073250876).
- ASI tutorial (2025, arXiv 2504.06548) with design, magnonics and neuromorphic computing. Not in OpenAlex.
- "Snakes in the Plane: Controllable Gliders in a Nanomagnetic Metamaterial". Likely the preprint of the on-site gliders paper. Not in OpenAlex under this title.
- "Freezing and melting of vortex ice". Not in OpenAlex.
- "Artificial spin ice: The unhappy wanderer", a commentary. Not in OpenAlex.

## Could not be checked
- The search engine returns ResearchGate pages only with a domain filter, and only about 9 per query. Coverage of this slice is therefore partial.
- I did not fetch ResearchGate pages. Authors, venues and years come from search-result titles and snippets only.
- Five finds have `openalex_id` null because OpenAlex title searches found nothing. For those, `first_author` is null, and the year is null for two of them.
- The preprint and book-chapter versions of Nomura et al. (W2899296163, W4233348479) exist in OpenAlex but are not listed separately.
