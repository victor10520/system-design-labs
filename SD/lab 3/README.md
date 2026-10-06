# Laboratorul 3: Desenează limita Dashboard-ului (Baseline Architecture)

## 1. Diagrama System Context (Nivelul 1 C4)

### Diagrama Mermaid System Context

```mermaid
flowchart LR
    user["Utilizator autentificat"]
    pid["Personal Investment Dashboard"]
    provider["Market Data Provider"]

    user -->|"Răsfoiește acțiuni, citește prețuri, gestionează Watchlist"| pid
    pid -->|"Solicită date de piață"| provider
    provider -->|"Furnizează date de piață"| pid
```

### Semnificația elementelor și a relațiilor

* **Utilizator autentificat:** Actorul uman (investitor sau trader) care interacționează direct cu aplicația pentru a consulta cotațiile, graficele de istoric, a căuta instrumente financiare și a administra propria listă de urmărire (*Watchlist*).
* **Personal Investment Dashboard:** Sistemul proiectat, privit ca o cutie neagră (*black box*). Toate detaliile interne de execuție, procesele și bazele de date sunt ascunse la acest nivel.
* **Market Data Provider:** Sistem extern independent care furnizează cotații de piață și date istorice. Are propriile politici de disponibilitate, latență și limitare a numărului de cereri.
* **Relația PID $\rightarrow$ Furnizor:** Sistemul preia periodic datele necesare pentru catalogul de 5.629 de acțiuni definit în Laboratorul 2.
* **Relația Furnizor $\rightarrow$ PID:** Răspunsurile furnizorului conțin cotații însoțite de momentul de observare al pieței (*provider timestamp*).

---

## 2. Diagrama Container (Nivelul 2 C4)

### Diagrama Mermaid Container

```mermaid
flowchart LR
    user["Utilizator autentificat"]
    provider["Market Data Provider"]

    subgraph pid["Personal Investment Dashboard"]
        web["Web App"]
        app["Dashboard Application"]
        db[("Database")]

        web -->|"Trimite cereri"| app
        app -->|"Citește și scrie date"| db
    end

    user -->|"Folosește aplicația"| web
    app -->|"Preia date de piață"| provider
```

### Responsabilitățile containerelor

Am ales cel mai mic set viabil de containere, conform principiului *Monolith First* (recomandat de Martin Fowler și Lecția 4), pentru a evita complexitatea și latența rețelei introduse de microservicii premature:

| Container | Rol și responsabilitate |
| :--- | :--- |
| **Web App** | Aplicație client (Single-Page Application) care rulează în browserul utilizatorului. Randează interfața grafică, preia acțiunile utilizatorului și afișează stările datelor (`normal`, `întârziat`, `indisponibil`). |
| **Dashboard Application** | Aplicația principală de server (monolit modular). Primește solicitările de la Web App, verifică identitatea și permisiunile, procesează logica de business, citește datele din baza de date și rulează în fundal sincronizarea cu furnizorul extern. |
| **Database** | Stocarea de date persistentă a sistemului. Păstrează conturile de utilizatori, listele private Watchlist, datele de referință ale acțiunilor, cotațiile curente și istoricul pe 5 ani. Asigură tranzacții sigure pentru scrieri. |

### Decizii cheie de arhitectură:
1. **Decuplarea citirilor de furnizor:** Nicio acțiune a utilizatorului din `Web App` nu apelează direct `Market Data Provider`. Toate cererile trec prin `Dashboard Application`, care servește datele din stocarea locală.
2. **Păstrarea secretelor pe server:** Cheile de acces către API-ul furnizorului extern sunt stocate exclusiv pe server în `Dashboard Application`, nefiind expuse niciodată în codul sau browserul clientului.

---

## 3. Diagrama Component (Nivelul 3 C4)

Deschidem exclusiv containerul **Dashboard Application** pentru a evidenția componentele sale interne și responsabilitățile acestora. Containerele adiacente (`Web App`, `Database`) și sistemul extern (`Market Data Provider`) rămân în afara limitei containerului.

### Diagrama Mermaid Component

```mermaid
flowchart LR
    web["Web App"]
    db[("Database")]
    provider["Market Data Provider"]

    subgraph app["Dashboard Application"]
        auth["AuthService"]
        catalog["StockCatalogService"]
        watchlist["WatchlistService"]
        marketData["MarketDataService"]
        scheduler["SyncScheduler"]
        pClient["ProviderClient"]
        validator["RecordValidator"]
        writer["MarketDataWriter"]

        scheduler -->|"Inițiază sincronizare"| pClient
        pClient -->|"Trimite date brute"| validator
        validator -->|"Trimite date validate"| writer
        scheduler -->|"Înregistrează progres"| writer
    end

    web -->|"Autentificare"| auth
    web -->|"Căutare și filtrare"| catalog
    web -->|"Gestiune Watchlist"| watchlist
    web -->|"Citire prețuri și istoric"| marketData

    auth -->|"Verifică utilizatori"| db
    catalog -->|"Citește companii"| db
    watchlist -->|"Citește și scrie liste"| db
    marketData -->|"Citește prețuri și istoric"| db
    writer -->|"Salvează date și progres"| db

    pClient -->|"Cere date de piață"| provider
```

