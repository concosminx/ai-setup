# Plan de implementare – Trips (organizator de excursii pentru grupuri)

> **Statut:** plan aprobat pentru v1. Acest document este **sursa unică de adevăr** pentru
> scop, arhitectură și etapizare. Este scris pentru a fi executat de un agent AI împreună cu
> un dezvoltator. Nu depinde de niciun alt fișier și înlocuiește documentul de idei anterior.
>
> **Ce NU acoperă acest pas:** produsul nu se construiește încă. Aici se stabilește planul.

---

## 1. Context și obiectiv

Se construiește o aplicație web pentru grupuri de prieteni care organizează excursii, în
special escapade de weekend la munte. Problema concretă pe care o rezolvă:

> **„Cum organizăm o excursie cu 5–15 prieteni fără 200 de mesaje pe WhatsApp și un Excel pe
> care nimeni nu îl actualizează?”**

Pentru un cititor ne-tehnic: cineva creează o excursie, trimite un link în grupul de WhatsApp,
toți intră fără cont și fără instalare, fiecare adaugă ce a plătit, iar la final aplicația
spune exact cine cui îi dă bani, în sume rotunde, astfel încât toată lumea să fie pe zero.

- **Tip proiect:** unealtă pentru cercul propriu, construită astfel încât transformarea
  ulterioară în produs să rămână posibilă. Nu se construiește ca produs comercial acum.
- **Echipă:** un singur dezvoltator, part-time (seri), experimentat cu Spring Boot, competent
  (nu expert) cu React.
- **Buget de timp:** ~7,5 săptămâni de seri până la finalul etapei E4.
- **Formă de livrare:** aplicație web mobile-first, instalabilă ca PWA. Fără aplicație
  nativă, fără magazin de aplicații, fără mod offline.
- **Licență:** nedecisă. Repository privat deocamdată; decizia se ia doar dacă proiectul
  devine produs.
- **Limbă:** interfața și acest document sunt în română; codul, schema bazei de date și API-ul
  sunt în engleză (vezi §17 Glosar).

### 1.1. Pana (wedge): banii

Dacă v1 face un singur lucru impecabil, acela este **partea financiară**: cheltuieli,
împărțire flexibilă și decontare cu număr minim de transferuri. Motivul nu este că banii sunt
cea mai spectaculoasă funcție, ci că sunt **singura funcție cu un criteriu obiectiv de
corectitudine**: decontarea fie aduce pe toată lumea la zero, fie nu. Asta o face testabilă și
asta face aplicația demnă de încredere.

A doua prioritate este **coordonarea** (cumpărături, responsabilități), care înlocuiește
Excel-ul mort din grup.

Toate celelalte idei — program orar, hartă, chat, vot, AI — sunt **amânate explicit** (§16).

---

## 2. Principii arhitecturale

Aceste principii sunt obligatorii și guvernează toate deciziile de implementare.

```text
React (PWA, .jsx) – servit de aceeași aplicație
        |  același origin, fără CORS
REST API + sesiune pe server
        |
Servicii de aplicație  ->  TripAccess (autorizare)
        |
Domeniu (money / trip / prep)
        |
PostgreSQL (Flyway deține schema)
```

1. **Un participant la excursie nu este un utilizator al aplicației.** Unii oameni din
   excursie nu vor deschide niciodată aplicația, dar trebuie să apară în decont. Modelul
   separă `Participant` (domeniu) de `UserAccount` (identitate). Toate tabelele de domeniu
   referă `participant_id`, **niciodată** `user_id`.
2. **Trecutul nu se mișcă.** Cotele unei cheltuieli se calculează și se **materializează** la
   momentul introducerii. Adăugarea unei persoane în excursie nu rescrie retroactiv
   cheltuielile de ieri.
3. **Banii sunt numere întregi.** Sumele se stochează ca numere întregi de unități minime
   (`long`). `double` și `float` nu apar nicăieri în apropierea banilor. În interfață și în
   acest document se vorbește exclusiv în **lei (RON)**.
4. **Invariantul fundamental:** `SUM(cote) == suma cheltuielii`, exact, întotdeauna. Verificat
   în domeniu, acoperit cu teste bazate pe proprietăți și impus în baza de date.
5. **Modular monolith** Spring Boot: un singur proces, pachete pe domeniu, granițe verificate
   cu ArchUnit. Modulele pot fi extrase ulterior; nu se începe cu microservicii.
6. **Autorizarea este per-excursie, nu globală.** Rolul unei persoane este o proprietate a
   apartenenței ei la *acea* excursie. Orice operație care atinge o excursie trece printr-o
   singură poartă, `TripAccess`, verificată printr-un test care enumeră toate endpoint-urile.
7. **Un singur artefact.** Frontend-ul se construiește în JAR-ul backend-ului și este servit
   de aceeași aplicație. Același origin înseamnă zero configurare CORS și cookie-uri de
   sesiune `SameSite=Lax` care funcționează fără excepții.
8. **Deploy din prima zi.** E0 pune scheletul în producție, pe domeniul real, cu HTTPS.
   Fiecare etapă ulterioară se termină în producție.
9. **Simplitate agresivă.** Fără WebSocket, fără staging, fără Kubernetes, fără librărie de
   state global, fără chat. Fiecare lucru neconstruit este o decizie scrisă, nu o omisiune.

---

## 3. Registru de decizii

