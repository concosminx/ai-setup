# Plan de implementare – Registratură electronică + BPM + AI

> **Statut:** plan aprobat pentru MVP. Acest document este **sursa unică de adevăr** pentru
> arhitectură, scop și etapizare. Este scris pentru a fi executat de un agent AI împreună cu
> un dezvoltator. Nu depinde de niciun alt fișier.
>
> **Ce NU acoperă acest pas:** produsul nu se construiește încă. Aici se stabilește doar planul.

---

## 1. Context și obiectiv

Se construiește o aplicație de **registratură electronică** pentru instituții publice, cu
**motor de workflow (BPMN)** și, în faze ulterioare, componente de **inteligență artificială**
(OCR, clasificare, asistență la redactarea răspunsurilor).

- **Tip proiect:** produs real, dezvoltat incremental, pornind de la un MVP redus.
- **Licență:** AGPL-3.0 (permite dual licensing comercial ulterior). Flowable OSS este
  Apache-2.0, compatibil.
- **Mod de construcție:** deocamdată speculativ, pe baza legislației și a practicii de
  registratură (Legea 201/2003 privind registratura, OG 27/2002 privind petițiile, Legea
  544/2001 privind accesul la informații publice, normele de arhivare). Există un **punct de
  validare cu primul client real** imediat după MVP, înainte de a investi în Faza 2.
- **Instituție-țintă de referință:** primărie mică / comună (5–20 utilizatori, sute–mii de
  documente/an). Modelul de date se pregătește pentru instituții mai mari, dar MVP-ul nu le
  țintește.
- **Echipă:** un singur dezvoltator, part-time, experimentat cu tot stack-ul (fără spike-uri
  de învățare, doar un spike de validare tehnologică pentru Flowable 8).
- **Livrare:** **single-tenant, instalabilă** – o instanță separată per client, pornită cu
  `docker compose` pe un singur host. Fiecare instalare își definește propriile registre și
  fluxuri din back office; codul sursă rămâne unic.

Pentru un cititor ne-tehnic: aplicația înregistrează documentele care intră și ies din
instituție, le dă un număr oficial, le trimite pe un traseu de rezolvare (repartizare →
redactare răspuns → aprobare → expediere → închidere), urmărește termenele legale și
păstrează un istoric complet, verificabil, al fiecărei acțiuni.

---

## 2. Principii arhitecturale

Aceste principii sunt obligatorii și guvernează toate deciziile de implementare.

```text
BPMN definit în back office
        ↓ import + validare + versionare
   Flowable Engine
        ↓
   WorkflowService (adapter propriu)
        ↓
   REST API orientat pe domeniu
        ↓
   OpenUI5
```

1. **Aplicația controlează experiența și business-ul.** Flowable este motorul de workflow –
   nu aplicația de registratură și nu UI-ul final.
2. **Modular monolith** Spring Boot: un singur proces, pachete pe domeniu, granițe verificate
   cu ArchUnit. Modulele pot fi extrase ulterior, dar nu se începe cu microservicii.
3. **Flowable este ascuns după un adapter** `WorkflowService` cu operații de domeniu
   (`startProcess`, `getTasks`, `getTask`, `claimTask`, `completeTask`, `delegateTask`,
   `cancelProcess`, `getTimeline`). Nicio altă parte a codului nu atinge `RuntimeService`,
   `TaskService` etc. Adapterul este și **plasa de siguranță**: un downgrade la Spring Boot
   3.4 + Flowable 7 ar atinge doar modulul `workflow`.
4. **User Task ≠ business logic.** Flowable spune „task-ul X este activ pentru utilizatorul
   Y”. Aplicația decide ce date se afișează, ce formular apare, ce validări există, ce acțiuni
   sunt permise, ce business logic se execută, ce audit se înregistrează.
5. **Mapare `taskDefinitionKey` → UI.** `taskDefinitionKey` este identificatorul stabil între
   proces și interfață. Maparea `taskDefinitionKey → view/handler/form schema` este **fixă în
   cod** în MVP (configurabilă din back office abia în Faza 2).
6. **Task Action API cu acțiuni explicite.** UI-ul nu trimite `completeTask`, ci acțiuni:
   `ASSIGN`, `FORWARD`, `SEND_BACK`, `SUBMIT_FOR_APPROVAL`, `SAVE`, `APPROVE`, `REJECT`,
   `CLAIM`, `DELEGATE`, `ESCALATE`, `MARK_SENT`, `CANCEL`. Backend-ul validează acțiunea
   înainte să atingă Flowable.
7. **Task Handler Registry.** `taskDefinitionKey → handler`. Fiecare handler execută, în
   ordine: validare → autorizare → business logic → persistare → audit → setare variabile
   Flowable → complete task.
8. **Logică simplă vs. logică importantă.** Condițiile din BPMN sunt expresii pe variabile
   (`${approved == true}`), niciodată script arbitrar. Logica importantă (OCR, AI, semnătură,
   REST extern, generare document) este `Service Task` + `JavaDelegate` tipizat și testabil.
9. **Script/Groovy din BPMN importat = cod executabil.** În MVP este **complet interzis** la
   import. Revine în Faza 2 ca API controlat cu sandbox (fără acces la `ApplicationContext`).
10. **Separă datele business de datele workflow.** Tabele proprii
    (`workflow_definition`, `workflow_instance`, `workflow_task`). Aplicația nu depinde de
    schema internă Flowable.
11. **Business key** leagă Flowable de obiectul business: `PETITII-2026-000001`.
12. **Timeline propriu**, nu history-ul Flowable expus direct. Combină Flowable history +
    auditul aplicației + evenimente business (+ acțiuni AI în Faza 2).
13. **Audit separat de Flowable History.** Append-only. Câmpuri: cine, când, acțiune, obiect
    business, valoare veche, valoare nouă, `correlationId`, `workflow_instance`, `task`.
14. **AI nu decide direct workflow-ul** (Faza 2+): AI → recomandare + `confidence` → regulă
    business / utilizator → tranziție Flowable. Praguri configurabile pe tip de document.
