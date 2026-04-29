---
layout: page
title: Datu baseak
---



# TurisGipuzkoa: Datu-baseen Sinkronizazioa eta Kudeaketa Sistema
Proiektu honek Gipuzkoako turismo bulegoetako datuak kudeatzeko sistema oso bat inplementatzen du. Sistema honek datuak iturri desberdinetatik (MongoDB) SQL datu-base zentralizatu batera (MariaDB) migratzea, estatistikak automatikoki kalkulatzea eta datuen osotasuna bermatzea ahalbidetzen du.

# Aurkibidea
Datu-basearen Diseinua (MariaDB)

Sinkronizazio Prozesua (ETL)

Estatistikak eta Agregazioak

Prozesuen Automatizazioa eta Segurtasuna

## 1. Datu-basearen Diseinua (MariaDB)
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

## 2. Sinkronizazio Prozesua (ETL)

Datuak Node.js script baten bidez migratzen dira MongoDBtik (iturburua) MariaDBra (helburua).Datu-baseak egun guztietako eta urte guztietako datuak alderatzea saihesteko, uneko ordua aurreko orduarekin alderatzen duen baldintza bat gehitu dugu; horrela, datuak orduz ordu soilik tratatuko ditugu, bestela datu-baseak ezingo bailioke epe luzera eutsi.
![Captura 1](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102030.png)

# 2.1. Duplikatuak Saihestea eta Data eta Orduaren Zuzenketa

Scriptak SQL SELECT kontsulta bat egiten du datu berri bakoitza txertatu aurretik. Datuak (bulegoa, jatorria, data eta pertsona kopurua) jada existitzen badira, ez dira berriro txertatzen. Honek datuen bikoizketa saihesten du sinkronizazio errepikakorretan. Gainera, MongoDBk datak ISO formatuan gordetzen ditu (UTC). Scriptak ordu hauek zuzentzen ditu MariaDBrekin bateragarria den formatura pasatzeko (YYYY-MM-DD HH:MM:SS) eta ordu-eremu lokalera egokitzeko.

![Captura 2](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102114.png)

# 2.2 MariaDBrako Select eta Insert Into. 

Scriptean select bat diseinatu dugu, SQL datu-basean gehitu nahi ditugun datu guztiak hartu eta MongoDBko datuekin erlazionatzen dituena; ondoren, datu horiek MariaDBko dagokion taulan (stats_turism) gehitzen dira Insert into baten bidez.
![Captura 3](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102209.png)

# 2.3 Scriptaren Maiztasuna
"Aurreko zatian aipatu genuen bezala, gure scripta orduoro automatikoki exekutatzeko diseinatuta dago; horrela, azken orduko datuak soilik aztertzeko baldintza ere betetzen du, ahalik eta azkarren."
![Captura 4](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102244.png)





## 3. MongoDB-ko aggregateak
MongoDB aggregates:

1. Bisitari kopurua ordu-tarteka (Data funtzioak erabiliz)

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
  { $sort: { _id: 1 } }
])



2. Asteko egunaren araberako estatistikak

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


3. Hilabeteko sasoiaren araberako azterketa (Spring vs Winter)

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


4. Ordu eta jatorriaren arteko konbinazioa

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


5. Dataren araberako metrika konplexua: Goiztiarrak vs Berandu etorritakoak

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

6. unwind erabilera: Jatorrien zerrenda prozesatzen

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


7. Izenen transformazioa eta kalkuluak

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


8. Bulego bakoitzeko gailurra eta erregistroen arteko aldea

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


9. Jatorriaren araberako segmentazio matematikoa (Potentzia edo erro karratua)

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


10. Jatorri bakoitzeko bisitaririk gehieneko taldea 

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

Sistemak automatikoki kalkulatzen ditu turismo estatistikak, orduko txostenak errazteko.

# 3.1. stats_turism Taularen Egitura
Taula honek datu agregatuak gordetzen ditu bulego eta ordu bakoitzeko:

fecha_hora: Agregazioaren hasierako ordua.

visitantes_max: Ordu horretako talderik handiena.

visitantes_min: Ordu horretako talderik txikiena.

media_visitantes: Taldeen batezbesteko tamaina.

pais_mas_visitado: Ordu horretan bisitari gehien ekarri dituen herrialdea.

Honako irudian ikus daiteke taularen edukia datu historikoak txertatu ondoren:

Irudia 3: stats_turism taularen edukia (duplikaturik gabe eta ordenatuta).

# 3.2. Eagregazio Loka (Scripta)
Node.js scripta eguneratu da sinkronizazio bakoitzean estatistikak kalkulatzeko. INSERT ... ON DUPLICATE KEY UPDATE sintaxia erabiltzen da: ordu horretako estatistikak jada existitzen badira, eguneratu egiten dira; bestela, txertatu.

## 4. Prozesuen Automatizazioa eta Segurtasuna
Proiektuaren baldintzak betetzeko, automatizazio eta segurtasun plan bat diseinatu da, Word dokumentuko baldintzak jarraituz.

# 4.1. Automatizazioa
Node.js scripta etengabe exekutatzen da atzealdean (PM2 bezalako tresnekin edo Windows Task Scheduler-ekin), 10 segundoro (edo konfiguratutako denboran) sinkronizazioa eta estatistiken kalkulua egiteko.

# 4.2. Datu-basearen Segurtasuna (Backups)
Datuak babesteko, automatizatutako backup sistema bat ezarri da mysqldump komandoa erabiliz.

Irudia 4: Word dokumentuko backup-ak egiteko baldintzak (euskaraz).

Inplementazioa (Windows .bat Scripta):
Backup automatikoak egiteko script bat sortu da, Windows-eko Zereginen Planifikatzaileak orduoro exekutatzen duena:

Fragmento de código
@echo off
set USER=root
set PASSWORD=
set DB_NAME=turismo
set BACKUP_PATH=C:\backups\turismo

if not exist "%BACKUP_PATH%" mkdir "%BACKUP_PATH%"
set FILENAME=%DB_NAME%_%date:~-4%%date:~3,2%%date:~0,2%_%time:~0,2%%time:~3,2%.sql
"C:\xampp\mysql\bin\mysqldump.exe" -u %USER% --databases %DB_NAME% > "%BACKUP_PATH%\%FILENAME%"
4.3. Estrategia Alternatiboen Azterketa
mysqldump erabili ordez, Node.js scrip-a JSON fitxategiak esportatzeko konfigura daiteke. Estrategia hau aztertu da eta backup azkarrak egiteko edo datuak beste tresna batzuetara (Excel, adibidez) erraz eramateko baliagarria dela ondorioztatu da.
