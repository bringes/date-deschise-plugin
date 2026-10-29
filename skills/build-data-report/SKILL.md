---
name: build-data-report
description: >-
  Turn answers from Date deschise (Romanian open data queried with DuckDB) into a professional,
  self-contained report page: headline finding, KPI tiles, ECharts charts, tables, a playground
  link and CSV/JSON download behind every figure, and a licence-and-SQL provenance footer, with
  optional live refresh through duckdb-wasm. Use after querying the datasets when the answer has
  more than about five numbers, a comparison, a trend, a ranking or a map, or when the user asks
  for a report, dashboard, one-pager, chart, artifact, canvas or an HTML page from the data.
---

# Building a report from Date deschise

A pasted markdown table is where an answer goes to be forgotten. A page can be read, shared,
re-filtered and checked. This skill builds that page from queries you **have already run and
checked** with the `explore-datasets` skill. The page is never the place you try a query for the
first time.

Start from `templates/report.html` in this skill's directory. It is a complete, working report (BNR
exchange rates since 2015). It is one file with no build step, renders in light and dark mode, holds
up at phone width, and its live refresh has been tested against the production bucket. Copy it,
then replace the `REPORT` object. Everything below explains the decisions the template encodes, so
you can keep them when you change it.

## 1. Choose where the page goes

| You are in | Deliver as | Rows |
| --- | --- | --- |
| Claude with artifacts (claude.ai chat, Cowork, Claude Code's Artifact tool) | An artifact. Re-publishing the same file updates it in place | **Embedded**, `live: false` |
| ChatGPT canvas, or any other sandboxed preview pane | A canvas/preview, **and** offer the `.html` file | **Embedded**, `live: false` |
| A local agent with a filesystem (Claude Code without artifacts, Codex, an IDE) | A `.html` file in the user's working directory. Open it (`open`, `xdg-open`, `start`) and give the full path | Embedded, `live: true` |
| A page the user will host on their own site | A `.html` file | Embedded, `live: true` |

**Why embedded rows are the baseline everywhere.** A sandboxed viewer (an artifact or a canvas)
blocks `fetch` to other hosts, so a page that only queries live renders *empty* there and shows the
reader no error. You already ran the SQL. Ship its result in `rows`, and treat live refresh as an
upgrade that may not arrive. The bucket itself allows any origin (`Access-Control-Allow-Origin: *`),
so a local file or a page on any normal website **can** refresh live.

## 2. Shape the data in SQL, not in the page

- **Aggregate first.** Embed hundreds of rows, a few thousand at most. A daily series over ten years
  becomes a monthly one. A ranking becomes its top 20 plus an "Other" row.
- **Return chart-ready types.** Dates as `VARCHAR` labels (`strftime(d, '%Y-%m')`), measures as
  rounded `DOUBLE` (`round(x, 2)::DOUBLE`), and counts as integers. The same SQL runs in the page through
  Arrow, where `DATE` and `DECIMAL` arrive as awkward objects, while these types arrive as plain
  values. Rounding for display is fine. Do the arithmetic on `DECIMAL` before you cast.
- **Write tables as `datadirectory.<dataset>.<table>`, one statement per figure.** The same string
  then runs in the DuckDB CLI after the `ATTACH`, in the playground link and in the page's live
  refresh. Use no `ATTACH`, macro or temp view inside it.
- **Produce `rows` by running exactly that SQL** and pasting the JSON verbatim:

  ```bash
  duckdb -json -c "ATTACH 'ducklake:https://date-deschise-s3.bunnytech.app/catalog/catalog.ducklake' AS datadirectory (READ_ONLY);
                   <the figure's SQL>"
  ```

  From Python: `[dict(zip(r.columns, row)) for row in r.fetchall()]` with `r = con.sql(sql)`, then
  `json.dumps(..., default=str)`. Never retype numbers by hand.

## 3. Fill in `REPORT`

The template's `<script>` has three marked parts. Edit only part 2 (and the `UI` strings if the
language changes):

