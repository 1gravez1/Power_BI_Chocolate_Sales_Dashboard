# 🍫 Analiza Prodaje in Uspešnosti (Power BI Dashboard)

## 📌 Pregled projekta
Projekt obsega razvoj interaktivne in optimizirane analitične nadzorne plošče (dashboard) v Power BI za celostno spremljanje prodajnih pošiljk, stroškov, dobičkonosnosti ter uspešnosti prodajne ekipe. Razvoj sledi najboljšim praksam poslovne analitike: od čistega podatkovnega modeliranja in naprednega DAX-a do premišljene uporabniške izkušnje (UX/UI).

![Dashboard screenshot](dashboard_screenshot.png)

---

## 🏗️ 1. Podatkovno modeliranje (Star Schema)
Za zagotavljanje optimalne hitrosti delovanja in preprečevanje zmešnjave v podatkih je model organiziran v **Zvezdno shemo (Star Schema)**. Ta sistem strogo ločuje podatke na dve vrsti tabel:

* **Tabela dejstev (`Fact table`):** Središče zvezde predstavlja glavna in največja tabela `shipments`. Vsebuje transakcijske številke in dogodke, ki jih merimo (npr. beleženje vsakega nakupa/pošiljke s pripadajočo količino in zneski).
* **Dimenzijske tabele (`Dimension tables`):** Žarki zvezde, ki dajejo številkam podrobnosti in kontekst (*Kdo? Kaj? Kdaj? Kje?*). Uporabljene so namenske tabele za kupce (`people`), izdelke (`products`) in koledar (`calendar`).

---

## 📅 2. Upravljanje s časom & Časovna inteligenca (Time Intelligence)

### Uradna datumska tabela
V pogledu *Model View* je tabela `calendar` označena kot **"Mark as Date Table"**. S tem korakom Power BI eksplicitno določi to tabelo kot glavni referenčni vir za vse časovne izračune, s čimer se prepreči samodejno generiranje skritih tabel v ozadju in zagotovi 100 % točnost DAX časovnih funkcij.

### Power Query (PQ) transformacije
V Power Query editorju so bili v tabelo `calendar` dodani naslednji opisni stolpci za granularno filtriranje:
* `year`, `month`, `name of month`...
* **Pomembno:** Dodan je namenski stolpec `start of month`, ki služi kot ključni temelj za napredne vizualne časovne izračune.

---

## 🧮 3. Razvoj DAX Mer (Measures)

Vse kalkulacije so zaradi organiziranosti shranjene v namensko ustvarjeni prazni tabeli **`_Measures`** (vneseni preko gumba *Enter Data*).

### Osnovne mere:
* **Total Sales:** `SUM(shipments[Sales])` *(Format: Currency, brez decimalnih števil)*
* **Total Boxes:** `SUM(shipments[Boxes])`
* **Total Shipments:** `COUNTROWS(shipments)`

### Izračun stroškov in dobička (Vrstični kontekst -> Agregacija):
1. Najprej je bil v tabeli pošiljk ustvarjen kalkuliran stolpec `Costs`, ki s funkcijo `RELATED` prenese strošek posamezne škatle iz dimenzijske tabele produktov in ga pomnoži s številom škatel v pošiljki:
   $$\text{Costs} = \text{RELATED}(\text{products[Cost per box]}) \times \text{shipments[Boxes]}$$
2. **Total Costs:** `SUM(shipments[Costs])`
3. **Total Profit:** `[Total Sales] - [Total Costs]`

### LBS Analiza (Pošiljke z manj kot 50 škatlami):
* **LBS Count:** `CALCULATE([Total Shipments], shipments[Boxes] < 50)`
* **% LBS:** `DIVIDE([LBS Count], [Total Shipments])`

### Časovna primerjava (Month-over-Month v tabelah):
* **Total Sales (prev month):** `CALCULATE([Total Sales], PREVIOUSMONTH('calendar'[Date]))`
* **MoM Sales Change %:**
    ```dax
    MoM Sales Change % = 
    VAR this_month = [Total Sales]
    VAR prev_month = [Total Sales (prev month)]
    RETURN DIVIDE(this_month - prev_month, prev_month)
    ```

---

## 🎨 4. Oblikovanje strani & Napredne vizualizacije (Page Design)

