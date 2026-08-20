# OnSTOVE Starter Data Kit — Data Collection Methodology

**Notebook:** `OnSTOVE_Starter_Data_Kit_August_2026.ipynb` (v5.1)
**Purpose:** This document describes how each dataset in the kit is sourced, retrieved, and processed, so that outputs can be traced back to their origin and any manual fallback steps are clear.

The notebook runs in four steps: (1) install packages, (2) load downloader functions, (3) select a country and datasets and fetch them, (4) package everything into `<ISO3>_SDK.zip`. Every layer below is clipped to the target country's administrative boundary before being saved, unless noted otherwise.

---

## 1. Administrative boundary

- **Source:** GADM (via the `pygadm` package), admin level 0.
- **Method:** `pygadm.Items(admin=<ISO3>, content_level=0)` fetches the boundary polygon directly from the GADM API and reprojects it to EPSG:4326.
- **Reliability handling:** The GADM server frequently times out under load (notably from Colab IP ranges), so requests are retried up to 3 times with increasing backoff (5s, 10s, 15s). The boundary is cached in-session so it is fetched only once and reused by every other layer for clipping/bounding-box queries.
- **Manual override:** Users can supply their own boundary (e.g. a more accurate national shapefile) via `set_manual_boundary()`, which is then used in place of the GADM fetch for the rest of the session.
- **Output:** `Data/<ISO3>/Boundaries/<ISO3>_boundary.gpkg`

## 2. Population (2020)

- **Source:** WorldPop, `Global_2000_2020_1km` series.
- **Method:** Direct HTTPS download of the pre-built 1 km country GeoTIFF (`https://data.worldpop.org/GIS/Population/Global_2000_2020_1km/2020/<ISO3>/<iso3>_ppp_2020_1km_Aggregated.tif`). If the 1 km file is unavailable for a country, the notebook falls back to the 100 m resolution product from the equivalent `Global_2000_2020` folder.
- **Output:** `Data/<ISO3>/Population/<ISO3>_population_2020.tif`

## 3. Urban–rural status (settlement classification)

- **Source:** JRC Global Human Settlement Layer, GHS-SMOD (`GHS_SMOD_E2020_GLOBE_R2023A`, 1 km, Mollweide/Interrupted Goode Homolosine grid).
- **Method:** The full global GHS-SMOD raster (~160 MB) is downloaded once per session from the JRC data portal, cached locally, then clipped to the country boundary using `gdalwarp` (with a rasterio-based fallback if GDAL fails).
- **Note:** This layer also underpins the Travel Time calculation (see §8b): GHS-SMOD class 30 ("Urban Centre") is used as the Malaria Atlas Project's definition of a "large town."
- **Output:** `Data/<ISO3>/UrbanRural/<ISO3>_ghs_smod_2020.tif`

## 4. Medium-voltage (MV) grid lines

- **Source:** Gridfinder (CC-BY), hosted on Zenodo — a global medium-voltage line dataset derived from nighttime lights and OpenStreetMap.
- **Method:** The global Gridfinder GeoPackage (~724 MB) is downloaded once per session and cached. Before use, it is validated as a genuine, non-corrupted SQLite/GeoPackage file (checked via the SQLite magic header and a `PRAGMA quick_check`); a failed check triggers automatic re-download. The cached file is then bounding-box filtered and clipped to the exact country boundary with GeoPandas.
- **Output:** `Data/<ISO3>/MV_lines/<ISO3>_mv_lines.gpkg`

## 5. High-voltage (HV) transmission lines

