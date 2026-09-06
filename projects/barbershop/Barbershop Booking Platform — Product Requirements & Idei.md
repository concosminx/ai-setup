# Barbershop Booking Platform

> Sistem web pentru gestionarea programărilor unei frizerii / unui barbershop.
>
> Obiectivul inițial este să construim un MVP simplu, rapid și ușor de folosit, dar cu o arhitectură care să permită extinderea ulterioară către un produs SaaS multi-frizerie.

---

# 1. Viziunea produsului

Aplicația trebuie să rezolve problema principală:

> **Clientul trebuie să poată face o programare în cât mai puțini pași, fără să fie obligat să instaleze o aplicație mobilă.**

Experiența principală este web/mobile-web:

```text
Client
  │
  ▼
Link către pagina frizeriei
  │
  ▼
Alege serviciul
  │
  ▼
Alege barberul
  │
  ▼
Alege data și ora
  │
  ▼
Introduce numele + telefonul
  │
  ▼
Confirmă
  │
  ▼
Programare creată
```

Aplicația mobilă nativă NU este obligatorie pentru MVP.

---

# 2. Tipuri de utilizatori

## Client

Poate:

- vedea serviciile;
- vedea barberii;
- vedea disponibilitatea;
- crea o programare;
- anula o programare;
- modifica o programare;
- primi notificări;
- vedea eventual istoricul;
- vedea eventual un sistem de fidelizare.

Contul de client poate fi:

- inexistent;
- opțional;
- obligatoriu.

Decizia trebuie luată în etapa de requirements.

---

## Barber / Angajat

Poate:

- vedea propriul calendar;
- vedea programările;
- confirma / modifica / anula programări;
- marca o programare ca `COMPLETED`;
- marca un client ca `NO_SHOW`;
- vedea informații despre client;
- bloca anumite intervale;
- seta eventual pauze / indisponibilități.

---

## Manager

Poate administra:

- barberii;
- programul;
- serviciile;
- programările;
- clienții;
- zilele libere;
- locația;
- notificările;
- rapoartele.

---

## Owner / Administrator

Poate administra întreaga frizerie:

- utilizatori;
- roluri;
- locații;
- servicii;
- angajați;
- configurări;
- rapoarte;
- clienți;
- programări.

---

## Super Admin

Necesită doar dacă produsul devine multi-tenant / SaaS.

Poate administra:

- frizerii;
- conturi;
- abonamente;
- configurări globale;
- eventual billing;
- sistemul în ansamblu.

---

# 3. Model de business

Există două direcții posibile.

## Varianta A — Single Barbershop

Aplicația este destinată unei singure frizerii.

```text
Application
    │
    └── Barbershop
          ├── Locations
          ├── Barbers
          ├── Services
          ├── Clients
          └── Appointments
```

Avantaje:

- simplu;
- bun pentru MVP;
- mai ușor de implementat;
- potrivit pentru învățarea Spring Boot.

---

## Varianta B — SaaS / Multi-tenant

Mai multe frizerii folosesc aceeași platformă.

```text
Platform
│
├── Barbershop A
│   ├── Locations
│   ├── Barbers
│   ├── Services
│   ├── Clients
│   └── Appointments
│
├── Barbershop B
│   ├── Locations
│   ├── Barbers
│   ├── Services
│   ├── Clients
│   └── Appointments
│
└── Barbershop C
```

Avantaje:

- produs comercial posibil;
- proiect tehnic mai interesant;
- posibilitate de subscription;
- arhitectură mai apropiată de un produs real.

Recomandare:

> MVP-ul poate începe ca single-tenant, dar modelul de date și serviciile trebuie proiectate astfel încât trecerea la multi-tenant să fie posibilă.

---

# 4. Client Journey

## Programare nouă

Flux recomandat:

```text
Landing page
    ↓
Serviciu
    ↓
Barber
    ↓
Data
    ↓
Ora
    ↓
Date client
    ↓
Confirmare
    ↓
Programare
```

Alternativ:

```text
Serviciu
    ↓
"Orice barber disponibil"
    ↓
Data
    ↓
Ora
    ↓
Date client
```

---

# 5. Servicii

Un serviciu poate avea:

```text
Service
├── name
├── description
├── duration
├── price
├── active
└── displayOrder
```

Exemple:

| Serviciu | Durată | Preț |
|---|---:|---:|
| Tuns | 30 min | 50 lei |
| Tuns + barbă | 45 min | 70 lei |
| Barbă | 20 min | 30 lei |
| Tuns copii | 30 min | 40 lei |

Întrebări de decis:

