# Troubleshooting

Concrete failure modes for the `paper-search` CLI and how to recover. Many "errors" are
**upstream provider instability**, not bugs — the fix is usually to switch sources or add
a key, not to retry blindly.

## First: how failures surface

- `search` **does not crash** on a bad source. It puts the failure in the top-level
  `errors` map and reports `0` in `source_results` for that source. **Always read
  `errors`** before concluding anything.
- A source with `0` hits and **no** `errors` entry = "nothing indexed here for this
  query," not "this paper doesn't exist." Try another source.
- `download` → `{"status": "error", "message": "..."}` on failure.
- `read` → prints a JSON error object (not plain text) on failure. Note: some
  metadata-only sources (`pubmed`, `crossref`, `google_scholar`) instead return a plain
  **info message** explaining they have no full text — treat that as "no full text," not
  as the paper.
- Config/credential warnings print to **stderr** and are safe to ignore.

## Installation / invocation

| Symptom | Cause | Fix |
|---------|-------|-----|
| `command not found: paper-search` | Not installed or not on PATH | `uv tool install paper-search-mcp`; if `uv` missing, `curl -LsSf https://astral.sh/uv/install.sh \| sh` then retry. Or `pip install paper-search-mcp` and call `python3 -m paper_search_mcp.cli ...` |
| `realpath: command not found` (macOS, uvx) | uvx wrapper needs GNU coreutils | `brew install coreutils`, or use `uv tool install` / run from source instead |
| Slow install behind a proxy | uv defaults to public PyPI over a throttled proxy | Point uv at a faster index: `uv tool install paper-search-mcp --default-index <url>` (uv ignores pip.conf; use `UV_DEFAULT_INDEX` or `~/.config/uv/uv.toml`) |
| `{"error": "No valid sources selected"}` | Misspelled source name(s) | Use exact names from `paper-search sources`. Note `semantic` (not `semantic_scholar`), `google_scholar` (underscore), `europepmc`, `pmc` |

## Per-source instability (upstream)

| Source | Symptom | Cause | Workaround |
|--------|---------|-------|------------|
| `google_scholar` | 0 results / empty HTML | Bot-detection (CAPTCHA) | Prefer `arxiv,semantic,openalex`. Or set `PAPER_SEARCH_MCP_GOOGLE_SCHOLAR_PROXY_URL` to a proxy |
| `semantic` | HTTP 429 | Anonymous rate limit | Retry once; set `PAPER_SEARCH_MCP_SEMANTIC_SCHOLAR_API_KEY`. A rejected key (403) auto-retries key-less |
| `core` | 500 / timeout | Unauthenticated throttling | Set `PAPER_SEARCH_MCP_CORE_API_KEY` (free); connector retries w/ exponential backoff, falls back key-less on 401/403 |
| `openaire` | Transient 403 | IP session rate limiting | None needed — retries with escalating request profiles |
| `citeseerx` | 404 via web-archive redirect | PSU endpoint unstable | No workaround; returns empty gracefully |
| `base` | Search returns 0 | OAI-PMH needs institutional IP registration | Register at base-search.net for API access; else returns empty |
| `ssrn` | HTTP 403 | Cloudflare bot-detection | None; tries two endpoints, returns a clear message. Public PDFs only |
| `pmc` / `europepmc` | PDF download `ProxyError` | Local proxy blocks direct HTTPS PDF | Disable the proxy, or resolve the PDF via another OA source |
| `unpaywall` | Skipped entirely / disabled | `PAPER_SEARCH_MCP_UNPAYWALL_EMAIL` not set | Set a **real** email (placeholders like `test@example.com` are rejected, HTTP 422) |

## Wrong-result traps

- **Metadata presented as full text.** `pubmed`, `crossref`, `openalex`, `dblp`,
  `openaire`, `unpaywall`, `google_scholar` return **no PDF**. `download`/`read` on them
  either raise an error or return an explanatory message. Resolve full text via
  `core`/`pmc`/`europepmc`/`arxiv` instead, or fetch the result's `pdf_url` directly.
  Never summarize an abstract as if it were the paper.
- **Wrong identifier for the source.** A PMID won't work in `arxiv`; an arXiv ID won't
  work in `pubmed`. Match the `paper_id` to the `source` it came from, or convert the ID
  (see `sources.md`).
- **Keyword search on bioRxiv/medRxiv.** They don't keyword-search — the query is treated
  as a subject **category** and only the last ~30 days are browsed. Use `europepmc` for
  topic search over preprints.
- **`-y` year filter ignored on non-Semantic sources.** The `-y` flag only reaches
  `semantic`. For other sources, filter locally by each paper's `published_date` (there
  is no `year` field in the output).

## Recovery checklist

1. **Did it actually fail?** Read `errors` and `source_results`. A 0 isn't always an error.
2. **Right identifier?** Check format against `references/sources.md`.
3. **Equivalent source?** Swap sources (PubMed miss → `semantic`/`openalex`; full-text
   miss → `europepmc`/`core`).
4. **Rate-limited?** Wait briefly, retry once, then add the relevant key.
5. **Report the gap.** State which source failed, the error, and what you tried instead.
   A reported gap is useful; a silent one is misleading.
