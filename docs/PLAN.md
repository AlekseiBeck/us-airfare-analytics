# US Airfare Analytics — Project Plan

Nov 8, 2026 · @Alex Beck

## Overview

Build a public GitHub portfolio project that analyzes 10+ years of real U.S. airfare data to answer when, where and on which airline flights are cheapest, with Detroit (DTW) to New York as the featured case study.

- **Working repo name:** `us-airfare-analytics`
- **Stack:** Python, SQL (DuckDB), dbt, Tableau Public (or Power BI), GitHub Actions
- **Data:** free U.S. Department of Transportation (DOT) and Bureau of Transportation Statistics (BTS) public data, plus one optional Kaggle dataset
- **Deliverables:** a reproducible pipeline, tested dbt models, analysis notebooks, a public interactive dashboard, and a README that reads like an analyst's findings report

**How to use this plan with Claude Code**

1. Export this doc as Markdown and save it in the new repo as `docs/PLAN.md`.
2. Paste the starter prompt (near the end of this doc) into Claude Code.
3. Work one milestone at a time. Review each milestone's output before starting the next.
4. Where this plan says **verify**, have Claude Code check the live source (URL, schema, field name) before writing code against it. Government download pages change.

## Decisions and defaults

The plan assumes the defaults below. Change any of them before starting; each one changes only a single layer of the build.

| Decision | Default | Alternative | Why the default |
| --- | --- | --- | --- |
| Years | 2015 Q1 to the latest available month | Full history from 1993 | Long enough for trends, includes the COVID shock, keeps downloads manageable |
| Warehouse | DuckDB (local file) + Parquet | Google BigQuery free tier | Free, fast on a laptop, no cloud account needed |
| Transform layer | dbt Core with the `dbt-duckdb` adapter | Plain SQL scripts | dbt shows up in analytics job postings and gives you tests and documentation |
| BI tool | Tableau Public | Power BI Desktop (Windows only) | Free on Mac and Windows, and it gives you a public link to share with recruiters |
| Analysis | Jupyter notebooks (pandas, statsmodels, matplotlib) | Plain scripts | Notebooks render on GitHub |
| Scope of v1 | Seasonality + airline and airport comparison + DTW to NYC case study | Everything at once | Gets a finished, shareable project fast |
| Scope of v2 | Booking window (how far ahead to book), reliability, monthly 40% data |  | Builds on v1 without blocking it |

Open question: do you have a Windows machine? If you want Power BI instead of Tableau, only the dashboard milestone changes.

## Questions the project answers

Every chart, model and notebook should trace back to one of these five questions.

| # | Question | Coverage | Version |
| --- | --- | --- | --- |
| Q1 | Which quarter (and, from July 2025, which month) is cheapest to fly a given route? | Every U.S. domestic route | v1 |
| Q2 | Which airline and which airport in a metro area (for example JFK vs LGA vs EWR) is cheapest for a route? | Every route | v1 |
| Q3 | How have fares changed since 2015 after adjusting for inflation, and what did COVID do? | Every route | v1 |
| Q4 | How far ahead should you book? | All routes from July 2025 (official purchase-window field), plus about 16 major airports in 2022 (Kaggle) | v2 |
| Q5 | Are the cheap times also reliable, or do you pay for them in delays and cancellations? | Every flight by reporting airlines | v2 |

**Featured case study:** "When should I fly from Detroit to see my girlfriend in New York, and on which airline and airport?" Compare JetBlue to JFK against the other airlines into JFK, LGA and EWR.

## Data sources

All core data is free U.S. government data; nothing is scraped from airline or travel sites. Items marked **verify** must be checked against the live page before code is written against them.

