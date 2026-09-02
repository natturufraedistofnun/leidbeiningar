# Vegetation plots — data access

Directions for those who want to pull **vegetation plot data** — species cover in surveyed plots and their
subplots — out of Náttúrufræðistofnun's database. They are intended for anyone who already has a VPN
connection and the appropriate access rights (see Grunnleiðbeiningar). The data is in the plant schema
`plants`.

> **Before you start:** read Grunnleiðbeiningar first — they explain how to connect to the database (server,
> database name, login) and the basics of running a query in R. These directions assume an open connection
> called `con`. A short recap follows below.

---

## 1. What the data looks like

Vegetation data is collected in **plots** (*reitir*). A plot belongs to a **project**, is surveyed on a date,
and is divided into numbered **subplots** (*smáreitir*). In every subplot each species present is given a
**cover class** on the project's cover scale, and the cover of plant groups (vascular plants, mosses, lichens,
stones and so on) is recorded the same way. Subplot environment (soil, moisture, vegetation height) and
plot-level environment and classification (habitat type, plant community, coordinates) are recorded alongside.

Four tables carry this:

| Table | One row is… | Used for |
|---|---|---|
| `plants.plot` | one **survey of one plot** (plot number + date) | where, when, which project, habitat type, cover scale used |
| `plants.subplot` | one **subplot** in that survey | subplot environment |
| `plants.subplot_taxon` | one **species in one subplot** | species cover — the core data |
| `plants.subplot_group` | one **plant group in one subplot** | group cover (mosses, lichens, stones …) |

The chain is `plot` → `subplot` (on `plot_id`) → `subplot_taxon` and `subplot_group` (on `subplot_id`).

Four things to know up front:

1. **Data is selected by project.** Every plot has a `project_id` (→ `plants.project`): some twenty projects,
   from *Natura Island Land* (the largest) to small one-off surveys. Plots of the *Vöktun náttúruverndarsvæða*
   project also carry a monitoring site (`monitoring_site_id`), see section 9.
2. **Cover is a class on a scale, not a number.** `scale_value_id` → `plants.scale_value`: `code` is the
   recorded class (`+`, `2`, `15-25` …) and `value` a number the class stands for — a cover percentage on the
   Braun-Blanquet, Hults-Sernander and VöNá scales, an ordinal 1–3 on the *þekjuflokkar* scales. Six scales are
   in use and one project may use several, so `value` is only comparable within one scale; carry the scale
   along (`scale_value.scale_id` → `plants.scale`).
3. **Subplot `0` is not a subplot.** Almost every plot has a subplot numbered `0`, the *zero-plot*: it holds
   the species (and group cover) recorded for the plot **outside** the numbered subplots, and has no position
   or environment values. A plot's full species list includes it; per-subplot frequency or cover excludes it
   (`subplot_number <> '0'`).
4. **One row per species per subplot**, so a plot appears as many rows. Totals and frequencies are made with
   `GROUP BY`. A plot surveyed again gets a new `plot` row with the same `plot_number` and a new `field_date`.

---

## 2. Connecting (short recap)

```r
library(DBI)
library(RPostgres)

con <- dbConnect(
  RPostgres::Postgres(),
  dbname   = "nitest",
  host     = "postgresql.natt.local",
  port     = 5432,
  user     = "username",
  password = "password"
)
```

Everything below runs through `con`. When finished, disconnect: `dbDisconnect(con)`.

---

## 3. The project

Every query is filtered by a **project id** (`plot.project_id`), which is looked up by name first:

```r
dbGetQuery(con, "
  SELECT id, name, number, description
  FROM plants.project
  ORDER BY name")
```

The examples below use **28** (Vöktun náttúruverndarsvæða); replace it with the id found here.

Plots of that project also belong to a **monitoring site**, which is looked up the same way (section 9 uses
**96**, Brennisteinsalda í Landmannalaugum):

