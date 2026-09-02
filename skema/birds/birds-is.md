# Fuglavöktunargögn — aðgangur með SQL og R

Hagnýt leiðbeining fyrir starfsfólk náttúrustofa sem vill sækja **fuglavöktunargögn** úr gagnagrunni
Náttúrufræðistofnunar og vinna með þau í R (eða hvaða tóli sem talar SQL), í stað þess að afrita og líma úr
Vöktunarforritinu.

> **Áður en þú byrjar:** lestu fyrst almennu leiðbeininguna um gagnaaðgang — hún útskýrir hvernig þú tengist
> gagnagrunninum (þjónn, gagnagrunnsnafn, innskráning) og grunnatriði þess að keyra fyrirspurn í R. Þessi
> leiðbeining gerir ráð fyrir að þú sért þegar með virka tengingu sem heitir `con`. Stutt upprifjun fylgir hér
> að neðan.

---

## 1. Hvernig fuglavöktunargögnin líta út

Fuglagögnum er safnað sem **vettvangsathugunum**. Í heimsókn á svæði skráir athugandi talningaraðstæður og
síðan, fyrir hverja tegund sem sést, **talningu** (heildarfjölda, kyn, aldur, pör og fleira). Hreiður eru
skráð sérstaklega og heimsótt aftur yfir tímabilið.

Til eru **tvær tilbúnar sýnir (views)** sem gera einmitt þetta — þær sjá um samtenginguna fyrir þig og eru
auðveldasti staðurinn til að byrja:

| Sýn | Ein lína er… | Notaðu hana fyrir |
|---|---|---|
| `monitoring.site_bird_count_v` | ein **tegundatalning** í einni vettvangsheimsókn | hversu margir af hverri tegund, hvar, hvenær, af hverjum |
| `monitoring.site_bird_nest_monitoring_v` | einn **hreiðurvöktunaratburður** | innihald og afdrif hreiðra yfir tímabilið |

**Tvennt sem gott er að vita strax:**

1. **Þessar sýnir innihalda aðeins verkefnið „vöktun viðkvæmra vistgerða".** Þær eru forsíaðar við það
   verkefni, sem er nær örugglega gögnin sem þú vilt. (Ef þú þarft einhvern tíma fuglagögn úr *öðrum*
   verkefnum, sjá kaflann *Lengra komnir* aftast.)