| # | Decizie | Alternativă respinsă | Motiv |
|---|---|---|---|
| D1 | Unealtă pentru cercul propriu, construită ca să poată deveni produs | Produs comercial de la început | Bucla de feedback este reală, nu speculativă |
| D2 | Pana = banii; coordonarea a doua | Program / voturi / hartă | Singura funcție cu criteriu obiectiv de corectitudine |
| D3 | PWA mobile-first, fără offline | Aplicație nativă; web desktop-first | Momentele reale de folosire sunt pe telefon; offline este un proiect separat |
| D4 | `Participant` separat de `UserAccount` | Doar conturi; doar identitate din cookie | Singura variantă care păstrează și integritatea decontului, și intrarea fără fricțiune |
| D5 | v1 = 7 module (§4) | „MVP”-ul de 10 module din documentul de idei | 10 module solo, pe seri, înseamnă un an și zero module terminate |
| D6 | Cote materializate, înghețate la introducere | Calcul din reguli la citire | Altfel schimbarea grupului rescrie retroactiv deconturi deja închise |
| D7 | Stocare exactă în unități minime, **transferuri rotunjite la leu întreg** | Doar lei întregi (subunitățile pierdute la introducere) | Totalul aplicației trebuie să coincidă cu bonul; transferurile trebuie să fie sume transferabile |
| D8 | Decontare greedy, deterministă | Soluție optimă exactă | Minimizarea exactă este NP-hard; greedy dă cel mult n−1 transferuri și nimeni nu observă diferența |
| D9 | `Settlement` contează imediat, fără confirmare | Flux pending → confirmed | Între prieteni ceremonia nu se completează niciodată; urma de audit este suficientă |
| D10 | Fără integrări de plată | Revolut / bancă / transfer automat | Greutate reglementară disproporționată |
| D11 | Spring Boot 4.1 + Java 25 | Spring Boot 3.5 + Java 21 | Codul supraviețuiește primei excursii; migrarea amânată este temă, nu economie |
| D12 | Gradle (Kotlin DSL) | Maven | Preferința dezvoltatorului |
| D13 | Sesiuni pe server | JWT | Revocarea trebuie să funcționeze (eliminare din excursie, logout) |
| D14 | Link personal de revenire, fără email | Magic link pe email în v1 | Rezolvă pierderea dispozitivului cu zero infrastructură |
| D15 | React 19 + Vite, `.jsx` + `checkJs` | TypeScript complet; JS fără verificare | Protecție la granița API/bani, fără taxa de sintaxă TS |
| D16 | Fără live push; polling + refetch on focus | SSE / WebSocket | 8 oameni; o listă veche de 20 de secunde este acceptabilă |
| D17 | Blocare optimistă doar pe bani | Blocare peste tot; nicăieri | Bifarea este idempotentă și nu are ce să piardă; editarea unei cheltuieli are |
| D18 | Doar `OWNER` și `MEMBER` | Owner / Admin / Member + Trezorier | `ADMIN` nu are nicio atribuție distinctă la 8 oameni |
| D19 | Vizibilitate pe câmpuri amânată | Matrice de permisiuni în v1 | Muncă mare pentru un grup care oricum împarte cabana |
| D20 | Ștergere individuală = anonimizare | Ștergere fizică a participantului | Ștergerea ar rupe decontul celorlalți |
| D21 | Chat și evaluarea dificultății traseelor: **niciodată** | Amânate în v2 / v3 | Chat-ul pierde în fața WhatsApp; evaluarea de siguranță este răspundere pe date necontrolate |
| D22 | Fără excursie reală → **replay pe o excursie trecută** la finalul E2 | Termen auto-impus | Există adevăr de referință: decontul corect este deja cunoscut |
| D23 | E3 și E4 sunt condiționate de existența unei excursii reale | Construire pe speculație | Restul se construiește la cerere, nu în avans |

---

## 4. Scope v1

### 4.1. Ce intră

| Modul | Conținut |
|---|---|
| **Excursie + invitație** | Creare excursie, link de invitație revocabil, intrare cu nume, link personal de revenire |
| **Participanți** | Listă, status (confirmat / poate / nu vine), roluri, pre-creare de către organizator, fuzionare duplicate |
| **Cheltuieli** | Adăugare / editare / ștergere, un plătitor, împărțire egal / pe cote / procentual / pe sume exacte / doar între anumite persoane |
| **Decontare** | Solduri nete, plan de transferuri minimizat, sume în lei întregi, înregistrarea plăților efectuate |
| **Cumpărături** | Listă comună, cantitate, categorie, responsabil, bifare instantanee |
| **Responsabilități** | Task-uri cu responsabil și termen, pe același motor ca lista de cumpărături |
| **„Pentru tine”** | Pagină personală: ce ai de făcut, ce ai de adus, cât ai de dat, cât ai de primit |

### 4.2. Ce NU intră, deliberat

Program orar cu drag & drop · Hartă · Voturi · Meniu · Generare automată a listei din meniu ·
Checklist personal de bagaj · Cazare și repartizarea camerelor · Transport și împărțirea
combustibilului · Urmărirea bugetului pe categorii · Rezervări · Notificări · Chat · Vreme ·
Siguranța traseelor · Orice componentă AI · Memories · Next Trip · Monetizare · Integrări
externe.

**Regula de scope:** pentru orice lucru adus în v1, alt lucru iese din v1. Fără excepții.

---

## 5. Model de identitate

### 5.1. Cele două straturi

- **`Participant`** — o persoană din excursie. Are nume, rol, status și este ținta tuturor
  legăturilor de domeniu (cheltuieli, cote, task-uri, cumpărături). Există indiferent dacă
  omul deschide vreodată aplicația: organizatorul poate crea „Radu, nu folosește aplicații”.
- **`UserAccount`** — identitate persistentă, cu email, reutilizabilă între excursii. În v1
  este **opțională și practic neatinsă**; coloana `participant.user_account_id` există,
  nulabilă, pentru upgrade-ul din v1.1.

### 5.2. Fluxul de intrare

1. Organizatorul creează excursia și primește un **link de invitație**: `/join/{token}`.
2. Linkul ajunge în grupul de WhatsApp. Oricine îl deschide vede o previzualizare a excursiei
   (nume, destinație, perioadă, număr de participanți) — fără date financiare.
3. Introduce un nume → se creează un `Participant` → se emite o sesiune pe server.
4. Imediat după intrare, ecranul afișează proeminent **linkul personal de revenire**
   (`/me/{personal_token}`) cu îndemnul „trimite-ți-l ție pe WhatsApp”. Acesta restaurează
   identitatea pe orice dispozitiv, oricând.

### 5.3. Token-uri

Atât token-ul de invitație, cât și cel personal sunt **credențiale de tip bearer**:

- 128 de biți aleatori, generați criptografic, codificați URL-safe;
- stocați **doar sub formă de hash**, niciodată în clar — se tratează ca o parolă;
- **revocabile și regenerabile**, fără expirare pe timp;
- regenerarea invitației invalidează linkul vechi, fără a afecta participanții deja intrați.

Riscul acceptat și scris explicit: cine are linkul poate intra în excursie și vede toate datele
ei financiare. Este exact aceeași expunere ca a grupului de WhatsApp în care a fost trimis.

