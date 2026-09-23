# Laboratorul 2: Cuantifică citirile Dashboard-ului

## 1. Cerințe de calitate

### Latența citirilor

#### Definirea punctelor de măsurare

Măsurăm **latența observată de client** (*Client-observed latency*):

* **Punct de început (T0):** Momentul în care Utilizatorul autentificat inițiază acțiunea în interfață (trimiterea cererii HTTP de către browser).
* **Punct de sfârșit (T1):** Momentul în care datele corecte primite sunt complet randate și devin utilizabile vizual în browserul Utilizatorului.
* **Interval măsurat:** Include latența de rețea dus-întors, procesarea pe server, accesul la date și randarea în interfața grafică (UI).

##### Sursă:

> [1] *„Latency and response time are often used synonymously, but they are not the same. The response time is what the client sees: besides the actual time to process the request (the service time), it includes network delays and queueing delays. Latency is the duration that a request is waiting to be handled — during which it is latent, awaiting service.”
> „Due to this effect, it is important to measure response times on the client side.”*

#### Percentile pentru „majoritatea citirilor” și percepția de „imediat”

Media aritmetică ascunde cererile lente din coada distribuției. Prin urmare:

* Folosim percentila **p50** (mediana) pentru a monitoriza comportamentul tipic.
* Folosim percentila **p95** (95 din 100 de cereri) pentru a defini riguros cerința clientului ca „majoritatea citirilor corecte să se finalizeze în cel mult 2 secunde în perioadele aglomerate”.

##### Sursă:

> [1] *„However, the mean is not a very good metric if you want to know your 'typical' response time, because it doesn’t tell you how many users actually experienced that delay. Usually it is better to use percentiles... The median is also known as 50th percentile, and sometimes abbreviated as p50... In order to figure out how bad your outliers are, you look at higher percentiles: the 95th, 99th and 99.9th percentile are common (abbreviated p95, p99 and p999).”*

<img src="/Users/victorcoselev/Library/Application%20Support/marktext/images/870e20dd9a4ef49fe9810d5ce8fda12c3a1a63b7.png" title="" alt="" data-align="left" />

#### Comportamentul la limita de 2 secunde (Timeout)

* Pragul de **2 secunde (2.000 ms)** este limita maximă de toleranță operațională (*hard timeout*).
* Dacă o cerere atinge 2 secunde fără un răspuns valid, conexiunea este întreruptă controlat. Sistemul anulează procesarea și afișează imediat starea explicită `indisponibil`.
* Nu se permite continuarea procesării agățate în fundal și nici blocarea interfeței grafice a utilizatorului.

#### Regula de separare a rezultatelor

Un răspuns rapid de eroare (de exemplu, un `504 Gateway Timeout` returnat în 50 ms) respectă ținta numerică de timp, dar **nu este considerat o citire corectă**. Calitatea latenței se aplică exclusiv citirilor corecte finalizate.

#### Cerințe măsurabile:

1. **Pentru `Stock price` (Cerința: „să pară imediate”):**

   > *În perioadele de vârf de utilizare, latența observată de client pentru citirile corecte de `Stock price` trebuie să fie **p50 ≤ 150 ms** și **p95 ≤ 500 ms**.*

2. **Pentru celelalte citiri (`Overview`, `Filter`, `History`, `Watchlist`, `Search`):**

   > *În perioadele aglomerate ale zilei, latența observată de client pentru **95% dintre citirile corecte (p95)** trebuie să fie **≤ 2.000 ms**, iar orice cerere care depășește pragul de 2.000 ms trebuie întreruptă și afișată explicit ca `indisponibil`.*

##### Sursă:

> [4] *„A quality attribute scenario is a quality-attribute-specific requirement. It consists of six parts: source of stimulus, stimulus, artifact, environment, response, and response measure.”*

---

### Disponibilitate bazată pe timpul de funcționare (Uptime Availability)

#### Când este Dashboard-ul „utilizabil”

Dashboard-ul este considerat **utilizabil** dacă un Utilizator autentificat se poate autentifica și accesa aplicația, iar fluxurile de bază (`Overview`, `Watchlist`) returnează date valide sau stări de eroare controlate/etichetate conform regulilor de afișare (fără erori de infrastructură sau căderi 5xx netratate).

#### Definirea intervalelor pe o perioadă de 30 de zile

Considerăm o lună standard de 30 de zile (30 zile × 24 ore/zi = 720 ore):

* **Ore de tranzacționare:** Corespund programului burselor de acțiuni, Luni–Vineri, 09:00 – 17:00 (8 ore/zi).
  * Într-o lună de 30 de zile există **21 de zile de tranzacționare** (zile lucrătoare).
  * Timp măsurat tranzacționare = 21 zile × 8 ore = 168 ore = 10.080 minute = 604.800 secunde.
* **Restul zilei (ore non-trading și weekenduri):**
  * Timp măsurat non-trading = 720 ore − 168 ore = 552 ore = 33.120 minute = 1.987.200 secunde.

#### Ținte exprimate în număr de nouari și bugetul de indisponibilitate

Buget de indisponibilitate = Timp total măsurat × (1 − Țintă disponibilitate)

##### Sursă:

> [3] *„An error budget is 1 minus the SLO of the service. A 99.9% SLO service has a 0.1% error budget.”*

| Interval                  | Țintă selectată | Procent indisponibil | Calculul matematic          | Buget de indisponibilitate (în 30 de zile)    |
|:------------------------- |:---------------:|:--------------------:|:--------------------------- |:--------------------------------------------- |
| **Ore de tranzacționare** | **99.9%**       | 0.1%                 | 168 ore × 0.001 = 0.168 ore | **10 minute și 5 secunde** (604.8 s)          |
| **Restul zilei**          | **99.0%**       | 1.0%                 | 552 ore × 0.01 = 5.52 ore   | **5 ore, 31 minute și 12 secunde** (19.872 s) |

