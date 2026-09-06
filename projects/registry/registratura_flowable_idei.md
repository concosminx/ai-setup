# Idei și decizii arhitecturale rămase – Registratură + BPM + AI

## 1. Principiul central

Aplicația proprie controlează experiența și business-ul. Flowable este motorul de workflow, nu aplicația de registratură și nu UI-ul final.

```text
BPMN extern
   ↓
Import / validare
   ↓
Flowable Engine
   ↓
WorkflowService propriu
   ↓
REST API
   ↓
OpenUI5
```

Administratorul nu proiectează procesele în aplicație. Procesele sunt proiectate extern și importate ca BPMN XML.

---

## 2. Modular monolith la început

Pentru prima versiune este preferabil un Spring Boot modular monolith, nu microservicii.

Module recomandate:

```text
registration
document
case-management
workflow
ai
audit
notification
search
security
reporting
integration
```

Avantaj: tranzacții și debugging mai simple, dar modulele pot fi extrase ulterior.

---

## 3. Flowable trebuie ascuns în spatele unui adapter

Nu lăsa codul din toată aplicația să folosească direct `RuntimeService`, `TaskService` etc.

Creează:

```java
WorkflowService
```

cu operații precum:

```text
startProcess(...)
getTasks(...)
getTask(...)
claimTask(...)
completeTask(...)
delegateTask(...)
cancelProcess(...)
getTimeline(...)
```

Implementarea internă folosește Flowable.

Astfel, UI-ul și business-ul nu devin dependente de API-ul Flowable.

---

## 4. User Task ≠ Business Logic

Un User Task reprezintă o etapă în proces.

Exemplu:

```text
verifyDocument
```

Flowable spune:

> task-ul `verifyDocument` este activ pentru utilizatorul X.

Aplicația decide:

- ce date se afișează;
- ce documente se deschid;
- ce formular apare;
- ce validări există;
- ce acțiuni sunt permise;
- ce business logic se execută;
- ce audit se înregistrează.

---

## 5. Maparea task → UI OpenUI5

Folosește `taskDefinitionKey` ca identificator stabil.

Exemplu:

```text
verifyDocument
    ↓
DocumentVerification.view.xml

approveDocument
    ↓
DocumentApproval.view.xml

draftResponse
    ↓
ResponseDrafting.view.xml
```

Poți avea o configurație:

```json
{
  "verifyDocument": {
    "component": "DocumentVerification",
    "route": "/documents/{documentId}/verify"
  },
  "approveDocument": {
    "component": "DocumentApproval",
    "route": "/documents/{documentId}/approve"
  }
}
```

Astfel procesele pot fi schimbate fără să reconstruiești UI-ul generic.

---

## 6. Task Action API

Nu trimite din UI doar `completeTask`.

Folosește acțiuni explicite:

```http
POST /api/workflow/tasks/{id}/actions
```

Exemplu:

```json
{
  "action": "APPROVE",
  "comment": "Verificat",
  "data": {}
}
```

Alte acțiuni:

```text
APPROVE
REJECT
SEND_BACK
CLAIM
DELEGATE
ESCALATE
SAVE
FORWARD
CANCEL
```

Backend-ul validează acțiunea înainte să modifice Flowable.

---

## 7. Task Handler Registry

O idee foarte bună pentru control fin:

```text
TaskHandlerRegistry
       │
       ├── verifyDocument → DocumentVerificationHandler
       ├── approveDocument → DocumentApprovalHandler
       ├── draftResponse → ResponseDraftHandler
       └── signDocument → SignatureHandler
```

Fiecare handler poate face:

```text
validare
→ autorizare
→ business logic
→ persistare
→ audit
→ variabile Flowable
→ complete task
```

Asta permite ca fiecare task să aibă comportament controlat de aplicație.

---

## 8. Groovy pentru logică simplă

Groovy este potrivit pentru:

- reguli simple;
- calcule;
- transformări;
- condiții;
- routing;
- setarea variabilelor.

Exemplu:

```groovy
if (urgency == "URGENT") {
    execution.setVariable("priority", 100)
}
```

