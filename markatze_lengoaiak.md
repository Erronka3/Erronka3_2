---
layout: page
title: Markatze lengoaiak
---

# Turismo Analitika: Proiektuaren Egitura eta Garapen Teknikoa

Proiektu honek **turismo-fluxuak aztertzeko plataforma digital bat** aurkezten du. Dokumentazio honek sistemaren arkitektura teknikoa, fitxategien antolaketa eta datuen tratamendua azaltzen ditu.

---

## 1. Arkitektura Teknikoa
Aplikazioa bezeroaren aldeko (*client-side*) arkitektura baten gainean eraiki da, teknologia hauek erabiliz:

* **Datu-egitura (XML):** Informazio guztia formatu hierarkikoan gordetzen da.
* **Logika Dinamikoa (jQuery & AJAX):** Datuak modu asinkronoan kargatzen dira, webgunea freskatu beharrik gabe.
* **Bistaratze Grafikoa (Chart.js):** Datu estatistikoak modu bisualean interpretatzeko liburutegia.
* **Diseinu Arduratsua (CSS3):** *Flexbox* eta *Sticky* posizionamendua erabili dira.

---

## 2. Fitxategien Egitura eta Eginkizunak

### 📂 HTML (Egitura)
* **`sarrera.html`**: Hasiera orria eta proiektuaren testuingurua.
![Captura sarrera](img/Markatze%20lengoaiak/sarrera.png)

* **`3erronka.html`**: Dashboard nagusia (iragazkiak eta grafikoak).
![Captura dashboard](img/Markatze%20lengoaiak/3erronka.png)

* **`txostena.html`**: Open Data atala, JSON deskargekin.
![Captura txostena](img/Markatze%20lengoaiak/txostena.png)

### 🎨 CSS eta ⚙️ JS
* **`3erronka.css`**: Estilo fitxategi bateratua (kolore paleta berdea).
* **`3erronka.js`**: Logika nagusia (AJAX, Parsing eta Chart.js).
* **`opendata.js`**: Deskarga sistemarako logika gehigarria.

### 📊 Data
* **`datuak.xml`**: Proiektuaren "datu-basea".

---

## 3. Datuen Fluxua (Data Workflow)

1. **Eskaera (Request):** AJAX bidez `datuak.xml` fitxategia lortzen da.
2. **Prozesatzea (Parsing):** XMLa DOM objektu bihurtzen da eta datuak antolatzen dira.
3. **Iragaztea (Filtering):** Erabiltzaileak hautatzaileak aldatzean, datu espezifikoak erauzten dira.
4. **Eguneratzea (Rendering):** Grafikoa berriz marrazten da dinamikoki.

---

## 4. Funtzio Nagusiak

* **Datuen Bistaratzea:** Grafiko dinamikoak panel nagusian.
* **Iragazki Sistema:** Datuak **egunaren** edo **jatorriaren** arabera iragazi daitezke.
* **Open Data:** Datu gordinak **JSON formatuan** deskargatzeko aukera.
