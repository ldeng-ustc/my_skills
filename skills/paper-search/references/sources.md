# Sources reference

Capability matrix and identifier formats for every source in the `paper-search` CLI.
Names below are the **exact strings** to pass to `-s` / `download` / `read`. Reflects the
CLI's registered source list (`paper-search sources`).

## Capability matrix

Legend: ✅ reliable · ⚠️ works but upstream-dependent/unstable · ❌ not supported ·
🔑 needs key.

| Source (`-s` name) | Search | Download | Read | Notes |
|--------------------|:------:|:--------:|:----:|-------|
| `arxiv` | ✅ | ✅ | ✅ | Physics/math/CS/quant-bio. Rate-limited ~1 req/3s. |
| `pubmed` | ✅ | ❌ | ❌ | Biomedical citations/abstracts; **no PDF**. `read` returns an info message, not text. |
| `biorxiv` | ✅ | ✅ | ✅ | Biology preprints. **Browses by category within the last N days**, not by keyword. |
| `medrxiv` | ✅ | ✅ | ✅ | Health-sciences preprints. Same category/date model as bioRxiv. |
| `google_scholar` | ⚠️ | ❌ | ❌ | Bot/CAPTCHA blocked; unreliable. Needs proxy env var. |
| `iacr` | ✅ | ✅ | ✅ | Cryptology ePrint Archive. |
| `semantic` | ✅ | ✅ (OA) | ✅ (OA) | Semantic Scholar. Citations, author works, TLDRs. Key raises limits; supports `-y` year filter. |
| `crossref` | ✅ | ❌ | ❌ | 150M+ DOI metadata; info-only. `read` returns a message. Best for DOI → metadata. |
| `openalex` | ✅ | ❌ | ❌ | 250M+ works, all fields; metadata + citation data. `download` raises; `read` returns a message. |
| `pmc` | ✅ | ✅ (OA only) | ✅ (OA only) | PubMed Central OA full text (JATS). Proxy may block direct PDF. |
| `core` | ✅ | ✅ (record-dependent) | ✅ (record-dependent) | Repository OA full text. Free key strongly recommended. |
| `europepmc` | ✅ | ✅ (OA) | ✅ (OA) | PubMed+PMC+preprints in one index; **keyword search over preprints**. |
| `dblp` | ✅ | ❌ | ❌ | CS bibliography; metadata/venues only. Both `download` and `read` raise NotImplementedError. |
| `openaire` | ✅ | ❌ | ❌ | Open research graph; metadata. Both `download`/`read` raise NotImplementedError. Transient 403s auto-retry. |
| `citeseerx` | ⚠️ | ✅ (record-dependent) | ⚠️ | Endpoint intermittently redirects to web archive. |
| `doaj` | ✅ | ⚠️ (URL-dependent) | ⚠️ | Directory of OA journals. |
| `base` | ⚠️ | ✅ (record-dependent) | ✅ (record-dependent) | OAI-PMH needs institutional IP registration; else returns empty. |
| `zenodo` | ✅ | ✅ (record-dependent) | ✅ (record-dependent) | General-purpose OA repository. |
| `hal` | ✅ | ✅ (record-dependent) | ✅ (record-dependent) | French OA archive. |
| `ssrn` | ⚠️ | ⚠️ best-effort | ⚠️ best-effort | Bot-detection; public PDFs only, compliance-first. |
| `unpaywall` | ✅ (DOI lookup) | ❌ | ❌ | OA status + PDF **links** for a DOI. Requires email env var. Both `download`/`read` raise; gives links, not files. |
| `ieee` 🔑 | 🚧 | 🚧 | 🚧 | Skeleton; only registered if `IEEE_API_KEY` set. |
| `acm` 🔑 | 🚧 | 🚧 | 🚧 | Skeleton; only registered if `ACM_API_KEY` set. |

The default install registers **21 sources** (all rows above except `ieee`/`acm`, which
appear only when their key is set). Run `paper-search sources` to see the live list.

> **Not exposed by the CLI:** the package also ships `sci_hub` and `chemrxiv`
> connectors, but neither is registered in the CLI — you cannot select them via `-s`,
> `download`, or `read`. (Sci-Hub is reachable only through the package's MCP server.)

### Practical groupings

- **Full text you can actually fetch:** `arxiv, biorxiv, medrxiv, iacr, pmc, europepmc,
  core, zenodo, hal, semantic (OA), base, citeseerx, ssrn (best-effort)`.
- **Metadata / discovery only (no file):** `pubmed, crossref, openalex, dblp, openaire,
  google_scholar, unpaywall`. Their `search` results still include a `pdf_url`/`url`
  field you can fetch directly when populated.
- **DOI/OA resolvers:** `unpaywall` (needs email), then try `core`, `pmc`, `europepmc`.

### Preprint search caveat

`biorxiv` and `medrxiv` do **not** keyword-search: their `search` treats the query as a
**subject category** (lowercased, spaces→underscores) and browses the last ~30 days via
the bioRxiv/medRxiv details API. To search preprints by topic, use `europepmc` (indexes
both), or `semantic`/`openalex`, then take the resulting `10.1101/...` DOI back to
`biorxiv`/`medrxiv` for preprint-specific metadata/full text.

## Identifier formats

`<paper_id>` for `download`/`read` depends on the source. Safest path: copy the
`paper_id` from a `search` result of the **same** source. Common formats:

| Identifier | Format | Example | Used by |
|------------|--------|---------|---------|
| DOI | `10.xxxx/...` | `10.1038/nature12373` | crossref, openalex, unpaywall, semantic, biorxiv, medrxiv |
| arXiv ID | `YYMM.NNNNN` | `2106.12345`, `1706.03762` | arxiv, semantic |
| PMID | integer | `34567890` | pubmed, europepmc, semantic |
| PMCID | `PMC`+digits | `PMC7029759` | pmc, europepmc |
| OpenAlex ID | `W`+digits | `W2741809807` | openalex |
| S2 prefixed | `DOI:` / `PMID:` / `ARXIV:` | `ARXIV:2103.15348` | semantic |

Cross-referencing: Semantic Scholar accepts DOI/PMID/PMCID/arXiv via prefixes; OpenAlex
accepts `doi:`/`pmid:`. If one source has no record for an ID, converting it and trying
another source is usually faster than reformulating the query.

## Paper record fields

Each entry in `papers[]` is a flattened record with these keys (list fields are
`; `-joined strings): `paper_id, title, authors, abstract, doi, published_date, pdf_url,
url, source, updated_date, categories, keywords, citations, references, extra`.

There is **no `year` field** — derive the year from `published_date` (ISO `YYYY-MM-DD`,
or empty string when unknown).
