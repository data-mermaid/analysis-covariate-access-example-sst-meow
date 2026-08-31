# Accessing covariates and comparing them to MERMAID surveys

Worked examples showing how environmental covariates can be extracted with the
[`mermaidrcovariates`](https://data-mermaid.github.io/mermaidr-covariates/) package and
compared against coral reef survey data held in [MERMAID](https://datamermaid.org/).

Each document is a self-contained, reproducible example written for the MERMAID Analysis
Hub. They start from publicly available MERMAID benthic data, attach one or more
covariates to the survey locations and dates, and produce interactive figures.

## Rendered documents

| Document | Focus |
| --- | --- |
| [Coral cover and sea surface temperature across marine realms](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/coral-cover-covariates-analysis.html) | Global surveys, annual maximum SST, and Marine Ecoregions of the World (MEOW) realms |
| [Coral cover and Degree Heating Weeks in East Africa](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/coral-cover-covariates-DHW-East-Africa.html) | Kenya and Tanzania surveys and heat stress exposure |
| [Mapping survey sites by DHW exposure](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/map-comparison-DHW-East-Africa.html) | Two ways of mapping the same data, with plotly and with leaflet |
| [Index page](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/) | Links to all of the above |

The rendered HTML in `docs/` is what GitHub Pages serves, so it is committed to the
repository rather than ignored.

## The examples

### 1. Coral cover vs. maximum sea surface temperature, by marine realm

`analysis/coral-cover-covariates-analysis.qmd`

Uses global public benthic sample events. Each survey is matched to the MEOW realm it sits
in, then daily SST is extracted for the year preceding it.

- **Vector covariate:** `meow_boundaries`, attached with `attach_covariate_data()`, which
  matches each sample event to the polygon containing it in one call. Only the `REALM`
  column is requested.
- **Raster covariate:** `daily_sst`, with `n_days = 365`, `radius = 1000` m and
  `spatial_stats = "mean"`, then reduced with `summarise_zonal_statistics("max")`.
  The resulting variable is the **hottest day** of the year before each survey, spatially
  averaged within 1 km - an annual maximum, not an annual mean.
- Requests are sent in batches of 50 sample events, each batch summarised before the next
  is fetched, with a row-by-row fallback if a batch fails. Summarising inside the loop
  matters: at 365 days across ~5,000 surveys the raw daily series is roughly 1.8 million
  rows, and the reduction is per survey so the result is identical either way.
- **Outputs:** a hard coral cover histogram; the daily series and its summarised value for
  a worked example; an overall coral-cover-vs-SST scatter with correlation; the same
  faceted by realm; per-realm correlations and linear model summaries; and an interactive
  multi-realm plot with regression lines.

### 2. Coral cover vs. Degree Heating Weeks in East Africa

`analysis/coral-cover-covariates-DHW-East-Africa.qmd`

Narrows the same MERMAID export to Kenya and Tanzania and looks at accumulated heat stress
in the month before each survey.

- **Raster covariate:** `daily_dhw`, with `n_days = 30`, `radius = 1000` m and
  `spatial_stats = "mean"`, reduced with `summarise_zonal_statistics("max")` to the peak
  DHW each survey was exposed to.
- Thresholds and colours are defined once in the setup chunk (`alert_1`, `alert_2`,
  `col_red`, `col_amber`, `col_green`), so every figure uses the same values. 4 and 8 DHW
  are NOAA Coral Reef Watch Bleaching Alert Levels 1 and 2.
- **Outputs:** a hard coral cover histogram; a DHW histogram restricted to surveys
  exceeding 4 DHW; a coral-cover-vs-DHW scatter with the 4 and 8 DHW thresholds shaded;
  the same faceted by year for years that saw heat stress; and an interactive map of the
  survey sites with a dropdown to isolate a single survey year.

### 3. Mapping the same data two ways

`analysis/map-comparison-DHW-East-Africa.qmd`

A short supporting document rather than a new analysis. It reads the cached DHW extraction
from example 2 and maps those survey sites twice - once with **plotly** (`scattermapbox`)
and once with **leaflet** - using identical data and the same DHW colour banding, so the
two libraries can be compared directly.

Both maps carry a year selector, built quite differently: plotly needs one trace per year
plus hand-assembled `visible` vectors driven by an `updatemenus` dropdown, while leaflet
gets the same behaviour from `group =` and a single `addLayersControl()` call. The document
also notes which basemaps are free to use - CARTO and Stadia now require an API key, while
OpenStreetMap and the Esri basemaps do not.

Because it reads the cache rather than the API, it renders in seconds.

## Covariate access pattern

The two analysis documents use three functions from `mermaidrcovariates`:

1. [`list_covariates()`](https://data-mermaid.github.io/mermaidr-covariates/reference/list_covariates.html)
   to see what is available. Covariates are referred to by **id** (`daily_sst`,
   `daily_dhw`, `meow_boundaries`) rather than by title, since titles have changed
   upstream before.
2. [`get_zonal_statistics()`](https://data-mermaid.github.io/mermaidr-covariates/reference/get_zonal_statistics.html)
   for raster covariates. It returns one row per day per survey for the chosen window, so
   the result is a short time series per sample event rather than a single number.
3. [`summarise_zonal_statistics()`](https://data-mermaid.github.io/mermaidr-covariates/reference/summarise_zonal_statistics.html)
   to collapse each series to one value per survey - here the maximum.

Vector covariates take a different route:
[`attach_covariate_data()`](https://data-mermaid.github.io/mermaidr-covariates/reference/attach_covariate_data.html)
joins by location in a single call, with
[`list_datasets_for_covariate()`](https://data-mermaid.github.io/mermaidr-covariates/reference/list_datasets_for_covariate.html)
to see which columns are available. Note that its `columns` argument currently accepts
only **one** column at a time; passing a vector fails with "the condition has length > 1".

## Repository structure

```
analysis/
  _quarto.yml                                  Quarto project; renders to ../docs
  index.qmd                                    Landing page linking the documents
  coral-cover-covariates-analysis.qmd          Example 1 (SST and MEOW realms)
  coral-cover-covariates-DHW-East-Africa.qmd   Example 2 (DHW, East Africa)
  map-comparison-DHW-East-Africa.qmd           Example 3 (plotly vs leaflet mapping)
  footer.html                                  MERMAID logo footer, included in all
  2021MERMAIDLogoBlueTransp.png                Logo used by the footer
  draft/                                       Earlier working version of example 2
data/                                          Cached data (see below)
docs/                                          Rendered HTML, published via GitHub Pages
```

`data/TestPtsCovariates.csv` and `data/realms.RData` are earlier working files and are not
read by any of the current documents.

## Data and caching

None of the documents ship the data they analyse. Each wraps its API calls in a
cache-or-download pattern: if the expected `.rds` file exists in `data/` it is read from
disk, otherwise the data is pulled and then saved there for next time. Set
`force_download <- TRUE` in a setup chunk to bypass the cache.

Three cache files are used:

| File | Written by | Notes |
| --- | --- | --- |
| `mermaid_summ_ses.rds` | either analysis document | The global, unfiltered MERMAID export; shared between them |
| `coralMeowSst_365days_1000m.rds` | example 1 | Named after the request parameters, not the row count |
| `coral_data_dhw_30days.rds` | example 2 | Also read by example 3 |

Cache paths are built with `here::here("data", ...)`. Each document calls `here::i_am()`
in its setup chunk first, and that call is **load-bearing**: `_quarto.yml` lives in
`analysis/`, which makes that folder look like a project root, so during a render
`here::here()` would otherwise resolve to `analysis/` instead of the repository root and
the caches would not be found.

The cache files are listed in `.gitignore` (`data/*.rds`) and are **not** committed, so a
fresh clone downloads them on its first render. Budget for that: the 30-day DHW extraction
takes roughly 15 minutes, and the global 365-day SST extraction took about 15 hours.
Subsequent renders read from the cache and are near-instant. On a fresh clone, render
example 2 before example 3, since example 3 reads the cache example 2 creates.

## Prerequisites

R 4.5 or newer, [Quarto](https://quarto.org/), and:

```r
install.packages(c("here", "tidyverse", "plotly", "DT", "janitor", "leaflet",
                   "duckdb", "duckspatial"))

# not on CRAN
remotes::install_github("data-mermaid/mermaidr")
remotes::install_github("data-mermaid/mermaidr-covariates")
```

The R version matters. `attach_covariate_data()` reads the covariate parquet through
DuckDB, and the file uses native spatial geometry with CRS type modifiers, which needs
**duckdb 1.5 or later**. CRAN only builds Windows binaries for the current and previous R
releases, so on older R you are capped at duckdb 1.2.1 and the vector covariate example
will fail with `Type 'GEOMETRY' does not take any type modifiers`.

Not every document needs everything: `leaflet` is only used by example 3, and `janitor`
only by example 2. All documents use publicly available MERMAID data, so no API token is
needed.

## Reproducing

Open the `.Rproj` file, then either render a single document from RStudio, or from a
terminal at the repository root:

```
quarto render analysis/coral-cover-covariates-DHW-East-Africa.qmd
```

Output is written to `docs/`, as configured in `analysis/_quarto.yml`.

Two tips for the long first run. Chunk output is captured by knitr during a render, so
progress messages only appear in the finished HTML - to watch a long download live, run
the chunks interactively instead and render afterwards, when it becomes a cache hit. And
before any long render, this catches syntax errors in seconds:

```r
tmp <- tempfile(fileext = ".R")
knitr::purl("analysis/coral-cover-covariates-analysis.qmd", output = tmp, quiet = TRUE)
invisible(parse(tmp))
unlink(tmp)
```

## License

Released under the GNU Affero General Public License v3.0. See [LICENSE](LICENSE).
