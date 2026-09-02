# Bird monitoring — data access

Directions for those who want to pull **bird monitoring data** out of Náttúrufræðistofnun's database. They
are intended for anyone who already has a VPN connection and the appropriate access rights (see
Grunnleiðbeiningar). The data is in the bird schema `birds`; the ready-made views described here are in the
monitoring schema `monitoring`.

> **Before you start:** read Grunnleiðbeiningar first — they explain how to connect to the database (server,
> database name, login) and the basics of running a query in R. These directions assume an open connection
> called `con`. A short recap follows below.

---

## 1. What the data looks like

Bird data is collected as **field observations**. On a visit to a site the observer records the counting
conditions and then, for each species seen, a **count** (total, sex, age, pairs and so on). Nests are recorded
separately and revisited over the season.

Two ready-made views do the joins and are the easiest place to start:

| View | One row is… | Used for |
|---|---|---|
| `monitoring.site_bird_count_v` | one **species count** within one field visit | how many of each species, where, when, by whom |
| `monitoring.site_bird_nest_monitoring_v` | one **nest monitoring event** (a visit to a nest) | nest contents and outcomes over the season |

Two things to know up front:

1. **The views contain only project number 13098**, the monitoring project. Eleven sub-projects share that
   number (for example *Mófuglar*, *Vatnafuglar*, *Fjörur og grunnsævi*, *Vöktun kríu*, *Vöktun skúma*); the
   `project_id` column tells them apart (section 6). Bird data from other projects is reached through the base
   tables (section 8).