15. **Versionarea proceselor.** Un proces activ nu se modifică in-place. Instanțele deja
    pornite rămân pe versiunea cu care au început; cele noi folosesc versiunea activă.
16. **Metadata proprie în BPMN** (`registerKey`, `taskDefinitionKey`, `businessType`) este
    contractul dintre designerul de proces și aplicație.
17. **Fără ID-uri hardcodate în BPMN.** Totul prin variabile de proces.
18. **Event-driven doar unde are sens** (Faza 2: RabbitMQ pentru notificare, indexare, OCR,
    AI). Sincron pentru ce trebuie finalizat înainte de continuarea procesului.
19. **Idempotency** pe handlerele importante (idempotency key + verificări de stare).
20. **Concurrency:** claim → verificare stare/versiune → complete; optimistic locking pe
    operații critice.
21. **API-ul final nu seamănă cu Flowable.** `GET /api/inbox/tasks`, nu
    `GET /flowable/runtime/tasks`.

**Principiul de păstrat:**

```text
Flowable  = WHEN / WHERE / NEXT
Java      = HOW
OpenUI5   = WHAT USER SEES
AI        = WHAT IS PROBABLE / RECOMMENDED
Database  = WHAT IS TRUE
Audit     = WHAT HAPPENED
```

---

## 3. Registru de decizii

| # | Decizie | Motivare |
|---|---------|----------|
| D1 | Produs real, MVP-first | Cerere de piață pentru comune fără soluție decentă; risc redus prin livrare incrementală |
| D2 | Licență AGPL-3.0 | Software de infrastructură publică; păstrează codul deschis; permite dual licensing |
| D3 | Construcție speculativă pe reglementări + punct de validare cu primul client | Nu există încă un client; se evită supra-investiția înainte de feedback real |
| D4 | Profil de referință: primărie mică / comună | Cel mai simplu circuit real; model de date pregătit pentru primării mai mari |
| D5 | Single-tenant, instalabilă, `docker compose` pe un host | Comunele cer adesea date „la ele”; elimină complexitatea multi-tenancy din MVP |
| D6 | Java 21, Spring Boot 4, Flowable 8 OSS, Gradle, monorepo | Alegere deliberată; Flowable 8 este singurul compatibil cu Spring Boot 4 |
| D7 | OpenUI5 în JavaScript, build cu UI5 Tooling, servit static de Spring Boot | Fără server Node în producție; livrare simplă |
| D8 | Flowable ascuns după `WorkflowService` = strat de izolare și plan B | Downgrade la SB 3.4 + Flowable 7 ar atinge doar modulul `workflow` |
| D9 | Registre definibile din back office (parte standard + schemă atribute custom) | Un singur cod sursă, fluxuri și registre per client, fără fork |
| D10 | Numerotare per registru, reset anual, format configurabil | Practica de registratură; fiecare registru are contorul lui |
| D11 | Legătură registru → workflow opțională | Unele registre sunt pură evidență; au un ciclu de viață minim încorporat |
| D12 | Vocabular închis de pași de workflow (handlere în cod) | „Un cod, fluxuri per client” fără a construi o platformă low-code |
| D13 | Proces MVP în 6 noduri | Acoperă lanțul real fără clasificare AI și fără semnătură electronică reală |
| D14 | Import BPMN în MVP: validare + deploy + versionare + activare separată | Definițiile de proces sunt date gestionate la runtime, nu artefacte din repo |
| D15 | Script/Groovy interzis la import în MVP | Cod executabil din surse externe; sandbox abia în Faza 2 |
| D16 | Autentificare JWT stateless | Cerință explicită; UI5 trimite `Authorization: Bearer` |
| D17 | Rapoarte cu JasperReports (PDF + XLSX) | Cerință explicită; coloane custom adăugate programatic |
| D18 | Notificări e-mail sincrone (`JavaMailSender`), fără coadă în MVP | Un inbox pe care nu-l vede nimeni e inutil; coada vine în Faza 2 |
| D19 | Semnătură electronică = placeholder (upload PDF semnat + marcaj) | Integrarea reală este Faza 2 |
| D20 | „Expediere” = marcaj `sent` + canal + dată consemnate manual | Suficient pentru MVP |
| D21 | GDPR MVP: temei legal pe registru + „anonimizează petent” manual + audit mutații | Corect pentru un produs public real; retenția automată vine în Faza 2 |
| D22 | RO singura limbă, dar totul în resource bundles | A doua limbă devine aditivă |
| D23 | 8 etape M0–M7, secvențiale, cu estimări orientative | Ritm part-time variabil; cifrele sunt indicative |

---

## 4. Scope MVP

### În MVP

- Registre definibile din back office (parte standard + schemă atribute custom), numerotare
  per registru cu reset anual, reguli de termen (`due_date`).
- Înregistrare document(e) cu formular dinamic generat din schema registrului; petent
  (persoană/organizație); documente atașate în MinIO.
- Import / validare / deploy / versionare BPMN din back office; activare per registru.
- `WorkflowService` adapter + tabele proprii de workflow.
- Pornire proces la înregistrare, cu business key.
- Inbox OpenUI5 + Task Detail; renderer de form schema; view custom pentru „Redactare
  răspuns”.
- Task Action API + Task Handler Registry; catalog fix de pași (vezi §7).
- Audit propriu (append-only) + Workflow Timeline propriu.
- Autentificare JWT cu utilizatori locali + 5 roluri; `department` minim pentru rutare; flag
  `confidential` pe înregistrare.
- Căutare de bază pe DB (număr, petent, dată, status, text în descriere).
- SLA: `due_date` calculat; timere Flowable (reminder + escaladare) și/sau job programat;
  Inbox arată timpul rămas.
- Notificări e-mail sincrone (repartizare, cerere de aprobare, send-back, reminder,
  escaladare).
- Rapoarte JasperReports: export registru (PDF + XLSX) pe interval, „termene depășite”,
  contoare pe home cu drill-down.
