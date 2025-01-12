**Databázové technológie ETL projekt chinook**

**Autor**: Erik Kováč

Môj projekt je zameraný na spracovanie údajov z databázy Chinook prostredníctvom ETL procesu, pričom dáta sú organizované v rámci hviezdicovej schémy.

**Zdrojový dataset:** 
Chinook databáza je vzorová relačná databáza obsahujúca informácie o hudobných albumoch, skladbách, interpretoch, zákazníkoch, objednávkach a faktúrach.
Údaje z tejto databázy budú transformované a optimalizované na analytické účely prostredníctvom platformy Snowflake.

---
## **1. Úvod a popis zdrojových dát**
Cieľom semestrálneho projektu je analyzovať dáta v databáze Chinook, pričom sa zameriame na používateľov, ich preferencie a a kúpi skladieb. 
Táto analýza umožní identifikovať trendy v záujmoch používateľov, najpopulárnejšie položky (napríklad skladby alebo albumy) a správanie používateľov.
Dataset obsahuje tabulky:
- `playlist`: Informácie o playlistoch vytvorených užívatelmi.
- `playlisttrack`: Spojovacia tabuľka pre playlisty a skladby.
- `track`: Informácie o skladbách.
- `album`: Informácie o hudobných albumoch.
- `artist`: Informácie o interpretoch.
- `customer`: Informácie o zákazníkoch.
- `employee`: Informácie o zamestnancoch.
- `genre`: Informácie o žánroch skladieb.
- `invoice`: Informácie o fakturach a predajoch.
- `invoiceline`: Dodatočné informácie k faktúram.
- `mediatype`: Dodatočné informácie o type média skladby.
 
---
### **1.1 Dátová architektúra**

 Dáta sú usporiadané v relačnom modeli, ktorý je znázornený na **entitno-relačnom diagrame (ERD)**:

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/erd/Chinook_ERD.png" alt="ERD Schema">
  <br>
  <em> Entitno-relačná schéma Chinook </em>
</p>

---
## **2 Dimenzionálny model**

Navrhnutý bol **hviezdicový model (star schema)**, pre efektívnu analýzu kde centrálny bod predstavuje faktová tabuľka **`fact_invoice`**, ktorá je prepojená s nasledujúcimi dimenziami:
- **`dim_track`**: Zahŕna údaje o skladbách , albumoch , interpretoch a žánroch.
- **`dim_customer`**: Obsahuje informácie o zákazníkoch, ktorí vykonali nákupy.
- **`dim_employee`**: Obsahuje informácie o zamestnancoch, ktorí sa podieľali na transakciách.
- **`dim_billing_adress`**: Táto tabuľka obsahuje informácie o geografických lokalitách.
- **`dim_date`**: Táto tabuľka poskytuje podrobnosti o čase a dátumoch pre analýzu. 



<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/scd/star.png" alt="ERD Schema">
  <br>
  <em> Star schema Chinook </em>
</p>

---
## **3. ETL proces v Snowflake**
ETL proces v Snowflake zahŕňal tri hlavné fázy: extrakciu (Extract), transformáciu (Transform) a načítanie (Load). Tento postup umožnil spracovanie zdrojových dát zo staging vrstvy do viacdimenzionálneho modelu, ktorý je vhodný na analýzu a vizualizáciu.

---
### **3.1 Extract (Extrahovanie dát)**
Dáta zo zdrojových súborov vo formáte .csv boli uložené do Snowflake v dočasnom úložisku TEMP_STAGE. Pred týmto krokom bola inicializovaná databáza, dátový sklad a schéma. Následne boli údaje nahrané do staging tabuliek. Proces bol spustený pomocou nasledujúcich príkazov:

```sql
CREATE DATABASE TURTLE_CHINOOK;
CREATE OR REPLACE SCHEMA TURTLE_CHINOOK.staging;
```

Kroky extrakcie dát:

Vytvorenie staging tabuliek pre všetky zdrojové údaje (napr. zamestnanci, zákazníci, faktúry, skladby, žánre a ďalšie). Použitie príkazu COPY INTO na nahranie dát z .csv súborov do príslušných staging tabuliek.

Príklad pre tabuľku employee_stage:

```sql
CREATE OR REPLACE TABLE employee_staging (
    employee_id INT AUTOINCREMENT PRIMARY KEY,
    first_name  VARCHAR(50)  NOT NULL,
    last_name   VARCHAR(50)  NOT NULL,
    title       VARCHAR(200) NOT NULL,
    reports_to  INT,
    birth_date  DATE         NOT NULL,
    hire_date   DATE         NOT NULL,
    address     STRING       NOT NULL,
    city        VARCHAR(50)  NOT NULL,
    state       VARCHAR(50),
    country     VARCHAR(50)  NOT NULL,
    postal_code VARCHAR(25),
    phone       VARCHAR(30),
    fax         VARCHAR(30),
    email       VARCHAR(60)  NOT NULL,
    FOREIGN KEY (reports_to) REFERENCES employee_staging(employee_id)
);

COPY INTO employee_staging
FROM @PENGUIN_CHINOOK_STAGE/
FILES = ('employee.csv')
FILE_FORMAT = (
    TYPE = 'CSV',
    COMPRESSION = 'NONE',
    FIELD_DELIMITER = ',',
    FILE_EXTENSION = 'csv',
    FIELD_OPTIONALLY_ENCLOSED_BY = '"',
    SKIP_HEADER = 1,
    ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE);
```

