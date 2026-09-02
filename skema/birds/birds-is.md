# Fuglavöktun — aðgangur að gögnum

Leiðbeiningar fyrir þau sem vilja sækja **fuglavöktunargögn** úr gagnagrunni Náttúrufræðistofnunar.
Leiðbeiningarnar eru ætlaðar öllum sem þegar hafa VPN-tengingu og viðeigandi aðgangsheimildir
(sjá Grunnleiðbeiningar). Gögnin eru í fuglaskemanu `birds`; tilbúnu sýnirnar (views) sem hér er lýst eru í
vöktunarskemanu `monitoring`.

> **Áður en þú byrjar:** lestu fyrst Grunnleiðbeiningar — þær útskýra hvernig tengjast má gagnagrunninum
> (þjónn, gagnagrunnsnafn, innskráning) og grunnatriði þess að keyra fyrirspurn í R. Þessar leiðbeiningar gera
> ráð fyrir að þú sért þegar með virka tengingu sem heitir `con`. Stutt upprifjun fylgir hér að neðan.

---

## 1. Hvernig gögnin líta út

Fuglagögnum er safnað sem **vettvangsathugunum**. Í heimsókn á svæði skráir athugandi talningaraðstæður og
síðan, fyrir hverja tegund sem sést, **talningu** (heildarfjölda, kyn, aldur, pör o.fl.). Hreiður eru skráð
sérstaklega og heimsótt aftur yfir tímabilið.

Tvær tilbúnar sýnir sjá um samtengingarnar og eru auðveldasti staðurinn til að byrja:

| Sýn | Ein lína er… | Notuð fyrir |
|---|---|---|
| `monitoring.site_bird_count_v` | ein **tegundatalning** í einni vettvangsheimsókn | fjöldi hverrar tegundar, hvar, hvenær, af hverjum |
| `monitoring.site_bird_nest_monitoring_v` | einn **hreiðurvöktunaratburður** (heimsókn að hreiðri) | innihald og afdrif hreiðra yfir tímabilið |

Tvennt sem gott er að vita strax:

1. **Sýnirnar innihalda aðeins verkefnisnúmer 13098**, vöktunarverkefnið. Ellefu undirverkefni deila því
   númeri (t.d. *Mófuglar*, *Vatnafuglar*, *Fjörur og grunnsævi*, *Vöktun kríu*, *Vöktun skúma*); dálkurinn
   `project_id` greinir þau að (kafli 6). Fuglagögn úr öðrum verkefnum eru sótt í grunntöflurnar (kafli 8).
2. **Ein heimsókn myndar margar línur.** Ef 10 tegundir voru taldar í heimsókn birtist sú heimsókn sem 10
   línur í `site_bird_count_v`, ein á hverja tegund. Heildartölur eru því **lagðar saman**, aldrei fundnar með
   því að telja línur. Heimsókn þar sem engin tegund var skráð birtist samt einu sinni, með auða talningardálka.

---

## 2. Tenging (stutt upprifjun)

```r
library(DBI)
library(RPostgres)

con <- dbConnect(
  RPostgres::Postgres(),
  dbname   = "nitest",
  host     = "postgresql.natt.local",
  port     = 5432,
  user     = "notendanafn",        # sama innskráning og í Vöktunarforritinu
  password = "lykilord"
)
```

Allt hér að neðan keyrir gegnum `con`. Þegar það er búið þarf að aftengjast: `dbDisconnect(con)`.

---

## 3. Vöktunarsvæðið

Hver fyrirspurn er síuð eftir **svæðisauðkenni** (`monitoring_site_id`), sem fyrst er flett upp eftir nafni.
Svæði geta verið stigskipt (stærra svæði með undirsvæðum), svo leit getur skilað fleiri en einu:

```r
dbGetQuery(con, "
  SELECT s.id, s.name, s.region, p.name AS parent_site
  FROM monitoring.site s
  LEFT JOIN monitoring.site p ON p.id = s.parent_site_id
  WHERE s.name ILIKE '%hérað%'
  ORDER BY s.name")
```

