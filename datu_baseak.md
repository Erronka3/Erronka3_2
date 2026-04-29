---
layout: page
title: Datu baseak
---



# TurismoBulego: Datu-baseen Sinkronizazioa eta Kudeaketa Sistema
Proiektu honek turismo bulegoetako datuak kudeatzeko sistema oso bat inplementatzen du. Sistema honek datuak iturri desberdinetatik (MongoDB) SQL datu-base zentralizatu batera (MariaDB) migratzea, estatistikak automatikoki kalkulatzea eta datuen osotasuna bermatzea ahalbidetzen du.

# Aurkibidea

1. [Datu-basearen Diseinua (MariaDB)](#1-datu-basearen-diseinua-mariadb)
2. [Sinkronizazio Prozesua (ETL)](#2-sinkronizazio-prozesua-ETL)
3. [Aggregateak, Selectak eta SQL-ko LABURPEN taula](#3-aggregateak-selectak-eta-sql-ko-laburpen-taula)
4. [Prozesuen Automatizazioa eta Segurtasuna](#4-prozesuen-automatizazioa-eta-segurtasuna)


# 1. Datu-basearen Diseinua (MariaDB)
Sistemaren muina MariaDB SQL datu-basea da. Diseinua erlazionala da, datuen osotasuna bermatzeko eta erredundantzia minimizatzeko.

### Egitura 
1- provincias (Probintziak): Bulego bakoitzeko probintzien izenak gordetzen ditu.

2- offices (Bulegoak): Turismo bulegoak identifikatzen ditu eta probintzia bati lotzen dizkio.

3- origins (Jatorriak): Bisitarien jatorrizko herrialde edo eskualdeen zerrenda.

4- visits (Bisitak): Erregistro bakoitzak bisita talde bat adierazten du (data, pertsona kopurua, jatorria eta bulegoa).

5- stats_turism (Turismo Estatistikak): Bulego eta ordu bakoitzeko agregazioak (max, min, avg) gordetzen dituen taula.

Aurreko Erronkan erabilitako datu-basearen diseinua honako hau zen:
![Captura 5](img/Datu%20base/Captura%20de%20pantalla%202026-04-29%20102956.png)



Gaur egun, diseinua eguneratu da estatistiken taula berria integratzeko:


![Captura sarrera](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102616.png)

# 2. Sinkronizazio Prozesua (ETL)

Datuak Node.js script baten bidez migratzen dira MongoDBtik (iturburua) MariaDBra (helburua).Datu-baseak egun guztietako eta urte guztietako datuak alderatzea saihesteko, uneko ordua aurreko orduarekin alderatzen duen baldintza bat gehitu dugu; horrela, datuak orduz ordu soilik tratatuko ditugu, bestela datu-baseak ezingo bailioke epe luzera eutsi.
![Captura 1](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102030.png)

## 2.1. Duplikatuak Saihestea eta Data eta Orduaren Zuzenketa

Scriptak SQL SELECT kontsulta bat egiten du datu berri bakoitza txertatu aurretik. Datuak (bulegoa, jatorria, data eta pertsona kopurua) jada existitzen badira, ez dira berriro txertatzen. Honek datuen bikoizketa saihesten du sinkronizazio errepikakorretan. Gainera, MongoDBk datak ISO formatuan gordetzen ditu (UTC). Scriptak ordu hauek zuzentzen ditu MariaDBrekin bateragarria den formatura pasatzeko (YYYY-MM-DD HH:MM:SS) eta ordu-eremu lokalera egokitzeko.

![Captura 2](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102114.png)

## 2.2 MariaDBrako Select eta Insert Into. 

Scriptean select bat diseinatu dugu, SQL datu-basean gehitu nahi ditugun datu guztiak hartu eta MongoDBko datuekin erlazionatzen dituena; ondoren, datu horiek MariaDBko dagokion taulan (stats_turism) gehitzen dira Insert into baten bidez.
![Captura 3](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102209.png)

## 2.3 Scriptaren Maiztasuna
"Aurreko zatian aipatu genuen bezala, gure scripta orduoro automatikoki exekutatzeko diseinatuta dago; horrela, azken orduko datuak soilik aztertzeko baldintza ere betetzen du, ahalik eta azkarren."
![Captura 4](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102244.png)




# 3. Aggregateak, Selectak eta SQL-ko LABURPEN taula

## 3.1 MongoDB-ko aggregateak

1. Bisitari kopurua ordu-tarteka (Data funtzioak erabiliz)

```javascript
db.tourist_office.aggregate([
  {
    $project: {
      ordua: { $hour: "$timestamp" },
      numberOfVisitors: 1
    }
  },
  {
    $group: {
      _id: "$ordua",
      bisitariak_guztira: { $sum: "$numberOfVisitors" }
    }
  },
  {
    $sort: { _id: 1 }
  }
])
```
![Captura 3](img/Datu%20base/M1.png)

2. Asteko egunaren araberako estatistikak
```javascript
db.tourist_office.aggregate([
  {
    $project: {
      asteko_eguna: { $dayOfWeek: "$timestamp" },
      numberOfVisitors: 1
    }
  },
  {
    $group: {
      _id: "$asteko_eguna",
      bisita_kopurua: { $sum: 1 },
      batezbesteko_taldea: { $avg: "$numberOfVisitors" }
    }
  },
  { $sort: { _id: 1 } } // 1 (Igandea) - 7 (Larunbata)
])
```
![Captura 3](img/Datu%20base/M2.png)
3. Hilabeteko sasoiaren araberako azterketa (Spring vs Winter)
```javascript
db.tourist_office.aggregate([
  {
    $project: {
      sasoia: {
        $cond: {
          if: { $in: [{ $month: "$timestamp" }, [3, 4, 5]] },
          then: "Spring",
          else: "Other Season"
        }
      },
      numberOfVisitors: 1
    }
  },
  {
    $group: {
      _id: "$sasoia",
      pertsona_kopurua: { $sum: "$numberOfVisitors" }
    }
  }
])
```
![Captura 3](img/Datu%20base/M3.png)
4. Ordu eta jatorriaren arteko konbinazioa
```javascript
db.tourist_office.aggregate([
  {
    $match: {
      $expr: { $gte: [{ $hour: "$timestamp" }, 10] }
    }
  },
  {
    $group: {
      _id: "$origin",
      taldeak: { $sum: 1 }
    }
  },
  { $sort: { taldeak: -1 } }
])
```
![Captura 3](img/Datu%20base/M4.png)
5. Dataren araberako metrika konplexua: Goiztiarrak vs Berandu etorritakoak
```javascript
db.tourist_office.aggregate([
  {
    $project: {
      momentua: {
        $cond: [{ $lt: [{ $hour: "$timestamp" }, 12] }, "Goizez", "Arratsaldez"]
      },
      numberOfVisitors: 1
    }
  },
  {
    $group: {
      _id: "$momentua",
      guztira: { $sum: "$numberOfVisitors" }
    }
  }
])
```
![Captura 3](img/Datu%20base/M5.png)
6. unwind erabilera: Jatorrien zerrenda prozesatzen
```javascript
db.tourist_office.aggregate([
  { $project: { jatorri_zerrenda: ["$origin"], numberOfVisitors: 1 } }, 
  { $unwind: "$jatorri_zerrenda" },
  {
    $group: {
      _id: "$jatorri_zerrenda",
      bisita_kopurua: { $sum: 1 }
    }
  }
])
```
![Captura 3](img/Datu%20base/M6.png)
7. Izenen transformazioa eta kalkuluak
```javascript
db.tourist_office.aggregate([
  {
    $project: {
      _id: 0,
      herrialdea: { $toUpper: "$origin" },
      bulego_zenbakia: "$officeNumber",
      gastu_estimatua: { $multiply: ["$numberOfVisitors", 50] } 
    }
  },
  { $limit: 5 }
])
```
![Captura 3](img/Datu%20base/M7.png)
8. Bulego bakoitzeko gailurra eta erregistroen arteko aldea
```javascript
db.tourist_office.aggregate([
  {
    $group: {
      _id: "$officeNumber",
      talde_handiena: { $max: "$numberOfVisitors" },
      talde_txikiena: { $min: "$numberOfVisitors" },
      guztira: { $sum: "$numberOfVisitors" }
    }
  },
  { $addFields: { aldea: { $subtract: ["$talde_handiena", "$talde_txikiena"] } } }
])
```
![Captura 3](img/Datu%20base/M8.png)
9. Jatorriaren araberako segmentazio matematikoa (Potentzia edo erro karratua)
```javascript
db.tourist_office.aggregate([
  {
    $group: {
      _id: "$origin",
      batezbestekoa: { $avg: "$numberOfVisitors" }
    }
  },
  {
    $project: {
      _id: 1,
      balio_estatistikoa: { $sqrt: "$batezbestekoa" }
    }
  }
])
```
![Captura 3](img/Datu%20base/M9.png)
10. Jatorri bakoitzeko bisitaririk gehieneko taldea 
```javascript
db.tourist_office.aggregate([
  {
    $group: {
      _id: "$origin",
      talde_handiena: { $max: "$numberOfVisitors" },
      batezbestekoa: { $avg: "$numberOfVisitors" }
    }
  },
  {
    $project: {
      _id: 0,                   
      herrialdea: "$_id",       
      talde_handiena: 1,
      batezbestekoa: { $round: ["$batezbestekoa", 1] } 
    }
  },
  { $sort: { talde_handiena: -1 } } 
])
```
![Captura 3](img/Datu%20base/M10.png)
Sistemak automatikoki kalkulatzen ditu turismo estatistikak, orduko txostenak errazteko.

## 3.2 stats_turism Taularen Egitura
Taula honek datu agregatuak gordetzen ditu bulego eta ordu bakoitzeko:

1-fecha_hora: Agregazioaren hasierako ordua.

2-visitantes_max: Ordu horretako talderik handiena.

3-visitantes_min: Ordu horretako talderik txikiena.

4-media_visitantes: Taldeen batezbesteko tamaina.

5-pais_mas_visitado: Ordu horretan bisitari gehien ekarri dituen herrialdea.

---
![Captura 3](img/Datu%20base/S6.png)



## 3.3 MariaDB-ko Selectak

Datu-basearen potentzia aprobetxatzeko, estatistika-taula eta taula erlazionalak uztartzen dituzten kontsulta aurreratuak diseinatu dira:

1. Batez besteko orokorraren gainetik dauden bulegoen sailkapena
Kontsulta honek azpikontsulta bat erabiltzen du batez besteko globala kalkulatzeko eta bulego bakoitzaren errendimenduarekin alderatzeko.

```SQL
SELECT 
    o.nombre AS bulegoa, 
    st.media_visitantes, 
    st.fecha_hora
FROM stats_turism st
JOIN offices o ON st.office_id = o.id
WHERE st.media_visitantes > (
    SELECT AVG(media_visitantes) 
    FROM stats_turism
)
ORDER BY st.media_visitantes DESC;
```
![Captura 3](img/Datu%20base/S1.png)
2. Probintziako bisitari errekorra duen herrialdea
Lau taula lotzen ditu (provincias, offices, stats_turism) lurralde bakoitzean bisitari kopuru handiena (visitantes_max) ekarri duen herrialdea identifikatzeko.

```SQL
SELECT 
    p.nombre AS probintzia, 
    o.nombre AS bulegoa, 
    st.pais_mas_visitado, 
    MAX(st.visitantes_max) AS bisitari_errekorra
FROM stats_turism st
JOIN offices o ON st.office_id = o.id
JOIN provincias p ON o.provincia_id = p.id
GROUP BY p.nombre, o.nombre
ORDER BY bisitari_errekorra DESC;
```
![Captura 3](img/Datu%20base/S2.png)
3. "Ordu Kritikoen" detekzioa (Bisitari tarte handia)
Ordu berean jaso den talde handienaren eta txikienaren arteko aldea kalkulatzen du. Lan-karga bat-batekoak detektatzeko oso erabilgarria da.

```SQL
SELECT 
    fecha_hora, 
    office_id, 
    visitantes_max, 
    visitantes_min,
    (visitantes_max - visitantes_min) AS pertsona_aldea,
    media_visitantes
FROM stats_turism
WHERE (visitantes_max - visitantes_min) > 10
ORDER BY pertsona_aldea DESC;
```
![Captura 3](img/Datu%20base/S3.png)
4. Bulegoen egoera eguneratua (LEFT JOIN)
LEFT JOIN erabiltzeak aukera ematen du bulego guztiak ikusteko, baita azken orduan bisitaririk jaso ez dutenak ere (hauek "Daturik gabe" gisa agertuko dira).

```SQL
SELECT 
    o.nombre AS bulegoa, 
    IFNULL(st.pais_mas_visitado, 'Daturik gabe') AS herrialde_nagusia,
    IFNULL(st.media_visitantes, 0) AS orduko_batezbestekoa
FROM offices o
LEFT JOIN stats_turism st ON o.id = st.office_id 
    AND st.fecha_hora >= NOW() - INTERVAL 1 HOUR
ORDER BY orduko_batezbestekoa DESC;
```
![Captura 3](img/Datu%20base/S4.png)
5. Jatorriaren araberako analisi historiko metatua
Estatistikako taula erabili beharrean, visits eta origins taula erlazionaletara jotzen du herrialde bakoitzeko metatu historiko osoa kalkulatzeko.

```SQL
SELECT 
    ori.nombre AS jatorrizko_herrialdea, 
    COUNT(v.id) AS talde_kopurua, 
    SUM(v.number_of_visitors) AS bisitariak_guztira,
    ROUND(AVG(v.number_of_visitors), 2) AS taldeko_batezbestekoa
FROM visits v
JOIN origins ori ON v.origin_id = ori.id
GROUP BY ori.nombre
HAVING bisitariak_guztira > 0
ORDER BY bisitariak_guztira DESC;
```
![Captura 3](img/Datu%20base/S5.png)

# 4. Prozesuen Automatizazioa eta Segurtasuna
Proiektuaren baldintzak betetzeko, automatizazio eta segurtasun plan bat diseinatu dugu

## 4.1. Automatizazioa
Node.js scripta etengabe exekutatzen da atzealdean (Node-red --> MongoDB --> Script --> MariaDB), orduro sinkronizazioa eta estatistiken kalkulua egiteko.

## 4.2. Datu-basearen Segurtasuna (Backups)
Datuak babesteko, automatizatutako backup sistema bat ezarri da mysqldump komandoa erabiliz.

Inplementazioa (Windows .bat Scripta):
Backup automatikoak egiteko script bat sortu da, Windows-eko Zereginen Planifikatzaileak orduoro exekutatzen duena:
```SQL
@echo off
set USER=root
set PASSWORD=
set DB_NAME=turismo
set BACKUP_PATH=C:\backups\turismo_copia

if not exist "%BACKUP_PATH%" mkdir "%BACKUP_PATH%"
set FILENAME=%DB_NAME%_%date:~-4%%date:~3,2%%date:~0,2%_%time:~0,2%%time:~3,2%.sql
"C:\xampp\mysql\bin\mysqldump.exe" -u %USER% --databases %DB_NAME% > "%BACKUP_PATH%\%FILENAME%"
```
