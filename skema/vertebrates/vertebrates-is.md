# Hryggdýrasafn (safngripaskrá) — aðgangur að gögnum

Leiðbeiningar fyrir þau sem vilja sækja gögn úr **hryggdýrasafni** Náttúrufræðistofnunar.
Leiðbeiningarnar eru ætlaðar öllum sem þegar hafa SSL-tengingu og viðeigandi aðgangsheimildir
(sjá Grunnleiðbeiningar). Gögnin eru í hryggdýraskemanu `vertebrates`.

> **Áður en þú byrjar:** lestu fyrst grunnleiðbeiningarnar um aðgang og notkun — þær útskýra hvernig
> tengjast má gagnagrunninum (þjónn, gagnagrunnsnafn, innskráning) og grunnatriði þess að keyra
> fyrirspurn í R. Þessar leiðbeiningar gera ráð fyrir að þú sért þegar með virka tengingu sem heitir
> `con`. Stutt upprifjun fylgir hér að neðan.

<!-- TODO (yfirferð Önnu Sveinsdóttur, 2026-08-17): bæta við kafla um landupplýsingavinnuaðgang. -->

---

## 1. Hvernig gögnin líta út

Hryggdýraskemað `vertebrates` er einfalt: **ein miðlæg tafla, `vertebrates.objectcatalog`**, með einni
línu á hvern skráðan **safngrip** (eintak) — um 18.000 gripir, meira en 1.000 tegundir. Í kringum hana
eru uppflettitöflur (safn, staður, varðveittur hluti o.s.frv.) sem gefa kóðum í `objectcatalog` merkingu.

Nær alltaf er byrjað í `objectcatalog` og samtengt (join) við uppflettitöflurnar eftir þörfum.

---

## 2. Tenging (stutt upprifjun)

```r
library(DBI)
library(RPostgres)

con <- dbConnect(
  RPostgres::Postgres(),
  dbname   = "nitest",
  host     = "innra.natt.is",
  port     = 5432,
  user     = "notendanafn",
  password = "lykilord"
)
```

Allt hér að neðan keyrir gegnum `con` (þ.e. gagnagrunnstenginguna). Þegar það er búið þarf að
aftengjast: `dbDisconnect(con)`.

---

## 3. Miðlæga taflan — `objectcatalog`

Gagnlegustu dálkarnir, flokkaðir:

- **Auðkenni:** `id`, `catalogno` (safnnúmer), `collectionno`, `accession`, `othernumber`
- **Tegund:** `species` — þetta er **tegundarauðkenni** (sjá skref 4)
- **Dagsetning:** `year`, `month`, `day` (aðskildir heiltölureitir)
- **Staðsetning:** `locationname` (frjáls texti), `geom` (PostGIS-punktur, EPSG:4326),
  `locationid` (→ `location`), `location_method_id`, `location_uncertainty_km`, `country_code`
- **Eintakið:** `sex`, `agecategory`, `agedetail`, `coat_color`, `specimen_count`,
  `part_preserved_id` (→ `part_preserved`), `bandno` (merkjanúmer)
- **Varsla:** `repository` (→ `museum`), `exact_storage_location`, `exhibition_item`
- **Fólk:** `leg` (safnari), `det` (greinandi)
- **Athugasemdir:** `gen_comment`, `sampling_remarks`, `catalog_remarks`

Grunnfyrirspurn — listi yfir gripi með uppflettum gildum:

```r
specimens <- dbGetQuery(con, "
  SELECT o.catalogno, t.name AS species, o.year, o.sex, o.agecategory,
         o.locationname, m.museumname, pp.name AS part_preserved,
         st_y(o.geom) AS latitude, st_x(o.geom) AS longitude
  FROM vertebrates.objectcatalog o
  LEFT JOIN natura.taxon t             ON t.id  = o.species
  LEFT JOIN vertebrates.museum m       ON m.id  = o.repository
  LEFT JOIN vertebrates.part_preserved pp ON pp.id = o.part_preserved_id
  WHERE o.year = 2024
  ORDER BY species")

head(specimens)
```

---

## 4. Tegundanöfn

Dálkurinn `species` er **tegundarauðkenni** (ekki nafn). Samtengja þarf **`natura.taxon`** til að fá
latneska nafnið (`taxon.name`) — miðlæga flokkunarfræðin sem notuð er þvert á öll gögn
Náttúrufræðistofnunar:

```sql
LEFT JOIN natura.taxon t ON t.id = o.species      -- t.name = latneskt nafn
```