| Source | What it holds | Grain and coverage | How to get it | Version |
| --- | --- | --- | --- | --- |
| [DB1B Market + Ticket (O&D10)](https://transtats.bts.gov/DatabaseInfo.asp?QO_VQ=EFI) | 10% sample of domestic tickets: origin, destination, city market IDs, carriers, fare paid, passengers, miles | Quarterly, 1993 to June 2025 | Pre-zipped CSVs: `https://transtats.bts.gov/PREZIP/Origin_and_Destination_Survey_DB1BMarket_{YYYY}_{Q}.zip` and `..._DB1BTicket_{YYYY}_{Q}.zip`. **Verify** the latest quarter available. | v1 |
| [DB1C product (O&D40)](https://www.bts.gov/topics/airlines-and-airports/origin-and-destination-survey-data-product) | 40% sample of tickets, one row per passenger: total amount and tax, reporting/operating/marketing carrier, scheduled flight year and month, purchase window group, city market ID | Monthly, July 2025 to latest (July 2026 listed as of Nov 2026) | Zip per month from the BTS product page. Comma-delimited. Field definitions in the [record description PDF](https://www.bts.gov/sites/bts.dot.gov/files/2025-12/DB1C_Description.pdf). **Verify** the schema before modeling. | v2 |
| [On-Time Performance (Reporting Carrier)](https://www.transtats.bts.gov/Tables.asp?QO_VQ=EFD) | Every nonstop domestic flight by reporting carriers: scheduled and actual times, delays, cancellations, delay causes | Daily, per flight, 1987 to present | Monthly zips on TranStats. **Verify** the pre-zip URL pattern. | v2 |
| T-100 Domestic Segment | Seats, passengers, departures by carrier and route | Monthly, 1990 to present | TranStats download form (no stable pre-zip). Optional, for load-factor context. | v2 (optional) |
| CPI-U, all items (FRED series `CPIAUCSL`) | Monthly inflation index, to convert fares to constant dollars | Monthly | CSV from FRED. **Verify** the endpoint. | v1 |
| [Kaggle Expedia flight prices](https://www.kaggle.com/datasets/dilwong/flightprices) | One-way fares found on Expedia, with the search date and the flight date | Daily snapshots, 2022-04-16 to 2022-10-05, about 16 major airports | Kaggle API. **Verify** the airport list (expected to include DTW, JFK, LGA, EWR) and the license. Very large: filter with DuckDB, don't load it in pandas. | v2 (optional) |
| TranStats lookup tables + [OurAirports](https://ourairports.com/data/) | Carrier names, city market names, airport names, coordinates | Static | Downloaded once into `seeds/` | v1 |
| [BTS Average Fare tables](https://www.transtats.bts.gov/AverageFare/) | Official average fare by airport and year | Annual and quarterly | Used only to validate your own numbers | v1 |

Two facts that shape the design:

- From July 2025 the sample grew from 10% to 40% of tickets and moved to the new monthly DB1C format. Older and newer data need separate staging models that meet in one common table.
- DB1C includes a purchase window field, so the booking-window question (Q4) can use official data for all routes, not only the 2022 Kaggle sample.

## Architecture and tech stack

The pipeline is batch and fully reproducible: one `make all` goes from raw downloads to dashboard extracts.

&#91;embedded content: pipeline · sources to dashboard\]

Data flows left to right; the marts are the only layer that notebooks and Tableau read.

| Layer | Tool | Resume keyword it shows |
| --- | --- | --- |
| Ingest | Python (requests, pyarrow) | Python, ETL |
| Storage | Parquet files + DuckDB | SQL, data warehousing |
| Transform | dbt Core (`dbt-duckdb`, `dbt_utils`) | dbt, data modeling, data quality tests |
| Analysis | Jupyter, pandas, statsmodels, scipy, matplotlib | Python, statistics |
| BI | Tableau Public (or Power BI) | Tableau / Power BI, dashboards |
| Engineering | Git, GitHub Actions, Make, ruff | Version control, CI/CD |

## Repository structure

Raw and processed data never go into git. Everything in `data/` is rebuilt by running the pipeline.

```
us-airfare-analytics/
├── README.md                  # findings-first write-up (see GitHub section)
├── docs/
│   ├── PLAN.md                # this plan
│   ├── data_dictionary.md     # every mart column, defined
│   └── img/                   # dashboard screenshots for the README
├── pyproject.toml             # Python deps (use uv or pip)
├── Makefile                   # make ingest / make build / make test / make export
├── .gitignore                 # data/, *.duckdb, .env, target/, logs/
├── .env.example               # KAGGLE_USERNAME, KAGGLE_KEY (never commit .env)
├── ingest/
│   ├── config.py              # years, quarters, months, paths
│   ├── download_db1b.py       # O&D10 Market + Ticket, quarterly
│   ├── download_db1c.py       # O&D40, monthly (v2)
│   ├── download_ontime.py     # on-time performance, monthly (v2)
│   ├── download_cpi.py
│   ├── download_kaggle.py     # optional (v2)
│   └── to_parquet.py          # CSV -> Parquet, keep only needed columns
├── data/                      # gitignored
│   ├── raw/                   # zips as downloaded
│   ├── parquet/               # one file per source per period
│   └── warehouse.duckdb
├── dbt/
│   ├── dbt_project.yml
│   ├── profiles.yml           # points at data/warehouse.duckdb
│   ├── seeds/                 # carriers, city markets, airports, cleaning thresholds
│   ├── models/
│   │   ├── staging/           # stg_* : one per source, rename + type cast only
│   │   ├── intermediate/      # int_* : cleaning, unifying O&D10 + O&D40
│   │   └── marts/             # fct_* / agg_* : what notebooks and the dashboard read
│   └── tests/                 # custom data tests
├── notebooks/
│   ├── 01_data_quality.ipynb
│   ├── 02_seasonality.ipynb
│   ├── 03_carrier_airport_comparison.ipynb
│   ├── 04_dtw_nyc_case_study.ipynb
│   ├── 05_booking_window.ipynb        # v2
│   └── 06_reliability.ipynb           # v2
├── exports/                   # small CSV extracts for Tableau (committed, under 100 MB)
├── dashboard/
│   └── README.md              # link to Tableau Public + how the extracts map to sheets
└── .github/workflows/ci.yml   # lint + dbt build on a tiny sample
```

## Data model

The model has three dbt layers. Notebooks and the dashboard read only from marts, and every mart stays under Tableau Public's 15 million row limit per data source.

**Staging (`stg_`)**: one model per source. Rename columns to snake\_case and cast types. No business logic.

| Model | Key columns (verify names against the source) |
| --- | --- |
| `stg_db1b_market` | itin\_id, mkt\_id, year, quarter, origin, origin\_city\_market\_id, dest, dest\_city\_market\_id, ticketing\_carrier, operating\_carrier, reporting\_carrier, passengers, market\_fare, market\_distance, market\_coupons, bulk\_fare |
| `stg_db1b_ticket` | itin\_id, round\_trip, dollar\_cred, itin\_fare, fare\_per\_mile, bulk\_fare |
| `stg_db1c_ticket` (v2) | ticket fields from the DB1C record description: total\_amt, tax\_amt, carriers, coupon airports, scheduled flight year and month, purchase window group, city market ID |
| `stg_ontime` (v2) | flight\_date, reporting\_airline, flight\_number, origin, dest, crs\_dep\_time, dep\_delay, arr\_delay, cancelled, diverted, delay-cause minutes |
| `stg_cpi` | month, cpi |
| `stg_kaggle_fares` (v2, optional) | search\_date, flight\_date, starting\_airport, destination\_airport, airline, base\_fare, total\_fare, is\_nonstop |

**Intermediate (`int_`)**: cleaning, and bringing the old and new ticket formats together.

- `int_fares_od10`: join Market to Ticket on itin\_id and apply the cleaning rules.
- `int_fares_od40` (v2): derive each directional origin-to-destination trip from DB1C coupons using the trip-break fields. This is the hardest model in the project, so give it its own milestone and tests.
- `int_fares_unified`: one row per directional trip with: period\_year, period\_quarter, period\_month (null before July 2025), origin\_airport, dest\_airport, origin\_market, dest\_market, carrier, passengers, fare\_nominal, fare\_real (constant dollars), source (`od10` or `od40`), sample\_rate (0.10 or 0.40), purchase\_window\_group (null before July 2025).

**Marts (`agg_`, `dim_`)**: small, documented tables built for one question each.

| Mart | Grain | Main measures | Answers |
| --- | --- | --- | --- |
| `agg_route_carrier_quarter` | airport pair x carrier x quarter | estimated passengers, ticket count, median / p25 / p75 real fare | Q1, Q2, Q3 |
| `agg_market_carrier_quarter` | city market pair x carrier x quarter (all NYC airports as one market) | same as above | Q2 |
| `agg_route_seasonality` | airport pair x quarter-of-year | seasonal index = quarter median / that route's annual median, averaged over years | Q1 |
| `agg_route_carrier_month` (v2) | airport pair x carrier x month, from July 2025 | median real fare, passengers | Q1 |
| `agg_booking_window` (v2) | airport pair x purchase window group | median fare, ticket share | Q4 |
| `agg_reliability_route_carrier_month` (v2) | airport pair x carrier x month | flights, on-time %, cancel %, mean arrival delay | Q5 |
| `dim_carrier`, `dim_airport`, `dim_city_market`, `dim_period` | one row per item | names, codes, coordinates | all |

**dbt tests:** `not_null` and `unique` on keys; `dbt_utils.unique_combination_of_columns` on every mart grain; `accepted_values` for quarter (1 to 4) and source; ranges on fares and passengers; a reconciliation test that the passengers in `int_fares_od10` equal the staged passengers minus the documented exclusions.

## Cleaning rules and validation

Every threshold lives in a dbt seed (`seeds/cleaning_thresholds.csv`), so it can be changed in one place and quoted in the README. Interviewers often ask about these choices.

**Cleaning rules**

1. Drop fares under $50. These are mostly award tickets that carry only taxes and fees; BTS applies a similar floor in [its own average-fare method](https://transportation.libanswers.com/faq/166146).
2. Drop records the source flags as questionable (`dollar_cred = 0`) and bulk fares (`bulk_fare = 1`).
3. Drop extreme fares above a per-mile ceiling. Set the ceiling from the data's distribution in notebook 01, not by guessing.
4. Coach only: exclude itineraries with a first- or business-class coupon, using the fare class field in the DB1B Coupon table. **Verify** the fare-class codes.
5. Use the directional market fare (one-way trip within the itinerary), so round trips and one-ways compare fairly.
6. Weight every statistic by passengers. Estimate total passengers as sample passengers divided by the sample rate (0.10 or 0.40).
7. Report a route-carrier-period only if it has at least 30 sampled tickets (seed value). Smaller cells are hidden, not shown as zero.
8. Convert to constant dollars: real fare = nominal fare x (CPI in the base period / CPI in the ticket's period). The base period is the latest full year.
9. Flag 2020 Q2 to 2021 Q4 as COVID. Leave it out of seasonal averages by default, and show it in the trend charts.
10. Mark July 2025 (the move from a 10% to a 40% sample) as a series break in every time chart.

**Validation checks (notebook `01_data_quality`)**

- [ ] Row and passenger counts per quarter, before and after each rule, in one table
- [ ] Your average fare per airport-year against the [BTS Average Fare table](https://www.transtats.bts.gov/AverageFare/), with any gap explained by method differences
- [ ] Spot check: JetBlue (carrier code B6) shows up on DTW to JFK, and Delta (DL) on DTW to JFK and LGA
- [ ] No mart grain has duplicates (dbt test)
- [ ] Overlapping months: for routes with enough data, compare July to December 2025 medians in O&D40 with 2024 Q3 and Q4 in O&D10 to check the series join

## Analysis plan

Each notebook opens with its question, ends with a findings box of two or three sentences with numbers, and reads from marts only.

| Notebook | Question | Method | Output |
| --- | --- | --- | --- |
| `02_seasonality` | Q1: cheapest quarter (and month, from July 2025) | Seasonal index per route; distribution of the cheapest quarter across the top 500 routes by passengers; Kruskal-Wallis test across quarters (fares are skewed, so no t-tests on raw fares) | "Q\_ is cheapest on \_% of the top 500 routes" + a heatmap of route x quarter |
| `03_carrier_airport_comparison` | Q2: airline and airport within a metro area | Passenger-weighted medians with bootstrap 95% confidence intervals; a regression of log(real fare) on carrier, quarter and distance with route fixed effects (statsmodels) | Fare premium or discount per carrier versus the route average, with intervals |
| (inside 02 and 03) | Q3: trend since 2015 | Real median fare by year, national and per route; COVID band and series break marked | One trend line + per-route small multiples |
| `04_dtw_nyc_case_study` | The personal question | All of the above for DTW to JFK, LGA and EWR, by carrier and quarter, plus monthly from July 2025 | A one-paragraph recommendation, e.g. "Fly JetBlue to JFK in Q\_; expect a median of $\_ in today's dollars" |
| `05_booking_window` (v2) | Q4: how far ahead to book | Median fare by DB1C purchase window group per route; check against Kaggle 2022 days-before-departure curves for DTW-NYC | Booking-window curve + the cheapest window per route |
| `06_reliability` (v2) | Q5: cheap vs reliable | On-time % and cancel % by route, carrier, month; scatter of median fare against on-time % | "Cheap and reliable" quadrant per route |

Two statistics rules apply throughout:

- Use medians and percentiles, not means, for fares. Show intervals when comparing two groups.
- Use correlation language for associations. Only the regression with route fixed effects supports a "holding other things equal" claim, and even that is not causal.

## Dashboard spec

One Tableau Public workbook with four views. Every view is filtered by origin, destination and carrier, and defaults to DTW to New York.

| View | Built from | Contents |
| --- | --- | --- |
| 1. When to fly | `agg_route_seasonality`, `agg_route_carrier_quarter` | KPI tiles (median real fare, cheapest quarter, savings versus the priciest quarter); a quarter-by-year heatmap; the trend line with the COVID band and the July 2025 series break |
| 2. Who to fly | `agg_market_carrier_quarter` | Carrier and airport comparison: median fare as a dot with a p25-p75 bar per carrier and airport; passenger share |
| 3. National view | `agg_route_seasonality`, `dim_airport` | Map of the top routes coloured by cheapest quarter; a table of the routes with the biggest seasonal swing |
| 4. Book and rely (v2) | `agg_booking_window`, `agg_reliability_route_carrier_month` | Booking-window curve; fare against on-time % scatter |

**Export step:** `make export` writes each mart Tableau needs to `exports/*.csv`, because Tableau Public reads files rather than DuckDB. Keep each file under 100 MB so GitHub accepts it.

**Design:** muted grey for context and one accent colour for the selected route or carrier; titles that state the finding ("Q1 is the cheapest time to fly DTW to NYC"), not the topic; source and as-of date in each footer. Add a Tableau Public link and screenshots to the README.

## Milestones

v1 is milestones 0 to 5, about two to three weekends of work (a rough estimate). Commit at the end of every milestone with a clear message. Milestones 6 to 7 are v2.

**M0: Setup**

- [ ] Create the public GitHub repo `us-airfare-analytics` with a README stub, MIT license, Python `.gitignore` plus the data rules above
- [x] Python 3.11+ environment with duckdb, pandas, pyarrow, requests, tqdm, statsmodels, scipy, matplotlib, jupyter, dbt-duckdb, ruff
- [x] `make` targets: `ingest`, `build`, `test`, `export`, `all`
- [x] Save this plan as `docs/PLAN.md`

**M1: Ingest v1 data**

- [ ] Verify the PREZIP URL pattern and the last available quarter, then download DB1B Market and Ticket for 2015 Q1 to 2025 Q2 (Coupon too, only the columns needed for fare class)
- [ ] Downloads are idempotent: skip files already present, retry with backoff, log each file's size and row count
- [ ] Convert each CSV to Parquet with only the needed columns; delete the extracted CSV afterwards
- [ ] Download CPI and the lookup tables into seeds
- [ ] Check free disk space first: many quarters of ticket data add up to tens of gigabytes before compression

**M2: dbt models and tests**

- [ ] `dbt init`, profile pointing at `data/warehouse.duckdb`, sources reading the Parquet files with `read_parquet()`
- [ ] Staging, `int_fares_od10`, `int_fares_unified`, and the v1 marts and dims
- [ ] All tests from the data model section pass; `dbt docs generate` works
- [ ] GitHub Actions: ruff lint + `dbt build` against a tiny committed sample (a few thousand made-up rows in `dbt/seeds/sample/`), so CI runs without the real data

**M3: Analysis**

- [ ] Notebooks 01 to 04, each ending with a findings box that has real numbers
- [ ] Save the key charts as PNG in `docs/img/` for the README

**M4: Dashboard v1**

- [ ] `make export` builds the CSV extracts
- [ ] Views 1 to 3 in Tableau Public, published, link saved

**M5: Write-up and publish (v1 done)**

- [ ] README in the findings-first format (GitHub section below)
- [ ] `docs/data_dictionary.md` generated from the dbt docs
- [ ] Pin the repo on your GitHub profile; add the dashboard link to LinkedIn's Featured section
- [ ] Update your resume with the bullets below, using your real numbers

**M6 (v2): Monthly 40% data and the booking window**

- [ ] Read the DB1C record description; write `stg_db1c_ticket` and `int_fares_od40` with tests
- [ ] Join into `int_fares_unified`; build `agg_route_carrier_month` and `agg_booking_window`
- [ ] Notebook 05; optional Kaggle cross-check for DTW to NYC

**M7 (v2): Reliability**

- [ ] Verify the URL pattern, then ingest on-time data for the same years
- [ ] `agg_reliability_route_carrier_month`, notebook 06, dashboard view 4

## GitHub publishing and README

Yes, the whole project can go in a public GitHub repo. Publish the code, models, notebooks, small exports and write-up, and let anyone rebuild the data with `make all`.

**What goes in and what stays out**

| Item | In the repo? | Why |
| --- | --- | --- |
| Code, dbt models, notebooks with outputs, docs | Yes | This is the portfolio |
| Small mart extracts in `exports/` (under 100 MB each) | Yes | Lets people open the dashboard data without running the pipeline |
| Raw and Parquet DOT data, `warehouse.duckdb` | No | Too large for GitHub; rebuilt by the scripts |
| Kaggle data | No | Downloaded with your own Kaggle key; check the dataset's license before sharing any extract of it |
| `.env`, API keys | No | Commit only `.env.example` |
| Anything from work | No | Keep UWM code, data and naming entirely out of this repo |

**README outline (findings first: recruiters read the top third)**

1. Title + one-line summary + dashboard link + one hero chart
2. Key findings: three to five bullets, each with a number
3. The DTW to New York case study, in one paragraph
4. Data: sources, years, row counts, the July 2025 sample change
5. Architecture diagram + stack
6. Method: cleaning rules and why, statistics used, limitations
7. How to run it (`make all`), how long it takes, disk space needed
8. Repo map and links to the notebooks
9. What I'd do next

## Resume bullet templates

Fill in the brackets with your real results once the project is built. Never put in a number you didn't measure.

**US Airfare Analytics** | Python, SQL, DuckDB, dbt, Tableau, GitHub Actions | \[github link\] | \[month year\]

- Built an end-to-end analytics pipeline over \[N\] million U.S. DOT airline ticket records (2015 to \[year\]), loading raw government files into a DuckDB warehouse and modeling them with \[N\] dbt models and \[N\] data tests.
- Found that \[quarter/month\] is the cheapest time to fly on \[X\]% of the top 500 domestic routes, and that \[carrier/airport\] fares run \[X\]% below the route average after controlling for distance and season.
- Designed and documented cleaning rules (award-ticket floor, coach-only, minimum sample size, inflation adjustment) that removed \[X\]% of records, and validated results against BTS published average fares to within \[X\]%.
- Published an interactive Tableau Public dashboard for any route, carrier and airport, and automated lint and dbt builds with GitHub Actions CI.

Keep three of the four bullets on a one-page resume; the findings bullet (the second) is the one to keep no matter what.

## Starter prompt for Claude Code

Put this plan at `docs/PLAN.md` in an empty repo folder, open Claude Code there, and paste the prompt below.

```text
You are helping me build a data analyst portfolio project. The full plan is in docs/PLAN.md. Read it completely before doing anything.

Working rules:
1. Work one milestone at a time, in order (M0, M1, ...). At the end of each milestone, stop, summarize what you built and anything that differed from the plan, and wait for me to say "continue".
2. Anything marked "verify" in the plan: check the live source first (download page, URL pattern, record description, column names) and tell me what you found before writing code against it. If a source differs from the plan, follow the source and update docs/PLAN.md.
3. Never commit data files, *.duckdb files, .env or credentials. Set up .gitignore in M0 before the first commit.
4. Downloads must be idempotent, resumable and polite (one file at a time, retries with backoff). Before downloading, estimate the total size and check free disk space.
5. Prefer DuckDB SQL over pandas for anything over a few million rows.
6. Every dbt model gets a description and tests in a schema.yml. Run dbt build and fix failures before finishing a milestone.
7. Keep cleaning thresholds in dbt/seeds/cleaning_thresholds.csv, never hard-coded.
8. In notebooks, every chart has a title that states the finding, and every notebook ends with a findings box built from computed values, not typed numbers.
9. Don't invent results. If a number isn't computed yet, leave a placeholder.
10. Create a CLAUDE.md in the repo root that records these rules and the commands (make ingest/build/test/export/all), so later sessions follow them.

Start with M0.
```

Tip: when you start a new Claude Code session later, say "Read CLAUDE.md and docs/PLAN.md, then continue from the next unchecked milestone."

## Risks and gotchas

| Risk | What happens | Fallback |
| --- | --- | --- |
| BTS changes a URL or file layout | Download scripts fail | Scripts read URLs from `ingest/config.py`; fall back to the TranStats download form for one period |
| Disk space | Raw files fill a laptop drive | Download one year at a time; keep only Parquet with selected columns |
| DB1C trip logic is harder than expected | M6 drags on | Ship v1 on O&D10 only; v2 is optional for the resume |
| Quarterly data can't say "which week" | The case study answer is coarser than hoped | Use monthly O&D40 from July 2025, plus the Kaggle 2022 daily data for DTW to NYC |
| Small routes have few tickets | Noisy medians | The 30-ticket minimum hides those cells; say so in the dashboard footer |
| Tableau Public's 15 million rows per data source | Publish fails | Marts are aggregated; filter to routes above a passenger threshold |
| Carrier codes change over time (mergers) | Wrong airline comparisons | Map codes through the TranStats carrier history lookup |

Amadeus closed its free Self-Service flight APIs on 17 July 2026, so live price collection would need a paid API. The plan doesn't depend on it. Don't scrape JetBlue, Google Flights or Expedia yourself; it breaks their terms of service.

## Sources

- [BTS TranStats: Airline Origin and Destination Survey (DB1B)](https://transtats.bts.gov/DatabaseInfo.asp?QO_VQ=EFI)
- [BTS: O&D Survey Data product page (O&D10 and O&D40 downloads)](https://www.bts.gov/topics/airlines-and-airports/origin-and-destination-survey-data-product)
- [BTS: DB1C record description (40% sample)](https://www.bts.gov/sites/bts.dot.gov/files/2025-12/DB1C_Description.pdf)
- [BTS: DB1B record description](https://www.bts.gov/sites/bts.dot.gov/files/DB1B_Record_Description_Product%20for%20bookstore.pdf)
- [USDOT Library: average fares from DB1B, and the 10% to 40% change](https://transportation.libanswers.com/faq/166146)
- [BTS TranStats: Airline On-Time Performance tables](https://www.transtats.bts.gov/Tables.asp?QO_VQ=EFD)
- [BTS Average Fare tables](https://www.transtats.bts.gov/AverageFare/)
- [skynet R package: DB1B pre-zip download pattern](https://rdrr.io/cran/skynet/src/R/download_db1b.R)
- [Kaggle: Flight Prices (Expedia, 2022)](https://www.kaggle.com/datasets/dilwong/flightprices)
- [Tableau: Desktop Public Edition feature comparison (15M row limit)](https://help.tableau.com/v0.0/pro/desktop/en-us/desktop_comparison.htm)
- [AirLabs: Amadeus Self-Service shutdown](https://airlabs.co/amadeus-self-service-api-shutdown)
- [FlightConnections: DTW to JFK nonstop airlines](https://www.flightconnections.com/flights-from-dtw-to-jfk)
