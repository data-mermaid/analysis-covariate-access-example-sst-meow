# Accessing covariates and comparing them to MERMAID surveys

Worked examples showing how environmental covariates can be extracted with the
[`mermaidrcovariates`](https://data-mermaid.github.io/mermaidr-covariates/) package and
compared against coral reef survey data held in [MERMAID](https://datamermaid.org/).

Each document is a self-contained, reproducible example written for the MERMAID Analysis
Hub. All of them start from publicly available MERMAID benthic data, attach one or more
covariates to the survey locations and dates, and produce interactive figures.

## Rendered documents

| Document | Focus |
| --- | --- |
| [Coral cover and sea surface temperature across marine realms](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/coral-cover-covariates-analysis.html) | Global surveys, SST, and Marine Ecoregions of the World (MEOW) realms |
| [Coral cover and Degree Heating Weeks in East Africa](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/coral-cover-covariates-DHW-East-Africa.html) | Kenya and Tanzania surveys and heat stress exposure |
| [Mapping survey sites by DHW exposure](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/map-comparison-DHW-East-Africa.html) | Two ways of mapping the same data, with plotly and with leaflet |
| [Index page](https://data-mermaid.github.io/analysis-covariate-access-example-sst-meow/) | Links to all of the above |

The rendered HTML in `docs/` is what GitHub Pages serves, so it is committed to the
repository rather than ignored.

## The examples

### 1. Coral cover vs. sea surface temperature, by marine realm

`analysis/coral-cover-covariates-analysis.qmd`

Uses global public benthic sample events. Survey points are spatially joined to MEOW
realms, then daily SST is extracted for the year preceding each survey.

- Covariate (vector): MEOW boundaries, downloaded from the covariates STAC catalogue as a
  geoparquet file, joined with `sf`, then deleted again to save space.
- Covariate (raster): `Daily Sea Surface Temperature`, `n_days = 365`, `radius = 1000` m,
  `spatial_stats = "mean"`.
- Requests are sent in batches of 10 sample events, with a row-by-row fallback if a batch
  fails, so a single bad point does not abort the whole download.
- Outputs: a hard coral cover histogram, an overall coral-cover-vs-SST scatter with
  correlation, the same relationship faceted by realm, per-realm correlations and linear
  model summaries, and an interactive multi-realm plot with regression lines.

### 2. Coral cover vs. Degree Heating Weeks in East Africa

`analysis/coral-cover-covariates-DHW-East-Africa.qmd`

Narrows the same MERMAID export to Kenya and Tanzania and looks at accumulated heat stress
in the month before each survey.

- Covariate (raster): `Daily Global 5km Satellite Coral Bleaching Degree Heating Week`,
  `n_days = 30`, `radius = 1000` m, `spatial_stats = "mean"`. The covariate's full title is
  looked up by its `daily_dhw` id rather than typed out.
- `summarise_zonal_statistics("max")` reduces the 30-day series to the peak DHW each
  survey was exposed to.
- Outputs: a hard coral cover histogram, a DHW histogram restricted to surveys exceeding
  4 DHW, a coral-cover-vs-DHW scatter with the 4 and 8 DHW bleaching thresholds shaded,
  the same relationship faceted by year for years that saw heat stress, and an interactive
  map of the survey sites with a dropdown to isolate a single survey year.
- Thresholds and colours are defined once in the setup chunk (`alert_1`, `alert_2`,
  `col_red`, `col_amber`, `col_green`), so every figure uses the same values.

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

The two analysis documents follow the same three steps from `mermaidrcovariates`:

1. [`list_covariates()`](https://data-mermaid.github.io/mermaidr-covariates/reference/list_covariates.html)
   to see what is available and to look up a covariate's full title from its id.
2. [`get_zonal_statistics()`](https://data-mermaid.github.io/mermaidr-covariates/reference/get_zonal_statistics.html)
   to extract raster values around each survey location, for a chosen number of days
   before the survey date.
3. `summarise_zonal_statistics()` to reduce the daily series to a single value per survey
   (for example the maximum).

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
read by either of the current documents.

## Data and caching

Neither document ships the data it analyses. Both wrap their API calls in a
cache-or-download pattern: if the expected `.rds` file exists in `data/` it is read from
disk, otherwise the data is pulled and then saved there for next time.

Cache paths are built with `here::here("data", ...)`, so they resolve to this project's
`data/` folder whether you run chunks interactively, render the document, or work from the
R console.

The cache files are listed in `.gitignore` (`data/*.rds`) and are **not** committed, so a
fresh clone has to download them on its first render. Expect that to take a while - the
30-day DHW extraction for the East Africa example takes roughly 15 minutes; the global SST
example is larger still. Subsequent renders read from the cache and are near-instant.

## Prerequisites

R, [Quarto](https://quarto.org/), and:

```r
install.packages(c("here", "tidyverse", "mermaidr", "plotly", "DT", "janitor",
                   "rstac", "geoarrow", "arrow", "sf", "leaflet"))

# mermaidrcovariates is not on CRAN
remotes::install_github("data-mermaid/mermaidr-covariates")
```

Not every document needs everything. Only the SST example needs the spatial stack
(`rstac`, `geoarrow`, `arrow`, `sf`) - note that `sf` requires GDAL, GEOS and PROJ on your
system - and only the mapping comparison needs `leaflet`. The DHW example runs without
either.

All documents use only publicly available MERMAID data, so no API token is needed.

## Reproducing

Open the `.Rproj` file, then either render a single document from RStudio, or from a
terminal in the `analysis/` folder:

```
quarto render coral-cover-covariates-DHW-East-Africa.qmd
```

Output is written to `docs/`, as configured in `analysis/_quarto.yml`.

To force a fresh download instead of using the cache, delete the relevant file from
`data/`.

Note that `map-comparison-DHW-East-Africa.qmd` reads the cache created by
`coral-cover-covariates-DHW-East-Africa.qmd`, so render that one first on a fresh clone.

## License

Released under the GNU Affero General Public License v3.0. See [LICENSE](LICENSE).