- Prețul este fix?
- Durata este fixă?
- Poate fiecare barber să aibă alt preț?
- Poate fiecare barber să aibă altă durată?
- Există servicii disponibile doar pentru anumiți barberi?
- Există servicii inactive?

---

# 6. Barberi

Un barber poate avea:

```text
Barber
├── name
├── phone
├── email
├── photo
├── description
├── active
└── services
```

Trebuie decis:

- poate un barber să ofere doar anumite servicii?
- are fiecare barber program propriu?
- poate avea zile libere?
- poate avea concediu?
- poate bloca manual intervale?
- poate accepta automat programări?

---

# 7. Programul barberului

Trebuie să putem reprezenta:

```text
Luni       09:00 - 17:00
Marți      09:00 - 17:00
Miercuri   LIBER
Joi        12:00 - 20:00
Vineri     09:00 - 17:00
Sâmbătă    10:00 - 15:00
Duminică   LIBER
```

Dar și:

- pauză de masă;
- concediu;
- zi liberă;
- sărbătoare;
- program special;
- indisponibilitate temporară.

Exemplu:

```text
Program normal:
09:00 - 17:00

Pauză:
13:00 - 14:00

Concediu:
10.09.2026 - 15.09.2026
```

---

# 8. Sloturile disponibile

Sistemul trebuie să calculeze automat disponibilitatea.

Exemplu:

```text
Barber:
09:00 - 17:00

Serviciu:
45 minute

Programări existente:
10:00 - 10:45
13:00 - 13:45
```

Sistemul poate returna:

```text
09:00  disponibil
09:45  indisponibil
10:00  indisponibil
10:45  disponibil
11:30  disponibil
12:15  disponibil
13:00  indisponibil
13:45  disponibil
...
```

Disponibilitatea trebuie să țină cont de:

- programul barberului;
- durata serviciului;
- programările existente;
- pauze;
- concedii;
- zile libere;
- eventual timpul de buffer între clienți.

---

# 9. Programări

Entitatea principală:

```text
Appointment
├── id
├── client
├── barber
├── service
├── location
├── startTime
├── endTime
├── status
├── notes
├── createdAt
└── updatedAt
```

Statusuri posibile:

```text
PENDING
CONFIRMED
CANCELLED_BY_CLIENT
CANCELLED_BY_BARBER
NO_SHOW
COMPLETED
```

Trebuie decis dacă avem:

```text
PENDING
```

sau programarea este direct:

```text
CONFIRMED
```

---

# 10. Prevenirea double booking

Aceasta este o problemă critică.

Nu este suficient ca frontend-ul să spună:

> "10:00 este disponibil."

Pot exista doi clienți care apasă simultan.

Backend-ul trebuie să verifice din nou disponibilitatea în momentul creării programării.

Exemplu:

```text
Client A → 10:00
Client B → 10:00

       ↓

Backend

Client A → SUCCESS
Client B → CONFLICT
```

Trebuie tratată corespunzător concurența.

---

# 11. Client

Date minime:

```text
Client
├── id
├── name
├── phone
├── email?
├── createdAt
└── notes?
```

Nu este obligatoriu ca un client să aibă cont.

Varianta simplă:

```text
Nume
Telefon
```

Clientul poate primi un link securizat pentru gestionarea programării.

---

# 12. Cont client

Opțional.

Dacă există cont:

```text
Client Account
├── login
├── profile
├── upcoming appointments
├── appointment history
├── favorite barber
└── loyalty
```

Pentru MVP:

> Contul clientului poate fi amânat.

---

# 13. Anulare / reprogramare

Clientul trebuie eventual să poată:

```text
Anulează programarea
```

și:

```text
Modifică programarea
```

Regulă posibilă:

```text
Cu mai mult de 2 ore înainte
→ poate modifica / anula

Cu mai puțin de 2 ore
→ trebuie contactată frizeria
```

Regula trebuie să fie configurabilă.

---

# 14. No-show

O programare poate fi marcată:

```text
NO_SHOW
```

Acest lucru permite ulterior:

- statistici;
- identificarea clienților problematici;
- reguli de confirmare;
- eventual solicitarea unui avans;
- limitarea programărilor.

Exemplu:

```text
Client
├── 20 programări
├── 17 completed
├── 2 cancelled
└── 1 no-show
```

---

# 15. Notificări

Nu trebuie să depindem de WhatsApp.

Canale:

```text
Notification
│
├── SMS
├── Email
├── Push
└── WhatsApp (opțional ulterior)
```

Pentru MVP:

> SMS + Email

sunt suficiente.

Push poate fi adăugat dacă vom avea PWA / aplicație mobilă.

---

# 16. Notificări recomandate