- **Source:** OpenStreetMap, via the Overpass API, filtering strictly on the tag `power=line` (major/high-voltage transmission).
- **Method:** A bounding-box Overpass QL query is issued against the country's boundary extent. Because this endpoint is public and shared, the notebook rotates across five Overpass mirrors (overpass-api.de, overpass.kumi.systems, overpass.private.coffee, overpass.nchc.org.tw, overpass.osm.ch), retrying once per mirror on HTTP 429 and honouring the server's `Retry-After` header before moving to the next mirror. Returned ways are converted to LineStrings and clipped to the country boundary. If no HV lines are mapped in OSM for a country, an empty (but validly structured) GeoPackage is written so downstream steps do not break.
- **Important distinction:** `power=minor_line` (local/lower-voltage distribution) is deliberately excluded — this layer represents high-voltage transmission only, not the road network and not distribution lines. (An earlier notebook version mistakenly derived this layer from a filtered roads shapefile; that has been corrected.)
- **Manual fallback:** For countries where OSM's HV coverage is sparse, [EnergyData.info](https://energydata.info/) often hosts better-curated, voltage-graded national transmission datasets — noted in the generated Instructions file but not fetched automatically (no consistent per-country URL exists).
- **Output:** `Data/<ISO3>/HV_lines/<ISO3>_hv_lines.gpkg`

## 6. Nighttime lights (NTL)

- **Source:** WorldPop's VIIRS VNL v2.1 "No Vegetation Filtering" (NVF) product, 2020, 100 m resolution, served from `data.worldpop.org` (WorldPop's own repackaging of the EOG/Mines VIIRS product).
- **Method:** Direct, country-clipped GeoTIFF download (`.../<ISO3>/VIIRS/v1/nvf/<iso3>_viirs_nvf_2020_100m_v1.tif`), no authentication required. NVF (rather than a vegetation-filtered product) is used because it better represents electrification-related light emissions.
- **Output:** `Data/<ISO3>/NighttimeLights/<ISO3>_ntl_viirs_2020.tif`

## 7–8. Walking and motorized friction surfaces