### 5.4. Duplicate

Riscul acceptat: cineva intră de două ori și apar doi „Radu”, fiecare cu jumătate din datorii.
Există **prevenție** (linkul personal de revenire) și **remediu**:

> `POST /api/trips/{id}/participants:merge` — rezervată rolului `OWNER`. Mută toate
> cheltuielile, cotele, plățile, task-urile și produsele de la participantul sursă la cel
> țintă, apoi dezactivează sursa. Tranzacțională și auditată.

### 5.5. Roluri

| Rol | Poate |
|---|---|
| `OWNER` | Tot ce poate un `MEMBER`, plus: șterge excursia, revocă/regenerează invitația, fuzionează participanți, elimină membri, pre-creează participanți |
| `MEMBER` | Vede tot din excursie; adaugă și editează cheltuieli, plăți, produse, task-uri; își modifică propriul status |

`MEMBER` poate edita cheltuieli pentru că o aplicație de bani de grup în care doar
organizatorul introduce sume își ratează scopul. Enum-ul lasă loc pentru `ADMIN` atunci când o
excursie reală demonstrează nevoia.

---

## 6. Model de date

Toate cheile primare sunt **UUIDv7** (sortabile în timp, deci indexează bine, și
neenumerabile — relevant pentru că URL-urile ajung în grupuri de chat). Toate tabelele au
`created_at` și `updated_at`.

### 6.1. `trip`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `name` | text | „Weekend Bucegi” |
| `destination` | text | nulabil |
| `start_date`, `end_date` | date | nulabile; fără orar în v1 |
| `currency` | char(3) | implicit `RON`; **o singură monedă per excursie** |
| `minor_unit_scale` | int | implicit `100`; factorul dintre leu și unitatea minimă |
| `deleted_at` | timestamptz | nulabil |

### 6.2. `user_account`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `email` | citext | unic, nulabil — neutilizat în v1 |

### 6.3. `participant`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `trip_id` | uuid | FK → `trip` |
| `display_name` | text | |
| `role` | text | `OWNER` / `MEMBER` |
| `status` | text | `CONFIRMED` / `MAYBE` / `DECLINED` |
| `user_account_id` | uuid | FK nulabil — puntea către v1.1 |
| `personal_token_hash` | text | unic |
| `active` | boolean | `false` = a părăsit excursia; istoricul rămâne |
| `anonymized_at` | timestamptz | nulabil |

Constrângere: exact un `OWNER` activ per excursie.

### 6.4. `trip_invite`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `trip_id` | uuid | FK |
| `token_hash` | text | unic |
| `revoked_at` | timestamptz | nulabil |

### 6.5. `expense`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `trip_id` | uuid | FK |
| `paid_by_participant_id` | uuid | FK — **un singur plătitor** în v1 |
| `amount_minor` | bigint | `CHECK > 0` |
| `description` | text | |
| `category` | text | enum liber: cazare, mâncare, transport, activități, diverse |
| `spent_at` | date | |
| `split_mode` | text | `EQUAL` / `SHARES` / `PERCENT` / `EXACT` |
| `split_input` | jsonb | parametrii de intrare, pentru reeditare |
| `created_by_participant_id` | uuid | FK |
| `version` | int | blocare optimistă |
| `deleted_at` | timestamptz | ștergere logică |

Două cheltuieli separate dacă plătesc doi oameni. Plătitori multipli: v2.

### 6.6. `expense_share`

| Coloană | Tip | Note |
|---|---|---|
| `expense_id` | uuid | PK compus |
| `participant_id` | uuid | PK compus |
| `share_amount_minor` | bigint | `CHECK >= 0` |

**Invariantul central**, impus în trei locuri: în domeniu la construcție, într-un test bazat pe
proprietăți și printr-un **trigger `DEFERRABLE` în PostgreSQL** care refuză commit-ul dacă
`SUM(share_amount_minor) != expense.amount_minor`.

Persoanele „scutite” de o cheltuială (documentul de idei, §18) nu au nevoie de nicio funcție
specială: pur și simplu nu au rând în `expense_share`.

### 6.7. `settlement`

| Coloană | Tip | Note |
|---|---|---|
| `id` | uuid | PK |
| `trip_id` | uuid | FK |
| `from_participant_id` | uuid | FK |
| `to_participant_id` | uuid | FK, `CHECK <> from` |
| `amount_minor` | bigint | `CHECK > 0` |
| `settled_at` | date | |
| `recorded_by_participant_id` | uuid | FK — urma de audit |
| `note` | text | nulabil |

### 6.8. `shopping_item` și `task`

Același motor, două tabele:

| `shopping_item` | `task` |
|---|---|
| `name`, `quantity`, `unit`, `category` | `title`, `description` |
| `assignee_participant_id` | `assignee_participant_id` |
| `checked`, `checked_by`, `checked_at` | `status` (`TODO`/`DOING`/`DONE`), `done_by`, `done_at` |
| `estimated_amount_minor` (nulabil) | `due_date` |

### 6.9. Structura pachetelor

```text
ro.trips.app
  trip/        Trip, Participant, Invite, TripAccess, TripService
  money/       Expense, ExpenseShare, Settlement, Money,
               SplitCalculator, BalanceCalculator, SettlementPlanner
  prep/        ShoppingItem, Task
  dashboard/   modele de citire pentru /overview și /me
  identity/    UserAccount, sesiune, token-uri
  shared/      configurare, securitate, erori, UUIDv7
```

Reguli verificate cu **ArchUnit**:

- `money` nu depinde de `prep` și nici de `dashboard`;
- nimic nu depinde de `dashboard`;
- nicio entitate JPA nu apare în semnătura unui controller;
- niciun controller nu ajunge la un serviciu cu domeniu de excursie fără `TripAccess`;
- `double` și `float` nu apar în pachetul `money`.

---

## 7. Regulile banilor

Secțiunea aceasta este specificația executabilă a panei. Orice ambiguitate aici este un bug.

### 7.1. Reprezentare

- Suma se stochează ca **număr întreg de unități minime**: `1.600,00 lei` → `160000`.
- Tipul în domeniu este un value object `Money` (`long amountMinor`, `Currency`), fără
  aritmetică pe `double` nicăieri.
- API-ul expune `{"amount": "1600.00", "currency": "RON"}` — șir zecimal, nu `number`, ca să nu
  existe pierderi prin `IEEE 754` în JavaScript.
