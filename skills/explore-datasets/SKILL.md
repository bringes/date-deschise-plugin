---
name: explore-datasets
description: >-
  Find and query Date deschise — 30+ Romanian open datasets (plus European and global sets mirrored
  beside them) published as sorted Parquet and one DuckLake catalog on a public bucket, queried with
  DuckDB SQL over plain HTTPS, no key or account. Use when a question could be answered from public
  data: Romanian companies and the trade register, public budgets and spending, procurement,
  elections, laws in Monitorul Oficial, the power grid and energy market, exchange rates, air
  quality, weather, roads, tourism, timber permits, INS TEMPO or Eurostat statistics, census,
  earthquakes, buildings and maps — or when the user names "date deschise", "datepublice" or "the
  data directory", or asks which datasets exist.
---

# Exploring Date deschise

Date deschise is a directory of open datasets. Every table lives in one DuckLake catalog on a
public bucket, and DuckDB reads it over HTTPS: there is no API to authenticate against, no rate
limit to negotiate and no HTML worth scraping. This skill is the route from a question to a
checked, cited answer. When the answer has more than about five numbers, finish with the
`build-data-report` skill.

## The addresses

| What | Where |
| --- | --- |
| Every dataset, one line each, and how the directory works | `https://date-deschise.bunnytech.app/llms.txt` |
| One dataset in full — tables, every column, sort order, licence, SQL known to run | `https://date-deschise.bunnytech.app/datasets/<id>/llms.txt` |
| The catalogue as JSON — ids, tables, row counts, last ingest | `https://date-deschise-s3.bunnytech.app/public/index.json` |
| The whole directory as one DuckLake catalog | `ducklake:https://date-deschise-s3.bunnytech.app/catalog/catalog.ducklake` |
| A browser SQL console over every dataset | `https://date-deschise.bunnytech.app/playground?sql=<url-encoded SQL>` |
| The in-depth query guide (time travel, whole-catalogue search, maps) | `https://date-deschise.bunnytech.app/skills/query-datasets/SKILL.md` |

Add `?lang=ro` or `?lang=en` to either `llms.txt` for that language. The dataset list changes as
datasets are added, so read it live: never answer "we have no data on X" from memory.

## 1. Pick an engine for where you are running

You need **DuckDB 2.0 or newer**. The catalog is written by DuckDB 2.0, and 1.x cannot open it;
it fails with a storage-version error that looks like a corrupt file. Check before anything else:

```bash
duckdb -c "SELECT version()"                                # CLI
python3 -c "import duckdb; print(duckdb.__version__)"       # Python
```

Then use the first of these that applies:

- **A shell with network access** (Claude Code, Codex, Cowork, a terminal): use the DuckDB CLI or
  Python package. If neither is installed or the version is below 2.0, install it with what the
  machine already uses — `pip install "duckdb>=2.0"`, `brew install duckdb` or
  `winget install DuckDB.cli`. Ask the user before installing anything. Always pass `-c` to the CLI:
  a bare `duckdb` waits on stdin, which from a tool call looks like a hang.
- **A code sandbox whose network is restricted** (code execution in a chat app): `pip install duckdb`
  usually works there, but the bucket may be blocked, which shows up as a proxy 403, a DNS failure
  or "host not allowed". Tell the user that querying needs `date-deschise.bunnytech.app` and
  `date-deschise-s3.bunnytech.app` added to the sandbox's allowed domains, which is a setting they
  control. Meanwhile, carry on with the next route.
- **No code execution at all**: read the two `llms.txt` pages with your web-fetch tool, write the
  SQL, and give the user a playground link for each query (§5) so they can run it in one click.
  Never present numbers you did not get from a query that actually ran.

## 2. Find the dataset

1. **Read `https://date-deschise.bunnytech.app/llms.txt`.** It lists every dataset with its
   tables, row counts and as-of date. Skip it only when the dataset is obvious.
2. **Read that dataset's `https://date-deschise.bunnytech.app/datasets/<id>/llms.txt` before
   writing any SQL.** It is generated from the dataset's manifest, so it cannot describe a column
   that does not exist. It gives every column's meaning and unit, the physical sort order, the
   write mode, the licence and example queries known to run as printed. Writing SQL without it is
   how you get binder errors and plausible-looking wrong answers.

When you do not know which dataset or column holds something, search every column in the catalogue
instead of guessing. "Search every text column" in the query guide shows the SQL that does it. It
scans every column of every table in a few seconds.

## 3. Attach once, look, then answer

