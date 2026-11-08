# US Airfare Analytics

The full plan is in `docs/PLAN.md`. Read it before doing anything. Continue from the next unchecked milestone.

## Commands

Python runs through uv (`uv sync` installs the environment, `uv run <cmd>` runs inside it).

- `make ingest`: download raw data and convert it to Parquet
- `make build`: run the dbt models
- `make test`: run the dbt tests
- `make export`: write mart extracts to `exports/*.csv` for Tableau
- `make all`: all of the above, in order

## Working rules

1. Work one milestone at a time, in order. At the end of each milestone, stop, summarize what was built and anything that differed from the plan, and wait for the user to say "continue".
2. Anything marked "verify" in the plan: check the live source first (download page, URL pattern, record description, column names) and report what you found before writing code against it. If a source differs from the plan, follow the source and update `docs/PLAN.md`.
3. Never commit data files, `*.duckdb` files, `.env` or credentials.
4. Downloads must be idempotent, resumable and polite (one file at a time, retries with backoff). Before downloading, estimate the total size and check free disk space.
5. Prefer DuckDB SQL over pandas for anything over a few million rows.
6. Every dbt model gets a description and tests in a schema.yml. Run dbt build and fix failures before finishing a milestone.
7. Keep cleaning thresholds in `dbt/seeds/cleaning_thresholds.csv`, never hard-coded.
8. In notebooks, every chart has a title that states the finding, and every notebook ends with a findings box built from computed values, not typed numbers.
9. Don't invent results. If a number isn't computed yet, leave a placeholder.
10. Keep anything from the user's employer (code, data, naming) entirely out of this repo. It is public.
