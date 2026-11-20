# CreateX AI: Qualified Lead Generation Agent

Finds social media marketing agencies (and agencies that run social media for clients) in a city,
researches each one, qualifies it against CreateX AI's ideal customer, finds the decision maker, and
writes a ranked Excel sheet your team can work from.

**Input:** a location and an agency size (small / mid / large).
**Output:** an `.xlsx` with qualified leads ranked A (hot) to D (not a fit), each with what the agency does,
how they work, estimated number of social media clients, decision maker and contacts, why they fit,
a pitch angle and a personalised opening line.

---

## How it works

| Stage | What happens |
|---|---|
| 1. Discover | Claude searches the web with 5 query phrasings for the city and size. Optional: Google Places and your own CSV list. Results are de-duplicated by domain; directories (Clutch, Justdial, etc.) are dropped. |
| 2. Crawl | Reads each agency's own website: home, about, team, services, work/clients and contact pages. Respects robots.txt and waits between requests. Collects publicly listed emails, phones and social links. |
| 3. Qualify | Claude reads the site content and fills a fixed schema: services, how they work, social media focus, Reels/short-form work, client mix, estimated client count and team size (with the basis), size, people named, red flags, pitch angle, opening line, and source links. Invented emails or phones are removed automatically. |
| 4. Decision makers | For promising leads with nobody named on the site, Claude searches public sources for the founder or director (only people with a source URL are kept). Optional: Hunter.io finds a verified work email. |
| 5. Score and export | A transparent 0-100 score (see below), tiers A-D, and the Excel report. Agencies are remembered so later runs can skip them. |

## Setup (one time)

```bash
cd createx_leadgen
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                     # then put your keys in .env
```

Keys in `.env`:

- `ANTHROPIC_API_KEY` (required). **Web search must be switched on for your organization in the Claude Console** (Settings > Privacy), otherwise discovery and people search return nothing.
- `GOOGLE_PLACES_API_KEY` (optional). Adds agencies from Google Maps, with ratings and phone numbers. Needs "Places API (New)" enabled in Google Cloud.
- `HUNTER_API_KEY` (optional). Adds verified work emails for decision makers.

The default model is set in `config.json` (`models`). Change it to any Claude model your account can use.

## Run it

```bash
# Preview the output format first (fictional data, no keys, no cost)
python run_leadgen.py --demo

# Real run
python run_leadgen.py --location "Hyderabad" --scale mid --target 25

# Your own list as well (e.g. from your outreach spreadsheet)
python run_leadgen.py --location "Hyderabad" --scale small --seed my_agencies.csv

# Next week: skip agencies you've already seen
python run_leadgen.py --location "Hyderabad" --scale mid --exclude-seen
```

Options: `--scale small|mid|large` (also accepts low/medium/high), `--target N` qualified leads wanted
(the agent researches about 3x as many agencies), `--out file.xlsx`, `--no-web`, `--no-places`,
`--no-people`, `--config custom.json`, `-v` for detailed logs.

Seed CSV columns: `name, website, location` (see `seeds_example.csv`).

## The Excel report

| Sheet | Use |
|---|---|
| Qualified Leads | Your working list, Tier A then B, best first. Edit **Status**, **Owner** (dropdowns), **Next step**, **Notes**. Hover over the client count, team size, size and decision-maker cells to see the basis and source. |
| Run Summary | What was searched, counts per tier, API usage, how to use the sheet, and the scoring rubric. |
| All Leads | Every agency researched, including disqualified ones and why. |
| Score Breakdown | The points behind each score. |
| Evidence | Source links for the key claims and people. Check these before a call. |

## How the score works (edit weights in `config.json`)

| Signal | Points |
|---|---|
| Social media is core / major / minor | 25 / 18 / 6 |
| Reels and short-form video work: strong / some | 20 / 12 |
| Content produced in-house | 5 |
| Estimated client count fits the size searched | 15 in range, 10 overlapping |
| Agency size matches the search | 10 exact, 4 one size off |
| Client mix: B2C/D2C / mixed / B2B | 10 / 6 / 2 |
| Contactability: decision maker / email / phone | 6 / 4 / 2 |
| Activity: good Google reviews / recent activity | 3 / 2 |
| Each red flag | -5 (max -15) |

Tiers: **A >= 70**, **B >= 50**, **C >= 30**, D below. Qualified = A and B.
Agencies one size away from your search are capped at B, two sizes away at C.
Agencies with no social media work, directories, SaaS tools and job sites are disqualified.

## Cost and speed

Each run makes roughly: 5 discovery calls with web search, one qualification call per agency researched,
and one people search per promising lead without a named contact. A `--target 25` run researches about
75 agencies, so expect on the order of 100+ API calls and a similar number of web searches.
The Run Summary sheet shows the exact usage for every run; check current pricing in the Claude Console.
Start with `--target 10` to calibrate. Results are cached (pages 14 days, analysis 30 days), so re-runs
and `--exclude-seen` runs are much cheaper.