## La creare

```text
Programarea ta a fost confirmată.

Data: 10 septembrie
Ora: 14:30
Barber: Andrei
Serviciu: Tuns + Barbă
```

## Reminder

Cu 24h înainte:

```text
Reminder:
Ai o programare mâine la 14:30.
```

Eventual încă unul:

```text
Cu 2 ore înainte.
```

## Anulare

```text
Programarea ta din 10 septembrie, ora 14:30,
a fost anulată.
```

## Modificare

```text
Programarea ta a fost modificată.

Noua oră: 15:00
```

---

# 17. Arhitectura notificărilor

Business logic-ul nu trebuie să depindă de un provider concret.

Conceptual:

```text
Appointment Service
        │
        ▼
 Notification Service
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
  SMS Email Push WhatsApp
```

Abstracție:

```java
interface NotificationProvider {
    void send(Notification notification);
}
```

Implementări:

```text
SmsNotificationProvider
EmailNotificationProvider
PushNotificationProvider
WhatsAppNotificationProvider
```

Astfel putem schimba providerul fără să modificăm logica de programări.

---

# 18. Admin Panel

Adminul trebuie să aibă un dashboard.

Exemplu:

```text
---------------------------------------------
Dashboard
---------------------------------------------

Astăzi

Programări:       32
Finalizate:       25
Anulate:           3
No-show:           2

---------------------------------------------

Programări următoare

09:00  Andrei  - Tuns
09:30  Mihai   - Tuns + Barbă
10:00  Alex    - Barbă
...
```

---

# 19. Calendar

Adminul / barberul trebuie să poată vedea:

```text
        Luni   Marți   Miercuri   Joi   Vineri

Andrei   ███     ███      -        ███    ███
Mihai    ███     -        ███      ███    ███
Alex     ███     ███      ███       -     ███
```

Posibilități:

- zi;
- săptămână;
- lună;
- filtrare după barber;
- filtrare după locație.

---

# 20. Programare manuală

Angajatul trebuie să poată crea o programare pentru client.

Exemplu:

```text
Client:
Ion Popescu

Serviciu:
Tuns

Barber:
Andrei

Data:
10.09.2026

Ora:
15:30
```

Acest lucru este important deoarece nu toate programările vin din website.

---

# 21. Clienți / CRM

Adminul poate vedea:

```text
Client
├── date contact
├── programări
├── ultima vizită
├── număr vizite
├── no-show count
├── total cheltuit
└── notes
```

Exemplu:

```text
Ion Popescu

Vizite:        18
No-show:        1
Ultima vizită: 02.09.2026
Total:          1.080 lei
```

---

# 22. Fidelizare

Posibil sistem:

```text
Client
   ↓
Programări finalizate
   ↓
Puncte
   ↓
Beneficii
```

Exemple:

- fiecare vizită = 1 punct;
- 10 vizite = serviciu gratuit;
- discount după anumite praguri;
- client fidel al lunii.

Această funcționalitate poate fi lăsată pentru o versiune ulterioară.

---

# 23. Review-uri

Opțional.

După o programare `COMPLETED`:

```text
Cum a fost experiența?

★★★★★

Comentariu:
______________
```

Review-ul poate fi asociat cu:

- frizeria;
- locația;
- barberul;
- serviciul.

Trebuie decis dacă review-urile sunt publice.

---

# 24. Plăți

MVP:

> plata se face la frizerie.

Ulterior:

```text
Online booking
       ↓
Payment
       ↓
Appointment confirmed
```

Posibile funcționalități:

- avans;
- plată integrală;
- card;
- Apple Pay / Google Pay;
- refund.

Plățile online nu sunt necesare pentru prima versiune.

---

# 25. Protecția împotriva no-show-urilor

Ulterior putem introduce:

```text
Client cu multe NO_SHOW
        ↓
Solicită confirmare
        ↓
SMS / cod
        ↓
eventual avans
```

Exemplu de regulă:

```text
NO_SHOW >= 3
    ↓
booking requires deposit
```

---

# 26. Rapoarte

Dashboard pentru owner:

## Programări

```text
Programări / zi
Programări / săptămână
Programări / lună
```

## Venituri

```text
Venit zilnic
Venit lunar
Venit per barber
Venit per serviciu
```

## Ocupare

```text
Andrei    82%
Mihai     71%
Alex      64%
```

## Clienți

```text
Clienți noi
Clienți recurenți
Clienți pierduți
No-show rate
```

---

# 27. Multi-location

Dacă o frizerie are mai multe locații:

```text
Barbershop
│
├── Location A
│   ├── Barbers
│   └── Services
│
└── Location B
    ├── Barbers
    └── Services
```