#### Justificarea diferențierii țintelor (analiza ROI)

* **Orele de tranzacționare:** Deciziile financiare se iau în timp real; indisponibilitatea în timpul pieței deschise cauzează pierderi directe de capital, blocaje critice și daune reputaționale majore. Ținta de **99.9%** este imperativă.
* **Restul zilei:** Piețele sunt închise, utilizatorii doar consultă portofoliile, iar tranzacțiile nu se execută. O țintă de **99.0%** este optimă prin prisma ROI: oferă o fereastră de peste 5 ore/lună pentru backup-uri, actualizări batch și mentenanță fără costuri disproporționate de infrastructură de înaltă disponibilitate 24/7.

#### Cerințe măsurabile:

> 1. *În intervalul orelor de tranzacționare (Luni–Vineri, 09:00–17:00), pe parcursul oricărui interval de 30 de zile, timpul de funcționare utilizabil al Dashboard-ului trebuie să fie de **cel puțin 99.9%** (buget maxim de indisponibilitate: **10 minute și 5 secunde**).*
> 2. *În afara orelor de tranzacționare (nopți și weekenduri), pe parcursul oricărui interval de 30 de zile, timpul de funcționare utilizabil al Dashboard-ului trebuie să fie de **cel puțin 99.0%** (buget maxim de indisponibilitate: **5 ore, 31 minute și 12 secunde**).*

##### Sursă:

> [2] *„...as we build systems, cost does not increase linearly as reliability increments—an incremental improvement in reliability may cost 100x more than the previous increment. The costliness has two dimensions: The cost of redundant machine/compute resources... The opportunity cost... We strive to make a service reliable enough, but no more reliable than it needs to be.”*

---

### Consistență

#### A. Pentru `Stock price` (Model: Bounded Staleness & Actualitate)

* **Vechime maximă acceptată și etichetare:**
  * Furnizorul are o întârziere estimată de piață de aproximativ 15 minute (+ 1 minut marjă tehnică de transfer).
  * Dacă vechimea datelor primite este ≤ 16 minute, cotația este afișată normal alături de ora furnizorului.
  * Dacă vechimea este între **16 și 30 de minute**, cotația este afișată, dar **trebuie etichetată vizibil ca `întârziat`** împreună cu ora furnizorului.
* **Momentul când rezultatul devine indisponibil:**
  * Dacă datele furnizorului depășesc o vechime de **30 de minute** sau dacă furnizorul nu a livrat o cotație, rezultatul devine strict **`indisponibil`**.
* **Interdicție:** Sistemul nu va afișa niciodată valoarea `0` pentru un preț lipsă sau expirat și nu va declara un preț vechi ca fiind curent.

> *Cerință măsurabilă:*  
> *La orice citire a unui `Stock price`, dacă vechimea datelor primite de la furnizor este ≤ 30 minute, sistemul returnează cotația însoțită de ora furnizorului (marcată explicit ca `întârziat` dacă depășește 16 minute); dacă vechimea depășește 30 de minute sau datele lipsesc, citirea trebuie să returneze rezultatul `indisponibil` și nu valoarea `0`.*

#### B. Pentru modificarea `Watchlist`-ului (Model: Read-Your-Own-Writes)

* **Comportament:**
  * Modificarea Watchlist-ului (adăugare/ștergere simbol) este o scriere confirmată.
  * Odată ce Dashboard-ul a confirmat salvarea modificării, **următoarea citire a Watchlist-ului efectuată de acel Utilizator trebuie să reflecte imediat modificarea respectivă**.
  * Nu este permis ca o citire ulterioară a aceluiași utilizator să afișeze o stare din cache anterioară salvării (citire monotonă).
  * Watchlist-ul este strict privat (izolat între utilizatori).

> *Cerință măsurabilă:*  
> *După ce Dashboard-ul confirmă succesul modificării Watchlist-ului unui Utilizator autentificat, 100% dintre citirile ulterioare ale Watchlist-ului efectuate în cadrul sesiunii acelui Utilizator trebuie să afișeze modificarea salvată (garanție Read-Your-Own-Writes).*

---

### Surse și cercetare

1. **Martin Kleppmann: Designing Data-Intensive Applications (DDIA)**  
   *URL:* [Cartea în format PDF](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-\(2017\).pdf) *(Capitolul 1 – Describing Performance & Percentiles, pag. 11–14)*  
   Sursa fundamentează pentru  decizia de a măsura timpul observat de client (*response time*, T0–T1), care include întârzierile de rețea dus-întors și cozile de așteptare, subliniind că evaluarea performanței trebuie făcută pe partea de client. De asemenea, Kleppmann demonstrează că media aritmetică este o metrică neconcludentă deoarece ascunde anomaliile din coada distribuției cauzate de concurență și blocaje (*head-of-line blocking*). Acest lucru justifică utilizarea percentilei **p50 (mediana)** pentru a urmări experiența utilizatorului tipic și a percentilei **p95** pentru a garanta contractual respectarea limitei de 2 secunde în perioadele aglomerate. În plus, validează regula de separare a rezultatelor (un răspuns rapid de eroare nu este o citire validă).

2. **Google SRE Book: Embracing Risk (Cap. 3) & Service Level Objectives (Cap. 4)**  
   *URL:* https://sre.google/sre-book/embracing-risk/ și https://sre.google/sre-book/service-level-objectives/  
   Sursa confirmă principiul că o disponibilitate de 100% este o țintă nerealistă și contraproductivă, iar fiecare „nouă” suplimentar adăugat crește costul de infrastructură și redundanță în mod exponențial (până la 100x). Această analiză a riscului și a costurilor justifică decizia noastră de ROI de a diferenția țintele pe două intervale distincte: **99.9% în orele de tranzacționare** (când indisponibilitatea produce pagube financiare directe și blocaje critice) și **99.0% în restul zilei** (când piețele sunt închise, permițând ferestre de mentenanță fără costuri operaționale disproporționate).