- Ciclu de viață minim pentru registrele fără workflow:
  `ÎNREGISTRAT → LANSAT → REZOLVAT | RESPINS | ANULAT`.
- Observabilitate: `correlationId`, Actuator health, listă job-uri Flowable eșuate în back
  office.
- Împachetare: `docker compose` + script de instalare + import seed (registre-exemplu).

### În afara MVP (Faza 2+)

Clasificare AI, OCR, extragere metadate, RAG / draft automat, OpenSearch, semnătură
electronică reală, ingestie e-mail automată, portal cetățean, RabbitMQ / Outbox /
event-driven, Redis, multi-tenancy, integrare ANAF, LDAP/AD, ACL fin per registru /
compartiment, retenție GDPR automată, audit-de-acces, model hibrid de pas (form schema
configurabil din back office), API de import cu sandbox de script, dashboards avansate,
automatizare UI de testare.

---

## 5. Model de date

Migrări cu **Flyway** (`src/main/resources/db/migration`). Flowable își gestionează propriile
tabele (`flowable.database-schema-update=true` la bootstrap-ul instalării). `custom_data` este
`jsonb` (PostgreSQL 16).

### Business

**`register_definition`** – definiția unui registru
```text
id (uuid, pk)
key (varchar, unique)              -- ex. PETITII, SOLICITARI_544
name, description
number_format (varchar)            -- ex. {YYYY}/{NNNNNN}
number_reset (enum: YEARLY, NEVER)
due_date_rule (jsonb)              -- {type: CALENDAR_DAYS|BUSINESS_DAYS|NONE, value, reminder_pct, escalation_pct}
custom_fields (jsonb)             -- [{name,label,type,required,options?,multi?}]
legal_basis (text)                -- temei legal / scop prelucrare (GDPR)
active_workflow_definition_id (uuid, null, fk workflow_definition)
confidential_default (boolean)
status (enum: ACTIVE, DISABLED)
created_at, created_by, updated_at, updated_by
```
Tipuri de câmp custom suportate: `text` (scurt/lung), `number`, `date`, `boolean`,
`select` (opțiuni fixe, single/multi), `party_ref` (referință persoană/organizație).

**`register_counter`** – contor de numerotare
```text
register_definition_id (uuid, pk)
year (int, pk)
last_value (bigint)
-- alocare cu SELECT ... FOR UPDATE (single-tenant, volum mic)
```

**`register_entry`** – o înregistrare
```text
id (uuid, pk)
register_definition_id (uuid, fk)
number (varchar)                   -- formatat; unic per (register_definition_id)
sequence_value (bigint), sequence_year (int)
entry_date (timestamptz)
due_date (date, null)
due_date_overridden (boolean), due_date_override_reason (text, null)
description (text)
custom_data (jsonb)
requester_party_id (uuid, null, fk party)
department_id (uuid, null, fk department)   -- compartiment responsabil curent
assignee_user_id (uuid, null, fk app_user)  -- responsabil curent
confidential (boolean)
status (enum: REGISTERED, DISPATCHED, IN_PROGRESS, WAITING_APPROVAL,
              RESOLVED, REJECTED, CANCELLED, CLOSED)
resolution (enum, null: RESOLVED, REJECTED, CANCELLED)   -- registre fără workflow
resolution_note (text, null)
workflow_instance_id (uuid, null, fk workflow_instance)
business_key (varchar)             -- ex. PETITII-2026-000001
created_at, created_by, updated_at, updated_by, closed_at
```

**`party`** – petent
```text
id (uuid, pk)
type (enum: PERSON, ORG)
full_name (varchar)
national_id (varchar, null)        -- CNP / CUI, adesea absent
address (text, null), email (varchar, null), phone (varchar, null)
contact_person (varchar, null)     -- pentru ORG
anonymized (boolean), anonymized_at (timestamptz, null), anonymized_by (uuid, null)
created_at, created_by
```

**`document`**
```text
id (uuid, pk)
register_entry_id (uuid, fk)
direction (enum: INCOMING, OUTGOING, INTERNAL)
kind (enum: REQUEST, RESPONSE, ATTACHMENT, SIGNED_RESPONSE)
file_ref (varchar)                 -- cheie MinIO
filename, mime_type, size_bytes, checksum_sha256
signed (boolean), signed_file_ref (varchar, null)
uploaded_by (uuid), uploaded_at (timestamptz)
```
Tipuri permise: PDF, JPG, PNG, DOCX, ODT. Dimensiune max configurabilă în `instance_settings`.
Fără OCR / preview / scanare antivirus în MVP – se prevede un hook gol
`DocumentIntakePipeline` pentru Faza 2.

**`department`**
```text
id (uuid, pk), code (varchar, unique), name, parent_id (uuid, null), active (boolean)
```

**`app_user`**
```text
id (uuid, pk), username (varchar, unique), password_hash (bcrypt)
full_name, email, department_id (uuid, null)
roles (set: ADMIN, REGISTRATOR, SOLUTIONATOR, APROBATOR, VIZUALIZATOR)
enabled (boolean), failed_login_count (int), locked_until (timestamptz, null)
must_change_password (boolean), created_at
```

**`refresh_token`**
```text
id (uuid, pk), user_id (uuid, fk), token_hash (sha-256)
expires_at (timestamptz), revoked (boolean), created_at, user_agent (varchar, null)
```

**`email_log`**
```text
id (uuid, pk), sent_at, to_address, subject, template
register_entry_id (uuid, null), status (enum: SENT, FAILED), error (text, null)
```

**`instance_settings`** – configurare instanță (o singură înregistrare / key-value)
```text
institution_name, institution_address, institution_cui
default_number_format
smtp_host, smtp_port, smtp_user, smtp_from   -- parola din env
max_file_size_bytes, allowed_mime_types
sla_reminder_pct_default, sla_escalation_pct_default
```

### Workflow (proprii – nu schema Flowable)