### Nastavitve platna (Canvas)
Privzeta velikost strani (1280 x 720 px) je bila preko *Format -> Canvas Settings* nadgrajena na standardno Full HD ločljivost **1920 x 1080 px** za večjo preglednost in profesionalno postavitev.

### Reševanje konteksta filtra na ravni KPI kartic (*Reference Labels*)
Ker klasične časovne mere na samostojnih karticah ne delujejo pravilno zaradi odsotnosti vrstičnega konteksta mesecev, je bila logika razvita preko treh naprednih mer, ki dinamično izolirajo zadnji razpoložljiv mesec:
1. **Latest Date:** `LASTDATE('calendar'[Start of Month])`
2. **Total Sales Latest Month:**
    ```dax
    Total Sales Latest Month = 
    VAR ld = [Latest Date]
    RETURN CALCULATE([Total Sales], 'calendar'[Start of Month] = ld)
    ```
3. **Latest MoM Sales Change %:** (Uporabljeno kot *Reference label* pod glavnim KPI-jem):
    ```dax
    Latest MoM Sales Change % = 
    VAR ld = [Latest Date]
    VAR this_month_sales = [Total Sales Latest Month]
    VAR prev_month_sales = CALCULATE([Total Sales], 'calendar'[Start of Month] = EDATE(ld, -1))
    RETURN DIVIDE(this_month_sales - prev_month_sales, prev_month_sales)
    ```
* **Pogojno oblikovanje (Conditional Formatting):** Vrednost znotraj *Reference label* se dinamično obarva **rdeče**, če je stopnja rasti manjša od 0, in **zeleno**, če je večja od 0.

### Vizualni elementi:
* **Gauge chart (Merilnik):** Implementiran za prikaz `% Profit` in prilagojen celostni grafični podobi.
* **Dinamična analiza trendov (Field Parameter):** Namesto petih ločenih grafov je bil ustvarjen en sam linijski grafikon. Preko *Modeling -> New Parameter -> Fields* je bil ustvarjen parameter **`Measure Selector`**. Na X-os je umeščen datum, na Y-os pa `Measure Selector`, kar uporabniku omogoča dinamično preklapljanje med prikazanimi metrikami znotraj enega vizuala.
* **Histogram porazdelitve velikosti pošiljk:** Ustvarjen s pomočjo funkcije skupin (*grouping/binning*) nad poljem `Boxes`. Na X-osi so prikazani `bins (boxes)`, na Y-osi pa `Total Shipments`. Dodan je *Zoom slider* za interaktivno prilagajanje razpona X-osi.

---

## 🔄 5. Napredna UX interakcija s preklapljanjem tabel (Bookmarks)

Za maksimalni izkoristek prostora sta bili tabela prodajalcev in tabela produktov umeščeni neposredno ena na drugo, med njima pa uporabnik preklaplja s pritiskom na ikoni (`people` in `product`).

### Tehnična izvedba preko zaznamkov (Bookmarks):
1. **Indikator ciljev prodajalcev:** V tabeli prodajalcev je bila implementirana logika za doseganje ciljne dobičkonosnosti:
   `Profit Target Indicator = IF([% Profit] > [Profit Target], 2, 1)`
2. Z uporabo plošč **Selection** in **Bookmarks** (zavihek *View*) so bili vsi elementi (obe tabeli in obe ikoni) natančno poimenovani.
3. **Zaznamek 1 (Sales Table):** Tabela produktov je skrita, tabela prodajalcev je vidna. Zaznamek je shranjen.
4. **Zaznamek 2 (Product Table):** Tabela prodajalcev je skrita, tabela produktov je vidna. Zaznamek je shranjen.
5. Zaznamki so bili dodeljeni ikonam preko *Format -> Action -> Bookmark*.
* ⚠️ **Kritična opomba za pravilno delovanje:** Ker zaznamki privzeto shranijo tudi stanje podatkov (trenutne filtre), je bilo pri obeh zaznamkih ročno **odznačeno polje "Data"**. To zagotavlja, da se ob preklopu med tabelama vizualno spremeni oblika, vsi trenutno izbrani filtri na poročilu pa ostanejo nedotaknjeni.

---

## 🚀 Kako zagnati projekt?
1. Prenesite datoteko s končnico `.pbix` iz tega repozitorija.
2. Zaženite jo s programom **Power BI Desktop**.