```r
dbGetQuery(con, "
  SELECT DISTINCT s.id, s.name, s.region
  FROM monitoring.site s
  JOIN plants.plot p ON p.monitoring_site_id = s.id
  WHERE s.name ILIKE '%landmannalaug%'
  ORDER BY s.name")
```

---

## 4. Plots — `plants.plot`

The most useful columns, grouped:

- **Identity:** `id`, `plot_number` (unique within a project and survey date; the same number recurs when a
  plot is resurveyed), `field_date`, `project_id`, `area` (free-text area name)
- **Classification:** `habitat_type_id` (→ `conservation.habitat_type`), `plant_community_1_id` … `_3_id`
  (→ `plants.plant_community`), `surface_type_id` (→ `plants.surface_type`), `monitoring_site_id`
  (→ `monitoring.site`, monitoring plots only)
- **Size and scales:** `plot_size_id` (→ `plants.plot_size`, width × length in m), `subplot_size_id`
  (→ `plants.subplot_size`, in cm), `subplot_taxon_scale_id` and `subplot_group_scale_id` (→ `plants.scale`)
- **Position:** `coordinates_1` (PostGIS point, EPSG:4326 — the plot's position), `coordinates_2` (a second
  point, the far end of transect-shaped plots, where recorded), the same as plain numbers in `latitude_1`,
  `longitude_1`, `latitude_2`, `longitude_2`, plus `altitude`, `slope`, `slope_aspect`
- **Environment:** `precipitation`, `temperature_year`, `temperature_january`, `temperature_july`,
  `nitrogen`, `carbon`, `ph`, `plot_area`
- **Text:** `comments`, `images`

Plots of one project with the lookups resolved:

```r
plots <- dbGetQuery(con, "
  SELECT p.id AS plot_id, p.plot_number, p.field_date, p.area, s.name AS site, ht.name AS habitat_type,
         sc.name AS cover_scale, st_y(p.coordinates_1) AS latitude, st_x(p.coordinates_1) AS longitude
  FROM plants.plot p
  LEFT JOIN monitoring.site s ON s.id = p.monitoring_site_id
  LEFT JOIN conservation.habitat_type ht ON ht.id = p.habitat_type_id
  LEFT JOIN plants.scale sc ON sc.id = p.subplot_taxon_scale_id
  WHERE p.project_id = 28
  ORDER BY p.field_date, p.plot_number")

head(plots)
```

The view `plants.v_all_plot` returns the same rows with every lookup already resolved (`project_name`,
`habitat_type_name`, `plant_community_1_name`, `taxon_scale_name`, `plot_size_width`, `plot_size_length` …) —
the quickest plot-level table:

```r
plots <- dbGetQuery(con, "
  SELECT id, plot_number, field_date, project_name, habitat_type_name,
         plant_community_1_name, taxon_scale_name, plot_size_width, plot_size_length, altitude
  FROM plants.v_all_plot
  WHERE project_id = 28
  ORDER BY field_date, plot_number")
```

---

## 5. Species cover — `subplot` and `subplot_taxon`

The core pull: every species in every subplot of a project, with its cover class:

```r
cover <- dbGetQuery(con, "
  SELECT p.plot_number, p.field_date, sp.subplot_number, t.name AS species,
         sv.code AS cover_class, sv.value AS cover_value
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_taxon st ON st.subplot_id = sp.id
  LEFT JOIN natura.taxon t ON t.id = st.taxon_id
  LEFT JOIN plants.scale_value sv ON sv.id = st.scale_value_id
  WHERE p.project_id = 28
  ORDER BY p.plot_number, p.field_date, sp.subplot_number, species")

head(cover)
```

- `subplot.subplot_number` is text (`'0'`, `'1'`, `'2'` …); `position_length` and `position_width` place the
  subplot within the plot.
- `subplot_taxon.taxon_id` is a **taxon id**, not a name (section 6).
- `subplot_taxon.scale_value_id` → `plants.scale_value`: `code` (the class as recorded) and `value` (the
  number it stands for). The scales and their classes:

```r
scales <- dbGetQuery(con, "
  SELECT s.name AS scale, sv.code, sv.value
  FROM plants.scale s
  JOIN plants.scale_value sv ON sv.scale_id = s.id
  ORDER BY s.name, sv.value")
```

---

## 6. Species names

The `taxon_id` column is a **taxon id**, not a name. `natura.taxon` needs to be joined for the Latin name
(`taxon.name`) — the central taxonomy used across all of Náttúrufræðistofnun's data:

```sql
LEFT JOIN natura.taxon t ON t.id = st.taxon_id      -- t.name = Latin name
```

For the **Icelandic** name (where one exists), the preferred-name view is added:

```r
names_is <- dbGetQuery(con, "
  SELECT p.plot_number, t.name AS latin_name, v.icelandic_name, sv.code AS cover_class
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_taxon st ON st.subplot_id = sp.id
  LEFT JOIN natura.taxon t ON t.id = st.taxon_id
  LEFT JOIN natura.taxon_preferred_icelandic_name_v v ON v.taxon_id = st.taxon_id
  LEFT JOIN plants.scale_value sv ON sv.id = st.scale_value_id
  WHERE p.project_id = 28 AND p.plot_number = 'Lh-S1'
  ORDER BY p.field_date, sp.subplot_number, latin_name")
```

---

## 7. Common recipes

**Species frequency in a project** — in how many plots and subplots each species occurs (subplot `0`
excluded; the database does the counting):

```r
frequency <- dbGetQuery(con, "
  SELECT t.name AS species, count(DISTINCT sp.plot_id) AS plots, count(*) AS subplots
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_taxon st ON st.subplot_id = sp.id
  JOIN natura.taxon t ON t.id = st.taxon_id
  WHERE p.project_id = 28 AND sp.subplot_number <> '0'
  GROUP BY t.name
  ORDER BY plots DESC, subplots DESC")
```

**The full species list of one plot**, with the number of subplots each species was found in and whether it
was recorded outside the subplots only:

```r
plot_species <- dbGetQuery(con, "
  SELECT t.name AS species,
         count(*) FILTER (WHERE sp.subplot_number <> '0') AS subplots_present,
         bool_or(sp.subplot_number = '0') AS outside_subplots
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_taxon st ON st.subplot_id = sp.id
  JOIN natura.taxon t ON t.id = st.taxon_id
  WHERE p.project_id = 28 AND p.plot_number = 'BA-S2'
  GROUP BY t.name
  ORDER BY t.name")
```

**A plot over time.** A resurveyed plot has one `plot` row per survey date, so grouping by `field_date` puts
the surveys side by side (here plot Lh-S1 at Leirhnjúkur í Kröflu, surveyed in 2020 and 2024). The scale is
carried along, because cover values only compare within a scale:

```r
resurvey <- dbGetQuery(con, "
  SELECT p.field_date, t.name AS species, sc.name AS scale,
         count(*) AS subplots_present, round(avg(sv.value), 1) AS mean_cover_where_present
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_taxon st ON st.subplot_id = sp.id
  JOIN natura.taxon t ON t.id = st.taxon_id
  JOIN plants.scale_value sv ON sv.id = st.scale_value_id
  JOIN plants.scale sc ON sc.id = sv.scale_id
  WHERE p.project_id = 28 AND p.plot_number = 'Lh-S1' AND sp.subplot_number <> '0'
  GROUP BY p.field_date, t.name, sc.name
  ORDER BY t.name, p.field_date")
```

**Who surveyed each plot** — `plants.plot_staff` holds one row per person, joined here into one column:

```r
staff <- dbGetQuery(con, "
  SELECT p.plot_number, p.field_date,
         (SELECT string_agg(s.name, ', ' ORDER BY s.name)
            FROM plants.plot_staff ps JOIN natura.staff s ON s.id = ps.staff_id
           WHERE ps.plot_id = p.id) AS staff
  FROM plants.plot p
  WHERE p.project_id = 28
  ORDER BY p.field_date, p.plot_number")
```