**`workflow_definition`**
```text
id (uuid, pk)
key (varchar), name
register_key (varchar)             -- registrul pentru care e valabil
version (int)                      -- max(version pt key) + 1 la fiecare import valid
flowable_deployment_id (varchar)
flowable_process_definition_id (varchar)
bpmn_xml_ref (varchar)             -- XML original în MinIO
status (enum: DRAFT, VALIDATED, ACTIVE, RETIRED)
validation_report (jsonb)
created_at, created_by, activated_at, activated_by
```

**`workflow_instance`**
```text
id (uuid, pk)
workflow_definition_id (uuid, fk)
register_entry_id (uuid, fk)
business_type (varchar)            -- = register_key
business_id (uuid)                 -- = register_entry_id
flowable_process_instance_id (varchar)
status (enum: RUNNING, COMPLETED, CANCELLED, ERROR)
started_at, started_by, completed_at
```

**`workflow_task`** – oglinda task-urilor Flowable pentru UI
```text
id (uuid, pk)
workflow_instance_id (uuid, fk)
flowable_task_id (varchar)
task_definition_key (varchar)
name (varchar)
assignee_user_id (uuid, null), candidate_group (varchar, null)
status (enum: OPEN, CLAIMED, COMPLETED, CANCELLED)
due_date (timestamptz, null)
created_at, claimed_at, completed_at, completed_by (uuid, null)
outcome (varchar, null)            -- numele acțiunii cu care s-a finalizat
```
Sincronizare prin `FlowableEventListener` (task created / assigned / completed).

**`audit_event`** – append-only, fără UPDATE/DELETE
```text
id (uuid, pk)
occurred_at (timestamptz)
actor_user_id (uuid, null), actor_username (varchar)
action (varchar)                   -- ex. ENTRY_REGISTERED, TASK_COMPLETED, PARTY_ANONYMIZED
business_object_type (varchar), business_object_id (varchar)
register_entry_id (uuid, null), workflow_instance_id (uuid, null), workflow_task_id (uuid, null)
old_value (jsonb, null), new_value (jsonb, null)
correlation_id (varchar)
ip_address (varchar, null), details (text, null)
```

**`timeline_event`** – materializat pentru UI
```text
id (uuid, pk)
register_entry_id (uuid, fk)
occurred_at (timestamptz)
type (varchar)                     -- REGISTERED, DISPATCHED, TASK_STARTED, TASK_COMPLETED,
                                   -- EMAIL_SENT, SLA_REMINDER, ESCALATION, SIGNED, SENT,
                                   -- RESOLVED, CLOSED, ANONYMIZED
label (varchar), actor (varchar, null), payload (jsonb, null)
```

---

## 6. Contract BPMN

Definit ca document de referință pentru orice proces încărcat în back office.

### Nivel proces

- `process id` (cheia) – unic, `[a-z][a-zA-Z0-9_]*`.
- `extensionElements` cu `<registry:registerKey>` (obligatoriu, trebuie să corespundă unui
  `register_definition` existent) și `<registry:businessType>` (= `registerKey`).
- Exact un `startEvent` (none start), cel puțin un `endEvent`, graf conectat (fără noduri
  orfane).

### Variabile injectate la pornire

```text
registerEntryId (string, uuid)
registerKey (string)
businessKey (= PETITII-2026-000001)
documentIds (list<string>)
requesterName, requesterType (string)
dueDate (string, ISO date)
attr_<numeCâmpCustom>          -- pentru fiecare câmp din custom_fields al registrului
```

### Variabile setate de handlere pe parcurs

```text
assigneeUserId, departmentId          -- după repartizare
responseDocumentId                    -- după redactare
approved (boolean), rejectionReason   -- după aprobare
signedDocumentId                      -- după marcarea răspunsului semnat
sentChannel, sentDate                 -- după expediere
```

### Elemente permise

- `userTask` **doar** cu `taskDefinitionKey ∈` catalog (vezi §7).
- `serviceTask` **doar** cu `flowable:delegateExpression ∈` whitelist:
  `${notificaEmailDelegate}`, `${seteazaTermenDelegate}`, `${inchideInregistrareDelegate}`.
- `exclusiveGateway` cu `conditionExpression` doar de forma `${ident}` (boolean) sau
  `${ident (==|!=|>|<|>=|<=) literal}`.
- `boundaryEvent` cu `timerEventDefinition` (`timeDuration` / `timeDate`) pe `userTask`,
  cu țintă `notificaEmailDelegate` sau un pas de escaladare.

### Elemente interzise (import respins)

`scriptTask`, orice `<*:script>`, `flowable:class`, `flowable:expression` arbitrar,
`manualTask`, `receiveTask`, `businessRuleTask`, `callActivity`, `sendTask`, `userTask` fără
`taskDefinitionKey` din catalog, `delegateExpression` din afara whitelist.

### Pipeline de validare la import

```text
1. XML well-formed
2. Validare față de XSD BPMN 2.0
3. Parse cu Flowable model API: exact un process, start + end, graf conectat
4. Toate userTask au taskDefinitionKey ∈ catalog
5. Toate serviceTask au delegateExpression ∈ whitelist (fără flowable:class)
6. Fără elemente interzise
7. conditionExpression respectă pattern-ul permis
8. registerKey prezent și existent
   ↓ la succes
9. status = VALIDATED, XML în MinIO, deployment Flowable (proces NEactiv),
   workflow_definition version = max+1, validation_report salvat
   ↓ acțiune separată
10. Activare → register_definition.active_workflow_definition_id, status ACTIVE,
    versiunea anterioară RETIRED; instanțele în curs rămân pe versiunea lor
```

---

## 7. Catalog de pași de workflow (MVP)

Handlerele conțin business logic în cod. Un BPMN încărcat **doar recombină** acești pași
(altă ordine, alte condiții, alte SLA-uri, alți aprobatori). Pași noi = dezvoltare, nu
configurare. Pașii sunt **generici peste „o înregistrare dintr-un registru”** – nu le pasă
din ce registru vine; UI-ul task-ului afișează atributele custom prin schema registrului.

