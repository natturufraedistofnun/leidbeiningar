# Grunnleiðbeiningar fyrir náttúrustofur um aðgang og notkun á gögnum í gagnagrunnum Náttúrufræðistofnunar

Samkvæmt verklagsreglu Náttúrufræðistofnunar VLR 052 um móttöku náttúrufræðigagna frá náttúrustofum gerir
Náttúrufræðistofnun samning um móttöku og vistun gagna við náttúrustofur. Sérfræðingar náttúrustofa fá aðgang
til að vinna í gagnagrunnunum, skrá í þá og sækja gögn.

Þessar leiðbeiningar fjalla um hvernig hægt er að:

1. Tengjast gagnagrunni og gagnaskemum,
2. Opna og skoða gögn í gagnagrunnsviðmóti,
3. Tengjast grunninum í gegnum landupplýsingakerfi,
4. Tengjast grunninum í gegnum R.

Sérhæfðari leiðbeiningar um notkun R og SQL fylgja hverju gagnagrunnsskema.

## Markmið

Markmið þessara leiðbeininga er að auðvelda náttúrustofum að nálgast, sækja og vinna með eigin gögn og gögn úr
gagnagrunnum Náttúrufræðistofnunar.

Leiðbeiningarnar eru hagnýtar og miða fyrst og fremst að því að notendur geti fundið viðeigandi gögn, síað þau
eftir þörfum og flutt þau út til áframhaldandi vinnslu, til dæmis í Excel, QGIS eða R.

Leiðbeiningarnar eru ætlaðar starfsfólki náttúrustofa sem þarf að nota gögn Náttúrufræðistofnunar og
náttúrustofa í verkefnum sínum.

## Skilyrði um notkun gagna

Þegar nota skal gögn sem ekki eru á forræði viðkomandi notanda skal áður kynna sér gögnin vel og leita álits
forráðamanna gagnanna eða yfirmanna viðkomandi stofnana (þ.e. Náttúrufræðistofnun og náttúrustofur). Þegar
vinna á með viðkvæm gögn, t.d. útbreiðslu fágætra tegunda, þarf að skoða það sérstaklega með viðkomandi
sérfræðingum á Náttúrufræðistofnun.

## Aðgangur að gögnum

Til að nálgast gögnin þarf að tengjast Náttúrufræðistofnun með SSL og fá uppsettan aðgang með tilheyrandi
aðgangsorðum í viðeigandi gagnaskema. Forstöðumaður hverrar náttúrustofu óskar eftir aðgangi fyrir viðeigandi
starfsmann og sendir beiðni á netfangið <gagnagrunnar@natt.is>. Viðkomandi starfsmaður fær sendar
leiðbeiningar um uppsettningu og aðgangsorð fyrir gagnagrunninn.

Lýsigögn eru í vinnslu en verða aðgengileg inni í grunninum og tengjast þá hverjum dálk.

Þrjár leiðir til að tengjast gagnagrunninum:

### Tenging við gagnagrunnsviðmót Náttúrufræðistofnunar (notað fyrir innslátt og einfalt export):

1. Tengjast Náttúrufræðistofnun
2. Opna tengil á setup-skrá sem viðkomandi hefur fengið sendan frá Náttúrufræðistofnun
3. Kerfið sett upp (install)
4. Tvísmellt á tengil (flýtileið) sem birtist á skjáborði
5. Notendanafn og lykilorð
6. Ef engin gögn birtast er smellt á Refresh
7. Almennar notendaleiðbeiningar eru í viðmótinu (Help > Help) m.a. fyrir:
   1. Innslátt gagna
   2. Leit og síun
   3. Afritun gagna til notkun í excel

### Tenging við landupplýsingakerfi (QGIS)

1. Tengjast Náttúrufræðistofnun
2. Sjá leiðbeiningar á GitHub: <https://github.com/natturufraedistofnun/leidbeiningar>

### Tenging við grunninn með R:

1. Tengjast Náttúrufræðistofnun
2. Gagnagrunnsþjónn (host): `innra.natt.is`
3. Gagnagrunnsnafn (dbname): `nitest` eða `lmigis`
4. Port: `5432`
5. Notendanafn og lykilorð

**Dæmi um notkun í R:**

```r
# Setja upp nauðsynlega pakka
install.packages("DBI")
install.packages("RPostgres")

# Tengjast gagnagrunni
library(DBI)

con <- dbConnect(RPostgres::Postgres(),
                 dbname = 'nitest',
                 host = 'innra.natt.is',
                 port = 5432,
                 user = 'notendanafn',
                 password = 'lykilorð')

# Skoða hvaða töflur og sýnir eru aðgengilegar í tilteknu skema (hér monitoring)
dbGetQuery(con, "
  SELECT table_name, table_type
  FROM information_schema.tables
  WHERE table_schema = 'monitoring'
  ORDER BY table_name")

# Sækja gögn úr töflu
data <- dbGetQuery(con, "SELECT * FROM monitoring.site")

# Skoða fyrstu línur gagnanna
head(data)

# Aftengjast gagnagrunni
dbDisconnect(con)
```

## Aðstoð

Ef eitthvað er óljóst eða vantar skal endilega hafa samband í netfangið <gagnagrunnar@natt.is>.