Í dæmunum hér að neðan er notað **188** (Úthérað í Múlaþingi); í stað þess kemur auðkennið sem hér fannst.

---

## 4. Fuglatalningar — `site_bird_count_v`

Aðalsýnin er `monitoring.site_bird_count_v`. Gagnlegustu dálkarnir, flokkaðir:

- **Hvenær / hvar:** `field_date`, `year`, `latitude`, `longitude`, `monitoring_site_id`
- **Talningin:** `total`, `males`, `females`, `pairs`, `adults`, `juveniles`, `chicks`, `nests`, `present`,
  `breeding`
- **Tegund:** `taxon_id` — **tegundarauðkenni**, ekki nafn (kafli 5)
- **Verkefni:** `project_id` (→ `birds.project`, undirverkefnið)
- **Hver:** `staff` (nöfn athugenda, aðskilin með kommu)

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

> **Sía þarf eftir `monitoring_site_id`.** Sýnin hefur líka dálk sem heitir `site_id`, en það er *fugla*-svæðið
> (innri undireining), **ekki** vöktunarsvæðið.

---

## 5. Tegundanöfn

Dálkurinn `taxon_id` er **tegundarauðkenni**, ekki nafn. Samtengja þarf **`natura.taxon`** til að fá latneska
nafnið (`taxon.name`) — miðlæga flokkunarfræðin sem notuð er þvert á öll gögn Náttúrufræðistofnunar:

```sql
LEFT JOIN natura.taxon t ON t.id = c.taxon_id      -- t.name = latneskt nafn
```

Fyrir **íslenskt** nafn (þar sem það er til) er sýninni fyrir valið nafn bætt við:

```r
counts_is <- dbGetQuery(con, "
  SELECT c.field_date, t.name AS latin_name, v.icelandic_name, c.total
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  LEFT JOIN natura.taxon_preferred_icelandic_name_v v ON v.taxon_id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024")
```

---

## 6. Algeng dæmi

**Heildarfjöldi talinn á hverja tegund, eitt svæði, eitt ár** (gagnagrunnurinn leggur saman):

```r
by_species <- dbGetQuery(con, "
  SELECT t.name AS species, sum(c.total) AS total_counted
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024
  GROUP BY t.name
  ORDER BY total_counted DESC NULLS LAST")
```

**Ein tegund yfir tíma** (hér spói, *Numenius phaeopus*):

```r
trend <- dbGetQuery(con, "
  SELECT year, sum(total) AS total_counted
  FROM monitoring.site_bird_count_v c
  JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND t.name = 'Numenius phaeopus'
  GROUP BY year ORDER BY year")

plot(trend$year, trend$total_counted, type = "b",
     xlab = "Ár", ylab = "Fjöldi spóa talinn")
```

**Hvaða undirverkefni eiga gögn á svæði** — `project_id` samtengt við `birds.project`:

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

**Vista í skrá** (Excel opnar CSV beint; `UTF-8` heldur íslenskum stöfum réttum):

```r
write.csv(by_species, "bird_counts_2024.csv", row.names = FALSE, fileEncoding = "UTF-8")
```

**Athuganir á korti með `sf`.** Fuglaskráningar geyma staðsetningu sem einföld `latitude`/`longitude` gildi
(gráður, WGS84), ekki sem landfræðilegan dálk, svo punktarnir eru búnir til í R:

```r
library(sf)
pts <- dbGetQuery(con, "
  SELECT field_date, taxon_id, total, latitude, longitude
  FROM monitoring.site_bird_count_v
  WHERE monitoring_site_id = 188 AND latitude IS NOT NULL AND longitude IS NOT NULL")

pts_sf <- st_as_sf(pts, coords = c("longitude", "latitude"), crs = 4326)
plot(st_geometry(pts_sf))
# st_write(pts_sf, "birds.gpkg")   # opnaðu í QGIS
```

---

## 7. Hreiðurvöktun

Hreiður eru í `monitoring.site_bird_nest_monitoring_v` — ein lína á hvern **hreiðurvöktunaratburð** (hreiður
er heimsótt nokkrum sinnum yfir tímabilið og birtist því einu sinni fyrir hverja heimsókn). Sýnin er byggð á
vettvangsheimsóknunum, svo **heimsóknir án hreiðra birtast líka**, sem línur með alla hreiðurdálka auða: sía
þarf með `nest_id IS NOT NULL` til að halda aðeins hreiðrum.