- Interfața afișează `Intl.NumberFormat('ro-RO')`: `1.600,00 lei`.
- La granița dintre API și frontend există **o singură funcție de parsare** care transformă
  suma primită într-un obiect `Money` și aruncă zgomotos dacă formatul nu este cel așteptat.

### 7.2. Moduri de împărțire

| Mod | Intrare | Calcul |
|---|---|---|
| `EQUAL` | lista de participanți | rest maxim (§7.3) pe ponderi egale |
| `SHARES` | ponderi întregi (ex. 2 persoane într-o cameră = 2) | rest maxim pe ponderi |
| `PERCENT` | procente, sumă = 100 | rest maxim pe procente |
| `EXACT` | sume explicite | validare: suma introdusă == totalul cheltuielii |

În toate cazurile rezultatul este **materializat** ca rânduri `expense_share` în unități
minime. Modul și parametrii se păstrează în `split_mode` / `split_input` doar pentru
redeschiderea formularului la editare — **niciodată** pentru recalcul la citire.

### 7.3. Rotunjire: metoda restului maxim

Pentru o sumă `A` împărțită după ponderile `w1..wn` cu `W = Σwi`:

```text
base_i  = floor(A * w_i / W)
rest    = A - Σ base_i                       // 0 <= rest < n
frac_i  = A * w_i / W - base_i                // partea fracționară
```

Se adaugă câte **o unitate minimă** primilor `rest` participanți, ordonați descrescător după
`frac_i`, cu **departajare după `participant_id` crescător**. Ordonarea explicită este
obligatorie: fără ea, aceeași cheltuială s-ar putea împărți diferit de la o rulare la alta, iar
aplicația ar părea că inventează bani.

Exemplu: `1.600,00 lei` la 7 persoane → 228,58 / 228,58 / 228,57 × 5, total exact `1.600,00`.

### 7.4. Solduri

Pentru fiecare participant `i`:

```text
net_i = platit_i - datorat_i + trimis_i - primit_i
```

unde `platit_i` = suma cheltuielilor plătite de `i`, `datorat_i` = suma cotelor lui `i`,
`trimis_i` / `primit_i` = plățile înregistrate. `net_i > 0` înseamnă **are de primit**.

Din construcție, `Σ net_i = 0`. Aceasta este a doua proprietate testată.

Soldurile se calculează cu **SQL scris de mână prin `JdbcClient`**, nu prin JPA: sunt agregări,
nu grafuri de obiecte. Două interogări, verificabile la citire.

### 7.5. Rotunjirea soldurilor la leu întreg

Documentul de idei arată transferuri în lei rotunzi și are dreptate: nimeni nu virează
`170,43 lei`. Soluția **nu** este rotunjirea transferurilor după calcul (ar strica echilibrul),
ci rotunjirea **soldurilor**, păstrând suma zero, înainte de planificare:

```text
b_i = floor(net_i / 100)                     // lei, rotunjire spre -infinit
f_i = net_i - 100 * b_i                      // rest in [0, 100)
K   = (Σ f_i) / 100                          // intreg, pentru ca Σ net_i = 0
```

Se adaugă `+1 leu` celor `K` participanți cu `f_i` cel mai mare, departajare după
`participant_id`. Rezultatul: solduri în lei întregi care **însumează tot zero**, fiecare la
mai puțin de un leu distanță de soldul exact.

Planul de transferuri se calculează **pe soldurile rotunjite**, deci toate transferurile sunt
sume rotunde și decontarea închide exact.

Consecință scrisă explicit în interfață: diferențele sub un leu sunt absorbite în rotunjire.
Soldul exact rămâne vizibil lângă cel rotunjit, pentru cine vrea să verifice.

### 7.6. Planul de decontare

Algoritm greedy pe soldurile rotunjite:

1. Se separă creditorii (`net > 0`) de debitori (`net < 0`).
2. Ambele liste se sortează descrescător după valoare absolută, **departajare după
   `participant_id`** — determinismul este cerință, nu detaliu: altfel lista se rearanjează
   între telefoanele a doi oameni și nimeni nu mai are încredere în ea.
3. Se potrivește repetat cel mai mare debitor cu cel mai mare creditor, transferul fiind
   `min(datorie, creanță)`; se scade din ambii și se elimină cel ajuns la zero.
4. Rezultat: cel mult `n − 1` transferuri.

Minimizarea exactă a numărului de transferuri este NP-hard. Greedy nu este întotdeauna optim și
în practică diferența nu apare la 8 persoane.

Planul de decontare este **recalculat, nu stocat**. Înregistrarea unei plăți (`settlement`)
schimbă soldurile, deci schimbă planul.

### 7.7. Ce nu face aplicația cu banii

- Nu inițiază și nu procesează plăți. Transferul se face în aplicația bancară.
- Nu are curs valutar și nici mai multe monede într-o excursie.
- Nu „închide” excursia în v1: soldurile rămân vii la nesfârșit. Când arhivarea va exista,
  ea va însemna **fără cheltuieli noi**, niciodată istoric recalculat.

---

## 8. API REST

Stil orientat pe resurse sub `/api/trips/{tripId}/…`, plus câteva **acțiuni de domeniu
explicite** acolo unde se petrece o tranziție de stare, nu o editare de câmp.

### 8.1. Identitate și excursie

| Metodă | Cale | Note |
|---|---|---|
| `POST` | `/api/trips` | creează excursia; autorul devine `OWNER` |
| `GET` | `/api/trips` | excursiile la care am acces |
| `GET` `PATCH` `DELETE` | `/api/trips/{id}` | `DELETE` doar `OWNER`, cascadă completă |
| `GET` | `/api/invites/{token}` | previzualizare publică, fără date financiare |
| `POST` | `/api/invites/{token}/accept` | `{ displayName }` → creează participantul, deschide sesiunea, **returnează linkul personal o singură dată** |
| `POST` | `/api/trips/{id}/invite:regenerate` | `OWNER` |
| `DELETE` | `/api/trips/{id}/invite` | revocare |
| `POST` | `/api/sessions/resume/{personalToken}` | restaurează identitatea pe alt dispozitiv |
| `POST` | `/api/sessions/logout` | |

### 8.2. Participanți

