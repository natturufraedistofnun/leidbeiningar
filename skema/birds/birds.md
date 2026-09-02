# Bird monitoring data — accessing it with SQL and R

A practical guide for nature-centre staff who want to pull **bird monitoring data** out of
Náttúrufræðistofnun's database and analyse it in R (or any tool that speaks SQL), instead of copy-pasting
from the Monitoring application.

> **Before you start:** read the general data-access intro first — it explains how to connect to the
> database (server, database name, login) and the basics of running a query in R. This guide assumes you
> already have a working connection object called `con`. A minimal reminder is included below.

---

## 1. What the bird monitoring data looks like

Bird data is collected as **field observations**. On a visit to a site, an observer records the counting
conditions and then, for each species seen, a **count** (totals, sex, age, pairs, and so on). Nests are
recorded separately and revisited over the season.

There are **two ready-made views** built for exactly this — they do the joins for you and are the easiest
place to start:

| View | One row is… | Use it for |
|---|---|---|
| `monitoring.site_bird_count_v` | one **species count** within one field visit | how many of each species, where, when, by whom |
| `monitoring.site_bird_nest_monitoring_v` | one **nest monitoring event** | nest contents and outcomes over the season |

**Two things to know up front:**

1. **These views only contain the "monitoring of vulnerable habitats" project.** They are pre-filtered to
   that project, which is almost certainly the data you want. (If you ever need bird data from *other*
   projects, see [Power users](#7-power-users-the-base-tables) at the end.)
2. **One visit produces many rows.** If 10 species were counted on a visit, that visit appears as 10 rows in
   `site_bird_count_v` (one per species). So when you want a total, **add the rows up** — don't assume one
   row per visit.

---

## 2. Connecting (quick reminder)

```r
library(DBI)
library(RPostgres)

con <- dbConnect(
  RPostgres::Postgres(),
  dbname   = "nitest",
  host     = "postgresql.natt.local",
  port     = 5432,
  user     = "your_username",       # same login as the Monitoring app
  password = "your_password"
)
```

Everything below runs against `con`. When you're done: `dbDisconnect(con)`.

---

## 3. Step one — find your site

Every query is filtered by a **site id** (`monitoring_site_id`). Look yours up by name first. Sites can be
hierarchical (a larger area with sub-sites), so a search may return several:

```r
dbGetQuery(con, "
  SELECT s.id, s.name, s.region, p.name AS parent_site
  FROM monitoring.site s
  LEFT JOIN monitoring.site p ON p.id = s.parent_site_id
  WHERE s.name ILIKE '%hérað%'
  ORDER BY s.name")
```

Note the `id` you need. In the examples below we use **188** ("Úthérað í Múlaþingi") — replace it with yours.

---

## 4. Step two — pull bird counts

The workhorse is `monitoring.site_bird_count_v`. Here are the columns you'll use most:

- **When / where:** `field_date`, `year`, `latitude`, `longitude`, `monitoring_site_id`
- **The count:** `total`, `males`, `females`, `pairs`, `adults`, `juveniles`, `chicks`, `nests`, `present`,
  `breeding`
- **Species:** `taxon_id` (join to get the name — see step three)
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
> site (an internal sub-unit), **not** the monitoring site — don't filter on it by mistake.

---

## 5. Step three — species names

Species are stored as a `taxon_id`. Join **`natura.taxon`** to get the Latin name (`taxon.name`) — this is
the central taxonomy used across all of NÍ's data, and what analysis normally uses:

```sql
LEFT JOIN natura.taxon t ON t.id = c.taxon_id      -- t.name = Latin name
```

If you also want the **Icelandic** name, add the preferred-name view (it returns the Icelandic name where
one exists):

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

**Total counted per species, for one site, one year** — let the database do the summing:

```r
by_species <- dbGetQuery(con, "
  SELECT t.name AS species, sum(c.total) AS total_counted
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024
  GROUP BY t.name
  ORDER BY total_counted DESC NULLS LAST")
```

**One species over time (a simple trend):**

```r
whimbrel <- dbGetQuery(con, "
  SELECT year, sum(total) AS total_counted
  FROM monitoring.site_bird_count_v c
  JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND t.name = 'Numenius phaeopus'
  GROUP BY year ORDER BY year")

plot(whimbrel$year, whimbrel$total_counted, type = "b",
     xlab = "Year", ylab = "Whimbrel counted")
```

**Save to a file** (Excel opens CSV directly; `UTF-8` keeps Icelandic characters correct):

```r
write.csv(by_species, "bird_counts_2024.csv", row.names = FALSE, fileEncoding = "UTF-8")
```

**Put observations on a map with `sf`.** Bird records store position as plain `latitude`/`longitude` numbers
(degrees, WGS84), so build the points yourself:

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

For nests, use `monitoring.site_bird_nest_monitoring_v` — one row per **nest monitoring event** (a nest can
be visited several times in a season). Useful columns: `field_date`, `nest_number`, `taxon_id`,
`no_of_eggs`, `max_no_of_eggs`, `hatched`, `no_of_chicks_hatched`, `egg_stage`,
`date_of_hatching_or_failure`, `reason_for_failure`, and the per-visit `monitoring_no_of_eggs` /
`monitoring_no_of_chicks` / `date`.

```r
nests <- dbGetQuery(con, "
  SELECT n.field_date, t.name AS species, n.nest_number,
         n.no_of_eggs, n.hatched, n.no_of_chicks_hatched, n.reason_for_failure
  FROM monitoring.site_bird_nest_monitoring_v n
  LEFT JOIN natura.taxon t ON t.id = n.taxon_id
  WHERE n.monitoring_site_id = 188
  ORDER BY n.field_date")
```

---

## 8. Power users — the base tables

The two views cover almost everything, but if you need more (for example bird data from **other projects**,
or fields the views don't expose) you can query the base `birds` schema directly:

- `birds.observation` → `birds.observation_taxon` (the per-species counts) → `natura.taxon` (species).
- `birds.nest` → `birds.nest_monitoring` → `birds.nest_monitoring_event`.
- Observers: `birds.observation_staff` → `natura.staff`.
- A bird observation links to a monitoring site indirectly:
  `observation.site_id → birds.site.area_id → birds.area.monitoring_site_id`.

The base tables contain **all** bird projects. To reproduce exactly what the monitoring views show, keep only
the monitoring project:

```sql
WHERE observation.project_id IN (SELECT id FROM birds.project WHERE number = 13098)
```

---

## 9. Good to know

- **These are read-only queries.** You cannot change or delete anything through R/SQL — it's safe to explore.
- **One row = one species count.** Sum `total` (with `GROUP BY`) to get totals; don't count rows.
- **Blanks are `NA`.** Not every field is filled in on every visit, so expect missing values.
- **`monitoring_site_id`, not `site_id`,** identifies the monitoring site (see §4).
- **Coordinates are plain numbers** (`latitude`/`longitude`, WGS84 degrees), not a spatial column — build
  geometry in R with `sf` as shown, or in SQL if you prefer.
- **Only the "monitoring of vulnerable habitats" project** is in the ready-made views (§8 for the rest).

---

## Getting help

If a column is unclear, a query returns something unexpected, or you need data the views don't cover,
contact Náttúrufræðistofnun — we're happy to help, and your questions help us improve these guides.