**Save to a file** (Excel opens CSV directly; `UTF-8` keeps Icelandic characters correct):

```r
write.csv(frequency, "plant_frequency_p28.csv", row.names = FALSE, fileEncoding = "UTF-8")
```

**Plots on a map with `sf`.** The plot position is a PostGIS point (`coordinates_1`), so `sf` reads it
directly:

```r
library(sf)
plots_sf <- st_read(con, query = "
  SELECT p.plot_number, p.field_date, p.coordinates_1
  FROM plants.plot p
  WHERE p.project_id = 28 AND p.coordinates_1 IS NOT NULL")

plot(st_geometry(plots_sf))
# st_write(plots_sf, "plots.gpkg")   # open in QGIS
```

---

## 8. Group cover, subplot environment and habitat types

**Group cover** — `plants.subplot_group` records, per subplot, the cover of plant groups and ground types
(`group_type.name`: *Háplöntuþekja*, *Mosaþekja*, *Fléttuþekja*, *Grýtniþekja* …) on the plot's group scale
(`plot.subplot_group_scale_id`):

```r
groups <- dbGetQuery(con, "
  SELECT p.plot_number, sp.subplot_number, g.name AS group_type, sv.code AS cover_class, sv.value AS cover_value
  FROM plants.plot p
  JOIN plants.subplot sp ON sp.plot_id = p.id
  JOIN plants.subplot_group sg ON sg.subplot_id = sp.id
  JOIN plants.group_type g ON g.id = sg.group_type_id
  LEFT JOIN plants.scale_value sv ON sv.id = sg.scale_value_id
  WHERE p.project_id = 28 AND p.plot_number = 'BA-S2'
  ORDER BY sp.subplot_number, g.name")
```

**Subplot environment** — the view `plants.v_subplot` returns one row per subplot with the lookups resolved
(`moisture_name`, `soil_type_name`, `topography_name`, `analyser_name`) next to the measurements
(`vegetation_height_1` … `_4`, `soil_depth`, `total_cover`, `biomass`, `soil_temperature_1` … `_3`, `ph`,
`permafrost`):

```r
environment <- dbGetQuery(con, "
  SELECT p.plot_number, v.subplot_number, v.position_length, v.position_width, v.moisture_name,
         v.soil_type_name, v.topography_name, v.vegetation_height_1, v.soil_depth, v.total_cover, v.analyser_name
  FROM plants.plot p
  JOIN plants.v_subplot v ON v.plot_id = p.id
  WHERE p.project_id = 28 AND p.plot_number = 'BA-S2'
  ORDER BY v.subplot_number")
```

**Habitat types** — `plot.habitat_type_id` points at the habitat typology in `conservation.habitat_type`
(`name`, `isl_code`, EUNIS codes):

```r
habitats <- dbGetQuery(con, "
  SELECT ht.name AS habitat_type, ht.isl_code, ht.eunis_2020_code, count(*) AS plots
  FROM plants.plot p
  JOIN conservation.habitat_type ht ON ht.id = p.habitat_type_id
  WHERE p.project_id = 28
  GROUP BY 1, 2, 3
  ORDER BY plots DESC")
```

---

## 9. Monitoring sites — `monitoring.site_plant_subplot_v`

Plots of the monitoring project carry a `monitoring_site_id`. For those, the ready-made view
`monitoring.site_plant_subplot_v` joins site → plot → subplot: one row per **subplot**, with `site_name`,
`habitat_type` and the subplot environment resolved. It exposes `subplot_id`, so species are joined on it:

```r
site_cover <- dbGetQuery(con, "
  SELECT v.plot_number, v.field_date, v.subplot_number, t.name AS species,
         sv.code AS cover_class, sv.value AS cover_value
  FROM monitoring.site_plant_subplot_v v
  JOIN plants.subplot_taxon st ON st.subplot_id = v.subplot_id
  LEFT JOIN natura.taxon t ON t.id = st.taxon_id
  LEFT JOIN plants.scale_value sv ON sv.id = st.scale_value_id
  WHERE v.site_id = 96
  ORDER BY v.field_date, v.plot_number, v.subplot_number, species")
```