3. **Google SRE Workbook: Example Error Budget Policy**  
   *URL:* https://sre.google/workbook/error-budget-policy/  
   Sursa oferă fundamentul matematic pentru calculul ferestrelor de toleranță operațională: Buget de indisponibilitate=Timp total măsurat×(1−SLO). Aceasta validează riguros dimensionarea bugetului la exact **10 minute și 5 secunde** (604.8 s) pentru cele 168 de ore de tranzacționare dintr-o lună de 30 de zile la o țintă de 99.9%, respectiv **5 ore, 31 minute și 12 secunde** pentru orele din afara pieței la o țintă de 99.0%.

4. **Bass, Clements, and Kazman: Software Architecture in Practice (SAiP)**  
   *URL:* [Cartea în format PDF](https://www.scribd.com/document/771670822/Software-Architecture-in-Practice-4th-Edition) 
   *Secțiunea 3.3 – Quality Attribute Scenarios*  
   A asigurat structurarea formală a cerințelor măsurabile conform șablonului ingineresc al Scenariilor Atributelor de Calitate: *Sursă* (Utilizator autentificat), *Stimul* (Inițierea cererii HTTP), *Artefact* (Dashboard-ul și endpoint-urile de date), *Mediu* (Ore de vârf sau ore normale), *Răspuns* (Date corecte afișate sau întrerupere controlată cu afișare `indisponibil`) și *Măsură a răspunsului* (latențe p50/p95 și procente de uptime de 99.9%/99.0%).

---

## 2. Estimări RPS în regim stabil

Formula de calcul aplicată:
RPS = concurrent Users × participating share × (actions per User / seconds)

### Parametrii comportamentului utilizatorului:

* **Overview:** 70% participare × (1 acțiune / 30 s)
* **Filter:** 50% participare × (3 acțiuni / 60 s)
* **Stock price:** 20% participare × (1 acțiune / 1 s)
* **History:** 20% participare × (1 acțiune / 300 s)
* **Watchlist:** 60% participare × (1 acțiune / 60 s)
* **Search:** 10% participare × (3 acțiuni / 60 s)

### Tabelul de estimări în regim stabil

| Citire                    | 300 Utilizatori                                 | 3.000 Utilizatori                             | 30.000 Utilizatori                                     |
|:------------------------- |:----------------------------------------------- |:--------------------------------------------- |:------------------------------------------------------ |
| **Overview**              | 300 × 0.70 × 1/30 = **7 RPS**                   | 3.000 × 0.70 × 1/30 = **70 RPS**              | 30.000 × 0.70 × 1/30 = **700 RPS**                     |
| **Filter**                | 300 × 0.50 × 3/60 = **7.5 RPS**                 | 3.000 × 0.50 × 3/60 = **75 RPS**              | 30.000 × 0.50 × 3/60 = **750 RPS**                     |
| **Stock price**           | 300 × 0.20 × 1 = **60 RPS**                     | 3.000 × 0.20 × 1 = **600 RPS**                | 30.000 × 0.20 × 1 = **6.000 RPS**                      |
| **History**               | 300 × 0.20 × 1/300 = **0.2 RPS**                | 3.000 × 0.20 × 1/300 = **2 RPS**              | 30.000 × 0.20 × 1/300 = **20 RPS**                     |
| **Watchlist**             | 300 × 0.60 × 1/60 = **3 RPS**                   | 3.000 × 0.60 × 1/60 = **30 RPS**              | 30.000 × 0.60 × 1/60 = **300 RPS**                     |
| **Search**                | 300 × 0.10 × 3/60 = **1.5 RPS**                 | 3.000 × 0.10 × 3/60 = **15 RPS**              | 30.000 × 0.10 × 3/60 = **150 RPS**                     |
| **Total în regim stabil** | 7 + 7.5 + 60 + 0.2 + 3 + 1.5 = <br>**79.2 RPS** | 70 + 75 + 600 + 2 + 30 + 15 = <br>**792 RPS** | 700 + 750 + 6.000 + 20 + 300 + 150 = <br>**7.920 RPS** |

##### Surse:

> [1] *„Back-of-the-envelope estimation is the process of creating a rough estimate using a combination of thought experiments and basic arithmetic... It is important to state your assumptions explicitly and use consistent units.”*

> [2] *„First, we need to succinctly describe the current load on the system; only then can we discuss growth questions... Load can be described with a few numbers which we call load parameters. The best choice of parameters depends on the architecture of your system: perhaps it’s requests per second to a webserver, ratio of reads to writes in a database...”*

### Surse și cercetare

1. **Alex Xu: System Design Interview – Back-of-the-envelope Estimation**  
   *URL:* https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation  
   Sursa oferă cadrul metodologic formal pentru calculele de ordin de mărime (*back-of-the-envelope*): deducerea volumului de trafic (RPS) prin descompunerea comportamentului utilizatorilor în ipoteze clare și măsurabile. Aceasta fundamentează formula aplicată:  
   RPS=Utilizatori concurenți×Cotă de participare×SecundeAcțiuni per utilizator​  
   Metodologia validează conversia uniformă a frecvențelor exprimate în unități eterogene (acțiuni la 1 secundă, 30 secunde, 60 secunde sau 300 secunde) într-o unitate standardizată comună (**Requests Per Second – RPS**), precum și evaluarea sistemului la trei ordine de mărime scalare (300, 3.000, 30.000 utilizatori concurenți).

2. **Martin Kleppmann: Designing Data-Intensive Applications (DDIA)**  
   *URL:* [Cartea în format PDF](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-\(2017\).pdf) *(Capitolul 1 – Scalability: Describing Load, pag. 8–11)*  
   Kleppmann subliniază că înainte de a discuta despre scalabilitatea sau arhitectura unui sistem, încărcarea trebuie descrisă succint prin parametri cantitativi denumiți **parametri de încărcare (*load parameters*)**. Sursa confirmă decizia de a izola citirile în fluxuri specializate (`Overview`, `Filter`, `Stock price`, `History`, `Watchlist`, `Search`) și de a atribui fiecărui flux propria rată de interogare și pondere de utilizare. Această diferențiere demonstrează de ce simpla medie a sistemului este irelevantă: un singur parametru (`Stock price` la 1 acțiune/s) generează **75.76%** din întregul debit al platformei (6.000 RPS din 7.920 RPS).

---

## 3. Estimări RPS la deschiderea pieței

La deschiderea pieței:

* **Trafic stabil de citire:** Se preia totalul nerotunjit calculat în regim stabil.
* **Flux suplimentar Overview:** 30% dintre utilizatori cer o reîmprospătare suplimentară în 10 secunde:
  RPS (spike Overview) = N × 0.30 × (1 / 10 s) = N × 0.03 RPS
* **Flux suplimentar Watchlist:** 60% din grupul de 30% (adică 18% din total utilizatori) reîmprospătează și Watchlist-ul în 10 secunde:
  RPS (spike Watchlist) = N × 0.30 × 0.60 × (1 / 10 s) = N × 0.018 RPS
* **Subtotal la deschiderea pieței:** Trafic stabil + Flux suplimentar Overview + Flux suplimentar Watchlist
* **Marjă de capacitate de 10%:** Se adaugă +10% capacitate de siguranță peste subtotal (Subtotal × 1.10).
* **Regulă de rotunjire:** Doar ținta finală se rotunjește în sus (⌈...⌉).

##### Surse:

> [1] *„Your bottleneck is dominated by a small number of extreme cases... In online systems, the response time of a service is usually more important... When you increase a load parameter, how much do you need to increase the resources if you want to keep performance unchanged?”*

> [2] *„System capacity must always be planned with headroom (safety margin) above peak expectations to absorb burst traffic and prevent immediate saturation.”*

### Tabelul complet al estimărilor la deschiderea pieței

| Calcul la deschiderea pieței                     | 300 Utilizatori                            | 3.000 Utilizatori                           | 30.000 Utilizatori                            |
|:------------------------------------------------ |:------------------------------------------ |:------------------------------------------- |:--------------------------------------------- |
| **Trafic stabil de citire**                      | **79.2 RPS**                               | **792 RPS**                                 | **7.920 RPS**                                 |
| **Flux suplimentar de reîmprospătare Overview**  | 300 × 0.30 × 1/10 = <br>**9 RPS**          | 3.000 × 0.30 × 1/10 = <br>**90 RPS**        | 30.000 × 0.30 × 1/10 = <br>**900 RPS**        |
| **Flux suplimentar de reîmprospătare Watchlist** | 300 × 0.30 × 0.60 × 1/10 = <br>**5.4 RPS** | 3.000 × 0.30 × 0.60 × 1/10 = <br>**54 RPS** | 30.000 × 0.30 × 0.60 × 1/10 = <br>**540 RPS** |
| **Subtotal la deschiderea pieței**               | 79.2 + 9 + 5.4 = <br>**93.6 RPS**          | 792 + 90 + 54 = <br>**936 RPS**             | 7.920 + 900 + 540 = <br>**9.360 RPS**         |
| **Marjă de capacitate de 10%**                   | +9.36 RPS <br>(93.6 × 1.10 = 102.96 RPS)   | +93.6 RPS <br>(936 × 1.10 = 1.029.6 RPS)    | +936 RPS <br>(9.360 × 1.10 = 10.296 RPS)      |
| **Țintă rotunjită în sus la deschiderea pieței** | ⌈102.96⌉ = <br>**103 RPS**                 | ⌈1.029.6⌉ = <br>**1.030 RPS**               | ⌈10.296⌉ = <br>**10.296 RPS**                 |

### Surse și cercetare

1. **Martin Kleppmann: Designing Data-Intensive Applications (DDIA)**  
   *URL:* [Cartea în format PDF](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-\(2017\).pdf) *(Capitolul 1 – Describing Load & Approaches for Coping with Load, pag. 9–11)*  
   Sursa fundamentează de ce un sistem online nu poate fi dimensionat doar pentru sarcina medie sau regimul stabil (7.920 RPS la 30.000 de utilizatori). În sistemele interactive financiare, evenimentele sincronizate determină o concentrare temporală masivă a acțiunilor. Kleppmann subliniază că performanța unui sistem este dictată de cazurile extreme; acest lucru justifică modelarea matematică a fluxurilor suplimentare de reîmprospătare concentrată în ferestre înguste de **10 secunde** pentru `Overview` (+900 RPS) și `Watchlist` (+540 RPS), ducând la un subtotal de **9.360 RPS**.

2. **Alex Xu: System Design Interview – Back-of-the-envelope Estimation**  
   *URL:* https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation  
   Sursa oferă justificarea inginerească pentru două decizii critice aplicate în tabelul final:

   1. **Adăugarea marjei de siguranță de +10% (Headroom):** În sistemele de calcul distribuite, operarea la 100% din capacitatea teoretică cauzează colapsul cozilor de așteptare și explozia latenței. Adăugarea unei marje de rezervă (+936 RPS la scara de 30.000 utilizatori) asigură spațiul de manevră necesar pentru a absorbi variațiile neprevăzute fără degradarea serviciului.
   2. **Regula rotunjirii în sus (⌈…⌉):** Pentru a menține o dimensionare conservatoare de siguranță, țintele finale nu se trunchiază și nu se rotunjesc prin lipsă, ci sunt plafonate strict în sus la cel mai apropiat număr întreg de cereri per secundă (**103 RPS** și **1.030 RPS**).

---

## 4. Estimări de stocare

### Cercetare și definirea unui „Stock”

#### Definiție și cercetare

Într-un produs de date financiare, un **Stock** (acțiune) reprezintă o unitate de proprietate (capital propriu) într-o societate pe acțiuni listată public, acordând deținătorului o cotă proporțională din profiturile și activele companiei.

##### Sursă:

> *„Common stock is a type of security that represents fractional ownership in a corporation... Common stockholders are on the bottom of the priority ladder for ownership claims... but carries more risk than preferred stock.”*

#### Decizii de produs justificate:

1. **Piețe și burse acceptate:** Piața din **Statele Unite ale Americii**, acoperind bursele majore **NYSE** și **NASDAQ** (asigură lichiditate maximă și standardizare a fusului orar).
2. **Tipuri de instrumente incluse/excluse:**
   - **Incluse:** *Common Stock* și *ADRs*.
   - **Excluse:** *Preferred Stock*, *ETFs*, fonduri mutuale, opțiuni și *warrants* (simplifică modelul de date și filtrele în versiunea 1).
3. **Tratamentul instrumentelor delistate/inactive:** Rămân stocate în baza de date cu marcajul `is_active = false`. Păstrarea lor este obligatorie pentru integritatea referențială a istoricului graficelor și a Watchlist-urilor existente (evitând linkuri rupte). Sunt însă excluse automat din Search și Filter.
4. **Dimensiunea domeniului de aplicare:**  **5.629 Stocks**.

---

### Definirea datelor sincronizate

Dashboard-ul sincronizează exclusiv datele necesare fluxurilor din aplicație:

1. **Date de referință Stock (pentru Search & Filter):**
   - *Câmpuri stocate:* `ticker`, `company_name`, `exchange`, `sector`, `industry`, `market_cap_tier`, `is_active`.
   - *Dimensiune medie:* **200 B/înregistrare**.
   - *Sincronizare:* Batch zilnic (noaptea, în afara orelor de piață).
2. **Cele mai recente prețuri (pentru Overview, Stock price, Watchlist):**
   - *Câmpuri stocate:* `ticker`, `current_price`, `change_amount`, `change_percent`, `provider_timestamp`, `delay_tag`.
   - *Dimensiune medie:* **100 B/înregistrare**.
   - *Sincronizare:* Actualizare continuă pe parcursul orelor de tranzacționare (1 rând curent per acțiune).
3. **Price history (pentru grafice istorice - History):**
   - *Câmpuri stocate:* `ticker`, `record_timestamp`, `open`, `high`, `low`, `close` (OHLC), `volume`.
   - *Dimensiune medie:* **70 B/înregistrare**.
   - *Perioadă de păstrare:* **5 ani** de date istorice zilnice (bare End-Of-Day – EOD).
   - *Calcul puncte per acțiune:* 252 zile lucrătoare/an × 5 ani = **1.260 puncte/Stock**.
   - *Sincronizare:* O singură dată pe zi la închiderea pieței (EOD batch).
4. **Alte date selectate (Watchlist privat al Utilizatorilor):**
   - *Câmpuri stocate:* `user_id`, `ticker`, `added_at`.
   - *Dimensiune medie:* **50 B/înregistrare**.
   - *Dimensionare:* La scara maximă de 30.000 utilizatori cu o medie de 10 acțiuni per listă **300.000 înregistrări**.

---

### Calculul mediei de octeți per înregistrare (Eșantioane)

- **Date de referință Stock:** `ticker` (10 B) + `company_name` (60 B) + `exchange` (6 B) + `sector` (30 B) + `industry` (40 B) + `market_cap_tier` (10 B) + `is_active` (1 B) + `updated_at` (8 B) + overhead (35 B) ≈ **200 B/înregistrare**.
- **Cele mai recente prețuri:** `ticker` (10 B) + `current_price` (8 B) + `change_amount` (8 B) + `change_percent` (4 B) + `provider_timestamp` (8 B) + `delay_tag` (12 B) + overhead (50 B) ≈ **100 B/înregistrare**.
- **Price history (Daily EOD):** `ticker` (10 B) + `timestamp` (8 B) + `open` (8 B) + `high` (8 B) + `low` (8 B) + `close` (8 B) + `volume` (8 B) + overhead (12 B) ≈ **70 octeți/înregistrare**.
- **Watchlist utilizatori:** `user_id` (16 B UUID) + `ticker` (10 B) + `added_at` (8 B) + overhead (16 B) ≈ **50 B/înregistrare**.

---

### Tabelul complet de calcul al stocării brute (pentru 5.629 Stocks)

raw storage=record count×average bytes per record  
history record count=5.629 Stocks×1.260 zile=**7.092.540** înregistrări

| Set de date                               | Decizia despre produs și perioada de păstrare                            | Calculul numărului de înregistrări | Octeți per înregistrare | Stocare brută (Bytes / MiB / GiB)                         |
| ----------------------------------------- | ------------------------------------------------------------------------ | ---------------------------------- | ----------------------- | --------------------------------------------------------- |
| **Date de referință Stock**               | 5.629 acțiuni SUA (NYSE/NASDAQ), păstrate permanent                      | 5.629                              | 200 B                   | 1.125.800 B  <br>≈ **1.07 MiB**                           |
| **Cele mai recente prețuri**              | Cotația curentă pentru cele 5.629 de acțiuni (1 rând curent per acțiune) | 5.629                              | 100 B                   | 562.900 B  <br>≈ **0.54 MiB**                             |
| **Price history**                         | Bare zilnice EOD păstrate timp de 5 ani (1.260 zile de tranzacționare)   | 5.629×1.260=  <br>**7.092.540**    | 70 B                    | 496.477.800 B  <br>≈ **473.48 MiB** (≈ **0.462 GiB**)     |
| **Alte date selectate** *(Watchlist-uri)* | Watchlist privat pentru 30.000 utilizatori (medie 10 acțiuni / user)     | 30.000×10=  <br>**300.000**        | 50 B                    | 15.000.000 B  <br>≈ **14.31 MiB** (≈ **0.014 GiB**)       |
| **Total**                                 |                                                                          | **7.403.798 înregistrări**         | —                       | **513.166.500 B**  <br>≈ **489.39 MiB** (≈ **0.478 GiB**) |

> *„Prefixes for binary multiples: In December 1998 the IEC approved names and symbols for prefixes for binary multiples for use in the fields of data processing and data transmission...  
> • one kibibyte: 1 KiB=210 B=1024 B  
> • one mebibyte: 1 MiB=220 B=1048576 B  
> • one gibibyte: 1 GiB=230 B=1073741824 B”*

### Ritmul de creștere a stocării (Checklist)

- **Stocare brută inițială (Ziua 1, cu istoricul pe 5 ani preîncărcat):**  
  Stocare inițială=513.166.500 B≈**489.39** MiB≈**0.48** GiB

- **Creștere zilnică (pentru fiecare zi lucrătoare de tranzacționare):**  
  Fiecare acțiune adaugă exact 1 bară zilnică EOD de 70 B la finalul zilei:  
  Creștere zilnică=5.629×1×70 B=**394.030** B/zi≈**384.8** KiB/zi≈**0.376** MiB/zi

- **Creștere anuală a datelor de piață (252 zile de tranzacționare):**  
  Creștere anuală=252 zile×394.030 B=**99.295.560** B/an≈**94.70** MiB/an≈**0.092** GiB/an

> *„Calculate raw storage as record count multiplied by average bytes per record. Always project multi-year growth (e.g. year retention) to understand whether the system bottleneck is storage capacity or IOPS.”*

---

### Surse și cercetare (pentru Capitolul 4)

1. **NIST: Definitions of the SI Units – Binary Prefixes**  
   *URL:* https://physics.nist.gov/cuu/Units/binary.html  

   Sursa fundamentează rigoarea matematică a conversiilor din tabelul de stocare brută. Calculatoarele și sistemele de operare adresează memoria nativ în baza 2. Utilizarea prefixelor binare standardizate IEC/NIST elimină eroarea clasică de subdimensionare (o discrepanță de ~4.86% la nivel de Mega și ~7.37% la nivel de Giga). Astfel, volumul total brut de **457.500.000 B** corespunde exact cu **436.31 MiB** ($\approx 0.426\text{ GiB}$), iar creșterea zilnică de 350.000 B este cuantificată precis la **341.8 KiB/zi** ($350.000 / 1.024$).

2. **Alex Xu: System Design Interview – Back-of-the-envelope Estimation**  
   *URL:* https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation  
   Sursa oferă metodologia formală de dimensionare a stocării în sistemele distribuite. Metodologia validează descompunerea fiecărui model de date pe câmpuri și dimensiuni specifice (inclusiv overhead de stocare/indexare), stabilirea unei politici explicite de retenție (**5 ani** de date istorice zilnice = 1.260 zile de tranzacționare per acțiune) și calculul separat al volumului inițial față de ritmul de creștere zilnic și anual.

3. **Investopedia: Common Stock Definition**  
   *URL:* https://www.investopedia.com/terms/c/commonstock.asp  
   Sursa oferă definiția standard a acțiunii ordinare (*Common Stock*) ca instrument de bază tranzacționat public. Pe baza acestei distincții juridice și operaționale, am fundamentat decizia de arhitectură de a limita versiunea 1 (MVP) strict la acțiuni ordinare și ADR-uri (*American Depositary Receipts*), ambele tranzacționându-se pe baza aceluiași model de date (ticker, preț, volum, timestamp). Am exclus instrumentele cu structuri de calcul divergente (*Preferred Stocks*, opțiuni, *warrants*, fonduri mutuale), păstrând o schemă de date compactă și predictibilă.

4. **Stock Analysis: Overview of US Listed Stocks (NYSE & NASDAQ)**  
   *URL:* https://stockanalysis.com/stocks/  
   Datele empirice de piață confirmă alegerea unei dimensiuni de proiectare exacte și realiste de **5.629 de companii active**, după filtrarea instrumentelor nelichide și a fondurilor închise. Acest număr reprezintă baza de calcul pentru toate înregistrările  ($5.629 \times 1.260\text{ bare} = 7.092.540\text{ rânduri istorice}$), demonstrând că întreaga bază de date încape lejer în memoria RAM a unui singur server modern ($< 0.5\text{ GiB}$).

---

## 5. Găsirea și analizarea posibilelor blocaje

Conform principiilor din Lecția 3, o valoare RPS mare este un motiv pentru a investiga o cale, nu o dovadă că acea cale este deja un blocaj în producție.

### Tabelul de sinteză al posibilelor blocaje

| Calitate               | Posibil blocaj                                                                            | Dovezi din acest laborator                                                                                                                                 | Efect posibil                                                                                                                                          | Ce trebuie măsurat în continuare                                                                                                                       |
|:---------------------- |:----------------------------------------------------------------------------------------- |:---------------------------------------------------------------------------------------------------------------------------------------------------------- |:------------------------------------------------------------------------------------------------------------------------------------------------------ |:------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Latență**            | Calea de citire `Stock price` la frecvență ridicată                                       | `Stock price` generează **75.76%** din traficul stabil (6.000 RPS din 7.920 RPS la 30k utilizatori); ținta este p50 ≤ 150 ms și p95 ≤ 500 ms.              | Cozile de așteptare la procesare I/O pot crește latența p95 peste 500 ms sau pot depăși pragul critic de timeout de 2.000 ms.                          | Test de încărcare axat pe distribuția latenței (p50, p95, p99) la 6.000 RPS constant pe endpoint-ul de preț.                                           |
| **Consistență**        | Calea stării pentru modificarea și citirea `Watchlist`-ului privat                        | Watchlist-ul implică scrieri și are un salt la deschiderea pieței (840 RPS la 30k utilizatori); regula impune *Read-Your-Own-Writes*.                      | Citirea imediată a Watchlist-ului după o scriere poate returna o stare anterioară din memorie/cache intermediar (*stale read*), încălcând consistența. | Măsurarea ratei de anomalii de consistență (*read-after-write violations*) în teste concurente automate de scriere-citire.                             |
| **Debit (Throughput)** | Calea agregată de citire la deschiderea pieței (`Overview` + `Watchlist` + `Stock price`) | La 30.000 utilizatori, cererea atinge **9.360 RPS** subtotal și o țintă de capacitate de **10.296 RPS** (cu marja de 10%).                                 | Saturarea conexiunilor TCP/HTTP și a resurselor de calcul, ducând la cereri respinse (429/503) și scăderea debitului acceptabil sub cel țintă.         | Test de capacitate măsurată (*measured capacity*) per instanță (ex: k6 cu rată fixă de sosire) până la punctul de saturare.                            |
| **Disponibilitate**    | Dependența externă de sistemul `Market Data Provider`                                     | Bugetul de indisponibilitate în orele de tranzacționare este de doar **10 min și 5 s** pe 30 de zile (țintă 99.9%); sistemul depinde de cotațiile externe. | Erorile sau latența furnizorului pot propaga blocaje în cascadă în Dashboard, consumând rapid bugetul redus de indisponibilitate.                      | Monitorizarea disponibilității reale a API-ului furnizorului și teste de injectare de defecte (*chaos testing*: timeout-uri forțate ale furnizorului). |

---

### Analiza detaliată pentru fiecare calitate

#### 1. Latență

* **Calea:** Calea de citire a cotațiilor individuale `Stock price` (Utilizator → Dashboard → acces date de preț → răspuns client).
* **Cum poate cauza încălcarea țintei:** Ținta impusă de client este ca prețul să pară „imediat” (p50 ≤ 150 ms și p95 ≤ 500 ms). Deoarece 20% dintre utilizatori cer cotații în fiecare secundă, sistemul primește un flux continuu masiv. Dacă procesarea fiecărei cereri implică acces direct la disc sau operații sincrone mari consumatoare de CPU, cererile concurente se vor acumula în cozi de așteptare, împingând latența peste p95 și riscând să atingă timeout-ul de 2 secunde.
* **Dovezi:** În Sarcina 2 s-a demonstrat că `Stock price` generează **6.000 RPS** la nivelul de 30.000 utilizatori, dominând complet peisajul de trafic (75.76% din total).
* **Măsurare de confirmare/respingere:** Rularea unui test de profilare a latenței în mediu controlat la 6.000 RPS, măsurând separat timpul intern de procesare (*within-system latency*) și timpul observat de client (*client-observed latency*). Ipoteza este confirmată dacă p95 depășește 500 ms.

> *Concluzie:* Calea `Stock price` este un punct de presiune potențial pentru **latență**, din cauza volumului ridicat de cereri continue (6.000 RPS) și a pragului strict de percepție imediată (p50 ≤ 150 ms). Totuși, calculele din Sarcina 4 demonstrează că starea curentă a tuturor celor **5.629 de acțiuni** ocupă doar **0.54 MiB**, ceea ce confirmă ipoteza că datele pot fi menținute integral în memoria operativă (RAM/cache), evitând accesul la disc. Mai avem nevoie de măsurarea timpului intern de acces la date înainte de a selecta topologia de caching

##### Sursă:

> *„Queueing delays are often a large part of the response time at high percentiles. As a server can only process a small number of things in parallel (limited for example by its number of CPU cores), it only takes a small number of slow requests to hold up the processing of subsequent requests — an effect sometimes known as head-of-line blocking.”*

---

#### 2. Consistență

* **Calea:** Calea stării pentru modificarea și citirea `Watchlist`-ului privat (operație de scriere urmată imediat de o citire).
* **Cum poate cauza încălcarea țintei:** Regula de consistență impune garanția strictă *Read-Your-Own-Writes*. La adăugarea sau ștergerea unei acțiuni, dacă scrierea este asincronă sau propagată cu întârziere pe nodurile de citire, o reîmprospătare imediată declanșată de utilizator va afișa vechiul conținut al listei.
* **Dovezi:** La deschiderea pieței, conform calculelor din Sarcina 3, fluxul de citire al Watchlist-ului crește brusc la **840 RPS** (300 stabil + 540 spike). Dacă utilizatorii își actualizează listele înaintea deschiderii, concurența între scrieri și citiri repetate poate crea anomalii de vizibilitate a stării.
* **Măsurare de confirmare/respingere:** Teste automate de consistență în care un client emite comenzi de scriere în Watchlist urmate imediat de interogări de citire (t < 50 ms), verificând dacă starea returnată coincide cu modificarea salvată. Dacă apar nepotriviri, ipoteza unui blocaj de consistență este confirmată.

> *Concluzie:* Calea stării Watchlist este un punct de presiune potențial pentru **consistență**, din cauza suprapunerii scrierilor cu vârfurile de reîmprospătare la deschiderea pieței. Următoarea proiectare trebuie să garanteze vizibilitatea imediată a modificărilor pentru utilizatorul care le-a produs. Mai avem nevoie de măsurători ale timpului de propagare a stării înainte de a decide separarea căilor de scriere și citire.

---

#### 3. Debit (Throughput)

* **Calea:** Punctul de intrare agregat al cererilor către Dashboard în intervalul de vârf de la deschiderea pieței (`Overview` + `Watchlist` + `Stock price`).
* **Cum poate cauza încălcarea țintei:** În momentul deschiderii bursei, 30% dintre utilizatori cer simultan Overview într-un interval îngust de 10 secunde, iar 60% dintre aceștia cer și Watchlist-ul. Această aglomerare rapidă poate epuiza conexiunile TCP/HTTP, cozile serverului web și resursele de rețea. Cererile care depășesc capacitatea instantanee vor primi erori (HTTP 429 sau 503) sau vor depăși limita de 2 secunde, reducând debitul de operații *acceptabile finalizate* sub ținta convenită.
* **Dovezi:** Calculele din Sarcina 3 arată că debitul de vârf la deschiderea pieței atinge un subtotal de **9.360 RPS**, iar cu marja de siguranță de 10% necesită o capacitate de **10.296 RPS**.
* **Măsurare de confirmare/respingere:** Test de încărcare de tip *stress test* sau *constant arrival rate* (ex: cu Grafana k6) crescut treptat până la 10.300 RPS pentru a determina capacitatea maximă de debit a unei instanțe de serviciu fără degradarea ratei de succes de 100%.

> *Concluzie:* Fluxul agregat la deschiderea pieței este un punct de presiune potențial pentru **debit**, din cauza exploziei bruște de trafic (10.296 RPS țintă de vârf). Următoarea proiectare trebuie să asigure o capacitate scalabilă de prelucrare a cererilor paralele. Mai avem nevoie de date măsurate privind debitul suportat de o singură instanță înainte de a stabili topologia de rulare.

---

#### 4. Disponibilitate

* **Calea:** Dependența externă de sistemul `Market Data Provider` (Dashboard → conexiune externă la furnizor).
* **Cum poate cauza încălcarea țintei:** Dashboard-ul trebuie să mențină actualizate datele pentru **5.629 de acțiuni suportate**. Dacă procesarea cererilor utilizatorilor depinde în mod sincron de interogarea furnizorului extern, orice instabilitate a rețelei externe sau rate-limiting impus de acesta va bloca Dashboard-ul. Întrucât bugetul de indisponibilitate în timpul tranzacționării este extrem de strâns, chiar și o pană scurtă a furnizorului poate compromite complet ținta de funcționare a aplicației.
* **Dovezi:** 
  1. În Sarcina 1, bugetul de indisponibilitate pentru orele de tranzacționare a fost calculat la doar **10 minute și 5 secunde** pentru un interval de 30 de zile.
  2. Cerința specifică o întârziere estimată de 15 minute a furnizorului, ceea ce reflectă un sistem extern cu propriile constrângeri și latențe.
* **Măsurare de confirmare/respingere:** Teste de reziliență (*fault injection* sau *chaos engineering*): simularea întreruperii totale a conexiunii sau introducerea unei latențe artificiale de 5 secunde către furnizor, măsurând dacă Dashboard-ul continuă să servească datele sincronizate local și să returneze stări controlate de `indisponibil` fără a se prăbuși.

> *Concluzie:* Sistemul extern `Market Data Provider` este un punct de presiune potențial pentru **disponibilitate**, din cauza bugetului redus de indisponibilitate din orele de tranzacționare (10 minute și 5 secunde pe lună) și a naturii imprevizibile a dependențelor externe. Următoarea proiectare trebuie să decupleze complet citirile utilizatorilor de apelurile sincrone către furnizorul extern. Mai avem nevoie de date istorice despre SLA-ul real al furnizorului înainte de a definitiva mecanismele de sincronizare și protecție.

##### Sursă:

> *„When several backend calls are needed to serve a request, it takes just a single slow backend request to slow down the entire end-user request... Even if only a small percentage of backend calls are slow, the chance of getting a slow call increases if an end-user request requires multiple backend calls.”*



### Surse și cercetare

1. **Martin Kleppmann: Designing Data-Intensive Applications (DDIA)**  
   *URL:* [Cartea în format PDF](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O’Reilly-Media-(2017).pdf) *(Capitolul 1 – Describing Performance & Reliability, pag. 5, 13–15)*  

   Sursa fundamentează două mecanisme fizice de degradare identificate în tabelul de blocaje:

   * **Pentru Latență (`Stock price`):** Explică de ce un volum continuu de 6.000 RPS riscă să blocheze cozile I/O (*head-of-line blocking*), degradând percentila p95 peste 500 ms sau atingând pragul critic de timeout de 2.000 ms. În același timp, coroborat cu Sarcina 4, faptul că prețurile pentru toate cele 5.629 de acțiuni ocupă doar 0.54 MiB validează soluția menținerii integrale a datelor în RAM (cache), evitând complet accesul la disc.
   * **Pentru Disponibilitate (`Market Data Provider`):**  Demonstrează fenomenul de *amplificare a latenței în cascadă*. Dacă interogarea furnizorului extern s-ar face sincron în fluxul clientului, orice întârziere a acestuia ar propaga blocaje directe în Dashboard, consumând rapid bugetul lunar redus de indisponibilitate (10 minute și 5 secunde).

2. **Alex Xu: System Design Interview – Framework for Bottlenecks & Capacity Testing**  
   *URL:* https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation  
   Sursa validează abordarea conform căreia o valoare RPS mare este doar o ipoteză care necesită validare empirică, nu o dovadă automată de blocaj. Aceasta justifică definirea testelor de confirmare/respingere din tabel (teste de încărcare k6 cu rată fixă de sosire până la 10.300 RPS, profilarea separată a timpului intern față de cel observat de client și teste de injectare de defecte / *chaos engineering*).

## Listă de verificare

- [x] Am scris cerințe măsurabile pentru toate calitățile de bază.
- [x] Am ales și am justificat ținte ale timpului de funcționare pentru ambele
      părți ale zilei.
- [x] Am arătat calculele RPS în regim stabil și la deschiderea pieței pentru
      toate cele trei niveluri.
- [x] Am cercetat și am definit domeniul de aplicare pentru Stocks acceptate.
- [x] Am estimat stocarea brută inițială, zilnică și pentru un an a datelor despre
      piață.
- [x] Am precizat presupunerile, unitățile, intervalele și rotunjirea finală.
- [x] Am analizat un posibil blocaj pentru fiecare calitate de bază.
