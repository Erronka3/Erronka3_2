---
layout: page
title: Datu baseak
---



TurisGipuzkoa: Datu-baseen Sinkronizazioa eta Kudeaketa Sistema
Proiektu honek Gipuzkoako turismo bulegoetako datuak kudeatzeko sistema oso bat inplementatzen du. Sistema honek datuak iturri desberdinetatik (MongoDB) SQL datu-base zentralizatu batera (MariaDB) migratzea, estatistikak automatikoki kalkulatzea eta datuen osotasuna bermatzea ahalbidetzen du.

Aurkibidea
Datu-basearen Diseinua (MariaDB)

Sinkronizazio Prozesua (ETL)

Estatistikak eta Agregazioak

Prozesuen Automatizazioa eta Segurtasuna

1. Datu-basearen Diseinua (MariaDB)
Sistemaren muina MariaDB SQL datu-basea da. Diseinua erlazionala da, datuen osotasuna bermatzeko eta erredundantzia minimizatzeko.

Egitura Euskaraz
provincias (Probintziak): Probintzien izenak gordetzen ditu.

offices (Bulegoak): Turismo bulegoak identifikatzen ditu eta probintzia bati lotzen dizkio.

origins (Jatorriak): Bisitarien jatorrizko herrialde edo eskualdeen zerrenda.

visits (Bisitak): Erregistro bakoitzak bisita talde bat adierazten du (data, pertsona kopurua, jatorria eta bulegoa).

stats_turism (Turismo Estatistikak): Bulego eta ordu bakoitzeko agregazioak (max, min, avg) gordetzen dituen taula.

Aurreko Erronkan erabilitako datu-basearen diseinua honako hau zen:

![Captura sarrera](img/Datu%20base/Captura%20de%20pantalla%202026-04-27%20102616.png)


Gaur egun, diseinua eguneratu da estatistiken taula berria integratzeko:

Irudia 2: Gaur egungo datu-basearen diseinu erlazionala.

2. Sinkronizazio Prozesua (ETL)
Datuak Node.js script baten bidez migratzen dira MongoDBtik (iturburua) MariaDBra (helburua). Prozesu honek bi erronka nagusi konpondu ditu:

2.1. Duplikatuak Saihestea
Scriptak SQL SELECT kontsulta bat egiten du datu berri bakoitza txertatu aurretik. Datuak (bulegoa, jatorria, data eta pertsona kopurua) jada existitzen badira, ez dira berriro txertatzen. Honek datuen bikoizketa saihesten du sinkronizazio errepikakorretan.

2.2. Data eta Orduaren Zuzenketa
MongoDBk datak ISO formatuan gordetzen ditu (UTC). Scriptak ordu hauek zuzentzen ditu MariaDBrekin bateragarria den formatura pasatzeko (YYYY-MM-DD HH:MM:SS) eta ordu-eremu lokalera egokitzeko.

3. Estatistikak eta Agregazioak
Sistemak automatikoki kalkulatzen ditu turismo estatistikak, orduko txostenak errazteko.

3.1. stats_turism Taularen Egitura
Taula honek datu agregatuak gordetzen ditu bulego eta ordu bakoitzeko:

fecha_hora: Agregazioaren hasierako ordua.

visitantes_max: Ordu horretako talderik handiena.

visitantes_min: Ordu horretako talderik txikiena.

media_visitantes: Taldeen batezbesteko tamaina.

pais_mas_visitado: Ordu horretan bisitari gehien ekarri dituen herrialdea.

Honako irudian ikus daiteke taularen edukia datu historikoak txertatu ondoren:

Irudia 3: stats_turism taularen edukia (duplikaturik gabe eta ordenatuta).

3.2. Eagregazio Loka (Scripta)
Node.js scripta eguneratu da sinkronizazio bakoitzean estatistikak kalkulatzeko. INSERT ... ON DUPLICATE KEY UPDATE sintaxia erabiltzen da: ordu horretako estatistikak jada existitzen badira, eguneratu egiten dira; bestela, txertatu.

4. Prozesuen Automatizazioa eta Segurtasuna
Proiektuaren baldintzak betetzeko, automatizazio eta segurtasun plan bat diseinatu da, Word dokumentuko baldintzak jarraituz.

4.1. Automatizazioa
Node.js scripta etengabe exekutatzen da atzealdean (PM2 bezalako tresnekin edo Windows Task Scheduler-ekin), 10 segundoro (edo konfiguratutako denboran) sinkronizazioa eta estatistiken kalkulua egiteko.

4.2. Datu-basearen Segurtasuna (Backups)
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