| `taskDefinitionKey` | UI | Acțiuni | Efect handler |
|---|---|---|---|
| `repartizare` | form generic (alege responsabil: user sau compartiment) | `ASSIGN`, `FORWARD`, `SEND_BACK` | setează `assigneeUserId`/`departmentId`, `entry.status = IN_PROGRESS`, e-mail responsabil |
| `redactareRaspuns` | view custom (upload document răspuns + editor text) | `SUBMIT_FOR_APPROVAL`, `SAVE` | atașează `document(kind=RESPONSE)`, setează `responseDocumentId`, `entry.status = WAITING_APPROVAL` |
| `aprobare` | view răspuns + istoric | `APPROVE`, `SEND_BACK` (cu motiv) | setează `approved`, `rejectionReason`; e-mail soluționator la send-back |
| `expediere` | form (canal + dată) | `MARK_SENT` | setează `sentChannel`, `sentDate`; opțional atașează `SIGNED_RESPONSE` |
| `verificare` | form generic (din form schema) | `APPROVE`, `SEND_BACK` | pas generic opțional de control |

**Service tasks (whitelist):**

| `delegateExpression` | Efect |
|---|---|
| `${notificaEmailDelegate}` | trimite e-mail din șablon; eșecul NU blochează procesul (log + `timeline_event` WARN) |
| `${seteazaTermenDelegate}` | (re)calculează `due_date` din regula registrului sau dintr-o variabilă |
| `${inchideInregistrareDelegate}` | `entry.status = CLOSED`, `closed_at = now`, `timeline_event CLOSED` |

**Închiderea** procesului: `endEvent` → `inchideInregistrareDelegate` (sau boundary implicit
dacă BPMN-ul nu-l include, aplicat de `WorkflowService` la `COMPLETED`).

### Proces MVP demonstrativ (6 noduri)

```text
START
  → [userTask] repartizare
  → [userTask] redactareRaspuns
  → [userTask] aprobare
  → <exclusiveGateway ${approved == true}>
        ├── true  → [userTask] expediere → [serviceTask] inchideInregistrare → END
        └── false → [userTask] redactareRaspuns   (buclă de corectare)
```

### Registre fără workflow – ciclu de viață încorporat

Nu folosește Flowable. Tranziții manuale prin API:

```text
REGISTERED
  → dispatch(responsabil)   → DISPATCHED
  → resolve(resolution, note) → RESOLVED | REJECTED | CANCELLED
```
Audit și timeline se aplică identic.

---

## 8. API REST (orientat pe domeniu)

Autentificare:
```text
POST /api/auth/login      -> { accessToken, refreshToken, expiresIn }
POST /api/auth/refresh
POST /api/auth/logout
GET  /api/auth/me
```

Registratură:
```text
GET   /api/registers                       -- definiții active (pentru formularul de înregistrare)
POST  /api/registers/{key}/entries         -- multipart: date + fișiere -> { number, businessKey, entryId }
GET   /api/entries                         -- căutare/filtrare (register, status, petent, interval, număr, text)
GET   /api/entries/{id}
PATCH /api/entries/{id}/due-date           -- override cu motiv
POST  /api/entries/{id}/anonymize          -- GDPR
POST  /api/entries/{id}/documents          -- multipart
GET   /api/entries/{id}/documents/{docId}  -- download (stream)
GET   /api/entries/{id}/timeline
POST  /api/entries/{id}/dispatch           -- doar registre fără workflow
POST  /api/entries/{id}/resolve            -- doar registre fără workflow: { resolution, note }
```

Inbox / task:
```text
GET  /api/inbox/tasks           -- task-urile mele + ale grupurilor mele; filtre: urgente|astazi|intarziate|toate
GET  /api/inbox/tasks/{id}      -- Task View Model: form schema + date entry + documente + acțiuni + context
POST /api/inbox/tasks/{id}/claim
POST /api/inbox/tasks/{id}/actions   -- { action, comment, data }
GET  /api/inbox/count
```

Back office:
```text
GET|POST|PUT /api/admin/registers
POST /api/admin/workflow-definitions/import        -- multipart BPMN -> validation_report
GET  /api/admin/workflow-definitions?registerKey=
POST /api/admin/workflow-definitions/{id}/activate
GET  /api/admin/workflow-definitions/{id}/xml
GET|POST|PUT /api/admin/users
POST /api/admin/users/{id}/reset-password
GET|POST|PUT /api/admin/departments
GET|PUT /api/admin/settings
POST /api/admin/seed/import
GET  /api/admin/jobs/failed
POST /api/admin/jobs/{id}/retry
```

Rapoarte / dashboard:
```text
GET /api/reports/register-export?registerKey=&from=&to=&format=pdf|xlsx
GET /api/reports/overdue
GET /api/dashboard/summary
```

Operațional: `/actuator/health`, `/actuator/metrics`, `/actuator/info`.

---

## 9. Ecrane OpenUI5

- **Login** – utilizator/parolă; ecran de schimbare forțată a parolei la prima autentificare.
- **Home / Dashboard** – contoare (în lucru / întârziate / de aprobat / de semnat), acces
  rapid la Inbox, căutare globală. Drill-down = listă filtrată.
- **Inbox** – tab-uri (Urgente / Astăzi / Întârziate / Toate); listă: număr, obiect, pas,
  termen rămas, petent. Sursă: `/api/inbox/tasks` (niciodată Flowable direct).
- **Task Detail** – zonă de context (vizualizare PDF, metadate, câmpuri custom, petent),
  formular dinamic din form schema per `taskDefinitionKey`, view custom pentru
  `redactareRaspuns` (upload + editor text simplu), butoane de acțiune din Task View Model,
  timeline lateral.
- **Înregistrare** – alegere registru → formular standard + câmpuri custom dinamice + upload
  documente + căutare/creare petent.
