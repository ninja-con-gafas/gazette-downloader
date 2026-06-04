# Maharashtra eGazette Name-Change Archive Toolkit

A practical workflow to collect Maharashtra eGazette PDFs and search them locally on Linux.

# Legal And Operational Caution

This software is provided strictly for lawful, personal, research, archival, interoperability, and administrative use. The author does not encourage, solicit, or authorise unlawful scraping, unauthorised access, circumvention of access controls, denial-of-service behaviour, copyright infringement, privacy violations, or any use contrary to the target website's applicable terms, policies, or governing law. Users are solely responsible for verifying that their access, download volume, storage, processing, and downstream use of any data or documents are lawful in their jurisdiction and permitted by the relevant website owner or public authority.

To the maximum extent permitted by applicable law, the author shall not be liable for any direct, indirect, incidental, consequential, special, exemplary, punitive, regulatory, or economic loss or damage, including without limitation service disruption, blocked access, corrupted files, missed records, false search results, inaccurate extractions, loss of opportunity, legal claims, enforcement action, or data-protection violations arising out of or related to the use or inability to use this software.

Use the downloader responsibly. Respect the portal's stability, avoid aggressive parallelism, and keep a copy of raw HTML responses for audit and debugging.

## Scope

This project automates two tasks:

1. Submit search requests to the Maharashtra eGazette portal.
2. Download each PDF returned for the selected date windows.

The collection flow relies on the ASP.NET Web Forms pattern used by the site: a search request returns a results page with hidden fields and row-specific `__doPostBack(...)` targets; each row target must then be submitted back to the same endpoint to obtain the corresponding PDF.

## What the project does?

- Accepts a user-specified date range.
- Splits the range into contiguous one-year windows to reduce the cost of large search requests.
- Submits one search per window.
- Extracts hidden ASP.NET state and all `View` postback targets from the results page.
- Downloads every PDF for that window.
- Stores files locally for later keyword search.

## Why yearly windows are used?

Large date ranges make the search response slow and state-heavy. Splitting the original range into one-year chunks reduces request size and keeps each search-response page smaller and easier to process reliably.

Example for `2019-08-01` to `2026-06-03`:

- `2019-08-01` to `2020-07-31`
- `2020-08-01` to `2021-07-31`
- `2021-08-01` to `2022-07-31`
- `2022-08-01` to `2023-07-31`
- `2023-08-01` to `2024-07-31`
- `2024-08-01` to `2025-07-31`
- `2025-08-01` to `2026-06-03`

## Requirements

- Linux
- Python 3.10+
- `requests`
- `beautifulsoup4`
- Valid session cookies and request headers captured from a working browser session
- Local disk space for PDFs

Install Python dependencies:

```bash
python3 -m pip install requests beautifulsoup4
```

## Project structure

```text
.
├── downloader.ipynb
├── README.md
└── gazette_output/
    ├── window_01_..._search_response.html
    ├── w01_001_...pdf
    └── ...
```

## Core workflow

1. Start with the working request payload captured from the browser.
2. Keep the original `cookies`, `headers`, and base `search_form_data` unchanged.
3. Replace only the date fields for each yearly window.
4. Submit the search request.
5. Extract:
   - hidden ASP.NET fields,
   - row `View` targets from anchors like `javascript:__doPostBack('ctl00$CPH$GridView2$ctl02$LinkButton1','')`.
6. Replay one postback per row to download the PDF.
7. Save all PDFs with unique names.

## Usage

Run the downloader after setting the following variables in the notebook:

- `cookies`
- `headers`
- `search_form_data`
- `user_start_date`
- `user_end_date`

The script should:

- write the raw search responses to `gazette_output/`,
- download PDFs into the same directory,
- print per-window and overall summaries in the output cell.

## Important operational notes

- The site uses ASP.NET Web Forms state. Hidden fields from the search-results page must be reused for row downloads.
- The row `View` link is not a direct static PDF URL; it is a postback target submitted to the same page.
- If the server starts returning HTML instead of PDF, save the response and inspect it before retrying.
- Recreate the browser session cookies if the session expires.
- Add polite delays between requests to reduce the chance of throttling.

