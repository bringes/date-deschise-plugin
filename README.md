# Date deschise — Romanian open data for Claude and Codex

![Date deschise](assets/logo.svg)

A plugin that lets your agent answer questions from
[Date deschise](https://date-deschise.bunnytech.app), a directory of 30+ Romanian open datasets,
and turn the answers into professional, sourced reports. It covers the company register, public
budget execution, procurement, elections, Monitorul Oficial, the power grid and energy market,
exchange rates, air quality, weather, roads, tourism, timber permits, INS TEMPO and Eurostat
statistics, earthquakes, buildings, and more.

Every dataset is published as physically sorted Parquet plus one DuckLake catalog on a public
bucket. Your agent queries it with DuckDB over plain HTTPS. There is no API key, no account and no
rate limit.

## What it adds

| Skill | What your agent learns |
| --- | --- |
| `explore-datasets` | The route from a question to a checked, cited answer: read the dataset's own page, attach the catalog, look before aggregating, scan rather than sample, fold Romanian diacritics, filter on the sort key, and carry the dataset, licence, SQL and as-of date with every number. It also covers what to do in a sandbox without network access. |
| `build-data-report` | How to deliver an answer as a page rather than a paragraph: an artifact, a canvas or a local HTML file with a headline finding, KPI tiles, charts, tables, a query link and CSV/JSON download behind every figure, a provenance footer, and an optional live refresh with duckdb-wasm. It ships a tested template, `skills/build-data-report/templates/report.html`. |

Try asking:

- *How has the EUR/RON exchange rate moved since 2015? Build me a report.*
- *Which Romanian public institutions spent the most last year?*
- *What datasets do you have about air quality in Bucharest?*

## Install

**Claude Code**

```bash
claude plugin marketplace add bringes/date-deschise-plugin
claude plugin install date-deschise@date-deschise
```

**Codex**

```bash
codex plugin marketplace add bringes/date-deschise-plugin
```

Then install **Date deschise** from the Plugins directory in the Codex app or ChatGPT desktop app.

Querying needs **DuckDB 2.0 or newer**: the CLI (`brew install duckdb`,
`winget install DuckDB.cli`) or the Python package (`pip install "duckdb>=2.0"`). The agent checks
the version and asks before installing anything.

## What it reads, runs and sends

The plugin contains instructions and one HTML template. It runs no hooks, starts no servers, asks
for no credentials and stores nothing. When the agent follows the instructions:

- **It reads** `https://date-deschise.bunnytech.app` (the dataset pages, `llms.txt` and the query
  guide) and `https://date-deschise-s3.bunnytech.app` (the public catalog and Parquet files). These
  are plain HTTPS GETs and range requests for public data. Your questions and your SQL are not sent
  anywhere; DuckDB runs on your machine and only requests the files it needs.
- **It may install DuckDB** with your package manager, after asking you.
- **Report pages it builds** load ECharts and duckdb-wasm from `https://cdn.jsdelivr.net` (pinned
  versions, ECharts with a subresource-integrity hash). If you click *Refresh from the lakehouse*,
  the page downloads the DuckDB engine and data files from `https://date-deschise-s3.bunnytech.app`.
  Each figure's *Open the query in the playground* link opens
  `https://date-deschise.bunnytech.app/playground` with that figure's SQL in the URL.

## Data licences

The datasets carry their own licences, and most require attribution. Each dataset's page states its
licence and attribution line, and the plugin's instructions make the agent copy both into every
answer and report. This plugin's own instructions and template are MIT-licensed; see `LICENSE`.

## Support

Questions and bugs: open an issue at
[github.com/bringes/date-deschise-plugin](https://github.com/bringes/date-deschise-plugin/issues),
or write to hello@bunnytech.app. To add a dataset to the directory, see
[date-deschise.bunnytech.app/contribute](https://date-deschise.bunnytech.app/contribute).