| Metodă | Cale | Note |
|---|---|---|
| `GET` `POST` | `/api/trips/{id}/participants` | `POST` = pre-creare de către `OWNER` |
| `PATCH` | `/api/trips/{id}/participants/{pid}` | nume, status |
| `POST` | `/api/trips/{id}/participants:merge` | `{ sourceId, targetId }`, `OWNER` |
| `DELETE` | `/api/trips/{id}/participants/{pid}` | dezactivare, nu ștergere |
| `POST` | `/api/trips/{id}/participants/{pid}:anonymize` | §12.3 |

### 8.3. Bani

| Metodă | Cale | Note |
|---|---|---|
| `GET` `POST` | `/api/trips/{id}/expenses` | |
| `GET` `PUT` `DELETE` | `/api/trips/{id}/expenses/{eid}` | `PUT` cere `version`; conflict → `409` |
| `GET` | `/api/trips/{id}/balances` | solduri exacte și rotunjite |
| `GET` | `/api/trips/{id}/settlement-plan` | recalculat la fiecare cerere |
| `GET` `POST` | `/api/trips/{id}/settlements` | |
| `DELETE` | `/api/trips/{id}/settlements/{sid}` | |

### 8.4. Pregătire

| Metodă | Cale | Note |
|---|---|---|
| `GET` `POST` | `/api/trips/{id}/shopping-items` | |
| `PATCH` `DELETE` | `/api/trips/{id}/shopping-items/{iid}` | |
| `PUT` | `/api/trips/{id}/shopping-items/{iid}/status` | **idempotent**, `{ checked, by }` |
| `GET` `POST` | `/api/trips/{id}/tasks` | |
| `PATCH` `DELETE` | `/api/trips/{id}/tasks/{tid}` | |
| `PUT` | `/api/trips/{id}/tasks/{tid}/status` | **idempotent** |

### 8.5. Două endpoint-uri agregate, deliberat ne-REST

Ecranul principal și pagina „Pentru tine” asamblează fiecare date din cinci module. Naiv, asta
înseamnă cinci-șase cereri, pe telefon, pe semnal de munte. Prin urmare:

| Metodă | Cale | Servește |
|---|---|---|
| `GET` | `/api/trips/{id}/overview` | ecranul principal: următoarele evenimente, sumar financiar, progres cumpărături, progres task-uri |
| `GET` | `/api/trips/{id}/me` | pagina personală: ce am de făcut, ce am de adus, cât am de dat, cât am de primit |

Sunt **excepții intenționate** de la stilul REST, notate aici ca atare, pentru ca cineva să nu
le „curețe” peste șase luni.

### 8.6. Convenții

- **DTO-uri explicite** (`record`) per endpoint. Entitățile nu traversează niciodată granița
  controller-ului.
- **Erori: RFC 9457 Problem Details**, suportat nativ de Spring. Frontend-ul tratează o
  singură formă de eroare.
- **Validare** Bean Validation la margine; invarianții de bani în domeniu.
- **Contract: springdoc-openapi**, din care se generează **clientul TypeScript (`.d.ts`)**
  pentru aplicația React (§9.2).
- **Fără versionare de API.** Client și server se livrează împreună.
- **Fără paginare** în v1; listele au plafon dur. O excursie are 8 oameni și 40 de cheltuieli.

---

## 9. Frontend

### 9.1. Stack

| Aspect | Alegere |
|---|---|
| Nucleu | React 19.2 + Vite, fișiere `.jsx` |
| Rutare | React Router |
| Stare de server | **TanStack Query** |
| Stare de client | `useState` + un singur context (excursia curentă, participantul curent). Fără Redux, fără Zustand |
| Stilizare | Tailwind v4 + primitive Radix pentru dialog, select, popover, checkbox, toast |
| Formulare | React Hook Form — doar pentru formularul de cheltuială/împărțire |
| PWA | `vite-plugin-pwa`: manifest, iconițe, precache doar pentru app shell, prompt de instalare. **Fără cache de API.** |
| Limbă | Doar română, șirurile centralizate într-un singur modul, fără librărie de i18n |
| Formatare | `Intl.NumberFormat('ro-RO')`, `Intl.DateTimeFormat('ro-RO')` |

### 9.2. Tipuri fără TypeScript

Codul se scrie în `.jsx`, fără adnotări de tip. Protecția vine din două lucruri:

1. `checkJs` activat, plus declarațiile `.d.ts` generate din OpenAPI pentru clientul de API.
   Redenumirea unui câmp în backend se aprinde roșu în editor imediat, iar autocomplete-ul
   cunoaște forma răspunsurilor.
2. **Validare la execuție la granița banilor**: o funcție de parsare care transformă suma
   primită de la API într-un `Money` și aruncă zgomotos dacă nu are forma așteptată.

### 9.3. Ecrane

Documentul de idei propunea cinci taburi. Cu programul amânat, rămân patru:

| Tab | Rută | Conținut |
|---|---|---|
| 🏔️ Excursia | `/t/:tripId` | ecranul principal (`/overview`) |
| 💰 Bani | `/t/:tripId/money` | cheltuieli, adăugare cheltuială, solduri, plan de decontare, plăți |
| 🛒 Pregătire | `/t/:tripId/prep` | cumpărături + responsabilități |
| 👥 Grup | `/t/:tripId/group` | participanți, status, invitație |

Plus, în afara taburilor:

- `/join/:token` — previzualizare + introducerea numelui + afișarea linkului personal;
- `/t/:tripId/me` — **„Pentru tine”**, accesibilă din orice tab, cea mai importantă pagină
  pentru un participant obișnuit;
- `/` — lista excursiilor mele.

### 9.4. Reguli de interacțiune

- Bifarea unui produs sau task folosește **actualizare optimistă**: se schimbă instantaneu,
  se reconciliază când răspunde serverul. Fără asta, lista este inutilizabilă pe wi-fi de
  magazin.
- Formularul de cheltuială arată **în timp real** cum se împarte suma, pe fiecare persoană,
  înainte de salvare. Este locul în care aplicația își câștigă sau își pierde credibilitatea.
- Pe conflict `409` la editarea unei cheltuieli: refetch și mesaj explicit — „Andrei a
  modificat între timp această cheltuială, verifică suma”.

---

## 10. Autentificare, autorizare, concurență

### 10.1. Sesiune