## Searching the downloaded PDFs with `pdfgrep`

`pdfgrep` is a grep-like command-line tool for searching text directly inside PDF files, with support for regular expressions, page-number output, case-insensitive matching, and common grep-style options.

### Install

Debian/Ubuntu:

```bash
sudo apt update
sudo apt install pdfgrep
```

### Basic examples

Search a surname in English across all downloaded PDFs:

```bash
pdfgrep -i -n "Darekar" gazette_output/*.pdf
```

Search a surname in Devnagri across all downloaded PDFs:

```bash
pdfgrep -i -n "दरष कर" gazette_output/*.pdf
```

Count matches per file:

```bash
pdfgrep -r -i -c "Darekar" gazette_output/
```

Show some surrounding context:

```bash
pdfgrep -r -i -n -C 2 "Darekar" gazette_output/
```

## Note on a Critical PDF Search Pitfall: Devanagari Text That Looks Correct but Does Not Match

A serious pitfall in searching Devanagari text inside PDFs is that the text can **look visually correct** to a human reader but still be encoded incorrectly for machine search. In practical terms, a word such as `दरेकर` may render on screen in a way that appears correct, but the underlying extracted text may actually be something else such as `दरष कर`, so a search for the proper spelling does not match the PDF text stream.

### What is happening technically?

PDF is primarily a page-description format, not a semantic text format. A PDF does not necessarily store "letters" in the same way a plain Unicode text file does; instead, it stores drawing instructions and references to glyphs in fonts, and text-extraction tools reconstruct Unicode text by relying on font encodings and optional mapping tables such as `ToUnicode`.

If the embedded font uses a broken, incomplete, ambiguous, or custom character mapping, the PDF viewer can still display the glyphs correctly because it knows how to draw them, but text extraction can return the wrong Unicode characters. In that case, searching tools such as `pdfgrep`, `pdftotext`, or PDF parsers are not "seeing" the same text that a human eye sees on the page; they are only seeing the Unicode values reconstructed from those internal mappings.

## Why this is common in Devanagari PDFs?

Devanagari is especially vulnerable because it is a shaped script. Many visible characters are formed from multiple code points, reordered vowel signs, conjuncts, ligatures, and glyph substitutions. In such scripts, the relationship between what is displayed and what is encoded is more complex than in plain Latin text, so extraction failures are more common when the PDF’s internal font mapping is poor.

A second complication is legacy Indian publishing practice. Some PDFs use embedded fonts with custom or non-standard encodings rather than proper Unicode text, particularly in older workflows. In those files, what appears to be Devanagari text may internally map to unrelated code points or garbage sequences, which makes full-text indexing and keyword search unreliable or impossible without additional conversion.

## Why `दरेकर` fails but `दरष कर` matches?

This usually means the PDF’s extracted text stream is already corrupted or mis-mapped. The rendered glyphs are visually acceptable, but the underlying Unicode sequence that search software receives is not the proper string `दरेकर`; it may instead contain substitute characters, broken ligature expansion, reordered marks, missing combining signs, or entirely wrong code points from a bad `ToUnicode` mapping.

Therefore, when `pdfgrep` searches, it matches the extracted text layer, not the visible appearance. If the text layer says `दरष कर`, then only `दरष कर` will match, even if the page visually appears to say `दरेकर` to a human reader.

## How to verify the issue?

A reliable diagnostic test is to copy the suspicious word from the PDF and paste it into a plain text editor. If the pasted text is broken, substituted, split incorrectly, or differs from what is seen on the page, then the PDF’s text layer or font mapping is defective, and search/indexing tools will inherit the same defect.

## What this means for keyword search?

This issue creates false negatives. A correct keyword may return no results even though the name is visibly present in the document, because the searchable layer does not contain the correct Unicode sequence.

For name search projects such as gazette lookup, this means exact-name search alone is unsafe. The search process should include likely broken variants, split forms, alternate spellings, and copied raw text from a known matching PDF page.

## Practical mitigation strategy

Use a layered search strategy:

- Search the correct spelling first.
- If no result appears, copy the visually displayed text from the PDF and search for the pasted form.
- Search for shorter fragments, for example surname stems or initial syllables.