The view contains only plots that have both a monitoring site and a habitat type; everything else is reached
through `plants.plot` (section 4). Filter on `site_id`.

---

## 10. Power users — the base tables and their keys

- `plot.project_id` → `project.id`; `plot.monitoring_site_id` → `monitoring.site.id`;
  `plot.habitat_type_id` → `conservation.habitat_type.id` (`habitat_type_id_old` → `plants.habitat_type`, the
  older classification); `plot.plant_community_1_id` … `_3_id` → `plant_community.id`;
  `plot.surface_type_id` → `surface_type.id`; `plot.plot_size_id` → `plot_size.id`;
  `plot.subplot_size_id` → `subplot_size.id`; `plot.subplot_taxon_scale_id` and
  `plot.subplot_group_scale_id` → `scale.id`; `plot.locality_id` → `locality.id`.
- `subplot.plot_id` → `plot.id`; `subplot.analyser_id` → `natura.staff.id`; `subplot.moisture_id`,
  `soil_type_id`, `topography_id` → the lookup tables of the same name.
- `subplot_taxon.subplot_id` → `subplot.id`; `subplot_taxon.taxon_id` → `natura.taxon.id`;
  `subplot_taxon.scale_value_id` → `scale_value.id`; `scale_value.scale_id` → `scale.id`.
- `subplot_group.subplot_id` → `subplot.id`; `subplot_group.group_type_id` → `group_type.id`;
  `subplot_group.scale_value_id` → `scale_value.id`.
- `plot_staff.plot_id` → `plot.id`, `plot_staff.staff_id` → `natura.staff.id`.
- Images: `plot_image` (`plot_id`), `subplot_image` (`subplot_id`). Trails at plots: `plot_trail` (`plot_id`)
  → `trail_dimension` (`trail_id`).
- Views: `v_all_plot`, `v_subplot` and `v_subplot_group` return the base rows with names resolved.
  `v_subplot_taxon` also joins species tags from the `factsheets` schema and returns one row per tag, so a
  species with several tags appears more than once — for counts use `subplot_taxon` directly.
- `plants_taxon` is the schema's species checklist, not a join target. `surtsey_locate` is a separate
  yearly species list for Surtsey, unrelated to the plots.

---

## 11. Good to know

- **These are read-only queries.** Nothing can be changed or deleted through R/SQL — exploring is safe.
- **One row = one species in one subplot.** Frequencies and totals are made with `GROUP BY`; do not count
  rows as plots.
- **Subplot `0` (the zero-plot) holds what was recorded outside the numbered subplots.** Include it for a
  plot's species list; exclude it (`subplot_number <> '0'`) for per-subplot frequency or cover.
- **Cover is a class:** `scale_value.code` is what was recorded, `scale_value.value` the number it stands for,
  and `value` only compares within one scale (section 5).
- **`taxon_id` is a taxon id**, not a name — join `natura.taxon` (section 6).
- **`plot_number` is unique only within a project and survey date.** A resurvey repeats the number with a
  new `field_date`; the plot's `id` is unique.
- **Empty fields come back as `NA`.**
- **Position is a PostGIS point** (`coordinates_1`, EPSG:4326), read directly with `sf` (section 7) or as
  `st_y()`/`st_x()` in SQL; `coordinates_2` exists for transect-shaped plots, and the same coordinates are in
  `latitude_1`/`longitude_1` as plain numbers. For GIS work save the `sf` object as a GeoPackage and open it
  in QGIS.

---

## Getting help

If a column is unclear, a query returns something unexpected, something is missing from these directions or
something is unclear, contact Náttúrufræðistofnun at <gagnagrunnar@natt.is>.