- **Sesiuni pe server**, Spring Session JDBC, cookie `httpOnly` + `Secure` + `SameSite=Lax`.
- Subiectul autentificat este un tip cu două variante: `USER:{uuid}` sau
  `GUEST:{participant_uuid}`.
- Revocarea este un `DELETE` în tabela de sesiuni. Eliminarea din excursie și logout-ul
  funcționează imediat — motivul principal pentru care nu se folosește JWT.

### 10.2. Autorizare

Rezolvarea este întotdeauna aceeași: **subiect + excursie → apartenență + rol**. Nu există
roluri globale. Toate operațiile cu domeniu de excursie trec prin componenta `TripAccess`, nu
prin `@PreAuthorize` împrăștiat prin cod.

Există un **test care enumerează prin reflecție toate metodele de controller cu domeniu de
excursie** și verifică faptul că fiecare respinge un ne-membru. Un endpoint nou care uită
poarta pică în CI, nu în producție.

### 10.3. Concurență

Două situații diferite, două răspunsuri diferite:

| Situație | Tratament |
|---|---|
| Doi oameni bifează în magazin | **Fără blocare.** Scriere idempotentă de stare (`PUT …/status`). Două bifări concurente dau același rezultat; nu există conflict. |
| Doi oameni editează aceeași cheltuială | **Blocare optimistă** (`@Version`) → `409`. Singurul loc din aplicație care merită această complexitate. |

**Prospețime:** fără push. TanStack Query cu refetch la revenirea în fereastră, plus polling la
~20 de secunde cât timp un ecran de excursie este deschis. Consecință acceptată: cineva poate
vedea o bifă veche de 20 de secunde. Pentru acest produs este perfect în regulă — iar pretenția
contrară este exact felul în care un proiect de weekend capătă un strat de WebSocket.

---

## 11. Testare

Proiectul are o separare neobișnuit de curată între cod unde bug-urile sunt catastrofale și cod
unde sunt cosmetice. Efortul de testare este dezechilibrat intenționat.

### 11.1. Proprietăți (jqwik) — nucleul

Se scriu **înainte** de codul de decontare. Sunt forma executabilă a §7.

| # | Proprietate |
|---|---|
| P1 | Pentru orice sumă și orice configurație de împărțire: `Σ cote == suma`, exact |
| P2 | Pentru orice set de cheltuieli și plăți: `Σ solduri == 0` |
| P3 | Aplicarea transferurilor propuse aduce fiecare sold exact la zero |
| P4 | Aplicarea transferurilor **rotunjite la leu** aduce fiecare sold rotunjit la zero, iar diferența față de soldul exact este sub un leu |
| P5 | Determinism: aceleași intrări produc aceeași listă de transferuri, în aceeași ordine |

Câteva sute de linii care acoperă toată pana, inclusiv cazurile la care nimeni nu s-ar gândi
manual: un singur participant, cheltuială de zero, toată lumea pe zero, o singură persoană care
plătește tot.

### 11.2. Integrare — Testcontainers, PostgreSQL real, HTTP real

Intrare prin invitație · revenire prin link personal · fuzionare participanți · CRUD cheltuieli
cu `409` pe versiune învechită · `/overview` · `/me` · testul de autorizare prin reflecție
(§10.2).

**Fără H2.** Aceeași bază de date în dezvoltare, în teste și în producție: invarianții de bani
merită constrângeri la nivel de bază de date, iar constrângerile care există doar în producție
sunt constrângeri netestate.

### 11.3. Frontend — trei scenarii Playwright

Intrare în excursie → adăugare cheltuială cu împărțire personalizată → vizualizarea decontului.
Atât. Fără teste unitare de componente, fără snapshot-uri, fără suită React Testing Library:
pe bugete de seri, costă mai mult decât aduc.

### 11.4. Ce nu se testează

Componentele de interfață, CSS-ul, manifestul PWA, orice modul amânat.

### 11.5. Definiția lui „gata” pentru o funcție

- proprietățile și testele de integrare trec;
- niciun `double` în apropierea banilor;
- autorizarea trece prin `TripAccess`;
- funcționează pe un viewport real de telefon;
- migrarea Flyway este în repository.

---

## 12. Confidențialitate și date

### 12.1. Ce se amână intenționat

**Vizibilitate pe câmpuri.** În v1, **toată lumea dintr-o excursie vede tot din acea excursie**.
O matrice de permisiuni (ascunde bugetele individuale, ascunde camerele) atinge fiecare
interogare și fiecare endpoint — muncă mare pentru un grup care oricum împarte cabana. Decizie
de v2, nu omisiune.

### 12.2. Părăsirea excursiei

Cine datorează 200 de lei nu poate pur și simplu să dispară: ștergerea rândului ar rupe
decontul celorlalți. Prin urmare: **părăsirea dezactivează apartenența**; rândul `participant`
și istoricul financiar rămân. Persoana pierde accesul, nu mai apare în împărțiri noi, iar
trecutul rămâne intact.

### 12.3. Ștergerea datelor

- **Ștergerea excursiei** (doar `OWNER`): ștergere fizică, cascadă completă.
- **Ștergere individuală: anonimizare, nu ștergere.** Numele devine „Participant șters”,
  legătura cu contul se elimină, rândurile financiare își păstrează `participant_id`. Decontul
  rămâne echilibrat și nu mai rămâne nicio dată cu caracter personal. Costă un endpoint tocmai
  pentru că modelul separă `Participant` de `UserAccount` (§5.1).

### 12.4. Poziție

Cât timp aplicația servește cercul propriu, aparatul formal GDPR (politică de confidențialitate,
DPA, proces documentat pentru cereri) nu este necesar — dar **designul este capabil de ștergere
din prima zi**. Dacă aplicația se deschide către străini, **analiza GDPR este o condiție
înainte de lansare, nu după**.

Consecințe imediate:

- **Fără analytics și fără tracking terț** în v1. Cookie-ul de sesiune este strict necesar,
  deci **nu este nevoie de niciun banner de cookie-uri.**
- **Fără date cu caracter personal în log-uri**: doar identificatori, niciodată nume sau sume —
  Sentry le-ar primi.

---

## 13. Build, livrare, operare

### 13.1. Structura repository-ului

```text
trips/
  backend/        Gradle (Kotlin DSL), Spring Boot
  frontend/       Vite, React, .jsx
  compose.yaml    app + postgres + caddy
  .github/workflows/
```