Clientul trebuie să aleagă:

```text
Locație
↓
Barber
↓
Serviciu
↓
Slot
```

Trebuie decis dacă:

- serviciile diferă între locații;
- prețurile diferă;
- barberii lucrează în mai multe locații.

---

# 28. Configurarea frizeriei

Adminul trebuie să poată configura:

```text
Barbershop
├── name
├── logo
├── phone
├── email
├── address
├── openingHours
├── bookingRules
├── cancellationRules
└── notificationSettings
```

---

# 29. Reguli de booking

Configurabile:

```text
Minimum booking notice:
2 ore

Maximum booking horizon:
30 zile

Cancellation deadline:
2 ore

Slot interval:
15 minute

Maximum appointments/day:
opțional
```

Exemplu:

```text
Astăzi 14:00

Clientul nu poate face programare la:
14:15

dacă minimum notice = 2h.

Prima disponibilitate:
16:00
```

---

# 30. Slot interval vs. durata serviciului

Trebuie separat:

```text
Service duration:
45 minute

Slot interval:
15 minute
```

Asta înseamnă că putem avea:

```text
09:00
09:15
09:30
09:45
10:00
...
```

dar o programare de 45 minute ocupă:

```text
09:00 - 09:45
```

---

# 31. Buffer între programări

Opțional:

```text
Tuns:
30 min

Buffer:
10 min
```

Rezultă:

```text
09:00 - 09:30  client
09:30 - 09:40  buffer
09:40           următor client
```

Poate fi configurabil per serviciu.

---

# 32. Link pentru gestionarea programării

În loc să obligăm clientul să aibă cont:

```text
SMS
  ↓
"Gestionează programarea"
  ↓
secure booking link
```

Clientul poate:

- vedea programarea;
- anula;
- reprograma.

Link-ul trebuie să fie securizat și greu de ghicit.

---

# 33. Securitate

Backend:

```text
Spring Security
```

Autentificare pentru staff:

```text
JWT / Session
```

Roluri:

```text
ROLE_OWNER
ROLE_MANAGER
ROLE_BARBER
ROLE_ADMIN
```

Clientul poate utiliza:

- token temporar;
- magic link;
- eventual OTP.

---

# 34. API

Exemple:

```text
GET    /api/services
GET    /api/barbers
GET    /api/availability
POST   /api/appointments

GET    /api/appointments/{id}
PATCH  /api/appointments/{id}
DELETE /api/appointments/{id}
```

Admin:

```text
GET    /api/admin/appointments
POST   /api/admin/appointments
GET    /api/admin/clients
GET    /api/admin/barbers
POST   /api/admin/barbers
PUT    /api/admin/barbers/{id}
```

---

# 35. Model de date — prima schiță

Posibile entități:

```text
Barbershop
Location
User
Role
Barber
Client
Service
BarberService
WorkingSchedule
ScheduleException
Appointment
Notification
NotificationTemplate
Review
LoyaltyAccount
LoyaltyTransaction
Payment
```

Nu toate trebuie implementate în MVP.

---

# 36. MVP propus

Prima versiune ar trebui să conțină DOAR:

## Client

- pagină publică;
- servicii;
- barberi;
- disponibilitate;
- creare programare;
- confirmare;
- anulare;
- reprogramare.

## Staff

- login;
- calendar;
- programări;
- creare programare manuală;
- modificare programare;
- anulare;
- `COMPLETED`;
- `NO_SHOW`.

## Admin

- CRUD servicii;
- CRUD barberi;
- program barber;
- configurare frizerie;
- clienți.

## Notifications

- SMS confirmare;
- SMS reminder;
- email opțional.

---

# 37. Ce NU intră în primul MVP

Pentru a nu transforma proiectul într-un monstru:

- aplicație iOS;
- aplicație Android;
- WhatsApp;
- marketplace;
- plăți online;
- abonamente;
- loyalty;
- review-uri;
- AI;
- marketing automation;
- facturare;
- contabilitate;
- multi-tenant complet.

Acestea pot deveni versiuni ulterioare.

---

# 38. Versiunea 2

După MVP:

```text
CRM
├── customer history
├── statistics
└── customer segmentation

Notifications
├── SMS
├── Email
└── WhatsApp

Loyalty
├── points
└── rewards

Payments
└── deposits

Reports
├── revenue
├── occupancy
└── barber performance
```

---

# 39. Versiunea 3 — SaaS

Dacă produsul devine comercial:

```text
Platform
│
├── Tenant management
├── Subscription
├── Billing
├── Multi-location
├── Custom branding
├── Custom booking page
└── Analytics
```