Rovnaký prístup sa aplikoval na všetky ostatné zdrojové dáta, pričom pre každý súbor bola vytvorená štruktúrovaná staging tabuľka.
---
### **3.2 Transfor (Transformácia dát)**
Transformácia dát zahŕňala čistenie, obohacovanie a reorganizáciu údajov do dimenzií a faktových tabuliek, ktoré podporujú viacdimenzionálnu analýzu.

Príklad transformácie:

Dimenzia dim_date: Táto dimenzia obsahuje informácie o dátumoch súvisiacich s fakturačnými údajmi a zahŕňa odvodené atribúty ako rok, mesiac, deň, týždeň a štvrťrok.

```sql
CREATE OR REPLACE TABLE dim_date (
    date_id         INT AUTOINCREMENT PRIMARY KEY,
    timestamp       TIMESTAMP NOT NULL,
    years           INT       NOT NULL,
    months          INT       NOT NULL,
    month_as_string STRING    NOT NULL,
    days            INT       NOT NULL,
    day_as_string   STRING    NOT NULL,
    is_weekend      BOOLEAN   NOT NULL
);


INSERT INTO dim_date (timestamp, years, months, month_as_string, days, day_as_string, is_weekend)
SELECT invoice_date AS timestamp,
    EXTRACT(YEAR FROM invoice_date) AS years,
    EXTRACT(MONTH FROM invoice_date) AS months,
    CASE 
        WHEN EXTRACT(MONTH FROM invoice_date) = 1 THEN 'January'
        WHEN EXTRACT(MONTH FROM invoice_date) = 2 THEN 'February'
        WHEN EXTRACT(MONTH FROM invoice_date) = 3 THEN 'March'
        WHEN EXTRACT(MONTH FROM invoice_date) = 4 THEN 'April'
        WHEN EXTRACT(MONTH FROM invoice_date) = 5 THEN 'May'
        WHEN EXTRACT(MONTH FROM invoice_date) = 6 THEN 'June'
        WHEN EXTRACT(MONTH FROM invoice_date) = 7 THEN 'July'
        WHEN EXTRACT(MONTH FROM invoice_date) = 8 THEN 'August'
        WHEN EXTRACT(MONTH FROM invoice_date) = 9 THEN 'September'
        WHEN EXTRACT(MONTH FROM invoice_date) = 10 THEN 'October'
        WHEN EXTRACT(MONTH FROM invoice_date) = 11 THEN 'November'
        WHEN EXTRACT(MONTH FROM invoice_date) = 12 THEN 'December'
    END AS month_as_string,
    EXTRACT(DAY FROM invoice_date) AS days,
    CASE 
        WHEN EXTRACT(DOW FROM invoice_date) = 0 THEN 'Sunday'
        WHEN EXTRACT(DOW FROM invoice_date) = 1 THEN 'Monday'
        WHEN EXTRACT(DOW FROM invoice_date) = 2 THEN 'Tuesday'
        WHEN EXTRACT(DOW FROM invoice_date) = 3 THEN 'Wednesday'
        WHEN EXTRACT(DOW FROM invoice_date) = 4 THEN 'Thursday'
        WHEN EXTRACT(DOW FROM invoice_date) = 5 THEN 'Friday'
        WHEN EXTRACT(DOW FROM invoice_date) = 6 THEN 'Saturday'
    END AS day_as_string,
    CASE WHEN EXTRACT(DOW FROM invoice_date) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend
FROM invoice_staging;
```

Dimenzia dim_customer: Obsahuje údaje o zákazníkoch ako meno, adresa, mesto a krajina, odvodené zo staging tabuľky customer_stage.

```sql
CREATE OR REPLACE TABLE dim_customer (
    customer_id INT AUTOINCREMENT PRIMARY KEY,
    first_name  STRING NOT NULL,
    last_name   STRING NOT NULL,
    city        STRING NOT NULL,
    state       STRING,
    country     STRING NOT NULL,
    postal_code STRING,
    email       STRING);
```

---    
### **3.3 Load (Načítanie dát)**
Po úspešnom vytvorení dimenzií a faktových tabuliek boli staging tabuľky odstránené, aby sa optimalizovalo úložisko. Príklad čistenia staging tabuliek:

```sql
DROP TABLE IF EXISTS artist_staging;
DROP TABLE IF EXISTS album_staging;
DROP TABLE IF EXISTS customer_staging;
DROP TABLE IF EXISTS employee_staging;
DROP TABLE IF EXISTS genre_staging;
DROP TABLE IF EXISTS mediatype_staging;
DROP TABLE IF EXISTS playlist_staging;
DROP TABLE IF EXISTS track_staging;
DROP TABLE IF EXISTS playlisttrack_staging;
DROP TABLE IF EXISTS invoice_staging;
DROP TABLE IF EXISTS invoiceline_staging;
```

