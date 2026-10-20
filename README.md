<p align="center"><a href="https://apify.com/DevilScrapes/openlibrary-books-scraper"><img src="assets/banner.svg" alt="Open Library Scraper — Book Metadata in Bulk — Python example" width="100%"></a></p>

# Open Library Scraper in Python: book metadata and ISBNs to pandas, JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/openlibrary-books-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/openlibrary-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/openlibrary-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This is a compact Python client for the **[Open Library Books Scraper](https://apify.com/DevilScrapes/openlibrary-books-scraper)** running on Apify. You give it a query such as an author and a series name, an ISBN or a subject tag, and it returns a table of matching works from the Internet Archive's open catalogue: title and subtitle, author names, first publication year, how many editions exist, the ISBNs, publishers, languages, subject tags and a cover image link. Apify runs the queries and the paging; the script here starts a run, waits, and saves `results.json` and `results.csv`. List columns stay as lists in JSON, which is what you want when the next step is `DataFrame.explode`.

> **Actor:** [Open Library Scraper — Book Metadata in Bulk](https://apify.com/DevilScrapes/openlibrary-books-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [openlibrary-scraper-javascript](https://github.com/devilscrapes/openlibrary-scraper-javascript)

## 🔥 What you get

- **Works, not editions**: each row is one Open Library work, with `edition_count` telling you how many printings and translations sit behind it.
- **Point the query at one field**: `searchField` sends your text to title, author, subject or ISBN matching, or leaves it free-form with `all`.
- **A language filter**: a three-letter code such as `eng` or `spa` in `language` is passed to Open Library's search as a language filter.
- **Identifiers you can join on**: the work key, a canonical `work_url`, every ISBN-10 and ISBN-13 Open Library lists, and the numeric `cover_id`.
- **A ready cover link**: `cover_url_l` is built from the cover ID and points at the large JPEG on Open Library's cover service.
- **Reader signals where they exist**: average rating, rating count and an `ebook_access` value that says whether a scan is public, borrowable or absent.
- **Bounded subject lists**: subjects are cut at 30 per work, so one heavily tagged classic cannot bloat a CSV column.

## 🎯 Use cases

### Enrich a spreadsheet of titles with ISBNs

Loop over a column of titles, run one small query per title with `searchField` set to `title`, and keep the top match. The `isbns` list and `first_publish_year` turn a loose reading list into something a library system or bookshop feed will accept.

### Build a subject corpus for research

Query a subject such as `science fiction`, explode `subjects`, and you have a co-occurrence table of tags in a notebook. Add `first_publish_year` and you can watch how a genre's vocabulary shifts from decade to decade.

### Find the most reprinted works by an author

Run an author search and sort by `edition_count`. The works with the most editions are usually the ones still in print or widely translated, which is a cheap proxy for staying power when you curate a collection.

### Refresh catalogue metadata on a schedule

A weekly cron job that re-queries your watch list and diffs `ratings_count`, `edition_count` and `ebook_access` shows which titles gained a new edition or a borrowable scan since the last pass.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/openlibrary-scraper-python.git
cd openlibrary-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Open Library Scraper — Book Metadata in Bulk](https://apify.com/DevilScrapes/openlibrary-books-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without the file writing and argument parsing, the client comes down to this. `call()` waits for the run to end and `iterate_items()` yields the stored rows lazily, a page at a time.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/openlibrary-books-scraper").call(run_input={'searchQuery': 'isaac asimov foundation', 'searchField': 'all', 'maxResults': 10, 'language': '', 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

The script forwards `input.json` exactly as it is on disk. The sample is the Actor's own default: a free-text search for Asimov's Foundation books, capped at 10 works, with no language filter and no proxy. That costs $0.20 for the start plus $0.015 for the rows. `searchQuery` is the one required field. Switch `searchField` to `isbn` for a single-book lookup, or to `subject` for a topical pull, and raise `maxResults` toward 1,000 once the query returns what you expect.

```json
{
  "searchQuery": "isaac asimov foundation",
  "searchField": "all",
  "maxResults": 10,
  "language": "",
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchQuery` | string | yes | `"isaac asimov foundation"` | — |
| `searchField` | string | no | `"all"` | `all`, `title`, `author`, `subject`, `isbn` |
| `maxResults` | integer | no | `30` | — |
| `language` | string | no | `""` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each row is one work. You can rely on `openlibrary_key`, `title` and `work_url`; anything else may be null or an empty list, because catalogue records vary a lot in completeness. `edition_count` falls back to 0 when Open Library has no figure, and ratings are often missing for older or obscure titles.

| Field | Type | Always present |
|---|---|---|
| `openlibrary_key` | string | yes |
| `title` | string | yes |
| `subtitle` | string or null | no |
| `authors` | array | no |
| `first_publish_year` | integer or null | no |
| `edition_count` | integer | no |
| `languages` | array | no |
| `subjects` | array | no |
| `isbns` | array | no |
| `publishers` | array | no |
| `cover_id` | integer or null | no |
| `cover_url_l` | string or null | no |
| `ratings_average` | number or null | no |
| `ratings_count` | integer or null | no |
| `ebook_access` | string or null | no |
| `work_url` | string | yes |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.0015 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/openlibrary-books-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

With pandas (`pip install pandas`) the list columns become easy to work with. This recipe writes an ISBN-13 lookup table, one line per ISBN with its work and authors, to `isbn13_index.csv`, then prints the subject tags that appear across the most works.

```python
import json

import pandas as pd

books = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))

# One row per ISBN-13, pointing back to its work: a lookup table for catalogue joins.
isbn = books[["openlibrary_key", "title", "authors", "first_publish_year", "isbns"]].explode("isbns")
isbn = isbn[isbn["isbns"].astype(str).str.fullmatch(r"97[89]\d{10}")]
isbn["authors"] = isbn["authors"].map(lambda names: "; ".join(names or []))
isbn.rename(columns={"isbns": "isbn13"}).to_csv("isbn13_index.csv", index=False)

# Which subject tags dominate the result set, weighted by how often each work was published.
tags = books[["subjects", "edition_count"]].explode("subjects").dropna(subset=["subjects"])
top = tags.groupby("subjects")["edition_count"].agg(works="count", editions="sum")
print(f"{len(isbn)} ISBN-13 rows written")
print(top.sort_values("works", ascending=False).head(15).to_string())
```

## ❓ FAQ

### Do I need an Open Library API key?

No. Open Library's search has no key, and this Actor needs none either. Your Apify token is the only secret involved.

### Can I get the book's description or blurb?

Not from this Actor. It reads Open Library's search results, which do not carry long descriptions. Each row gives you `work_url`; a second request to that work's record is the place to fetch them.

### Why does the same book appear without some ISBNs?

ISBNs are only as complete as the catalogue. Many older works were entered without them, and the Actor returns exactly the list Open Library has for the work, ISBN-10 and ISBN-13 mixed.

### What happens if Open Library returns an error mid-run?

Paging stops at that page and the run finishes with the rows it already saved; it is not retried. For a scheduled job, compare the row count with `maxResults` and rerun when a broad query comes back unexpectedly short.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/openlibrary-books-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Open Library Scraper — Book Metadata in Bulk on Apify Store](https://apify.com/DevilScrapes/openlibrary-books-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/openlibrary-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Goodreads Book & Reviews Scraper](https://apify.com/DevilScrapes/goodreads-book-reviews-scraper) — reader reviews to set next to Open Library's catalogue data.
- [Crossref Works Scraper — DOIs, Authors & Citations](https://apify.com/DevilScrapes/crossref-works-scraper) — DOI metadata for journal articles and scholarly books.
- [arXiv Scraper — Search & Export Paper Metadata](https://apify.com/DevilScrapes/arxiv-papers-scraper) — preprint records when the reading list turns academic.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/openlibrary-books-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Open Library or the Internet Archive.