## Data use rules (please keep)

- The agent only collects **business information that the agency or person has published**: company
  websites, public search results and Google Maps. It does not scrape LinkedIn; it records profile links
  that appear in search results.
- It never guesses emails. Emails come from the agency's own site or Hunter's verified lookup.
- Every person and claim has a source link in the Evidence sheet.
- India's Digital Personal Data Protection Act applies to personal data such as names and emails. Use
  contacts only for relevant B2B outreach, honour any request to stop contacting someone, and delete data you
  no longer need. Get legal advice if you plan large-scale automated outreach.

## Known limits

- Client counts and team sizes are **estimates** from public content (logos, case studies, team pages).
  The confidence level and basis are shown; treat "low" confidence as a guess.
- Websites built entirely in JavaScript, or behind bot protection, return little text. These show a
  "Very little readable text" note and usually land in a lower tier. Add them manually or via the seed CSV.
- Discovery quality depends on what is findable online in that city; Google Places plus a seed list gives
  the best coverage.

## Getting more leads

Each run saves a **new workbook** (`..._2.xlsx`, `..._3.xlsx`); earlier files are never overwritten.
To find agencies you have not analysed yet:

```bash
# New agencies only, same city (skips everything analysed in earlier runs)
python run_leadgen.py --location "Hyderabad" --scale mid --target 25 --exclude-seen

# Search the city area by area. This is the best way to find agencies a city-wide search misses.
python run_leadgen.py --location "Hyderabad" --scale mid --target 25 --exclude-seen \
    --areas "Gachibowli, Madhapur, Kukatpally, Secunderabad, Banjara Hills, Kondapur"

# Other agency sizes
python run_leadgen.py --location "Hyderabad" --scale small --target 25 --exclude-seen
```

How it works: with `--exclude-seen`, agencies analysed before are removed from the list, are not counted
towards your target, and are named in the search prompt so Claude looks for different ones.
`--areas` searches each neighbourhood separately (two search phrasings per area). `--fresh` ignores cached
discovery results and searches again. An agency whose site could not be read, or whose analysis failed, is not
counted as seen, so it is retried next time.

`--target N` sets the size of the pool researched (about 3 x N agencies). The number of qualified leads is whatever
share of that pool fits; about 1 in 10 is typical, so use a larger target or several area runs for more.

## Troubleshooting

**`tool_choice: type "tool" and "any" are not supported for this model` (HTTP 400 on every analysis call)**
Claude 5.5-generation models reject "forced" tool calls. Version 1.0 of this agent forced one, so every analysis
failed and the spreadsheet came out empty. Fixed in this version: the agent now asks for structured output the
supported way and no longer forces anything. If you still see this error you are running the old files; replace the
whole folder with the new download.

**The run stops with "Stopped: N analysis requests failed and none succeeded"**
The API refused the requests, so no spreadsheet is written. The message shows the first error. Common causes:
a wrong model name in `config.json`, an invalid or exhausted API key, or web search not enabled in the Claude Console.

**Qualified Leads sheet is empty**
It now says so, and "All Leads" shows the reason for each agency. The `Analysis` column tells you whether an agency
was analysed (`OK`), failed because of an API error (`Failed`), or could not be read (`Not analysed (site unreadable)`).
Sites that block crawlers (HTTP 403) cannot be read; add them by hand through the seed CSV if they matter.

**Re-running after a failed run costs very little**
Discovery (7 days) and website pages (14 days) are cached in `.leadgen_cache.sqlite`, so a re-run only pays for the
analysis. Failed agencies are not remembered as "seen". If you ran an earlier version, clear its history before using
`--exclude-seen`:

```bash
sqlite3 .leadgen_cache.sqlite "DELETE FROM seen;"
```

**Thinking and cost settings** (`config.json`)
5.5 models think before answering. `effort` sets how much (`low`, `medium`, `high`; `null` leaves the model default)
and `max_tokens` leaves room for thinking plus the answer. If a model rejects `effort`, the agent drops it
automatically and carries on.

## Tests

```bash
python -m unittest discover tests -v     # offline, no keys needed
```

## Project layout

```
run_leadgen.py        command-line entry point
config.json           models, queries, size definitions, scoring weights, owners, statuses
leadgen/discovery.py  web search, Google Places, seed CSV, de-duplication
leadgen/crawler.py    website reading and contact extraction
leadgen/extract.py    qualification schema and prompt
leadgen/people.py     decision-maker search and Hunter emails
leadgen/scoring.py    score, tiers, size cap
leadgen/excel.py      the Excel report
leadgen/pipeline.py   runs the stages in order
leadgen/demo.py       fictional preview data
```
# Leadgen