- **Source:** Malaria Atlas Project (MAP) global friction surfaces, January 2020 vintage, served via `data.malariaatlas.org` (the project's OGC Web Coverage Service; the older `malariaatlas.org/geoserver` endpoint has been decommissioned).
- **Method:** A WCS 2.0.1 `GetCoverage` request is built per country using its bounding box, returning a bbox-clipped GeoTIFF directly — no global raster download is needed for these two layers.
- **Output:** `Data/<ISO3>/WalkingFriction/<ISO3>_walking_friction.tif`, `Data/<ISO3>/MotorizedFriction/<ISO3>_motorized_friction.tif`

## 8b. Travel time to nearest large town

- **Source:** Not an external dataset — this is calculated in-notebook, replicating the method behind MAP's "Accessibility to Cities" surface (Weiss et al. 2018, *Nature*, doi:10.1038/nature25181).
- **Method:**
  1. "Large towns" are defined per MAP's own criterion (built-up areas with ≥1,500 people/km² and ≥50,000 total population), which corresponds directly to GHS-SMOD class 30 ("Urban Centre"), already fetched in §3. If a country has no Urban Centre cells, the notebook falls back to class 23 ("Dense Urban Cluster").
  2. The Motorized Friction surface (§8) is converted from minutes-per-metre into a per-pixel cost (minutes to cross that cell), using each row's true cell width/height in metres (computed geodetically with `pyproj.Geod`, accounting for latitude).
  3. A multi-source Dijkstra least-cost accumulation (`skimage.graph.MCP_Geometric`) is run outward from every "town" cell simultaneously, producing, for every pixel, the cumulative travel time to its nearest large town.
- **Use in OnSTOVE:** This layer proxies travel time to LPG resupply points, since LPG distribution infrastructure tends to be located in or near larger towns.
- **Output:** `Data/<ISO3>/TravelTimeToTowns/<ISO3>_travel_time_to_towns.tif`

## 9. Forest cover (optical)

- **Source:** Hansen/GLAD Global Forest Change (GFC), `treecover2000`, v1.8, GFW/Google-hosted 10°×10° tiles (`storage.googleapis.com/earthenginepartners-hansen`).
- **Method:** The notebook determines which 10° tiles intersect the country's bounding box and downloads each (tiles are cached across countries to avoid re-downloading shared tiles). For countries spanning multiple tiles, a GDAL VRT (a lightweight XML index, not a full in-memory mosaic) is built with `gdalbuildvrt`, and `gdalwarp` streams the country clip directly off disk. This avoids loading multi-gigabyte tile arrays fully into memory, which was a known cause of Colab runtime crashes in earlier notebook versions.
- **Note:** Because of its size, this layer is provided as a separate "Forest Cover Fetch" cell that can be re-run independently, so a crash here does not take the rest of the download session down with it.
- **Output:** `Data/<ISO3>/ForestCover/<ISO3>_forest_cover_2020.tif`

## 9b. Forest cover, radar (optional)

- **Source:** JAXA ALOS PALSAR-2 FNF4 (Forest/Non-Forest, 4-class), 2021, 25 m resolution, accessed via Google Earth Engine (`JAXA/ALOS/PALSAR/YEARLY/FNF4`).
- **Method:** Requires a one-time Earth Engine authentication (browser-based login and a registered Google Cloud project — the only layer in the kit needing authentication, which is why it is optional). The country's bounding box is split into ~1°×1° tiles to stay under Earth Engine's synchronous download size limits; each tile is fetched via `getDownloadURL`, mosaicked with a GDAL VRT, and clipped to the country boundary. Output classes: 1 = Dense Forest, 2 = Non-dense Forest, 3 = Non-Forest, 4 = Water.
- **Why offered alongside Hansen:** PALSAR is L-band radar (not optical), so it sees through cloud/haze cover — useful in persistently cloudy tropical regions (e.g. the Congo Basin) where optical products like Hansen can have data gaps.
- **Licensing note:** JAXA prohibits commercial use of PALSAR/FNF data without consent; acceptable for research/planning use but should be flagged if outputs are used commercially.
- **Output:** `Data/<ISO3>/ForestCover/<ISO3>_forest_cover_palsar_fnf_2021.tif`

## 10. Livestock density

- **Source:** FAO Gridded Livestock of the World, GLW4 (2020 release), species: cattle, goats, sheep.
- **Method:** Downloaded directly from FAO's Google Cloud Storage bucket (`storage.googleapis.com/fao-gismgr-glw4-2020-data`), which serves the rasters without the authentication/HTML redirect gate present on FAO's public catalog (`data.apps.fao.org`). Each species' global raster is cached once, then clipped to the country boundary. A content-type check catches cases where a bad species code returns an HTML error page instead of a TIFF.
- **Output:** `Data/<ISO3>/Livestock/<ISO3>_livestock_{cattle,goats,sheep}.tif`

## 11. Relative Wealth Index (household wealth proxy)

- **Source:** Meta's Relative Wealth Index (RWI), distributed via the Humanitarian Data Exchange (HDX); WorldPop's poverty index rasters as a secondary fallback.
- **Method (in order of preference):**
  1. Check HDX for a per-country RWI CSV resource (many countries were added individually after the original 93-country release) via the CKAN `package_show` API.
  2. If not found, fetch the full 93-country RWI zip. HDX's direct download URL returns an HTTP 202 "consent gate" for programmatic access, so the notebook instead resolves the actual S3 storage URL via the CKAN `resource_show` API and downloads from there, extracting the matching country CSV by ISO2 code.
  3. If the country is not covered by either RWI route, fall back to WorldPop's poverty index raster (tried for 2020, 2015, then 2010).
  4. If none succeed, the notebook prints manual-download links (HDX RWI page and WorldPop poverty hub) rather than failing silently.
- **Output:** `Data/<ISO3>/WealthIndex/<ISO3>_relative_wealth_index.csv` (or `..._worldpop_poverty_index.tif` if the raster fallback was used)

## 12. Temperature (full Global Solar Atlas package)

- **Source:** Global Solar Atlas v2 (World Bank/ESMAP, via `api.globalsolaratlas.info`), full country GIS package.
- **Method:** Global Solar Atlas has no public listing API, but each country's package is served at a predictable static URL keyed to the country's display name. The notebook builds a list of candidate name spellings (from `pycountry`'s standard/common/official names, plus a manual override table for countries where GSA's naming diverges, e.g. `COD → Democratic-Republic-of-the-Congo`, `CIV → Ivory-Coast`) and tries each until one succeeds. The **entire** unzipped folder is kept — GHI, DNI, DIF, GTI, PVOUT, TEMP, ELE, OPTA rasters plus metadata — not just the temperature layer, matching what a manual download from the Global Solar Atlas website would provide.
- **Manual fallback:** If no name variant resolves, the notebook prints a link to [globalsolaratlas.info/download](https://globalsolaratlas.info/download) with instructions on where to place the manually downloaded folder.
- **Output:** `Data/<ISO3>/Temperature/<Name>_GISdata_LTAy_YearlyMonthlyTotals_GlobalSolarAtlas-v2_GEOTIFF/`

## 13. Blank socio-economic and techno-economic CSVs

- **Source:** Not downloaded — generated in-notebook from a fixed parameter list built into the OnSTOVE specification.
- **Method:** Two CSVs are written with the standard OnSTOVE parameter names, data types, and units pre-filled and the `Value` column left blank for the user to populate: a socio-economic file (population, electrification rates, mortality/morbidity rates, cost-of-illness, VSL, discount rate, weighting parameters, etc.) and a techno-economic file (per-stove costs, lifetimes, fuel properties, emissions intensities, and current urban/rural shares, across the ten cookstove technologies modelled — from traditional biomass through to electric cooking — plus grid generation-mix parameters for the Electricity option).
- **Output:** `Data/<ISO3>/Specifications/<ISO3>_socioeconomic.csv`, `Data/<ISO3>/Specifications/<ISO3>_technoeconomic.csv`

## 14. Specification templates (Technoeconomic and Prep-file workbooks)

- **Source:** Fixed, blank Excel workbook templates (including "Read Me" and "Parameter Legend" tabs), embedded directly in the notebook as base64-encoded strings.
- **Method:** These are not downloaded from any external source and are not country-specific — the same template applies to any country. They are decoded and written to disk locally, so this step has no internet dependency and cannot fail due to a network or source-availability issue.
- **Output:** `Data/<ISO3>/Specifications/<ISO3>_technoeconomic_TEMPLATE.xlsx`, `Data/<ISO3>/Specifications/<ISO3>_prep_file_TEMPLATE.xlsx`

## Layer requiring manual collection

| Layer | Why it's manual |
|---|---|
| **Electricity transformers** | No public API provides a consistent, country-by-country transformer dataset. Users are directed to [EnergyData.info](https://energydata.info/) or their own national utility data. This instruction is written automatically into the `Instructions.txt` file packaged with the output zip. |

---

## Packaging

Once all selected layers have been fetched for a country, Step 4 zips the entire `Data/<ISO3>/` folder into `<ISO3>_SDK.zip` and auto-generates an `Instructions.txt` covering: which datasets succeeded/failed in that run, the manual steps still required (filling in the specification CSVs/templates, sourcing transformer data), and a link to the OnSTOVE documentation.

## General reliability patterns used throughout

- **Boundary caching:** The GADM boundary is fetched once per session and reused by every downstream layer, rather than being re-fetched per dataset.
- **Retry/backoff and multi-mirror logic:** Applied to GADM and Overpass, both of which are public, rate-limited, or occasionally unstable services.
- **Streaming/VRT-based clipping for large rasters:** Used for Gridfinder, Hansen forest cover, and PALSAR FNF, to avoid holding country-sized or global arrays fully in memory (a known cause of Colab crashes).
- **File-integrity checks before use:** Applied to Gridfinder (SQLite/GeoPackage validation) and the RWI zip (zip-signature check), so a corrupted or partial cached download is detected and re-fetched rather than silently propagating errors downstream.