- `title`: **the finding, stated as a fact with its number** ("A euro costs 19% more lei than in
  January 2015"), not the topic ("Exchange rates"). Compute the number from the rows, and re-check
  it after any change to the SQL.
- `lede`: one sentence on what is measured, over what period, and how to read it.
- `asOf`: the dataset's "data as of" date from its page, or `lastIngestAt` in `index.json`.
- `sources`: per dataset, its id, name, licence, licence URL and attribution, **copied exactly**
  from the dataset's `llms.txt`.
- `kpis`: three or four numbers that carry the story. Each one is computed from a figure's rows, so
  a live refresh updates it too.
- `figures`: each one has an `id`, a `kind`, a `title`, an optional `subtitle` (the unit) and
  `caption` (a caveat), its `sql` and its `rows`.
  - `line`: `x` plus up to four `series` (`{key, label}`). Optional `ranges` chips
    (`{label, last: n}`) re-slice the embedded rows, which is instant and works in a sandbox.
  - `bar`: a horizontal ranking from `x` and one series, sorted descending, labelled when there are
    15 bars or fewer.
  - `table`: `columns` (`{key, label, digits}`). Numbers are right-aligned in tabular figures.
- `lang` and `locale`: write the page in the reader's language. For a Romanian reader use `'ro'` and
  `'ro-RO'` (`1.234,56`) and translate the `UI` strings with it. Keep SQL, column keys and dataset
  ids in English, and reproduce labels published by the source exactly as published.

Keep `<meta charset="utf-8">`. Romanian labels (`Bistrița-Năsăud`, `Mureș`) turn into mojibake
without it, most visibly in a `file://` preview.

## 4. Chart standards

The template already applies these. Keep them when you add a figure:

- **Pick the form for the job.** Use a line for change over time, a sorted horizontal bar for a
  ranking, a KPI tile for a single number (never a one-bar chart), and a table when readers need
  exact values. Use a stacked area only for up to five parts of a whole. Pies and donuts are almost
  never the answer.
- **One y-axis.** Never a dual axis. Put two measures of different scales in two charts, or index both
  to 100 at the start.
- **Colour follows the entity.** EUR stays blue in every chart on the page. The categorical slots
  are `--series-1…4`, tuned for colour-blind separation in both themes, so add no fifth hue:
  fold the rest into "Other" or split into small multiples.
- **Bars start at zero.** A line may zoom to its range, but then say so in the caption.
- **Label selectively.** Use end labels on lines and values on short bar lists, never a number on
  every point. The tooltip and the table carry the rest.
- **Thin marks and quiet chrome.** Use 2px lines, hairline solid gridlines, no shadows on marks,
  and axis text in the muted token.
- **Format every number** with `fmt()` (it uses `Intl.NumberFormat` in the page's locale), with
  units in the subtitle, never repeated on every tick.

**Maps.** Never use a tile map (Leaflet, Mapbox, OpenStreetMap tiles): tiles, stylesheets and style
JSON are all blocked in a sandboxed viewer, and the result is a grey box. Draw an inline SVG
choropleth from boundaries you embed. The query guide's "Maps: inline SVG, never tiles" section
(`https://date-deschise.bunnytech.app/skills/query-datasets/SKILL.md`) gives the recipe: sources,
simplification, projection and name matching.

## 5. Provenance is part of the page

Every report carries:

- per figure, a **Data** menu offering *Open the query in the playground* (the exact SQL, one click
  to the same rows), *Copy SQL*, and *CSV* / *JSON* downloads with the row count shown;
- a **Method and sources** section that states each figure's SQL in full;
- a footer with each dataset's page, licence (linked) and attribution, plus the as-of date.

A number without its licence line and as-of date is not an answer anyone can use. Do not remove
any of these to save space.

**Downloads inside an artifact.** A sandboxed viewer makes `<a download>` inert, silently. The
template saves through the host's `downloads` capability when one exists (in Claude, publish the
artifact with the `downloads` capability enabled). Where no such capability exists, it copies the
file to the clipboard and says so. It offers CSV and JSON, which every host accepts.

## 6. Live refresh

`REPORT.live = true` shows *Refresh from the lakehouse*. When clicked, the page loads DuckDB 2.0
compiled to WebAssembly, attaches the catalog, re-runs every figure's SQL, re-renders the charts
and KPIs, and switches the badge from *Embedded data · as of …* to *Live from the lakehouse*. The
production run takes about seven seconds. If it fails, the embedded figures stay and the reader is
told why. A blocked fetch surfaces in DuckDB as "Cannot open database … read-only mode", which
reads like a missing file. Do not repeat that wording to the reader.

The `ENGINE` constants pin the DuckDB 2.0 engine the directory publishes beside its data, because
npm's own duckdb-wasm build cannot load the extensions this catalog needs. When you build a page,
compare them with the "In a browser: duckdb-wasm" section of the query guide and copy the current
values if they differ. Do not enable COOP/COEP headers on a page that refreshes live: they block
the cross-origin reads the data arrives on.

## 7. Check it before you hand it over

Do every step. Each catches a failure that has happened before:

1. **Open it in a real browser** (or a headless one) and read the console. Zero errors.
2. **Re-derive every number in the title, lede and KPIs** from the rows and compare them with your
   query output. A headline written before the final query is the most common wrong number.
3. **Check the footer.** Licence and attribution match the dataset page character for character,
   and the as-of date is right.
4. **Check both themes and a 375px width.** No horizontal page scroll, no clipped labels, and a
   wide table scrolls inside its own card.
5. **Check the diacritics.** `ă â î ș ț` render correctly.
6. **Click one playground link and one download**, and confirm the rows match the figure.
7. If the page is `live`, **click refresh once** and confirm the numbers match the embedded ones.

Then deliver it. Give an artifact's link, or a file's full path. In the reply, state the
finding in one or two sentences with its as-of date, so the answer survives even if the page is
never opened.