Nu transforma Groovy în limbajul principal al business-ului.

---

## 9. Java/Spring pentru logică importantă

Pentru operații complexe folosește `Service Task` + `JavaDelegate`.

Exemple:

```text
OCR
AI
semnătură electronică
REST extern
ANAF
generare document
acces DB
calcul complex
integrare e-mail
```

Exemplu BPMN:

```xml
<serviceTask
    id="classifyDocument"
    flowable:delegateExpression="${documentClassificationDelegate}" />
```

Avantajul este că business logic rămâne testabilă și tipizată în Java.

---

## 10. Nu permite Groovy arbitrar din import

Dacă BPMN-urile pot fi importate de utilizatori, tratează scripturile ca executable code.

Pipeline recomandat:

```text
Upload BPMN
   ↓
XML validation
   ↓
BPMN validation
   ↓
Allowed task types
   ↓
Script validation
   ↓
Security checks
   ↓
Version approval
   ↓
Deploy
```

Ideal, scripturile au acces la un API controlat de aplicație, nu la întregul Spring ApplicationContext.

---

## 11. Separă datele business de datele workflow

Business DB:

```text
registration
document
person
organization
case
response
attachment
```

Workflow:

```text
workflow_definition
workflow_instance
workflow_task
```

Flowable are propriile tabele.

Nu face aplicația dependentă de schema internă Flowable.

---

## 12. Tabele proprii recomandate

### workflow_definition

```text
id
key
name
version
flowable_deployment_id
active
created_at
```

### workflow_instance

```text
id
workflow_definition_id
business_type
business_id
flowable_process_instance_id
status
started_at
completed_at
```

### workflow_task

```text
id
workflow_instance_id
flowable_task_id
task_definition_key
assignee
status
due_date
completed_at
```

Acestea permit UI-ului să lucreze cu propriul model.

---

## 13. Business Key

La pornirea procesului, folosește business key.

Exemplu:

```text
PETITION-2026-12345
```

sau:

```text
REGISTRATION-2026-12345
```

Asta face legătura clară între Flowable și obiectul business.

---

## 14. Workflow Timeline propriu

Nu expune istoricul Flowable direct.

Construiește timeline-ul propriu:

```text
✓ Înregistrat
✓ Clasificat
✓ Repartizat Juridic
✓ Verificat
● Aprobare
○ Redactare răspuns
○ Semnare
○ Expediere
○ Închidere
```

Poți combina:

- Flowable history;
- auditul aplicației;
- evenimentele business;
- acțiunile AI.

---

## 15. Audit separat de Flowable History

Flowable History este util pentru workflow, dar auditul juridic/business trebuie să fie al aplicației.

Audit event:

```text
who
when
action
business_object
old_value
new_value
ip/device context dacă este necesar
workflow_instance
task
correlation_id
```

Pentru documente oficiale, auditul trebuie tratat ca parte importantă a domeniului.

---

## 16. AI nu trebuie să decidă direct workflow-ul

Model recomandat:

```text
AI
 ↓
recomandare + confidence
 ↓
Business Rule / User
 ↓
Flowable transition
```

Exemplu:

```text
AI:
categorie = JURIDIC
confidence = 0.94
```

Apoi regula aplicației:

```text
confidence >= 0.90
→ repartizare automată

confidence < 0.90
→ User Task "Verificare clasificare"
```

---

## 17. AI în interiorul task-ului

Când utilizatorul deschide un task:

```text
OpenUI5
   ↓
GET task
   ↓
Spring
   ├── workflow
   ├── document
   ├── AI
   └── business
   ↓
Task View Model
```

UI poate afișa:

```text
Document PDF
Metadata
OCR text
AI summary
AI classification
AI confidence
Suggested department
Previous correspondence
Related cases
```

Utilizatorul rămâne în UI-ul tău.

---

## 18. Form Schema

O idee foarte bună pentru a evita codarea fiecărui formular de la zero:

```text
taskDefinitionKey
       ↓
form schema
       ↓
OpenUI5 renderer
```

Exemplu:

