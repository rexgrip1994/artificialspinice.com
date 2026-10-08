# artificialspinice.com

**A searchable map of the research literature on artificial spin ice:** the papers, theses and reviews of the
field, how they cite each other, and what each paper actually did.

**→ [artificialspinice.com](https://artificialspinice.com)**

| | |
|---|---|
| Works | 925 (incl. 61 theses and 52 reviews) |
| Citation links within the field | 14,694 |
| Paper cards | 578 |
| Last updated | October 8, 2026 (refreshed weekly) |

This repository holds the **published, generated website** only. It is rebuilt automatically, so please do not
edit files here or open pull requests. Corrections, missing papers and questions are very welcome by email:
**info@artificialspinice.com**.

## What you can do on the site

- **Search** every work by title, author, journal or year (press `/` anywhere).
- **Explore the citation map:** papers as nodes and citations as lines. Use the network or by-year layout, colour
  by topic, lattice, year or open access, hover a paper to see its neighbours, and open a local graph around any
  paper. Theses (triangles) and reviews and book chapters (hexagons) are drawn in their own shapes; a dashed line
  joins a thesis to the papers of its author.
- **Rankings** by citations from within the field (the default), per year since publication, recent attention
  (cited by papers of the last three years) or global citations, with separate lists for reviews and theses.
- **Timeline** of the field and **Statistics** (e.g. where the work is published).
- **Paper pages:** full author list, the official abstract, every free version found (publisher open access,
  arXiv, repository copies), supplementary information, figures from openly licensed papers, the citation
  neighbourhood, and a **paper card**.

## Paper cards

A paper card records what a paper did: lattice, material, island geometry, fabrication, state preparation,
measurement techniques, simulation methods, observables and the main result. Every value is backed by a
one-sentence quote from the paper, with its page. Cards are drafted and reviewed by AI from the paper's text, and
each quote is checked word for word against that text before it is published. Values marked *AI-reviewed* have
not yet been checked by a person.

## Where the data comes from

- **[OpenAlex](https://openalex.org)**: the works and most citation links (CC0; Priem, Piwowar & Orr,
  [arXiv:2205.01833](https://arxiv.org/abs/2205.01833)). Thank you, OpenAlex.
- **[Crossref](https://www.crossref.org)**: publication dates and publisher-deposited reference lists, which
  fill citation links OpenAlex lacks.
- **[arXiv](https://arxiv.org)**: preprints and their abstracts (metadata CC0). ORCID and Semantic Scholar help
  find theses and free versions.
- **The papers' own full texts** (open copies, read privately, never hosted here) for paper cards and for
  citations that no database lists.
- New papers are found every week and added only after the curator has reviewed them.

## Licences and reuse

- No PDFs are hosted. The site links to every free version it finds.
- Figures are shown only from papers under an open licence (CC BY, BY-SA, BY-NC, BY-NC-SA, CC0), with credit,
  licence and a note that they are resized. Other papers link to where their figures can be seen.
- Abstracts other than arXiv's belong to the publishers or authors and are shown to identify the paper.
- Corresponding-author emails appear as printed in the papers. Authors can ask for removal by email.
- Third-party components: IBM Plex and STIX Two fonts (SIL Open Font License, in `assets/fonts`) and
  [d3](https://d3js.org) (ISC licence).

## Disclaimer

artificialspinice.com was vibe-coded by Ondrej Brunn, with the help of Anthropic's Claude Opus 5.5. Much of the
content, from paper classifications to statistics, was produced or assisted by AI, and AI makes mistakes. Please
treat it as a starting point and check the original papers before relying on it.

## What is in this repository

| File | Content |
|---|---|
| `index.html` | the whole web app (HTML, CSS and JavaScript in one file) |
| `data.js` | works and citation links |
| `authors.js`, `versions.js` | author lists; merged records (preprint and journal versions of one work) |
| `access.js` | free versions, arXiv identifiers and abstracts |
| `cards.js`, `figs/`, `figmeta.js` | paper cards; openly licensed figures and their credits |
| `theses.js`, `contacts.js` | thesis ↔ paper links; corresponding-author contacts |
| `assets/` | self-hosted fonts and d3 |

Status: public beta. Contact: **info@artificialspinice.com**