2. **One visit produces many rows.** If 10 species were counted on a visit, that visit appears as 10 rows in
   `site_bird_count_v`, one per species. Totals are therefore **summed**, never found by counting rows. A visit
   with no species recorded still appears once, with the count columns empty.

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
  user     = "username",           # same login as the Monitoring app
  password = "password"
)
```

Everything below runs through `con`. When finished, disconnect: `dbDisconnect(con)`.

---

## 3. The monitoring site

Every query is filtered by a **site id** (`monitoring_site_id`), which is looked up by name first. Sites can be
hierarchical (a larger area with sub-sites), so a search may return several:

```r
dbGetQuery(con, "
  SELECT s.id, s.name, s.region, p.name AS parent_site
  FROM monitoring.site s
  LEFT JOIN monitoring.site p ON p.id = s.parent_site_id
  WHERE s.name ILIKE '%hérað%'
  ORDER BY s.name")
```

The examples below use **188** (Úthérað í Múlaþingi); replace it with the id found here.

---

## 4. Bird counts — `site_bird_count_v`

The main view is `monitoring.site_bird_count_v`. The most useful columns, grouped:

- **When / where:** `field_date`, `year`, `latitude`, `longitude`, `monitoring_site_id`
- **The count:** `total`, `males`, `females`, `pairs`, `adults`, `juveniles`, `chicks`, `nests`, `present`,
  `breeding`
- **Species:** `taxon_id` — a **taxon id**, not a name (section 5)
- **Project:** `project_id` (→ `birds.project`, the sub-project)
- **Who:** `staff` (observer names, comma-separated)

```r
counts <- dbGetQuery(con, "
  SELECT c.field_date, c.year, t.name AS species,
         c.total, c.males, c.females, c.pairs, c.staff
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188
  ORDER BY c.field_date, species")

head(counts)
```

> **Filter on `monitoring_site_id`.** The view also has a column called `site_id`, but that is the *bird*
> site (an internal sub-unit), **not** the monitoring site.

---

## 5. Species names

The `taxon_id` column is a **taxon id**, not a name. `natura.taxon` needs to be joined for the Latin name
(`taxon.name`) — the central taxonomy used across all of Náttúrufræðistofnun's data:

```sql
LEFT JOIN natura.taxon t ON t.id = c.taxon_id      -- t.name = Latin name
```

For the **Icelandic** name (where one exists), the preferred-name view is added:

```r
counts_is <- dbGetQuery(con, "
  SELECT c.field_date, t.name AS latin_name, v.icelandic_name, c.total
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  LEFT JOIN natura.taxon_preferred_icelandic_name_v v ON v.taxon_id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024")
```

---

## 6. Common recipes

**Total counted per species, one site, one year** (the database does the summing):

```r
by_species <- dbGetQuery(con, "
  SELECT t.name AS species, sum(c.total) AS total_counted
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024
  GROUP BY t.name
  ORDER BY total_counted DESC NULLS LAST")
```

**One species over time** (here whimbrel, *Numenius phaeopus*):

```r
trend <- dbGetQuery(con, "
  SELECT year, sum(total) AS total_counted
  FROM monitoring.site_bird_count_v c
  JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND t.name = 'Numenius phaeopus'
  GROUP BY year ORDER BY year")

plot(trend$year, trend$total_counted, type = "b",
     xlab = "Year", ylab = "Whimbrel counted")
```

**Which sub-projects have data for a site** — `project_id` joined to `birds.project`:

```r
projects <- dbGetQuery(con, "
  SELECT p.name AS project, count(DISTINCT c.observation_id) AS visits,
         min(c.year) AS first_year, max(c.year) AS last_year
  FROM monitoring.site_bird_count_v c
  JOIN birds.project p ON p.id = c.project_id
  WHERE c.monitoring_site_id = 188
  GROUP BY p.name
  ORDER BY visits DESC")
```

**Save to a file** (Excel opens CSV directly; `UTF-8` keeps Icelandic characters correct):

```r
write.csv(by_species, "bird_counts_2024.csv", row.names = FALSE, fileEncoding = "UTF-8")
```

**Observations on a map with `sf`.** Bird records store position as plain `latitude`/`longitude` numbers
(degrees, WGS84), not as a geometry column, so the points are built in R:

```r
library(sf)
pts <- dbGetQuery(con, "
  SELECT field_date, taxon_id, total, latitude, longitude
  FROM monitoring.site_bird_count_v
  WHERE monitoring_site_id = 188 AND latitude IS NOT NULL AND longitude IS NOT NULL")

pts_sf <- st_as_sf(pts, coords = c("longitude", "latitude"), crs = 4326)
plot(st_geometry(pts_sf))
# st_write(pts_sf, "birds.gpkg")   # open in QGIS
```

---

## 7. Nest monitoring

Nests are in `monitoring.site_bird_nest_monitoring_v` — one row per **nest monitoring event** (a nest is
visited several times over a season, so it appears once per visit). The view is built from the field visits,
so **visits without nests appear too**, as rows with every nest column empty: filter on `nest_id IS NOT NULL`
to keep nests only.

Useful columns: the nest (`nest_number`, `taxon_id`, `no_of_eggs`, `max_no_of_eggs`, `hatched`,
`no_of_chicks_hatched`, `egg_stage`, `date_of_hatching_or_failure`, `reason_for_failure`) and the visit
(`date`, `monitoring_no_of_eggs`, `monitoring_no_of_chicks`, `monitoring_method`, `monitoring_event_id`).
The event id is what was observed on the visit — incubating, chicks hatched, nest robbed and so on; the name
is in `birds.nest_monitoring_event`.

Site 188 has no nest monitoring, so this example uses **28** (Ástjörn í Jökulsárgljúfrum):

```r
nests <- dbGetQuery(con, "
  SELECT n.nest_number, t.name AS species, n.date AS visit_date, e.name AS event,
         n.monitoring_no_of_eggs, n.monitoring_no_of_chicks, n.hatched, n.no_of_chicks_hatched
  FROM monitoring.site_bird_nest_monitoring_v n
  LEFT JOIN natura.taxon t ON t.id = n.taxon_id
  LEFT JOIN birds.nest_monitoring_event e ON e.id = n.monitoring_event_id
  WHERE n.monitoring_site_id = 28 AND n.nest_id IS NOT NULL
  ORDER BY n.nest_number, n.date")
```

---

## 8. Power users — the base tables

The two views cover almost everything, but bird data from **other projects**, or columns the views do not
expose, is reached in the base `birds` schema:

- `birds.observation` (the visit) → `birds.observation_taxon` (one row per species counted, `observation_id`)
  → `natura.taxon` (`taxon_id`).
- `birds.nest` (`observation_id`) → `birds.nest_monitoring` (`nest_id`; one row per nest visit, with `date`,
  `no_of_eggs`, `no_of_chicks`). Its `monitoring_event_id` → `birds.nest_monitoring_event`, the lookup of
  event types.
- Observers: `birds.observation_staff` (`observation_id`, `staff_id`) → `natura.staff`.
- Sub-project: `observation.project_id` → `birds.project` (`name`, `number`).
- The link to a monitoring site is indirect: `observation.site_id` → `birds.site.id`, `site.area_id` →
  `birds.area.id`, `area.monitoring_site_id` → `monitoring.site.id`.

The base tables contain **all** bird projects. To reproduce what the monitoring views show, only project
number 13098 is kept:

```r
base <- dbGetQuery(con, "
  SELECT o.field_date, p.name AS project, t.name AS species, ot.total
  FROM birds.observation o
  JOIN birds.project p ON p.id = o.project_id
  JOIN birds.observation_taxon ot ON ot.observation_id = o.id
  LEFT JOIN natura.taxon t ON t.id = ot.taxon_id
  JOIN birds.site s ON s.id = o.site_id
  JOIN birds.area a ON a.id = s.area_id
  WHERE a.monitoring_site_id = 188 AND p.number = 13098 AND o.year = 2024
  ORDER BY o.field_date")
```

Lookup tables for the coded columns: `activity`, `counting_method`, `observation_method`, `count_quality`,
`distance_range`, `family_composition`, `observation_category` (all in `birds`, joined on the matching
`<name>_id` column).

---

## 9. Good to know

- **These are read-only queries.** Nothing can be changed or deleted through R/SQL — exploring is safe.
- **One row = one species count.** Sum `total` (with `GROUP BY`) for totals; do not count rows. A visit with
  no species rows appears once, with the count columns empty.
- **Empty fields come back as `NA`.**
- **`monitoring_site_id`, not `site_id`,** identifies the monitoring site (section 4).
- **Coordinates are plain numbers** (`latitude`/`longitude`, WGS84 degrees); the bird data has no PostGIS
  geometry column. For GIS work the points are built as in section 6, or saved as a GeoPackage and opened in
  QGIS.
- **Only project number 13098** is in the ready-made views; the sub-project is in `project_id` (section 6),
  other projects are in the base tables (section 8).

---

## Getting help

If a column is unclear, a query returns something unexpected, something is missing from these directions or
something is unclear, contact Náttúrufræðistofnun at <gagnagrunnar@natt.is>.
