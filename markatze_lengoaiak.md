---
layout: page
title: Markatze lengoaiak
---

# Turismo Analitika: Proiektuaren Egitura eta Garapen Teknikoa

Proiektu honek **turismo-fluxuak aztertzeko plataforma digital bat** aurkezten du. Dokumentazio honek sistemaren arkitektura teknikoa, fitxategien antolaketa eta datuen tratamendua azaltzen ditu.

---

## 1. Arkitektura Teknikoa

Aplikazioa bezeroaren aldeko (*client-side*) arkitektura baten gainean eraiki da, teknologia hauek erabiliz:

* **Datu-egitura (XML):** Informazio guztia formatu hierarkikoan gordetzen da, beste sistema batzuekin interoperabilitatea bermatzeko.
* **Logika Dinamikoa (jQuery & AJAX):** Datuak modu asinkronoan kargatzen dira, webgunea freskatu beharrik gabe erabiltzailearen esperientzia hobetzeko.
* **Bistaratze Grafikoa (Chart.js):** Datu estatistikoak (maximoak, minimoak eta batez bestekoak) modu bisualean interpretatzeko liburutegia.
* **Diseinu Arduratsua (CSS3):** Flexbox eta Sticky posizionamendua erabili dira interfaze garbi eta moldagarri bat lortzeko.

---

## 2. Fitxategien Egitura eta Eginkizunak

Proiektua modulu hauetan banatuta dago:

###  HTML (Egitura)
* **`sarrera.html`**: Hasiera orria. Proiektuaren helburu estrategikoak eta testuingurua aurkezten ditu.
![Captura sarrera](img/Markatze%20lengoaiak/sarrera.png)

* **`3erronka.html`**: Dashboard nagusia. Hemen kokatzen dira iragazkiak, grafiko interaktiboak eta eguneko datuen fitxak.
![Captura dashboard](img/Markatze%20lengoaiak/3erronka.png)

* **`txostena.html`**: Open Data atala. Datuak JSON formatuan deskargatzeko gunea.
![Captura txostena](img/Markatze%20lengoaiak/txostena.png)

###  CSS (Diseinua)
* **`3erronka.css`**: Estilo fitxategi bateratua. Kolore paleta berdea erabili da jasangarritasunaren irudia indartzeko eta osagaien itxura definitzen du.

### JavaScript (Logika)
* **`3erronka.js`**: Fitxategi nagusia. XML datuen karga (AJAX), datuen parseatzea eta grafikoaren eguneraketa kudeatzen ditu.
* **`opendata.js`**: Deskarga sistemaren logika gehigarria kudeatzeko erabilgarria.

###  Data (Iturria)
* **`datuak.xml`**: Proiektuaren "datu-basea". Ofizina, data, bisitari kopurua eta jatorria biltzen dituen fitxategia.

---

## 3. Datuen Fluxua (Data Workflow)

Aplikazioak prozesu hau jarraitzen du datuak erakusteko:

1. **Eskaera (Request):** Orrialdea kargatzean, JavaScript-ak AJAX eskaera bat egiten du `datuak.xml` fitxategia lortzeko.
2. **Prozesatzea (Parsing):** XML testua DOM objektu bihurtzen da. Algoritmoak datuak array-etan antolatzen ditu.
3. **Iragaztea (Filtering):** Erabiltzaileak hautatzaileak aldatzean, logika honek datu espezifikoak erauzten ditu.
4. **Eguneratzea (Rendering):** Chart.js liburutegiak grafikoa berriz marrazten du dinamikoki.

---

## 4. Funtzio Nagusiak

1. **Datuen Bistaratzea:** Panel nagusian, erabiltzaileak bisitari kopuruen grafiko dinamikoak ikus ditzake.
2. **Iragazki Sistema:** Datuak **egunaren** arabera edo **jatorriaren** (bulegoa) arabera iragazi daitezke.
3. **Open Data eta Deskargak:** Proiektuak gardentasuna sustatzen du; erabiltzaileak datu gordinak **JSON formatuan** deskarga ditzake `txostena.html` orrialdean.  