Fyrir **íslenskt** nafn (þar sem það er til):

```r
tegundir_is <- dbGetQuery(con, "
  SELECT o.catalogno, t.name AS latin_name, v.icelandic_name, o.year
  FROM vertebrates.objectcatalog o
  LEFT JOIN natura.taxon t ON t.id = o.species
  LEFT JOIN natura.taxon_preferred_icelandic_name_v v ON v.taxon_id = o.species
  WHERE o.year = 2024")
```

---

## 5. Algeng dæmi

**Fjöldi gripa á hverja tegund** (gagnagrunnurinn telur):

```r
by_species <- dbGetQuery(con, "
  SELECT t.name AS species, count(*) AS specimens
  FROM vertebrates.objectcatalog o
  JOIN natura.taxon t ON t.id = o.species
  GROUP BY t.name
  ORDER BY specimens DESC")
```

**Allir gripir af einni tegund** (hér kría, *Sterna paradisaea*):

```r
kria <- dbGetQuery(con, "
  SELECT o.catalogno, o.year, o.month, o.day, o.sex, o.locationname
  FROM vertebrates.objectcatalog o
  JOIN natura.taxon t ON t.id = o.species
  WHERE t.name = 'Sterna paradisaea'
  ORDER BY o.year")
```

**Vista í skrá** (Excel opnar CSV beint; `UTF-8` heldur íslenskum stöfum réttum):

```r
write.csv(by_species, "hryggdyr_tegundir.csv", row.names = FALSE, fileEncoding = "UTF-8")
```


**Gripir á korti með `sf`.** Taflan hefur landfræðilegan dálk (`geom`, PostGIS), svo `sf` les hann beint:

```r
library(sf)
sp <- st_read(con, query = "
  SELECT o.catalogno, t.name AS species, o.year, o.geom
  FROM vertebrates.objectcatalog o
  LEFT JOIN natura.taxon t ON t.id = o.species
  WHERE o.geom IS NOT NULL AND t.name = 'Sterna paradisaea'")

plot(st_geometry(sp))
# st_write(sp, "hryggdyr.gpkg")   # opnaðu í QGIS
```

---

## 6. Uppfletti- og stoðtöflur

Kóðadálkar í `objectcatalog` vísa í eftirfarandi töflur:

- `vertebrates.museum` — safn sem varðveitir gripinn (`repository` → `museum.id`): `museumname`, `museumcode`.
- `vertebrates.part_preserved` — hvaða hluti er varðveittur (`part_preserved_id` → `id`): `name`.
- `vertebrates.location` — staðir, stigskiptir (`locationid` → `location.locationid`, `parent` gefur yfirstað).
- `vertebrates.location_method` — hvernig staðsetning var ákvörðuð (`location_method_id` → `id`).
- `vertebrates.vertebrates_taxon` — tegundalisti safnsins (*ekki* nauðsynlegur fyrir nöfn; nota
  `natura.taxon` beint eins og að ofan).

Fyrir ítarlegri vinnu eru einnig sérhæfðar töflur, tengdar við `objectcatalog` um gripinn: `autopsy` og
`autopsymeasurement` (krufning/mælingar), `egg` og `eggmeasurement`, `bone`, `age_analysis`, `photo`,
`loan` (útlán) o.fl.

---

## 7. Gott að vita

- **Þetta eru lesfyrirspurnir (read-only).** Þú getur ekki breytt eða eytt neinu — óhætt er að skoða.
- **`species` er tegundarauðkenni**, ekki nafn — samtengdu `natura.taxon` (sjá skref 4).
- **Dagsetning er í þremur reitum** (`year`, `month`, `day`) og getur verið að hluta eða vantað.
- **Staðsetningin er í `geom`** (PostGIS-punktur, EPSG:4326). Í SQL má draga hnitin út með `st_y(geom)`
  (breidd) og `st_x(geom)` (lengd), eða lesa dálkinn beint með `sf` (sjá skref 5).
- **Tveir staðsetningarreitir:** frjáls texti `locationname` beint í `objectcatalog`, og stigskipt
  `location`-tafla um `locationid`.

---

## Aðstoð

Ef dálkur er óskýr, fyrirspurn skilar einhverju óvæntu, eitthvað vantar í leiðbeiningarnar eða eitthvað
er óljóst, hafðu endilega samband við Náttúrufræðistofnun í netfangið <gagnagrunnar@natt.is>.