O sarcină Gradle rulează build-ul Vite și copiază rezultatul în resursele statice ale
backend-ului. Rezultat: **un JAR, un container, aceeași aplicație servește și SPA-ul, și
API-ul.**

Acest lucru nu este cosmetic: același origin înseamnă **zero configurare CORS** și cookie-uri
`SameSite=Lax` care funcționează. Găzduirea separată (Vercel + API în altă parte) ar forța
CORS, domenii de cookie și `SameSite=None` fără niciun beneficiu la această scară. În
dezvoltare, serverul Vite face proxy pentru `/api` către `:8080` și proprietățile se păstrează.

### 13.2. Operare

| Aspect | Alegere |
|---|---|
| Gazdă | VPS mic (clasa Hetzner CX22, ~5 €/lună) |
| Compose | `app` + `postgres` + **Caddy** (TLS Let's Encrypt automat, ~5 linii de configurare) |
| HTTPS + domeniu | **Obligatorii**: instalarea PWA, service worker-ul și cookie-urile `Secure` le cer |
| CI | GitHub Actions: build, teste cu Testcontainers, publicare imagine în GHCR |
| Deploy | `docker compose pull && up -d` pe VPS, manual sau printr-un webhook minimal |
| Secrete | `.env` pe gazdă, niciodată în git |
| **Backup** | **`pg_dump` nocturn, offsite, plus o restaurare repetată efectiv înainte de prima excursie reală** |
| Observabilitate | Actuator health + log-uri JSON structurate + un ping de uptime gratuit |
| Urmărirea erorilor | **Sentry** (plan gratuit) sau GlitchTip auto-găzduit |

Fără staging, fără Kubernetes, fără Prometheus, fără Grafana. Omisiunile sunt intenționate și
sunt ceea ce face proiectul livrabil pe seri.

Două lucruri merită apărate explicit:

- **Backup cu restaurare repetată.** Aplicația ține evidența banilor reali dintre oameni
  reali. „Am backup-uri” care nu au fost niciodată restaurate nu înseamnă că ai backup-uri.
  Este o oră de muncă, o singură dată, și face diferența dintre un VPS pierdut ca inconvenient
  și un VPS pierdut ca sfârșit al proiectului.
- **Sentry.** Într-un PWA nu poți vedea consola Ioanei. Fără urmărirea erorilor, fiecare
  raport de bug este „nu a mers”, fără stack trace, și se ard seri ghicind.

---

## 14. Plan pe etape

Regula care guvernează etapizarea: **E0 ajunge în producție din prima zi, și fiecare etapă
ulterioară se termină în producție.** Fără „integrăm la final”.

### E0 — Fundația (~1 săptămână)

Repository, build Gradle care împachetează rezultatul Vite în JAR, PostgreSQL + Flyway,
`compose.yaml` cu Caddy, GitHub Actions, GHCR, primul deploy. **Criteriu:** endpoint de health
accesibil pe HTTPS, pe domeniul real.

### E1 — Identitate și excursie (~1,5 săptămâni)

`Trip`, `Participant`, `UserAccount` nulabil, token de invitație cu hash + revocare/regenerare,
flux de intrare, link personal de revenire, fuzionare participanți, roluri, `TripAccess` și
testul de autorizare prin reflecție. **Criteriu:** opt oameni pot intra într-o excursie de pe
telefoanele lor, iar unul își poate recăpăta identitatea pe alt dispozitiv.

### E2 — Banii (~2,5 săptămâni) — etapa protejată

`Expense`, `ExpenseShare`, cele patru moduri de împărțire, `Money`, trigger-ul de invariant în
PostgreSQL, soldurile în SQL, rotunjirea soldurilor la leu, planificatorul greedy, `Settlement`,
și **toate cele cinci proprietăți jqwik, scrise înainte de codul de decontare**.

**Criteriu (poarta de validare, D22):** o excursie trecută reală, al cărei decont este deja
cunoscut, este introdusă în aplicație, iar planul de decontare produs coincide cu ce s-a
întâmplat în realitate.

### E3 — Coordonare (~1,5 săptămâni) — condiționată

Cumpărături și responsabilități pe un singur motor, scrieri idempotente de stare, actualizare
optimistă în interfață, responsabil și termen.

### E4 — Ecranele (~1 săptămână) — condiționată

`/overview` și `/me`, manifest PWA + instalare, Sentry, formatare `ro-RO`, finisaj pe mobil,
repetiția restaurării din backup.

### E5 — Prima excursie reală

O excursie reală rulează pe aplicație. Se repară ce se strică.

### Ordinea nu este negociabilă

~7,5 săptămâni de seri, fără rezervă. Dacă realitatea intervine, **scope-ul se taie din E3 și
E4, niciodată din E2.**

Conform D23, **E3 și E4 se construiesc atunci când există o excursie reală în calendar**, nu
înainte. După E2 există un punct de decizie legitim: aplicația este deja un înlocuitor
funcțional de Tricount și poate rămâne acolo.

---

## 15. Criteriul de acceptanță

v1 este gata când următorul scenariu rulează cap-coadă, pe telefoane reale:

> Andrei creează *Weekend Bucegi, 18–20 septembrie* și trimite linkul în grupul de WhatsApp.
> Șapte oameni îl deschid pe telefon, își scriu numele și sunt înăuntru — fără cont, fără
> instalare. Andrei introduce cabana: 1.600 lei, împărțită la toți 8. Mihai introduce
> cumpărăturile: 480 lei. Ioana introduce benzina: 300 lei, împărțită doar la cei 4 din mașina
> ei. Radu introduce restaurantul: 640 lei pentru cei 6 care au fost. Fiecare deschide *Pentru
> tine* și vede ce are de plătit și ce are de adus. Duminică, aplicația arată: total 3.020 lei,
> și trei transferuri în lei întregi care aduc pe toată lumea la zero. Ioana trimite cei 150 de
> lei, marchează plata, iar soldul ei devine zero. Nimeni nu a deschis niciun Excel.

---

## 16. Riscuri

| # | Risc | Mitigare |
|---|---|---|
| R1 | **Nu există o dată reală de excursie**, deci nu există forță externă care să împingă proiectul | Replay pe o excursie trecută la finalul E2 (D22); E3/E4 condiționate de existența unei excursii reale (D23) |
| R2 | **Derivă de scope** înapoi către documentul de 44 de secțiuni | Lista de amânări face parte din plan; orice lucru adus în v1 scoate altceva din v1 (§4.2) |
| R3 | O aplicație de coordonare de grup **nu are valoare cu un singur utilizator**; nici cea mai bună implementare nu este dovedită până când 8 oameni nu o folosesc efectiv | Nu se poate simula. Risc numit, nu rezolvat |
| R4 | Participanți duplicat care fragmentează decontul | Link personal de revenire (prevenție) + fuziune de participanți (remediu), §5.4 |
| R5 | Linkul de invitație este o credențială purtătoare care circulă în chat | Stocat ca hash, revocabil și regenerabil; expunere egală cu a grupului în care a fost trimis; scris explicit în plan |
| R6 | Ecosistemul Spring Boot 4 este încă în urmă — exemplele și codul generat presupun Boot 3 (`spring-boot-starter-web` în loc de `spring-boot-starter-webmvc`) | Zgârieturi, nu blocaje, pentru un dezvoltator experimentat; alternativa de retragere este Boot 3.5 + Java 21 |
| R7 | Fără push pe iOS decât cu PWA instalat | Notificările sunt oricum amânate; când vor veni, întâi email sau in-app |
| R8 | Pierderea VPS-ului = pierderea evidenței banilor | Backup nocturn offsite **cu restaurare repetată efectiv** înainte de prima excursie |

---

## 17. După v1

### v1.1 — ieftine, valoare mare

| Funcție | Note |
|---|---|
| **Clonarea unei excursii** | Cea mai ieftină funcție cu valoare mare din tot documentul de idei: copiază participanți, task-uri, structură. Câteva zile de muncă și transformă o unealtă de o excursie în ceva reutilizabil. **Deasupra voturilor.** |
| **Vremea** | Open-Meteo: gratuit, fără cheie de API. Jumătate de zi pentru ceva chiar util înainte de un weekend la munte |
| **Voturi** | Ieftine, dar concurează cu un poll de WhatsApp gratuit pe care grupul îl folosește deja |
| **Buget pe categorii** | Rezultă aproape gratuit din modelul de cheltuieli |
| **Conturi cu email** | Magic link; `participant.user_account_id` există deja |

### v2 — funcții reale, cost real

Program orar cu drag & drop · Hartă · Cazare și camere · Transport și împărțirea
combustibilului · Rezervări · Memories · Notificări (constrâns: push pe iOS cere PWA instalat,
deci întâi email sau in-app) · Vizibilitate pe câmpuri (§12.1) · Plătitori multipli pe o
cheltuială · Arhivarea excursiei.

### v3 — AI

Scanarea bonurilor · Asistent de excursie · Plan generat dintr-o singură frază.

**Angajament arhitectural luat de pe acum**, ca interfața să nu se modeleze după un singur
furnizor: se definește un port intern `AiService`, iar furnizorul stă în spatele lui,
provider-agnostic (de exemplu prin OpenRouter). Scanarea bonurilor și asistentul sunt
consumatori ai acestui port, **niciodată ai SDK-ului unui furnizor anume.**

### Niciodată

- **Chat.** Grupul are deja WhatsApp și nu îl va părăsi. Un tab de chat pe care nu îl folosește
  nimeni face întreaga aplicație să pară abandonată, iar munca este surprinzător de mare (stare
  de citire, notificări, moderare, atașamente) pentru o funcție al cărei caz cel mai bun este
  paritatea cu ceva gratuit. În loc de asta: link către conversația de WhatsApp.
- **Evaluarea dificultății traseelor.** „Traseul este prea dificil pentru grupul vostru”
  înseamnă că aplicația face o afirmație de siguranță despre un munte, pe date pe care nu le
  controlează. Datele despre trasee, dacă vor exista vreodată, rămân **pur descriptive**
  (lungime, diferență de nivel, durată), fără nicio evaluare a capacității grupului.

### Amânate până când decizia D1 se schimbă

Monetizare (abonament, Group Pass) · integrări de rezervare și afiliere (Booking, Airbnb) ·
orice deschidere către utilizatori din afara cercului propriu.

---

## 18. Glosar

Codul, schema și API-ul sunt în engleză; interfața și acest document sunt în română.
Identificatorii în limbi amestecate (`cheltuiala.getParticipanti()`) îmbătrânesc prost, se bat
cu convențiile oricărei librării și fac codul ostil oricui altcuiva îl atinge vreodată,
inclusiv asistenței AI.

| Domeniu (engleză, în cod) | Interfață (română) |
|---|---|
| `Trip` | Excursie |
| `Participant` | Participant |
| `UserAccount` | Cont |
| `Membership` / `role` | Rol |
| `Invite` | Invitație |
| `Expense` | Cheltuială |
| `ExpenseShare` | Cotă / parte |
| `Settlement` | Plată / decontare |
| `Balance` | Sold |
| `SuggestedTransfer` | Transfer sugerat |
| `ShoppingItem` | Produs de cumpărat |
| `Task` | Responsabilitate |
| `minor units` | unități minime (subdiviziunea leului) |

---

## 19. Stack tehnic (referință)

| Strat | Tehnologie |
|---|---|
| Limbaj | Java 25 (LTS) |
| Framework | Spring Boot 4.1 (baza de rulare: Java 17+; atenție la redenumirea starter-elor, `spring-boot-starter-webmvc`) |
| Build | Gradle, Kotlin DSL |
| Bază de date | PostgreSQL 17+ |
| Migrări | Flyway, SQL versionat, `ddl-auto: validate` — Hibernate nu deține niciodată schema |
| Persistență | Spring Data JPA pentru CRUD; `JdbcClient` + SQL scris de mână pentru solduri și decontare |
| Sesiuni | Spring Session JDBC |
| Contract | springdoc-openapi → client TypeScript generat |
| Teste | JUnit 5, **jqwik** (proprietăți), Testcontainers, ArchUnit, Playwright |
| Frontend | React 19.2, Vite, React Router, TanStack Query, Tailwind v4, Radix, React Hook Form, `vite-plugin-pwa` |
| Infrastructură | Docker Compose, Caddy, VPS unic |
| CI/CD | GitHub Actions → GHCR |
| Erori | Sentry |