```json
{
  "task": "verifyDocument",
  "fields": [
    {
      "name": "category",
      "type": "select",
      "required": true
    },
    {
      "name": "observations",
      "type": "textarea"
    }
  ],
  "actions": [
    "APPROVE",
    "SEND_BACK"
  ]
}
```

Pentru task-urile foarte complexe poți avea UI5 views custom.

---

## 19. Două niveluri de UI

### Generic Task UI

Pentru task-uri simple:

```text
Form Schema
→ Dynamic UI5 form
```

### Custom Task UI

Pentru task-uri complexe:

```text
verifyDocument
→ DocumentVerification.view.xml
```

Astfel ai flexibilitate fără să construiești totul manual.

---

## 20. Versionarea proceselor

Nu modifica un proces activ în loc.

Model:

```text
petition-process v1
petition-process v2
petition-process v3
```

Procesele deja pornite rămân pe versiunea cu care au început.

Procesele noi folosesc versiunea activă.

---

## 21. Import BPMN

Poți avea:

```http
POST /api/workflow/definitions/import
```

care:

```text
primește BPMN XML
→ validează
→ extrage process definitions
→ verifică metadata
→ creează deployment Flowable
→ salvează workflow_definition
→ publică versiunea
```

Poți păstra BPMN XML original în object storage sau DB pentru audit/versionare.

---

## 22. Metadata proprie în BPMN

O idee utilă: procesele externe pot conține identificatori pe care aplicația îi interpretează.

De exemplu:

```text
taskDefinitionKey = verifyDocument
formKey = document-verification
businessType = PETITION
```

În felul acesta BPMN-ul devine contract între designer și aplicație.

---

## 23. Nu pune ID-uri de document hardcodate în BPMN

BPMN-ul trebuie să fie generic.

Corect:

```text
documentId = process variable
```

Nu:

```text
documentId = 12345
```

La pornirea procesului:

```json
{
  "registrationId": "REG-2026-12345",
  "documentId": "DOC-2026-77881"
}
```

---

## 24. Event-driven unde are sens

Pentru operații care nu trebuie să blocheze task-ul:

```text
Workflow
 ↓
Application Event
 ↓
RabbitMQ
 ↓
Worker
```

Exemple:

```text
notificare
indexare OpenSearch
OCR
generare preview
analytics
AI processing
```

Dar nu transforma fiecare operație într-un mesaj. Păstrează sincron pentru operațiile care trebuie finalizate înainte de continuarea procesului.

---

## 25. Outbox Pattern

Pentru evenimente importante:

```text
DB transaction
 ├── business update
 └── outbox event
          ↓
      publisher
          ↓
       RabbitMQ
```

Ajută la evitarea situației:

```text
DB salvat
dar mesajul nu a fost trimis
```

---

## 26. Idempotency

Orice handler important trebuie să poată trata repetarea.

Exemplu:

```text
approveDocument(documentId)
```

nu trebuie să aprobe de două ori documentul dacă request-ul este retrimis.

Folosește:

```text
idempotency key
business constraints
state checks
```

---

## 27. Concurrency

Protejează task-urile împotriva a două tab-uri sau doi utilizatori care încearcă să execute simultan același task.

Exemplu:

```text
Task OPEN
 ↓
claim
 ↓
version/state check
 ↓
complete
```

Pentru operații critice, folosește optimistic locking și verificări de stare.

---

## 28. SLA și escaladare

Folosește timer events în BPMN.

Exemplu:

```text
User Task: Verificare
        │
        ├── 2 zile → reminder
        │
        ├── 5 zile → escaladare
        │
        └── 10 zile → manager
```

UI-ul poate afișa:

```text
SLA: 1 zi 4 ore rămase
```

---

## 29. Inbox-ul OpenUI5

Unul dintre cele mai importante ecrane:

```text
┌─────────────────────────────────────────────┐
│ Inbox                                       │
├─────────────────────────────────────────────┤
│ Urgente   │ Astăzi │ Întârziate │ Toate   │
├─────────────────────────────────────────────┤
│ Petiția #12345  │ Verificare │ URGENT     │
│ Adresa #77881   │ Aprobare   │ 2h         │
│ Sesizarea #991  │ Răspuns    │ 1 zi       │
└─────────────────────────────────────────────┘
```