Gagnlegir dálkar: hreiðrið (`nest_number`, `taxon_id`, `no_of_eggs`, `max_no_of_eggs`, `hatched`,
`no_of_chicks_hatched`, `egg_stage`, `date_of_hatching_or_failure`, `reason_for_failure`) og heimsóknin
(`date`, `monitoring_no_of_eggs`, `monitoring_no_of_chicks`, `monitoring_method`, `monitoring_event_id`).
Atburðarauðkennið segir hvað sást í heimsókninni — leggst á, ungar klaktir, hreiður rænt o.s.frv.; nafnið er í
`birds.nest_monitoring_event`.

Svæði 188 hefur enga hreiðurvöktun, svo í þessu dæmi er notað **28** (Ástjörn í Jökulsárgljúfrum):

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

## 8. Lengra komnir — grunntöflurnar

Sýnirnar tvær ná yfir nánast allt, en fuglagögn úr **öðrum verkefnum**, eða dálkar sem sýnirnar birta ekki,
eru sótt í grunn-`birds`-skemað:

- `birds.observation` (heimsóknin) → `birds.observation_taxon` (ein lína á hverja talda tegund,
  `observation_id`) → `natura.taxon` (`taxon_id`).
- `birds.nest` (`observation_id`) → `birds.nest_monitoring` (`nest_id`; ein lína á hverja hreiðurheimsókn,
  með `date`, `no_of_eggs`, `no_of_chicks`). Dálkurinn `monitoring_event_id` → `birds.nest_monitoring_event`,
  uppflettitafla atburðartegunda.
- Athugendur: `birds.observation_staff` (`observation_id`, `staff_id`) → `natura.staff`.
- Undirverkefni: `observation.project_id` → `birds.project` (`name`, `number`).
- Tengingin við vöktunarsvæði er óbein: `observation.site_id` → `birds.site.id`, `site.area_id` →
  `birds.area.id`, `area.monitoring_site_id` → `monitoring.site.id`.

Grunntöflurnar innihalda **öll** fuglaverkefni. Til að endurskapa það sem vöktunarsýnirnar sýna er aðeins
verkefnisnúmeri 13098 haldið:

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

Uppflettitöflur fyrir kóðaða dálka: `activity`, `counting_method`, `observation_method`, `count_quality`,
`distance_range`, `family_composition`, `observation_category` (allar í `birds`, samtengdar um samsvarandi
`<nafn>_id`-dálk).

---

## 9. Gott að vita

- **Þetta eru lesfyrirspurnir (read-only).** Þú getur ekki breytt eða eytt neinu gegnum R/SQL — óhætt er að
  skoða.
- **Ein lína = ein tegundatalning.** Leggja þarf saman `total` (með `GROUP BY`) til að fá heildartölur; ekki
  telja línur. Heimsókn án tegundalína birtist einu sinni, með auða talningardálka.
- **Auðir reitir koma sem `NA`.**
- **`monitoring_site_id`, ekki `site_id`,** auðkennir vöktunarsvæðið (kafli 4).
- **Hnit eru einfaldar tölur** (`latitude`/`longitude`, WGS84 gráður); fuglagögnin hafa engan landfræðilegan
  PostGIS-dálk. Fyrir landupplýsingavinnu eru punktarnir búnir til eins og í kafla 6, eða vistaðir sem
  GeoPackage og opnaðir í QGIS.
- **Aðeins verkefnisnúmer 13098** er í tilbúnu sýnunum; undirverkefnið er í `project_id` (kafli 6), önnur
  verkefni eru í grunntöflunum (kafli 8).

---

## Aðstoð

Ef dálkur er óskýr, fyrirspurn skilar einhverju óvæntu, eitthvað vantar í leiðbeiningarnar eða eitthvað
er óljóst, hafðu endilega samband við Náttúrufræðistofnun í netfangið <gagnagrunnar@natt.is>.
