# Tengjast gagnagrunni NÍ frá R

## 1. Sækja RPostgreSQL pakkann

```r
install.packages("RPostgreSQL")
```

## 2. Keyra pakkann

```r
library(RPostgreSQL)
```

## 3. Segja R hvaða driver á að nota

```r
drv <- dbDriver("PostgreSQL")
```

## 4. Opna tengingu við PostgreSQL gagnagrunninn

```r
con <- dbConnect(
  drv,
  dbname = "nitest",
  host = "postgresql.natt.local",
  port = 5432,
  user = "notandanafn",
  password = "lykilorð"
)
```

## 5. Nota SQL til að sækja gögn úr PostgreSQL

```r
res <- dbGetQuery(
  con,
  "SELECT * FROM natura.taxon LIMIT 100"
)
```

## Nýrri aðferð – RPostgres

Einnig er hægt að nota `RPostgres` pakkann.

```r
install.packages("RPostgres")
```

```r
library(RPostgres)
```

Tenging við gagnagrunn:

```r
drv <- dbDriver("Postgres")

con <- dbConnect(
  drv,
  dbname = "nitest",
  host = "postgresql.natt.local",
  port = 5432,
  user = "notandanafn"
)
```