Task-urile vin din API-ul tău, nu direct din Flowable.

---

## 30. Dashboard workflow

Poți avea:

```text
În lucru             1.248
În întârziere          73
Urgente                21
Așteaptă aprobare      48
Așteaptă semnare       16
```

și drill-down până la:

```text
Proces
→ instanță
→ task
→ document
→ audit
```

---

## 31. Search unificat

OpenSearch poate indexa:

```text
număr registratură
număr document
subiect
conținut OCR
persoană
instituție
categorie
status
workflow
task
```

Căutarea trebuie să poată găsi documentul chiar dacă utilizatorul nu știe exact numărul.

---

## 32. AI + RAG

Pentru redactarea răspunsurilor:

```text
Petiție
 ↓
documente relevante
 ↓
regulamente/proceduri
 ↓
RAG
 ↓
draft răspuns
 ↓
User Task: Revizuire
 ↓
User
 ↓
Semnare
```

AI produce draftul, nu îl trimite automat fără control dacă procesul cere responsabilitate umană.

---

## 33. Document intelligence

Pipeline recomandat:

```text
Upload
 ↓
virus scan
 ↓
OCR
 ↓
document classification
 ↓
metadata extraction
 ↓
confidence
 ↓
human validation dacă este necesar
 ↓
indexare
```

---

## 34. Confidence-driven workflow

Una dintre cele mai bune utilizări AI + BPM:

```text
confidence >= 0.95
    → automat

0.70 - 0.95
    → verificare umană

< 0.70
    → task special / clasificare manuală
```

Pragurile trebuie configurabile pe tip de document.

---

## 35. Reguli business configurabile

Nu pune toate regulile în cod.

Separă:

```text
BPMN
Business Rules
Application Code
AI
```

Exemplu:

```text
BPMN:
cine urmează după verificare

Rule:
cine are dreptul să aprobe

Java:
cum se validează documentul

AI:
ce categorie este probabilă
```

---

## 36. Testare BPMN

Fiecare proces importat ar trebui să aibă teste.

Exemplu:

```text
given petition urgent
when process starts
then task verifyDocument is created

given verification approved
when task completed
then approveDocument becomes active
```

Poți testa procesul cu Spring + Flowable Test.

---

## 37. Contract BPMN

Definește o convenție internă:

```text
process key
taskDefinitionKey
variable names
business key
form key
action names
service delegates
script APIs
```

Exemplu:

```text
documentId
registrationId
caseId
priority
category
department
approved
rejectionReason
```

Asta devine contractul dintre echipa BPM și echipa Java/UI5.

---

## 38. Observability

Toate request-urile și workflow-urile trebuie să aibă:

```text
correlationId
traceId
processInstanceId
taskId
businessId
```

Astfel poți urmări:

```text
HTTP request
→ Spring
→ Flowable
→ AI
→ DB
→ RabbitMQ
```

într-un singur trace.

---

## 39. Health și monitoring

Monitorizează:

```text
Flowable jobs
failed jobs
dead letters
task backlog
SLA breaches
AI latency
OCR latency
RabbitMQ
PostgreSQL
OpenSearch
```

---

## 40. Retry controlat

Nu retrimite automat la infinit.

Model:

```text
attempt 1
attempt 2
attempt 3
      ↓
dead letter / manual retry
```

Pentru integrarea externă, păstrează statusul și eroarea tehnică.

---

## 41. API-ul final nu trebuie să semene cu Flowable

Evită:

```text
GET /flowable/runtime/tasks
```

Preferă:

```text
GET /api/inbox/tasks
GET /api/inbox/tasks/{id}
POST /api/inbox/tasks/{id}/actions
GET /api/cases/{id}/timeline
GET /api/cases/{id}/workflow
```

API-ul reprezintă domeniul tău.

---

## 42. Stack tehnic propus