- **Detaliu înregistrare** – date, documente, timeline complet, audit, status; acțiuni
  `due-date` și `anonymize` vizibile pe rol.
- **Căutare / registru** – filtre + tabel + export (PDF/XLSX).
- **Back office** – Registre (editor schemă), Definiții workflow (upload, versiuni, raport
  validare, activare, vizualizare XML), Utilizatori & roluri, Compartimente, Setări instanță,
  Job-uri eșuate, Import seed.

Toate textele în `i18n.properties` (RO). Build: `ui5 build` → `backend/src/main/resources/static`.

---

## 10. Cross-cutting

**Autentificare & autorizare.** JWT HS256, secret în `APP_JWT_SECRET`. Access token ~15 min;
refresh token ~8 h, stocat ca hash SHA-256 în `refresh_token` (revocabil). Spring Security
resource server; roluri în claim `roles`; `@PreAuthorize` pe endpointuri. Blocare: 5
autentificări eșuate → `locked_until = now + 15 min`. Logout = revocare refresh token +
discard pe client. Notă acceptată: token în `localStorage` implică risc XSS; mitigare prin
CSP strict + refresh scurt; mutarea pe cookie `httpOnly` este o opțiune de Faza 2.

**Migrări.** Flyway pentru schema aplicației. Flowable cu `database-schema-update=true` la
instalare controlată.

**`correlationId`.** `OncePerRequestFilter` preia/generează `X-Correlation-Id`, îl pune în
MDC, îl atașează la `audit_event` și îl setează ca variabilă de proces `correlationId`;
delegate-urile îl propagă în e-mailuri și loguri.

**Audit.** `AuditService.record(...)` apelat din handlere + un interceptor pe mutațiile
`register_entry`, `party`, `document`, `app_user`, `register_definition`,
`workflow_definition`. Append-only.

**Timeline.** `TimelineService` scrie `timeline_event` din aceleași puncte + dintr-un
`FlowableEventListener` (task created / completed).

**GDPR.** `legal_basis` pe definiția registrului, afișat la înregistrare. `anonymize`
înlocuiește câmpurile PII din `party` cu marcaje, păstrează `party.id` și înregistrarea
(număr, istoric workflow), scrie audit + `timeline_event ANONYMIZED`. Documentele rămân (nu
pot fi redate în MVP). Retenție automată și audit-de-acces → Faza 2.

**Localizare.** RO singura limbă livrată; toate textele în resource bundles (UI5 `i18n` +
backend `messages_ro.properties`). Timezone fix `Europe/Bucharest`, UTF-8 peste tot, format
dată/număr locale RO.

**E-mail.** `JavaMailSender` sincron, șabloane Thymeleaf. Declanșatoare: repartizare, cerere
de aprobare, send-back, SLA reminder, escaladare. `email_log` per mesaj. Eșecul e-mailului nu
blochează tranziția task-ului.

**SLA.** `due_date` la înregistrare din `due_date_rule`. BPMN poate avea boundary timers;
suplimentar, un job `@Scheduled` (la 1 h) scanează înregistrările cu proces activ și cele
fără workflow: reminder la `reminder_pct` (implicit 75% din interval), escaladare la depășire
(reasignare la `department.parent` + e-mail). Praguri configurabile per registru și global.

**Observabilitate.** Actuator `health` cu indicatori pentru DB, MinIO, SMTP, executorul async
Flowable; `metrics` Micrometer; listă job-uri Flowable eșuate în back office cu retry.
OpenTelemetry / Prometheus / UI dead-letter → Faza 2/3.

**Testare.**
- Unit: servicii de domeniu, `NumberGenerator`, validatorul BPMN, form schema mapper.
- Flowable process tests: fiecare proces seed pe căile happy + send-back.
- Integrare (Testcontainers PostgreSQL + MinIO): înregistrare → număr, upload → MinIO,
  import BPMN → deploy, start → `workflow_instance`.
- REST (MockMvc / WebTestClient): autentificare, RBAC (403), acțiuni de task.
- ArchUnit: `document` nu importă `workflow.flowable`; nimic în afara pachetului
  `workflow.flowable` nu importă `org.flowable.*`.
- Acceptanță: un test `@SpringBootTest` care rulează cei 12 pași din §12 cap-coadă, cu
  Greenmail pentru e-mail.
- CI: build Gradle + teste (host-ul se alege la crearea repo-ului de produs).

---

## 11. Plan pe etape

Secvențial. Estimările sunt **orientative**, în săptămâni part-time; ritmul variază.
Fiecare etapă se consideră încheiată doar când „Definition of Done” este integral îndeplinit.

### M0 – Fundație — *1–2 săpt.*
- **Obiectiv:** schelet rulabil, decizie tehnologică confirmată.
- **Livrabile:** monorepo (`backend/` Gradle + `ui/` UI5); Spring Boot 4 + **spike Flowable
  8** (proces minimal pornit și finalizat); `docker compose` (PostgreSQL 16, MinIO); Flyway;
  schelet auth JWT; Actuator health; ArchUnit; pipeline `ui5 build` → `static/`; CI build+test.
- **DoD:** `docker compose up` pornește aplicația; un proces BPMN de test rulează prin
  `WorkflowService`; `GET /api/auth/me` funcționează cu token; testele trec în CI.
- **Riscuri:** incompatibilități Flowable 8 / Jackson 3 → dacă blochează, downgrade la Spring
  Boot 3.4 + Flowable 7 (atinge doar modulul `workflow`). Decizia se ia la finalul M0.

### M1 – Registre & înregistrare — *2–3 săpt.*
- **Obiectiv:** definirea registrelor și înregistrarea documentelor.
- **Livrabile:** `register_definition` + `register_counter` + `register_entry`; `NumberGenerator`
  (per registru, reset anual, `SELECT ... FOR UPDATE`); schema câmpuri custom + formular
  dinamic UI5; `party` + căutare „soft”; `document` + stocare MinIO; editor de registru în
  back office; `POST /api/registers/{key}/entries`; ecranele Înregistrare și Detaliu
  înregistrare.