```sql
INSTALL ducklake; LOAD ducklake; INSTALL httpfs; LOAD httpfs;
ATTACH 'ducklake:https://date-deschise-s3.bunnytech.app/catalog/catalog.ducklake'
  AS datadirectory (READ_ONLY);
SELECT schema_name, table_name
FROM duckdb_tables() WHERE database_name = 'datadirectory' ORDER BY 1, 2;
```

- Tables are `datadirectory.<dataset_id_with_underscores>.<table>`: `bnr-exchange-rates` is the schema
  `datadirectory.bnr_exchange_rates`.
- `SHOW ALL TABLES` returns **nothing** for an attached DuckLake. Use `duckdb_tables()` as above.
- If you will run more than a couple of queries, download the catalog file once
  (`curl -fsSLo catalog.ducklake <catalog url>`) and attach the local copy. Only the catalog's
  metadata is copied, so the data stays remote and is still pruned, and every query starts faster.
- When a dataset's page prints a different setup (some tables are *federated*, queried live at their
  upstream source), use the setup it prints. Federated tables can take minutes per query, so filter
  them hard.

From Python the same statements go through `duckdb.connect().sql(...)`.

**Look before you aggregate.** `DESCRIBE` the table, read `LIMIT 20` rows (units, casing, codes and
empty strings are not in the schema), check `count(*)` and the range of the leading sort column.
Only then write the query that answers the question, and run it.

## 4. Rules that prevent wrong answers

- **Scan, never sample, before claiming absence.** A `LIMIT` shows a column's shape; it is not
  evidence about what a table contains. Every "there is no data on X" must come from a query that
  read every row of every column that could hold X.
- **Fold diacritics on both sides.** Romanian sources mix `Băicoi`, `BAICOI` and `Baicoi`:
  `WHERE lower(strip_accents(col)) LIKE '%baicoi%'`. Labels often carry codes (`'130981 ORAS
  BAICOI'`), so match with `LIKE '%…%'`, not equality, and also search the bare code.
- **Verify every hit.** Substring matches find numeric coincidences and same-named places in other
  counties. Check the county and code before you report one.
- **Separate "not published" from "not loaded".** Compare your `count(*)` with the row counts in
  the dataset's page before calling something a gap, and say which of the two you are reporting.
- **Filter on the leading sort column first.** The physical sort order plus Parquet statistics is
  the only index a lakehouse has. A predicate on the leading sort column skips whole files. Wrapping
  that column in a function or using `LIKE '%…'` on it prevents pruning, so the query reads every file.
- **Respect the write mode.** A `snapshot_history` table holds one row per entity *per snapshot*,
  so an aggregate without a snapshot-date filter double-counts.
- **Name your columns and aggregate in SQL.** `SELECT *` pays for every column, and a `GROUP BY`
  beats paging rows into your context by orders of magnitude.
- **Money is `DECIMAL`, large ids are `BIGINT`.** Do not cast them to floats to make arithmetic
  convenient. Round for display only.
- **Read the definition before naming a trend.** A dataset page quotes the source's methodology.
  When a definition constrains the reading (domicile vs. location, a boundary change), say so.

## 5. Deliver the answer with its provenance

Every answer carries:

- the dataset name and its page, `https://date-deschise.bunnytech.app/datasets/<id>`;
- the licence and the attribution line **exactly** as the dataset page states them;
- the SQL you ran;
- the date the data is as of (`lastIngestAt` / "data as of" on the dataset page).

Put a playground link beside every number you publish. The playground runs the query on arrival,
so the reader lands on the same rows:

```js
'https://date-deschise.bunnytech.app/playground?sql=' + encodeURIComponent(sql)
```

The playground accepts **one read-only statement** (`SELECT`, `WITH` or `DESCRIBE`). Tables must be
qualified as `datadirectory.<dataset>.<table>`, and it has no session macros or temp views. Drop
the `ATTACH`, keep the last statement, and inline any helper.

If no dataset covers the question, say so, name the tables you scanned to establish it, and point
at `https://date-deschise.bunnytech.app/contribute`. Do not answer from a web search and present the
result as though it came from here.

**More than about five numbers, a comparison, a trend, a ranking or a map?** Use the
`build-data-report` skill to turn the queries you have already run into a page.

## Going deeper

The query guide at `https://date-deschise.bunnytech.app/skills/query-datasets/SKILL.md` covers time
travel, reading one table's Parquet directly, the whole-catalogue value sweep, cross-dataset joins
and the cost model. It is long, so fetch it and read only the section you need. It describes an
installer that refreshes a locally installed copy. That installer is for people who use the guide
without this plugin; do not run it, because this plugin updates through its own directory.