---
## **4 Vizualizácia dát**
<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/ALL.PNG" alt="Dashboard">
  <br>
  <em> Dashboard Chinook datasetu </em>
</p>

---  

### **4.1 Songs Per Genre**
Transformácia dát zahŕňala čistenie, obohacovanie a reorganizáciu údajov do dimenzií a faktových tabuliek, ktoré podporujú viacdimenzionálnu analýzu.

Príklad transformácie:
Dimenzia dim_genre: Táto dimenzia uchováva informácie o jednotlivých hudobných žánroch a počte skladieb prislúchajúcich každému žánru. 
Obsahuje odvodené atribúty ako názov žánru, celkový počet skladieb a priemerná dĺžka skladieb v rámci žánru.

```sql
SELECT t.genre, COUNT(*) AS Sales_Count
FROM dim_track t
JOIN fact_invoice fi ON t.dim_track_id = fi.dim_track_id
GROUP BY t.genre
ORDER BY Sales_Count;
```

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/SongsPerGenre.PNG" alt="Songs Per Genre">
  <br>
  <em> Star schema Chinook </em>
</p>

---  

### **4.2 Number Of Songs Per Length**
 Táto vizualizácia analyzuje počet skladieb podľa dĺžky, pričom dĺžka je rozdelená na kategórie: dlhé, stredné a krátke. Pre každú kategóriu dĺžky vypočíta počet skladieb, ktoré do nej spadajú. Umožňuje identifikovať, ktoré dĺžky skladieb sú najbežnejšie a poskytuje prehľad o distribúcii skladieb podľa ich trvania.
 
```sql
SELECT dt.len, SUM(quantity) AS Quantity
FROM fact_invoice fi
JOIN dim_track dt ON fi.dim_track_id = dt.dim_track_id
GROUP BY dt.len;
```

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/NumberOfSongsPerLength.PNG" alt="ERD Schema">
  <br>
  <em> Star schema Chinook </em>
</p>

--- 

### **4.3 Songs In Price Bracket**
Táto vizualizácia poskytuje prehľad o rozdelení počtu skladieb podľa cenových kategórií. Výsledky sú zoradené podľa počtu skladieb v jednotlivých cenových kategóriách, konkrétne 0.99 a 1.99, čo umožňuje identifikovať, ktorá cenová kategória je najpopulárnejšia medzi zákazníkmi.
 
 ```sql
SELECT unit_price as Price, SUM(quantity) AS Quantity
FROM fact_invoice
GROUP BY unit_price;
```

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/SongsInPriceBracket.PNG" alt="ERD Schema">
  <br>
  <em> Star schema Chinook </em>
</p>


---    

### **4.4 Song Length Stats**
 Táto vizualizácia vypočíta priemernú dĺžku skladieb a celkovú dĺžku všetkých skladieb v databáze. Spojí údaje o skladbách s ich dĺžkou, aby sa získali štatistiky, ktoré umožňujú analyzovať, či priemerná dĺžka skladieb rastie alebo klesá. Táto vizualizácia poskytuje prehľad o trendoch v dĺžkach skladieb a ich vplyve na celkový čas trvania všetkých skladieb.
 
 ```sql
SELECT AVG(dt.milliseconds) / 1000 / 60
FROM fact_invoice fi
JOIN dim_track dt ON fi.dim_track_id = dt.dim_track_id
GROUP BY dt.len;



SELECT SUM(dt.milliseconds) / 1000 / 60
FROM fact_invoice fi
JOIN dim_track dt ON fi.dim_track_id = dt.dim_track_id
GROUP BY dt.len;
```

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/LengthStats.PNG" alt="Length stats">
  <br>
  <em> Star schema Chinook </em>
</p>


---    

### **4.5 Sales Stats**
 Táto vizualizácia analyzuje celkové predaje za jednotlivé roky, ako aj predaje za najnovší rok 2025. Pre každý rok vypočíta súčet hodnôt všetkých predajov, pričom porovnáva výsledky za celý časový rámec s predajmi v roku 2025. Výsledky sú zoradené chronologicky podľa rokov a zároveň umožňujú získať prehľad o najnovších trendoch v predajoch, čo umožňuje identifikovať zmeny v predajnom výkone v porovnaní s minulými rokmi.
 
 ```sql
SELECT dd.years, COUNT(*) AS Sales_Count
FROM fact_invoice t
JOIN dim_date dd ON dd.date_id = t.dim_date_id
GROUP BY dd.years;



SELECT dd.month_as_string AS month, COUNT(fi.fact_invoice_id) AS sales_count, dd.months
FROM fact_invoice fi
JOIN dim_date dd ON fi.dim_date_id = dd.date_id
WHERE dd.years = 2025
GROUP BY dd.month_as_string, dd.months
ORDER BY dd.months;
```

<p align="center">
  <img src="https://github.com/D-Duck/SCHL_ChinukETL/blob/main/png/Sales.PNG" alt="sales stats">
  <br>
  <em> Star schema Chinook </em>
</p>