- **DoD:** un registru creat din back office; o înregistrare cu fișier atașat primește număr
  și `due_date`; câmpurile custom se salvează în `custom_data` și se afișează.
- **Riscuri:** coloane dinamice în UI5; validarea schemei custom.

### M2 – Workflow core — *2–3 săpt.*
- **Obiectiv:** definițiile de proces gestionate la runtime.
- **Livrabile:** `WorkflowService` adapter + `workflow_definition` / `workflow_instance` /
  `workflow_task`; `POST /api/admin/workflow-definitions/import` cu pipeline complet de
  validare (§6); deploy + versionare + XML în MinIO; activare per registru; pornirea instanței
  la înregistrare cu business key; `FlowableEventListener` pentru oglindirea task-urilor.
- **DoD:** un BPMN invalid este respins cu raport; unul valid se deployează, se activează, și
  la o înregistrare nouă pornește o instanță cu `businessKey` corect.
- **Riscuri:** acoperirea completă a regulilor de validare; parsarea `conditionExpression`.

### M3 – Inbox & execuție task — *2–3 săpt.*
- **Obiectiv:** utilizatorul lucrează task-uri în UI-ul propriu.
- **Livrabile:** `GET /api/inbox/tasks` + `count`; ecran Inbox cu tab-uri; Task Detail cu
  renderer de form schema; Task Action API `POST /api/inbox/tasks/{id}/actions`; Task Handler
  Registry; handlerele `repartizare` și `verificare`; claim + verificări de concurență.
- **DoD:** un task `repartizare` apare în Inbox-ul responsabilului corect, se poate revendica
  și finaliza cu `ASSIGN`; acțiunile nepermise întorc 4xx.
- **Riscuri:** modelul candidate group ↔ compartiment; coerența `workflow_task` cu Flowable.

### M4 – Circuit complet petiție — *1–2 săpt.*
- **Obiectiv:** procesul demonstrativ funcțional cap-coadă.
- **Livrabile:** view custom `redactareRaspuns`; handler `aprobare` + gateway `SEND_BACK`;
  handler `expediere`; `inchideInregistrareDelegate`; `timeline_event` complet; procesul seed
  în 6 noduri.
- **DoD:** parcurgerea Repartizare → Redactare → Aprobare → (send-back o dată) → Aprobare →
  Expediere → Închidere, cu timeline complet.
- **Riscuri:** bucla de corectare (re-intrare în `redactareRaspuns`).

### M5 – SLA & notificări — *1–2 săpt.*
- **Obiectiv:** termene și e-mail.
- **Livrabile:** reguli `due_date` (calendar/lucrătoare); boundary timers + job `@Scheduled`;
  `notificaEmailDelegate` + șabloane Thymeleaf; `email_log`; ciclul de viață pentru registre
  fără workflow (`dispatch` / `resolve`); afișarea timpului rămas în Inbox.
- **DoD:** cu termen scurtat, se trimit reminder și escaladare (verificate cu Greenmail); un
  registru fără workflow parcurge `REGISTERED → DISPATCHED → RESOLVED`.
- **Riscuri:** calculul zilelor lucrătoare / sărbători legale (MVP: doar calendaristice +
  weekend; sărbătorile → configurare simplă).

### M6 – Rapoarte, GDPR, confidențial — *2–3 săpt.*
- **Obiectiv:** ieșiri oficiale și conformitate de bază.
- **Livrabile:** integrare JasperReports; export registru (PDF + XLSX) cu coloane standard +
  coloane din schema custom adăugate programatic; raport „termene depășite”; contoare pe home
  cu drill-down; acțiune „anonimizează petent”; flag `confidential` + filtrare la citire pe rol.
- **DoD:** export corect pe un interval, cu un câmp custom în coloane; anonimizarea șterge
  PII și lasă înregistrarea + numărul; o înregistrare confidențială nu apare pentru un
  `VIZUALIZATOR` neimplicat.
- **Riscuri:** coloane dinamice în JasperReports → PoC devreme; **fallback:** export XLSX
  programatic cu Apache POI pentru registrele cu multe câmpuri custom.

### M7 – Întărire & acceptanță — *1–2 săpt.*
- **Obiectiv:** produs livrabil pentru un pilot.
- **Livrabile:** audit complet pe toate mutațiile; `correlationId` end-to-end; listă job-uri
  eșuate + retry în back office; testul de acceptanță (cei 12 pași) automatizat; împachetare
  (`docker compose` + script de instalare + `POST /api/admin/seed/import`); `README`,
  `AGENTS.md`, ghid de operare.
- **DoD:** testul de acceptanță trece în CI; o instalare curată din script ajunge la primul
  registru funcțional în sub 30 de minute.
- **Riscuri:** deriva de scop; se închide strict pe scenariul de acceptanță.

**Total orientativ: ~12–20 săptămâni part-time.**

---

## 12. Scenariu de acceptanță (criteriul „MVP gata”)

Rulează și ca test de integrare end-to-end (Testcontainers + Greenmail).

1. Admin creează registrul „Petiții” cu câmpuri custom `urgență` (select) și `mod primire`
   (select), regulă termen +30 zile calendaristice; încarcă și activează BPMN-ul cu 6 pași.
2. Admin creează 4 utilizatori: `registrator`, `soluționator`, `aprobator`, `vizualizator`.
3. `registrator` înregistrează o petiție cu PDF atașat, petent persoană fizică → se alocă
   numărul `2026/000001`, `due_date` calculat, pornește instanța, business key
   `PETITII-2026-000001`.
4. `registrator` repartizează către `soluționator`.
5. `soluționator` redactează răspunsul, încarcă un DOCX, trimite la aprobare.
6. `aprobator` trimite înapoi cu motiv → task-ul revine la `soluționator` (test gateway).
7. `soluționator` corectează și retrimite → `aprobator` aprobă.
8. `soluționator`/`registrator` încarcă PDF-ul semnat (placeholder) și marchează expediat
   (canal: poștă) → handlerul închide înregistrarea.