```text
Frontend
    OpenUI5

Backend
    Java
    Spring Boot
    Spring Security
    Spring Data JPA

Workflow
    Flowable Open Source

Database
    PostgreSQL

Documents
    S3 / MinIO

Search
    OpenSearch

Messaging
    RabbitMQ

Cache
    Redis

AI
    Spring AI
    LLM provider
    OCR
    embeddings / vector search

Observability
    Micrometer
    OpenTelemetry
```

---

## 43. Structura proiectului Spring

O posibilă structură:

```text
src/main/java
└── ro.company.registratura
    ├── registration
    ├── document
    ├── casework
    ├── workflow
    │   ├── api
    │   ├── application
    │   ├── domain
    │   ├── flowable
    │   └── handler
    ├── ai
    ├── audit
    ├── notification
    ├── search
    ├── security
    └── common
```

În `workflow.flowable` stă integrarea efectivă cu Flowable.

---

## 44. Principiul care merită păstrat

```text
Flowable = WHEN / WHERE / NEXT

Java = HOW

OpenUI5 = WHAT USER SEES

AI = WHAT IS PROBABLE / RECOMMENDED

Database = WHAT IS TRUE

Audit = WHAT HAPPENED
```

Această separare va face aplicația mult mai ușor de întreținut.

---

## 45. MVP recomandat

Nu începe cu toate funcționalitățile.

### Faza 1

```text
Spring Boot
PostgreSQL
Flowable
OpenUI5
Document upload
Registrare
Workflow import
Task Inbox
Task UI
Complete / Reject
Audit
```

### Faza 2

```text
OCR
AI classification
metadata extraction
OpenSearch
notifications
SLA
delegation
```

### Faza 3

```text
RAG
AI copilot
draft response
advanced analytics
predictive SLA
duplicate detection
```

---

## 46. Primul proces demonstrativ

Proces ideal pentru PoC:

```text
START
  ↓
Înregistrare
  ↓
Clasificare
  ↓
Repartizare
  ↓
Verificare document
  ↓
     ┌──────────────┐
     │              │
  APROBAT        RESPINS
     │              │
     ↓              ↓
 Aprobare       Corectare
     │              │
     └───────┬──────┘
             ↓
       Redactare răspuns
             ↓
          Semnare
             ↓
         Expediere
             ↓
          Închidere
```

Acest proces testează aproape toate conceptele importante:

- User Tasks;
- Script Task;
- Service Task;
- variabile;
- gateway;
- UI custom;
- acțiuni;
- audit;
- documente;
- AI;
- SLA;
- versionare.

---

## 47. Decizia arhitecturală finală

Pentru proiectul descris, direcția recomandată este:

```text
                    ┌─────────────────────┐
                    │ External BPMN Tool  │
                    └──────────┬──────────┘
                               │ BPMN XML
                               ▼
                    ┌─────────────────────┐
                    │ BPMN Import/Deploy  │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │       Flowable      │
                    │   Workflow Engine   │
                    └──────────┬──────────┘
                               │
                       WorkflowService
                               │
                    ┌──────────▼──────────┐
                    │     Spring Boot     │
                    │ Business + API + AI │
                    └──────────┬──────────┘
                               │ REST
                               ▼
                    ┌─────────────────────┐
                    │       OpenUI5       │
                    │  Application UI     │
                    └─────────────────────┘
```

**Ideea-cheie:** Flowable nu trebuie să-ți dicteze aplicația. El execută procesul. Tu controlezi complet task-ul, UI-ul, business logic-ul, AI-ul și datele.

## 48. Următorul pas tehnic

Cel mai valoros PoC este să construiești efectiv:

```text
1. BPMN XML
2. Flowable Spring Boot starter
3. PostgreSQL
4. import/deploy BPMN
5. start process
6. GET /api/inbox/tasks
7. OpenUI5 Inbox
8. OpenUI5 Task Detail
9. APPROVE / REJECT
10. Groovy Script Task
11. JavaDelegate
12. audit
13. timeline
```

După ce acest flux funcționează, restul aplicației poate fi construit incremental în jurul lui.