Fiecare frizerie poate avea:

```text
myplatform.ro/barbershop-x
```

sau propriul domeniu.

---

# 40. Principii de UX

## Client

Trebuie să fie:

- mobile-first;
- rapid;
- fără cont obligatoriu;
- fără pași inutili;
- fără reclame;
- foarte clar.

Obiectiv:

> Programarea să poată fi făcută în aproximativ 30–60 secunde.

## Staff

Trebuie să fie:

- calendar-centric;
- rapid;
- ușor de folosit de pe desktop/tabletă;
- programările să fie vizibile imediat.

---

# 41. Principii tehnice

Backend:

```text
Java
Spring Boot
Spring Security
Spring Data JPA
Hibernate
PostgreSQL
```

Posibil:

```text
Flyway
Docker
JUnit
Mockito
Testcontainers
OpenAPI / Swagger
```

Frontend-ul poate fi:

```text
React / Next.js
```

sau, pentru un MVP foarte simplu:

```text
Thymeleaf
```

---

# 42. Arhitectură recomandată

Nu începem cu microservicii.

Recomandare:

> **Modular Monolith**

Exemplu:

```text
Spring Boot
│
├── authentication
├── users
├── barbershop
├── locations
├── services
├── schedules
├── appointments
├── clients
├── notifications
└── reporting
```

Ulterior anumite module pot deveni servicii separate.

---

# 43. Event-driven notifications

Programarea:

```text
AppointmentCreated
        │
        ▼
Notification Handler
        │
        ├── Confirmation SMS
        └── Confirmation Email
```

Reminder:

```text
Appointment
     │
     ▼
Scheduled Job
     │
     ▼
Reminder
```

Acest model ne permite să schimbăm providerii fără să modificăm `AppointmentService`.

---

# 44. Întrebări care trebuie rezolvate înainte de implementare

## Business

- [ ] Single barbershop sau SaaS?
- [ ] Una sau mai multe locații?
- [ ] Cine sunt utilizatorii?
- [ ] Clientul are cont?
- [ ] Clientul poate programa fără cont?

## Booking

- [ ] Clientul alege barberul?
- [ ] Există opțiunea "orice barber"?
- [ ] Serviciile au durată?
- [ ] Serviciile au preț?
- [ ] Barberii pot avea servicii diferite?
- [ ] Există buffer?
- [ ] Există pauze?
- [ ] Care este regula de anulare?
- [ ] Care este regula de reprogramare?

## Staff

- [ ] Barber
- [ ] Manager
- [ ] Owner
- [ ] Super Admin

## Notifications

- [ ] SMS
- [ ] Email
- [ ] Push
- [ ] WhatsApp ulterior

## Business rules

- [ ] No-show
- [ ] Confirmare
- [ ] Avans
- [ ] Limite de booking
- [ ] Program special
- [ ] Concedii

## Extra

- [ ] CRM
- [ ] Loyalty
- [ ] Review-uri
- [ ] Payments
- [ ] Reports
- [ ] Marketing

---

# 45. Întrebarea centrală

Înainte de orice cod trebuie să putem completa propoziția:

> **"Aplicația noastră ajută ______ să ______ fără ______."**

Exemplu:

> Aplicația noastră ajută **barbershop-urile mici și medii să-și gestioneze programările online fără să depindă de telefon, WhatsApp sau agende fizice.**

Această propoziție trebuie stabilită înainte să definim arhitectura finală.

---

# 46. Ordinea în care vom lua deciziile

Nu implementăm încă.

Ordinea recomandată:

```text
1. Business model
       ↓
2. User roles
       ↓
3. Client journey
       ↓
4. Booking rules
       ↓
5. Staff workflow
       ↓
6. Notifications
       ↓
7. Admin functionality
       ↓
8. MVP scope
       ↓
9. Domain model
       ↓
10. Database
       ↓
11. API
       ↓
12. Architecture
       ↓
13. Implementation
```

---

# 47. Obiectivul proiectului

La final vrem să avem un sistem în care:

```text
Client
  │
  ├── vede frizeria
  ├── vede serviciile
  ├── alege barberul
  ├── vede sloturile reale
  ├── face programarea
  └── primește confirmare/reminder
             │
             ▼
        Spring Boot
             │
       ┌─────┴─────┐
       │           │
   PostgreSQL   Notifications
       │
       ▼
     Staff
       │
       ├── calendar
       ├── clienți
       ├── programări
       └── administrare
```

Principiul principal:

> **Mai întâi construim cel mai simplu flux care rezolvă problema reală. Apoi adăugăm funcționalități care produc valoare, nu doar complexitate.**