### Responsabilitățile componentelor interne

| Componentă | Responsabilitate |
| :--- | :--- |
| **AuthService** | Verifică identitatea utilizatorului la fiecare solicitare protejată. |
| **StockCatalogService** | Asigură căutarea și filtrarea companiilor din catalog (cele 5.629 de acțiuni). |
| **WatchlistService** | Gestionează listele private de urmărire; verifică permisiunea de acces (proprietatea listei) și asigură consistența datelor private. |
| **MarketDataService** | Răspunde la cererile de preț și istoric ale clienților; evaluează prospețimea cotațiilor (stare normală, întârziată sau indisponibilă). |
| **SyncScheduler** | Coordonează sincronizarea periodică în fundal; deduplică operațiile și gestionează pauzele la erori. |
| **ProviderClient** | Gestionează comunicarea externă securizată cu furnizorul; deține credențialele de acces pe server și tratează pauzele de rețea sau limitările de cereri. |
| **RecordValidator** | Verifică integritatea datelor primite de la furnizor (câmpuri obligatorii, valori pozitive, timestamp-uri valide) și respinge înregistrările corupte. |
| **MarketDataWriter** | Salvează atomic în baza de date loturile de date validate împreună cu marcajul de progres al sincronizării. |

---

## 4. Diagrame de secvență

### A. Sincronizează datele de piață

#### Ipoteze inițiale:
1. Sincronizarea este pornită periodic de un timer intern în `SyncScheduler` (în fundal).
2. Credențialele de acces către furnizor sunt păstrate strict pe server în `ProviderClient`.
3. Dacă sincronizarea eșuează (limitare de cereri, timeout sau date invalide), **datele anterior salvate în baza de date rămân neschimbate** și continuă să fie servite.
4. Datele noi devin disponibile pentru citirile ulterioare numai după confirmarea salvării în baza de date.

```mermaid
sequenceDiagram
    autonumber
    participant Scheduler as SyncScheduler
    participant Client as ProviderClient
    participant Provider as Market Data Provider
    participant Validator as RecordValidator
    participant Writer as MarketDataWriter
    participant DB as Database

    Note over Scheduler: Declanșator periodic de fundal
    Scheduler->>Client: Solicită sincronizare date
    Client->>Provider: Cere date de piață
    
    alt Furnizorul limitează cererile (Rate limit)
        Provider-->>Client: Limită de cereri atinsă
        Client-->>Scheduler: Semnalează limitarea
        Scheduler->>Scheduler: Amână următoarea încercare
        Note over Scheduler,DB: Sincronizarea se oprește, datele vechi rămân în DB
    else Timp de așteptare expirat (Timeout)
        Client-->>Scheduler: Răspuns indisponibil
        Scheduler->>Scheduler: Programează reîncercare
        Note over Scheduler,DB: Sincronizarea se oprește, datele vechi rămân în DB
    else Furnizorul returnează date
        Provider-->>Client: Răspuns cu date de piață
        Client->>Validator: Trimite date pentru verificare
        alt Date invalide
            Validator-->>Scheduler: Respinge datele
            Note over Scheduler,DB: Datele sunt ignorate, stocarea nu este modificată
        else Date valide
            Validator->>Writer: Trimite date acceptate
            Writer->>DB: Salvare atomică date și progres
            DB-->>Writer: Salvare confirmată
            Writer-->>Scheduler: Sincronizare reușită
            Note over DB: Noile date devin disponibile pentru citiri
        end
    end
```

---

### B. Citește prețul unui Stock

#### Ipoteze inițiale:
1. Utilizatorul cere prețul unei acțiuni din interfață.
2. **Citirea utilizatorului NU declanșează o cerere nouă către furnizor.** Datele se citesc strict din stocarea locală.
3. Reguli de afișare bazate pe vechimea datelor (din Lab 2):
   * $\le$ 16 minute: preț afișat normal.
   * Între 16 și 30 minute: preț afișat cu eticheta `întârziat`.
   * Peste 30 minute: prețul este considerat expirat și se afișează `indisponibil` (niciodată `0`).
4. În caz de defectare a stocării sau simbol inexistent, utilizatorul vede explicit starea `indisponibil`.

```mermaid
sequenceDiagram
    autonumber
    actor User as Utilizator
    participant Web as Web App
    participant Auth as AuthService
    participant Market as MarketDataService
    participant DB as Database

    User->>Web: Solicită preț acțiune
    Web->>Auth: Verifică identitate utilizator
    
    alt Identitate invalidă
        Auth-->>Web: Utilizator neautentificat
        Web-->>User: Solicită autentificare
    else Identitate validă
        Auth->>Market: Transmite cerere preț
        Market->>Market: Verificare date intrare
        alt Simbol acțiune invalid
            Market-->>Web: Cerere invalidă
            Web-->>User: Afișează eroare intrare
        else Simbol valid
            Note over Market,DB: Citire din stocare locală (fără apel la furnizor)
            Market->>DB: Citește preț și moment furnizor
            
            alt Defectare stocare
                DB-->>Market: Eroare stocare
                Market-->>Web: Răspuns indisponibil
                Web-->>User: Afișează indisponibil
            else Preț lipsă
                DB-->>Market: Preț negăsit
                Market-->>Web: Răspuns indisponibil
                Web-->>User: Afișează indisponibil
            else Preț găsit
                DB-->>Market: Preț și moment furnizor
                alt Date mai vechi de 30 minute
                    Market-->>Web: Răspuns indisponibil
                    Web-->>User: Afișează indisponibil
                else Date între 16 și 30 minute
                    Market-->>Web: Preț cu marcaj întârziat
                    Web-->>User: Afișează preț întârziat
                else Date recente (sub 16 minute)
                    Market-->>Web: Preț curent
                    Web-->>User: Afișează preț curent
                end
            end
        end
    end
```