9. Timeline-ul arată toți pașii; auditul conține toate acțiunile cu autor și timp;
   s-au trimis e-mailuri la repartizare și la cererea de aprobare (verificate cu Greenmail).
10. Cu termenul scurtat prin configurare SLA, se trimite un e-mail de reminder.
11. Export registru (PDF + XLSX) pe interval, cu coloanele standard + `urgență`.
12. Un al doilea registru **fără workflow** („Corespondență”): înregistrare → lansare spre
    rezolvare → rezoluție „rezolvat”.

---

## 13. Riscuri și necunoscute

| Risc | Impact | Mitigare |
|---|---|---|
| Flowable 8 este nou (lansat feb. 2026) | mediu | Spike în M0; downgrade izolat la SB 3.4 + Flowable 7 în modulul `workflow` |
| Jackson 3 (Spring Boot 4 / Flowable 8) | mediu | Verificare serializare `custom_data` JSONB și DTO-uri în M0/M1 |
| Coloane dinamice în JasperReports | mediu | PoC în M6; fallback = export XLSX cu Apache POI |
| Build UI5 fără Node în producție | mic | Verificat în M0 (`ui5 build` → `static/`) |
| Vocabular închis vs. nevoi reale de proces la primul pilot | mediu | Acceptat conștient; un pas nou = dezvoltare, nu configurare |
| JWT în `localStorage` (XSS) | mic-mediu | CSP strict + refresh scurt; cookie `httpOnly` ca opțiune de Faza 2 |
| Zile lucrătoare / sărbători legale la SLA | mic | MVP: doar calendaristice + weekend; sărbători prin configurare simplă |
| Calitatea BPMN-urilor încărcate de clienți | mediu | Pipeline de validare strict + contract BPMN documentat |

---

## 14. După MVP

### Punct de validare (imediat după M7)
Pilot cu prima comună reală, 2–4 săptămâni. Feedback-ul intră într-un backlog de Faza 2.
Nu se investește în Faza 2 înainte de acest pas.

### Faza 2
- OCR (Tesseract sau serviciu extern) + `DocumentIntakePipeline` real.
- Clasificare AI **confidence-driven** – AI recomandă, regula/omul decide. Praguri
  configurabile per tip de document (ex. `≥0.95` automat, `0.70–0.95` verificare umană,
  `<0.70` clasificare manuală). AI nu declanșează direct tranziții Flowable.
- Extragere metadate din documente.
- OpenSearch – căutare unificată (număr registratură, număr document, subiect, conținut OCR,
  persoană, instituție, categorie, status, workflow, task).
- Notificări pe coadă: RabbitMQ + **Outbox pattern** (evenimentul se scrie în aceeași
  tranzacție cu update-ul business).
- Semnătură electronică reală.
- Ingestie e-mail automată (mailbox → înregistrare draft).
- LDAP / Active Directory.
- ACL fin per registru / compartiment; flag `confidential` extins.
- Retenție GDPR automată (programe de ștergere/anonimizare) + audit-de-acces.
- Model hibrid de pas: form schema, label-uri, SLA și rol-aprobator configurabile din back
  office (business logic rămâne în cod).
- API de import cu **sandbox de script** – acces la un API controlat al aplicației, nu la
  `ApplicationContext`.
- Retry controlat + UI de dead-letter; `Idempotency` explicit pe handlerele critice.

### Faza 3
- RAG pentru redactarea răspunsurilor: petiție + documente relevante + regulamente/proceduri
  → draft → User Task de revizuire → semnare. AI produce draftul, nu îl trimite automat.
- AI copilot în interiorul task-ului (rezumat, corespondență anterioară, cazuri conexe).
- Analytics / dashboards avansate; drill-down proces → instanță → task → document → audit.
- SLA predictiv, detecție duplicate.
- Portal cetățean (depunere online, urmărire status).
- Observabilitate completă: OpenTelemetry, trace unic HTTP → Spring → Flowable → AI → DB →
  RabbitMQ; Prometheus/Grafana.

---

## 15. Stack tehnic (referință)

```text
Frontend      OpenUI5 (JavaScript, UI5 Tooling), servit static de Spring Boot
Backend       Java 21, Spring Boot 4, Spring Security (resource server JWT), Spring Data JPA
Workflow      Flowable 8 Open Source (embedded, ascuns după WorkflowService)
Bază de date  PostgreSQL 16 (jsonb pentru custom_data); migrări Flyway
Documente     MinIO (API S3, AWS SDK v2)
Rapoarte      JasperReports 7 (+ Apache POI ca fallback pentru XLSX dinamic)
E-mail        JavaMailSender + Thymeleaf (sincron în MVP)
Build         Gradle, monorepo
Test          JUnit 5, AssertJ, Testcontainers, Greenmail, ArchUnit, Flowable process tests
Observabil.   Micrometer + Spring Boot Actuator
Ambalare      Docker Compose (un singur host, single-tenant)
```

### Structura backend (pachete pe domeniu, granițe ArchUnit)

```text
ro.<inst>.registratura
├── registration      -- register_definition, register_entry, NumberGenerator, înregistrare
├── document          -- document, stocare MinIO, DocumentIntakePipeline (hook)
├── party             -- persoane/organizații, anonimizare GDPR
├── workflow
│   ├── api           -- REST inbox/task/admin workflow
│   ├── application   -- WorkflowService, Task Handler Registry, handlere
│   ├── domain        -- workflow_definition/instance/task, contract BPMN
│   └── flowable      -- SINGURA zonă care atinge org.flowable.*
├── audit             -- AuditService, audit_event
├── timeline          -- TimelineService, timeline_event
├── notification      -- e-mail, șabloane, email_log
├── search            -- căutare pe DB (OpenSearch în Faza 2)
├── security          -- JWT, utilizatori, roluri, refresh token
├── reporting         -- JasperReports, export registru, overdue, dashboard
└── common            -- correlationId, settings, i18n, erori
```