2. **Ein heimsókn myndar margar línur.** Ef 10 tegundir voru taldar í heimsókn birtist sú heimsókn sem 10
   línur í `site_bird_count_v` (ein á hverja tegund). Þegar þú vilt heildartölu skaltu því **leggja línurnar
   saman** — ekki gera ráð fyrir einni línu á hverja heimsókn.

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
  user     = "notendanafn",         # sama innskráning og í Vöktunarforritinu
  password = "lykilord"
)
```

Allt hér að neðan keyrir gegn `con`. Þegar þú ert búin(n): `dbDisconnect(con)`.

---

## 3. Skref eitt — finndu svæðið þitt

Hver fyrirspurn er síuð eftir **svæðisauðkenni** (`monitoring_site_id`). Flettu þínu upp eftir nafni fyrst.
Svæði geta verið stigskipt (stærra svæði með undirsvæðum), svo leit getur skilað fleiri en einu:

```r
dbGetQuery(con, "
  SELECT s.id, s.name, s.region, p.name AS parent_site
  FROM monitoring.site s
  LEFT JOIN monitoring.site p ON p.id = s.parent_site_id
  WHERE s.name ILIKE '%hérað%'
  ORDER BY s.name")
```

Taktu eftir `id` sem þú þarft. Í dæmunum hér að neðan notum við **188** („Úthérað í Múlaþingi") — skiptu því
út fyrir þitt.

---

## 4. Skref tvö — sæktu fuglatalningar

Vinnuhesturinn er `monitoring.site_bird_count_v`. Hér eru dálkarnir sem þú notar mest:

- **Hvenær / hvar:** `field_date`, `year`, `latitude`, `longitude`, `monitoring_site_id`
- **Talningin:** `total`, `males`, `females`, `pairs`, `adults`, `juveniles`, `chicks`, `nests`, `present`,
  `breeding`
- **Tegund:** `taxon_id` (samtengdu til að fá nafnið — sjá skref þrjú)
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

> **Síaðu eftir `monitoring_site_id`.** Sýnin hefur líka dálk sem heitir `site_id`, en það er *fugla*-svæðið
> (innri undireining), **ekki** vöktunarsvæðið — ekki sía eftir því fyrir mistök.

---

## 5. Skref þrjú — tegundanöfn

Tegundir eru geymdar sem `taxon_id`. Samtengdu **`natura.taxon`** til að fá latneska nafnið (`taxon.name`) —
þetta er miðlæga flokkunarfræðin sem notuð er þvert á öll gögn NÍ, og það sem greining notar venjulega:

```sql
LEFT JOIN natura.taxon t ON t.id = c.taxon_id      -- t.name = latneskt nafn
```

Ef þú vilt líka **íslenska** nafnið skaltu bæta við sýninni fyrir valið nafn (hún skilar íslenska nafninu þar
sem það er til):

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

**Heildarfjöldi talinn á hverja tegund, fyrir eitt svæði, eitt ár** — láttu gagnagrunninn sjá um
samlagninguna:

```r
by_species <- dbGetQuery(con, "
  SELECT t.name AS species, sum(c.total) AS total_counted
  FROM monitoring.site_bird_count_v c
  LEFT JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND c.year = 2024
  GROUP BY t.name
  ORDER BY total_counted DESC NULLS LAST")
```

**Ein tegund yfir tíma (einföld þróun):**

```r
spoi <- dbGetQuery(con, "
  SELECT year, sum(total) AS total_counted
  FROM monitoring.site_bird_count_v c
  JOIN natura.taxon t ON t.id = c.taxon_id
  WHERE c.monitoring_site_id = 188 AND t.name = 'Numenius phaeopus'
  GROUP BY year ORDER BY year")

plot(spoi$year, spoi$total_counted, type = "b",
     xlab = "Ár", ylab = "Fjöldi spóa talinn")
```

**Vista í skrá** (Excel opnar CSV beint; `UTF-8` heldur íslenskum stöfum réttum):

```r
write.csv(by_species, "bird_counts_2024.csv", row.names = FALSE, fileEncoding = "UTF-8")
```

**Settu athuganir á kort með `sf`.** Fuglaskráningar geyma staðsetningu sem einföld `latitude`/`longitude`
gildi (gráður, WGS84), svo þú býrð til punktana sjálf(ur):

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

Fyrir hreiður skaltu nota `monitoring.site_bird_nest_monitoring_v` — ein lína á hvern
**hreiðurvöktunaratburð** (hreiður getur verið heimsótt nokkrum sinnum yfir tímabilið). Gagnlegir dálkar:
`field_date`, `nest_number`, `taxon_id`, `no_of_eggs`, `max_no_of_eggs`, `hatched`, `no_of_chicks_hatched`,
`egg_stage`, `date_of_hatching_or_failure`, `reason_for_failure`, og per-heimsókn gildin
`monitoring_no_of_eggs` / `monitoring_no_of_chicks` / `date`.

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

## 8. Lengra komnir — grunntöflurnar

Sýnirnar tvær ná yfir nánast allt, en ef þú þarft meira (til dæmis fuglagögn úr **öðrum verkefnum**, eða
dálka sem sýnirnar birta ekki) geturðu spurt grunn-`birds`-skemað beint:

- `birds.observation` → `birds.observation_taxon` (talningar á hverja tegund) → `natura.taxon` (tegund).
- `birds.nest` → `birds.nest_monitoring` → `birds.nest_monitoring_event`.
- Athugendur: `birds.observation_staff` → `natura.staff`.
- Fuglaathugun tengist vöktunarsvæði óbeint:
  `observation.site_id → birds.site.area_id → birds.area.monitoring_site_id`.

Grunntöflurnar innihalda **öll** fuglaverkefni. Til að endurskapa nákvæmlega það sem vöktunarsýnirnar sýna
skaltu halda aðeins vöktunarverkefninu:

```sql
WHERE observation.project_id IN (SELECT id FROM birds.project WHERE number = 13098)
```

---

## 9. Gott að vita

- **Þetta eru lesfyrirspurnir (read-only).** Þú getur ekki breytt eða eytt neinu gegnum R/SQL — óhætt er að
  skoða.
- **Ein lína = ein tegundatalning.** Leggðu saman `total` (með `GROUP BY`) til að fá heildartölur; ekki telja
  línur.
- **Auð gildi eru `NA`.** Ekki er fyllt út í alla reiti í hverri heimsókn, svo búastu við vantandi gildum.
- **`monitoring_site_id`, ekki `site_id`,** auðkennir vöktunarsvæðið (sjá §4).
- **Hnit eru einfaldar tölur** (`latitude`/`longitude`, WGS84 gráður), ekki landfræðilegur dálkur — búðu til
  rúmfræði í R með `sf` eins og sýnt er, eða í SQL ef þú kýst.
- **Aðeins verkefnið „vöktun viðkvæmra vistgerða"** er í tilbúnu sýnunum (sjá kaflann *Lengra komnir* fyrir hitt).

---

## Aðstoð

Ef dálkur er óskýr, fyrirspurn skilar einhverju óvæntu, eða þú þarft gögn sem sýnirnar ná ekki yfir, hafðu
samband við Náttúrufræðistofnun — við hjálpum með glöðu geði, og spurningar þínar hjálpa okkur að bæta þessar
leiðbeiningar.