---

### C. Modifică un Watchlist privat

#### Ipoteze inițiale:
1. Utilizatorul solicită adăugarea sau ștergerea unei acțiuni din Watchlist.
2. Se verifică obligatoriu identitatea, datele de intrare și permisiunea (utilizatorul poate modifica doar propria listă).
3. **Punctul de raportare a succesului:** Succesul este confirmat utilizatorului **doar după ce salvarea a reușit în baza de date**.
4. **Garanția Read-Your-Own-Writes:** Odată confirmată salvarea, următoarea citire a Watchlist-ului de către același utilizator reflectă garantat modificarea efectuată.

```mermaid
sequenceDiagram
    autonumber
    actor User as Utilizator
    participant Web as Web App
    participant Auth as AuthService
    participant Watch as WatchlistService
    participant DB as Database

    User->>Web: Solicită modificare Watchlist
    Web->>Auth: Verifică identitate utilizator
    
    alt Identitate invalidă
        Auth-->>Web: Utilizator neautentificat
        Web-->>User: Solicită autentificare
    else Identitate validă
        Auth->>Watch: Transmite solicitare modificare
        Watch->>Watch: Verifică date intrare și permisiuni
        alt Acțiune nepermisă
            Watch-->>Web: Acces interzis
            Web-->>User: Afișează eroare permisiune
        else Acțiune permisă
            Watch->>DB: Salvează modificare Watchlist
            
            alt Defectare stocare
                DB-->>Watch: Eroare stocare
                Watch-->>Web: Modificare eșuată
                Web-->>User: Afișează eroare salvare
            else Salvare confirmată
                DB-->>Watch: Modificare confirmată
                Note over Watch: Punct de raportare a succesului
                Watch-->>Web: Succes modificare
                Web-->>User: Afișează confirmare
                Note over User,DB: Următoarea citire a acestui utilizator reflectă modificarea
            end
        end
    end
```

---

## 5. Surse și cercetare

1. **Simon Brown: The C4 Model for Visualising Software Architecture**  
   *URL:* https://c4model.com/  
   Fundamentează descompunerea ierarhică a sistemului pe niveluri clare de abstractizare (System Context, Container, Component).
2. **Martin Fowler: MonolithFirst**  
   *URL:* https://martinfowler.com/bliki/MonolithFirst.html  
   Argumentează începerea proiectării cu un monolit modular bine delimitat înainte de a împărți sistemul în servicii distribuite.
3. **Martin Kleppmann: Designing Data-Intensive Applications (DDIA)**  
   *URL:* [Cartea în format PDF](https://0-lucas.github.io/digital-garden/99.-Books/Martin-Kleppmann---Designing-Data-Intensive-Applications_-O%E2%80%99Reilly-Media-(2017).pdf)  
   Fundamentează decuplarea citirilor utilizatorilor de dependențele externe lente și modelele de consistență (*Bounded Staleness* și *Read-Your-Own-Writes*).
4. **Google SRE Book: Addressing Cascading Failures**  
   *URL:* https://sre.google/sre-book/addressing-cascading-failures/  
   Explică mecanismele de protecție împotriva defectării furnizorilor externi prin timeout-uri și politici de reîncercare cu pauză.
5. **OWASP Authorization Cheat Sheet**  
   *URL:* https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html  
   Fundamentează verificarea explicită a identității și a permisiunilor înainte de procesarea oricărei modificări de date.

---

## 6. Listă de verificare (Checklist)

- [x] Am inclus o diagramă System Context, una Container și una Component.
- [x] Am inclus cele trei diagrame de secvență cerute (Sincronizare piață, Citire preț, Modificare Watchlist).
- [x] Toate blocurile Mermaid sunt simple, conțin doar denumirile, fără detalii de implementare premature.
- [x] Diagrama Component deschide numai Dashboard Application.
- [x] Fluxurile de secvență arată rezultate normale, acțiuni respinse și defectări.
- [x] Proiectarea acoperă citirile, datele private, sincronizarea cu furnizorul și defectările.
- [x] Am precizat ipotezele inițiale sub fiecare diagramă de secvență.
- [x] Am specificat clar că o citire a utilizatorului nu apelează furnizorul extern.
- [x] Am marcat punctul de raportare a succesului și comportamentul următoarei citiri la Watchlist (Read-Your-Own-Writes).
