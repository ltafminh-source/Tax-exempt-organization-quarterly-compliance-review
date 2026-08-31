# Quarterly Grantee Tax-Exempt & Public Support Check

Automates a quarterly compliance check on grant recipients: confirms tax-exempt status and pulls IRS Form 990 financial data (via ProPublica's Nonprofit Explorer) to flag organizations approaching the 33.33% public support threshold under IRC §509(a)(1)/170(b)(1)(A)(vi).

> **Compliance note:** This tool is AI-assisted and produces a starting point, not a final answer. Every output must be reviewed by a qualified person before being used in any grant or funding decision. See [Known Limitations](#known-limitations) below — some figures are estimates or intentionally left blank pending manual confirmation.

---

## What this does

For each grantee in the input tracker, the script:

1. Looks up the organization on ProPublica's Nonprofit Explorer by EIN
2. Pulls the most recent Form 990 filing
3. Extracts:
   - **Total Revenue** — Form 990, Part I, Line 12
   - **Total Public Support** — Schedule A, Part II, Section A, Line 6
   - **Total Support** (denominator) — Schedule A, Part II, Section B, Line 11
   - **Public Support Percentage** — Schedule A, Part II, Section C, Line 14
4. Flags whether the org is below the 33.33% public support threshold
5. Writes a formatted Excel report with conditional red/green highlighting

---

## Data sources

Two different ProPublica-adjacent sources are used, because no single one is both current and complete:

| Source | What it's good for | Limitation |
|---|---|---|
| ProPublica summary API (`.../nonprofits/api/v2/organizations/{ein}.json`) | Total Revenue — reliable, documented field (`totrevenue`) | Schedule A support-test fields aren't reliably exposed; data can lag 1–2 years behind the live site |
| IRS e-file XML via public AWS S3 (`s3://irs-form-990/{object_id}_public.xml`) | Schedule A Part II detail (Public Support, Total Support, Percentage) with real IRS element names | Requires scraping the ProPublica org page to find the filing's `object_id`; a small number of orgs (509(a)(2) filers, non-digitized/paper filers) won't have Part II data |

The `object_id` for a filing is the long number in a ProPublica filing URL, e.g.:
```
https://projects.propublica.org/nonprofits/organizations/{ein}/{object_id}/full
```

---

## Repository contents

| File | Purpose |
|---|---|
| `990_scrape_and_xml_pull.py` | Main pipeline — reads the tracker file, pulls data for every org, writes the formatted Excel report |
| `990_diagnostic_one_ein.py` | Debug tool — dumps every raw field ProPublica/IRS returns for a single EIN, for troubleshooting a specific org |
| `990_fetch_known_xml.py` | Debug tool — fetches and inspects a specific filing's raw IRS XML given a known object ID |
| `SOP_Public_Support_Percentage_Lookup.md` | Manual fallback procedure for confirming a single org's percentage by hand when automation can't resolve it |
| `Quarterly Grantee Tax-Exempt Check.xlsx` | Input tracker (not committed — see [Input file](#input-file)) |

---

## Setup

```bash
pip install requests pandas openpyxl
```

Requires network access to `projects.propublica.org` and `s3.amazonaws.com`. (Does not work in network-sandboxed environments — see [Notes on execution environment](#notes-on-execution-environment).)

---

## Input file

`Quarterly Grantee Tax-Exempt Check.xlsx`, with columns:

| Organization Name | EIN | Date Checked | Flag | Note |
|---|---|---|---|---|

The script reads `Organization Name` and `EIN` directly — no name-matching or search is performed, since EINs are already confirmed in this file.

---

## Usage

```bash
python 990_scrape_and_xml_pull.py
```

Processes every row in the input file and writes `990_public_support_results.xlsx` with columns:

```
Company Name | EIN | Most Recent Filing Year | Total Revenue | Total Public Support |
Public Support Percentage | Below 33.33% Threshold | Filing URL | Notes
```

Formatting:
- Wrapped text on every column
- Bold header row, frozen
- **Below 33.33% Threshold** cell: red if under 33.33%, green if at/above, uncolored if undetermined (check the Notes column)

The pipeline selects the newest fiscal-year filing listed on the live ProPublica organization page, then reads Schedule A values from that filing's IRS XML. If XML access is blocked or the filing does not contain Part II Line 14, the percentage remains blank and the Notes column explains why.

---

## Known limitations

- **Not every filing digitizes cleanly.** Some orgs file under the 509(a)(2) test (Schedule A Part III) instead of Part II, some have non-digitized/paper filings, and IRS XML element names can vary slightly by form year. When Schedule A data can't be confidently located, the script leaves Public Support Percentage blank and explains why in the Notes column — it does **not** guess or substitute a similarly-named field.
- **This pipeline is actively evolving.** Field-name assumptions were derived from the general IRS e-file schema and ProPublica's documentation, not verified against every org in the dataset. Spot-check results against the source PDF before relying on them, especially for orgs near the 33.33% threshold.
- **Rate limiting.** The script pauses briefly between requests to avoid hammering ProPublica/AWS. Processing the full list takes a while — this is intentional.
- **Manual fallback exists.** See `SOP_Public_Support_Percentage_Lookup.md` for the exact click-path to confirm any org's numbers by hand via ProPublica's website UI.

## Notes on execution environment

This script needs outbound network access to `projects.propublica.org` (HTML scraping) and `s3.amazonaws.com` (raw XML downloads). It will not run in network-restricted sandboxes. Google Colab or a local machine both work.

---

## Disclaimer

Data is sourced from public IRS filings via ProPublica and the IRS's public AWS dataset. Figures reflect the most recently *digitized* filing available at run time, which may lag behind an organization's most recently *filed* return. This tool does not constitute legal, tax, or financial advice — verify any figure used in a funding decision against the source filing.
