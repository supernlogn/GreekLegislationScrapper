# Greek Legislation Scraper

[![CI](https://github.com/supernlogn/GreekLegislationScrapper/actions/workflows/ci.yml/badge.svg)](https://github.com/supernlogn/GreekLegislationScrapper/actions/workflows/ci.yml)
[![Node.js 20+](https://img.shields.io/badge/node-%3E%3D20-brightgreen)](https://nodejs.org/)
[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](LICENSE)

Scrapes the Greek Government Gazette legislation catalogue at [search.et.gr](https://search.et.gr/el/search-legislation/), downloads every PDF listed for a given year, and keeps a local markdown index. Includes a daily checker that detects newly published PDFs and downloads only what is new.

## Requirements

- [Node.js](https://nodejs.org/) 20+
- npm 9+

## Setup

```bash
git clone https://github.com/supernlogn/GreekLegislationScrapper.git
cd GreekLegislationScrapper
npm install
npx playwright install chromium
```

---

## Scripts

### `scrape:all` — Download every year from 1833 to present

Loops through every year from 1833 to the current year and runs a full scrape for each one, reusing a single browser instance. Already-scraped years are automatically skipped (detected by the presence of their markdown file with data rows).

```bash
npm run scrape:all
```

**Resume from a specific year** (e.g. after an interruption):

```bash
npm run scrape:all -- --start-year 1950
```

Output will be written to:

```
downloads/
  listing-items-1833.md
  listing-items-1834.md
  ...
  listing-items-2026.md
  1833/
  1834/
  ...
  2026/
```

At the end a summary is printed showing how many years were scraped, skipped, empty, or failed. If any years failed, the error message tells you which `--start-year` to use to resume without re-downloading already-completed years.

---

### `scrape:table` — Full scrape for a given year

Downloads **all** PDFs for a year and writes a complete markdown listing. Use this once to build the initial archive.

```bash
npm run scrape:table -- <url> [output-path] [catalogue-value]
```

| Argument | Required | Description |
|---|---|---|
| `url` | yes | Legislation search page URL (include `?selectYear=YYYY`) |
| `output-path` | no | Path for the markdown file. Defaults to `listing-items.md` |
| `catalogue-value` | no | Force a specific legislation catalogue dropdown value |

**Example:**

```bash
npm run scrape:table -- "https://search.et.gr/el/search-legislation/?selectYear=2026" "downloads/listing-items-2026.md"
```

Output:

```
downloads/
  listing-items-2026.md   ← markdown table with links
  2026/
    20260100058.pdf
    20260100057.pdf
    ...
```

The markdown table looks like:

```
| Είδος | Αριθμός | Περιγραφή | Τίτλος ΦΕΚ | Ημερομηνία ΦΕΚ | Σελίδες | Λήψη | Σελιδοδείκτης |
| --- | --- | --- | ... |
| Ν. | 5294 | Κοινωνικό Κλιματικό Ταμείο... | Α 58/2026 | 08-04-2026 | 50 | [20260100058.pdf](2026/20260100058.pdf) | |
```

---

### `check:new` — Daily check for newly published PDFs

Compares the live website against your local listing and downloads only PDFs that are not already present. New rows are prepended to the top of the existing markdown file.

```bash
npm run check:new
```

- Reads `downloads/listing-items-{current-year}.md` to find already-known PDFs
- Scrapes the current year's page in full
- Downloads only new PDFs into `downloads/{year}/`
- Prepends new rows to the markdown listing
- Prints a summary of what was found and downloaded

**Example output:**

```
[2026-04-22T08:00:00.000Z] Checking for new legislation (2026)...
Known PDFs: 31
Scraped 33 total rows from website.

Found 2 new PDF(s):
  Downloading 20260100060.pdf... OK
  Downloading 20260100059.pdf... OK

Summary:
  New entries added : 2
  PDFs downloaded   : 2
  Listing updated   : downloads/listing-items-2026.md
```

If nothing is new:

```
No new PDFs found. Listing is up to date.
```

---

## Scheduling daily checks (Windows)

Run the setup script once to register a Windows Task Scheduler task that executes `npm run check:new` every day at 08:00:

```powershell
.\setup-scheduled-task.ps1
```

The task is named `GreekLegislationDailyCheck`. It uses `-StartWhenAvailable` so if the machine is off at the scheduled time it will run as soon as it comes back online.

**Other useful commands:**

```powershell
# Trigger immediately
Start-ScheduledTask -TaskName "GreekLegislationDailyCheck"

# View last run result
Get-ScheduledTaskInfo -TaskName "GreekLegislationDailyCheck"

# Remove the task
Unregister-ScheduledTask -TaskName "GreekLegislationDailyCheck" -Confirm:$false
```

---

## Project structure

```
scripts/
  utils.ts                ← shared pure functions (tested)
  exportListingTable.ts   ← full-scrape core logic + scrape:table CLI
  scrapeAllYears.ts       ← loops 1833–present (scrape:all)
  checkNewLegislation.ts  ← daily diff checker (check:new)
  __tests__/
    utils.test.ts         ← unit tests
.github/
  workflows/
    ci.yml                ← typecheck + tests on push / PR
setup-scheduled-task.ps1  ← registers Windows scheduled task
tsconfig.json
downloads/
  listing-items-2026.md   ← generated markdown index
  2026/                   ← downloaded PDFs
```

## Testing

Unit tests cover all pure utility functions (text normalisation, markdown building, PDF filename diffing). They run without a browser and complete in under a second.

```bash
# Run tests once
npm test

# Watch mode during development
npm run test:watch

# Type-check without emitting output
npm run typecheck
```

Tests live in [scripts/\_\_tests\_\_/utils.test.ts](scripts/__tests__/utils.test.ts). CI runs them against Node.js 20 and 22 on every push and pull request via [GitHub Actions](.github/workflows/ci.yml).

---

## Contributing

Contributions are welcome. Please follow these steps:

1. Fork the repository and create a feature branch from `main`.
2. Install dependencies: `npm install && npx playwright install chromium`
3. Make your changes.
4. Add or update tests in `scripts/__tests__/` for any logic you touch.
5. Ensure everything passes before opening a PR:
   ```bash
   npm run typecheck
   npm test
   ```
6. Open a pull request with a clear description of the change and why it is needed.

**Reporting bugs:** Please open a GitHub Issue and include the year URL you were scraping and the full console output.

---

## Notes

- The website renders the `#listing-items` table only after the legislation catalogue dropdown is selected and the search button is submitted. Both scripts handle this automatically and retry up to 5 times before failing.
- PDF downloads use exponential backoff with up to 4 attempts per file.
- The year is derived from the URL when using `scrape:table`; `check:new` always uses the current calendar year.
