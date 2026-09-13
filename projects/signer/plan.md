# Plan de implementare – Serviciu de semnare electronică

> **Statut:** plan aprobat, rezultat al unei sesiuni de interogare sistematică a cerințelor.
> Acest document este **sursa unică de adevăr** pentru arhitectură, scop și etapizare.
> Este scris pentru a fi executat de un agent AI împreună cu un dezvoltator.
> Nu depinde de niciun alt fișier.
>
> **Ce NU acoperă acest pas:** produsul nu se construiește încă. Aici se stabilește doar planul.

---

## 1. Context și obiectiv

Se construiește un **serviciu de semnare electronică** care primește documente prin API și le
returnează semnate — cu semnătură electronică calificată, avansată sau cu sigiliu electronic —
în formatele standardizate european (ETSI), cu audit probatoriu pe tot ce trece prin el.

Pentru un cititor ne-tehnic: aplicația primește un document, îl semnează cu certificatul digital
al unei persoane sau al unei organizații, îi atașează o dovadă a momentului semnării și toate
informațiile necesare pentru ca semnătura să poată fi verificată și peste zece ani, apoi
înregistrează operația într-un jurnal care nu poate fi modificat fără să se observe.

- **Tip proiect:** produs independent. Nu e o componentă a altui proiect și nu presupune nimic
  despre integratorii lui. Poate fi integrat ulterior cu alte sisteme, dar niciun element de
  arhitectură nu se subordonează acestei posibilități.
- **Licență:** **închis** (proprietar). Consecințele juridice ale dependenței LGPL sunt tratate
  în §17.
- **Echipă:** un singur dezvoltator, part-time, experimentat cu Java / Spring / Gradle. Fără
  spike-uri de învățare, cu două excepții tratate ca Etapa 0 (§18).
- **Orizont:** fără termen impus de un client. Construcție speculativă, cu MVP-ul tăiat astfel
  încât **prima livrare utilizabilă să fie verificabilă de dezvoltator singur, cu hardware real**.
- **Hardware disponibil:** token calificat real, folosit pentru validare încă din Etapa 0.
- **Livrare:** două ambalaje din același cod — server Linux (headless, automat) și aplicație
  Windows (cu utilizator prezent, dialog de PIN).

### Ce rezolvă produsul

1. **Semnare automată pe server**: un sistem generează documente și are nevoie ca ele să poarte
   sigiliul organizației, fără intervenție umană, cu cheia într-un HSM și secretele în Vault.
2. **Semnare asistată pe stația utilizatorului**: o persoană semnează cu tokenul ei calificat,
   introduce PIN-ul o dată și poate semna un lot fără să-l retasteze.
3. **Semnare distribuită**: documentul stă pe server, cheia stă pe stația utilizatorului, și
   doar amprenta circulă între ele.
4. **Validare**: verificarea unui document semnat, cu verdict conform raportului ETSI.
5. **Dovadă**: jurnal înlănțuit criptografic și sigilat temporal, exportabil și verificabil
   independent de aplicație.

### Ce NU este produsul

Nu este o platformă de fluxuri de semnare cu invitații pe e-mail, pagini de semnare pentru
utilizatori externi și urmărirea stadiului (categoria DocuSign / Adobe Sign). Aceea e o altă
categorie de produs — cu identitate a semnatarilor externi, sesiuni, notificări, consimțământ
și o suprafață de securitate complet diferită. Poate fi un produs bun; nu este **acest** produs,
iar amestecarea lor le strică pe amândouă. **Această limită este definitivă**, nu o etapă
amânată.

---

## 2. Principii arhitecturale

Aceste principii sunt obligatorii și guvernează toate deciziile de implementare.

```text
       API REST (OpenAPI ca sursă de adevăr)
                    ↓
       Autorizare pe profil de semnare
                    ↓
       SignatureEngine (adapter propriu)
                    ↓
   ┌────────────────┼────────────────┐
   ↓                ↓                ↓
KeyProvider    TrustServices     AuditLog
(PKCS11/HSM/    (TSA, LOTL,     (înlănțuit,
 MSCAPI/QTSP)    revocare)       sigilat temporal)
   ↓
DSS 6.x (izolat într-un singur modul)
```

1. **Un singur cod, două ambalaje.** Serverul Linux și aplicația Windows sunt același proiect,
   cu profiluri diferite și același API. Paritatea de funcționalitate este structurală, nu o
   promisiune care se erodează.

2. **DSS este ascuns după un adapter.** `eu.europa.esig.*` apare **exclusiv** în modulul
   `signer-engine-dss`. Granița e verificată automat cu ArchUnit, nu lăsată în seama disciplinei.
   Motive: API-ul DSS se schimbă între versiunile majore și nu trebuie să se vadă în contractul
   public; iar obligațiile de licență LGPL rămân confinate într-un singur modul (§17).

3. **Nucleul lucrează în doi pași.** Orice semnare este `prepare` (calculează amprenta de semnat)
   → semnătură brută → `complete` (asamblează documentul). Semnarea „dintr-o bucată” este doar
   înlănțuirea celor doi pași. Fără această formă, semnarea la distanță și modulul agent ar cere
   rescrierea nucleului.

4. **Tot ce e extern este interschimbabil.** Furnizorul de chei, furnizorul de secrete,
   autoritatea de timp — fiecare are o abstracțiune proprie cu mai multe implementări alese din
   configurare. Legarea directă de un furnizor transformă orice schimbare de context într-o
   rescriere.

5. **Nimic nu se etichetează „calificat” fără acoperire.** Profilul declară nivelul juridic
   țintit, iar aplicația **refuză să pornească** dacă furnizorul de chei configurat nu îl poate
   susține. Un instrument care produce documente ce *par* calificate și nu sunt este mai
   periculos decât unul care refuză.

6. **Fără audit scris, nu se semnează.** Auditul se scrie tranzacțional, înainte ca răspunsul să
   plece către apelant. O semnătură neînregistrată e mai periculoasă decât una care n-a avut loc.

7. **Cache-ul de secret și consimțământul sunt lucruri diferite.** PIN-ul poate fi reținut o oră;
   asta nu înseamnă că utilizatorul a consimțit la orice semnare din acea oră. Cele două se
   configurează separat.

8. **Orice intrare este ostilă.** Documentele vin de la sisteme externe și sunt procesate cu
   parsere PDF și XML. Limite dure peste tot, parsere blindate, curățare garantată.

9. **Degradarea nu se ascunde.** Dacă nivelul cerut nu poate fi atins, operația eșuează curat.
   Nu se livrează documente cu un nivel mai mic decât cel cerut, sperând că nimeni nu citește
   răspunsul.

10. **Specificația precede codul.** OpenAPI se scrie întâi și generează interfețele. La un produs
    al cărui *produs* este API-ul, specificația e sursa de adevăr, nu un derivat din adnotări.

---

## 3. Registru de decizii

| # | Decizie | Alegere | Motiv pe scurt |
|---|---|---|---|
| D01 | Natura produsului | Produs independent, închis, single-tenant | Fără cuplaj cu alte proiecte; multi-tenancy într-un serviciu cu chei = risc disproporționat |
| D02 | Forma clientului Windows | Aceeași aplicație Spring Boot, profil `desktop`, pe `127.0.0.1`, cu tray + dialog PIN | Un singur cod, paritate structurală; CLI și agent derivă din el |
| D03 | Bibliotecă criptografică | DSS 6.x, izolat după adapter `SignatureEngine` | PAdES/XAdES/LTA/LOTL corecte gratuit; reimplementarea = ani de muncă |
| D04 | Model de acces la cheie | `KeyProvider` pluggable: PKCS11, PKCS12, MSCAPI, REMOTE_QTSP | Evită legarea de un furnizor; acoperă și sigiliu, și semnătură de persoană |
| D05 | Nivel juridic | Declarat per profil, validat la pornire | Sigiliul automat e legitim; „semnătură calificată cu PIN din Vault” nu este |
| D06 | Cache PIN (desktop) | Sesiune PKCS#11 vie cu TTL absolut; PIN șters după login; mod `PIN` opțional | Secretul trăiește secunde, nu o oră |
| D07 | Formate | PAdES + XAdES + ASiC-E implementate; CAdES modelat, `NOT_IMPLEMENTED` | Adăugarea unui format e ieftină; adăugarea dimensiunii „format” în API e ruptură |
| D08 | Nivel implicit | B-LT, ridicare la LTA per profil | B-B produce semnături care se „strică” tăcut; LTA implicit e scump |
| D09 | Mecanica API | prepare/complete ca nucleu, one-shot derivat | Singura formă compatibilă cu agentul și cu semnarea la distanță |
| D10 | Execuție | Sincron sub prag configurabil, asincron peste, webhook opțional | Documentele mari și loturile nu încap într-o cerere sincronă |
| D11 | Descărcare de la URL | Doar din listă albă, goală din start; validare post-DNS; fără redirectări | Altfel SSRF; alternativa curată e `POST /documents` |
| D12 | Semnătură vizuală | Poziționare explicită + simbolică în MVP; câmp existent apoi; ancorare pe text ulterior | Ancorarea pe text e ce face serviciul utilizabil cu documente de lungime variabilă |
| D13 | Sursa imaginii | Inline **și** stocată în profil, cu referire pe nume | Integrarea simplă trimite un nume; cea sofisticată trimite tot obiectul |
| D14 | Autoritate de timp | Listă ordonată cu comutare automată, plafon de latență, metrici | Un singur TSA = un singur punct de cădere pentru toată semnarea |
| D15 | LOTL | Reîmprospătare în fundal, cache persistent, niciodată pe calea critică | Descărcarea inițială durează minute; nu poate bloca o semnare |
| D16 | Revocare | Cache cu `nextUpdate`; `FAIL` implicit pe LT/LTA | Un document „LT” fără dovezi de revocare nu e LT |
| D17 | Autentificare | Mod autonom cu chei de API proprii; OIDC/Keycloak activabil; mTLS opțional | Trebuie să funcționeze și fără infrastructură de identitate |
| D18 | Autorizare | Pe profil de semnare (`signer:use:<profil>`) | Altfel cine poate semna o factură poate semna un contract cu cheia directorului |
| D19 | Audit | Lanț de amprente, sigilat temporal periodic, append-only, export verificabil | Diferența dintre „pot verifica eu” și „poate verifica un terț” |
| D20 | Disponibilitatea auditului | Obligatoriu — fără scriere reușită, nu se semnează | O semnătură neînregistrată nu poate fi explicată niciodată |
| D21 | Retenția documentelor | `NONE / HASH_ONLY / FULL` per profil; `HASH_ONLY` implicit | Documentele semnate sunt aproape întotdeauna date personale |
| D22 | Bază de date | PostgreSQL pe server, H2 fișier pe desktop, migrări Flyway comune | Nimeni nu instalează PostgreSQL ca să semneze un PDF |
| D23 | Suprafață API | Semnare, validare, augmentare, semnături multiple, interogare, marcă temporală | Validarea e și infrastructura de testare, deghizată în funcționalitate |
| D24 | Fluxuri cu invitații | În afara scopului, definitiv | Alt produs, altă suprafață de securitate |
| D25 | Erori | Taxonomie proprie cu categorie de reîncercare, în Problem Details (RFC 9457) | Reîncercarea automată la „PIN greșit” blochează cardul |
| D26 | Versionare | `/api/v1` de la primul commit | Costă nimic acum, scump mai târziu |
| D27 | Ambalare Linux | Container + `compose`, cu alternativă `systemd` documentată | Containerul complică accesul la HSM; obligarea lui costă la primul client |
| D28 | Ambalare Windows | MSI prin `jpackage`, JRE inclus, serviciu + tray | Pe stații corporate nu ai voie să instalezi Java |
| D29 | Actualizare | Notificare, descărcare declanșată de utilizator, fără auto-update silențios | Auto-update pe stații unde se semnează calificat = vector de aprovizionare |
| D30 | Secrete | `SecretProvider`: env / file / vault; validare la pornire | Vault rezolvă *unde ții secretul*, nu *cine controlează cheia* |
| D31 | Structură | Multi-modul pe capabilități, granițe ArchUnit | Face verificabilă promisiunea de izolare a DSS |
| D32 | WYSIWYS | Metadate în dialog în MVP; `REQUIRE_CONFIRMATION` per profil; previzualizare ulterior | Cache-ul de PIN nu trebuie să însemne semnare oarbă |
| D33 | Concurență | Serializare per `KeyProvider`, fir dedicat pe token, coadă mărginită | Driverele PKCS#11 eșuează urât sub concurență |
| D34 | PDF criptat | Refuzat categoric | Simplu, curat, fără gestiunea unui secret în plus |
| D35 | Reîmprospătare LTA | Augmentare la cerere în MVP; planificator ca funcționalitate ulterioară | Un planificator care modifică arhiva e un sistem de arhivare, cu greutate proprie |
| D36 | Licență DSS | Dependență separată, fără shading, fără imagine nativă | Singura cale prin care un produs închis folosește LGPL corect |

---

## 4. Scope MVP

MVP-ul este **Etapa 1** din §18: semnarea unui PDF cu tokenul real, pe Windows.

### În MVP

- Nucleu de domeniu + adapter `SignatureEngine` peste DSS.
- `KeyProvider` PKCS#11 (token USB), cu autodetectare a driverului și suprascriere manuală.
- **PAdES-B-LT**, semnare și validare.
- Flux prepare/complete + one-shot sincron.
- Dialog de PIN cu metadatele cererii; cache de sesiune cu TTL, plafon de semnături și
  invalidare pe evenimente.
- Audit înlănțuit, sigilat temporal, în H2; verificator de lanț ca unealtă separată.
- `POST /api/v1/signatures`, `POST /api/v1/signatures/prepare`, `.../complete`,
  `POST /api/v1/validations`, `GET /api/v1/capabilities`, `GET /api/v1/diagnostics`.
- Autentificare cu token local pe `desktop`.
- Profiluri de semnare din configurare, cu validarea nivelului juridic la pornire.
- Listă de TSA-uri cu comutare; LOTL cu cache persistent.
- Ambalare MSI cu `jpackage`, serviciu Windows + tray.
- Testare: PKI generat, SoftHSM2, TSA locală, plus matrice manuală pe tokenul real.

### În afara MVP

- Server Linux, PostgreSQL, Vault, OIDC, HSM, sigiliu automat → Etapa 2.
- XAdES, ASiC-E, augmentare, semnături multiple, loturi, joburi asincrone, webhook,
  semnătură vizuală cu imagine, `POST /documents`, clienți generați → Etapa 3.
- Modul agent, REMOTE_QTSP, `REQUIRE_CONFIRMATION`, ancorare pe text, certificare PDF,
  metrici Prometheus, MSCAPI → Etapa 4.
- Previzualizare WYSIWYS, planificator de reîmprospătare, izolarea parsării, mod izolat,
  CAdES → Etapa 5.
- Fluxuri de semnare cu invitații → **niciodată**.

---

## 5. Model de domeniu

### 5.1 Profilul de semnare — conceptul central

Profilul este obiectul care leagă autentificarea de tot restul. Apelantul trimite un nume de
profil în loc de douăzeci de parametri, iar administratorul controlează politica fără să atingă
integratorii.

```yaml
signer:
  profiles:
    contracte-director:
      legalLevel: QUALIFIED_SIGNATURE      # QUALIFIED_SIGNATURE | QUALIFIED_SEAL
                                           # | ADVANCED_SIGNATURE | ADVANCED_SEAL | BASIC
      keyProvider: token-director
      format: PADES
      level: B_LT
      tsa: tsa-principal
      visualTemplate: director
      revocationPolicy: FAIL
      retain: HASH_ONLY                    # NONE | HASH_ONLY | FULL
      consent: REQUIRE_CONFIRMATION        # NONE | REQUIRE_CONFIRMATION
      maxDocumentSize: 20MB
      maxPages: 500
      allowMultipleSignatures: true
      archiveRefresh: false                # dacă true → retain trebuie să fie FULL
```

**Validarea la pornire** (regula D05, obligatorie): pentru fiecare profil se verifică că
`keyProvider` poate susține `legalLevel` declarat. Un `keyProvider` de tip `PKCS12` nu poate
susține `QUALIFIED_*`. Un `archiveRefresh: true` cu `retain` diferit de `FULL` este o
contradicție. În oricare din cazuri, **aplicația nu pornește** și scrie motivul explicit.

### 5.2 Entități de domeniu

| Entitate | Rol |
|---|---|
| `SigningProfile` | Politica de semnare, încărcată din configurare, validată la pornire |
| `SigningRequest` | Cererea primită: documente, profil, suprascrieri, cheie de idempotență |
| `SigningSession` | Starea unui flux prepare/complete: amprentă, certificat, TTL |
| `SigningJob` | Execuția asincronă: stare, progres, rezultate parțiale |
| `DocumentRef` | Document: conținut inline, referință temporară sau URL din listă albă |
| `SignatureResult` | Rezultat per document: format, nivel atins, amprente, termen de reîmprospătare |
| `AuditEntry` | Intrare de jurnal, cu amprenta intrării precedente |
| `AuditSeal` | Marca temporală aplicată periodic pe capătul lanțului |
| `KeyDescriptor` | Ce cheie s-a folosit: subiect, emitent, serie, valabilitate, tip de dispozitiv |
| `VisualTemplate` | Imagine + blocuri de text cu substituenți |
| `ApiKey` | Cheie de API: hash, expirare, permisiuni, ultima folosire |

### 5.3 Ciclul de viață al unei semnări

```text
  cerere → autentificare → autorizare pe profil → validare intrare (limite, tip, călire)
     → rezervare capacitate (semafor per KeyProvider)
     → prepare: calcul amprentă (SignatureEngine → DSS)
     → obținere semnătură brută (KeyProvider: token / HSM / QTSP)
     → complete: asamblare + marcă temporală + dovezi de revocare
     → validare a rezultatului propriu (semnează → validează)
     → AUDIT (tranzacțional, înainte de răspuns)
     → răspuns
```

Orice eșec pe traseu produce **o intrare de audit cu motivul** și o eroare tipizată. Fișierele
temporare se șterg pe toate căile, inclusiv pe cele de eroare.

---

## 6. Model de date

Migrări **Flyway**, SQL ținut portabil între PostgreSQL și H2. Divergențele se prind prin
rularea aceluiași set de teste pe ambele (§15).

### Audit — nucleul probatoriu

```sql
audit_entry
  id                BIGINT PK          -- secvențial, fără goluri
  occurred_at       TIMESTAMPTZ
  correlation_id    VARCHAR            -- același ca în API și în jurnalele structurate
  operation         VARCHAR            -- SIGN | VALIDATE | EXTEND | TIMESTAMP | ADMIN | ...
  outcome           VARCHAR            -- SUCCESS | FAILURE
  error_code        VARCHAR NULL
  principal         VARCHAR            -- cine a cerut
  principal_type    VARCHAR            -- API_KEY | OIDC | LOCAL | SYSTEM
  remote_addr       VARCHAR NULL
  profile           VARCHAR
  document_name     VARCHAR NULL       -- opțional per profil: poate fi dată personală
  input_digest      VARCHAR            -- SHA-256 al documentului de intrare
  output_digest     VARCHAR NULL       -- SHA-256 al documentului semnat
  signature_format  VARCHAR NULL
  signature_level   VARCHAR NULL       -- nivelul EFECTIV atins, nu cel cerut
  cert_subject      VARCHAR NULL
  cert_issuer       VARCHAR NULL
  cert_serial       VARCHAR NULL
  cert_not_after    TIMESTAMPTZ NULL
  tsa_url           VARCHAR NULL
  tsa_time          TIMESTAMPTZ NULL   -- timpul din marca temporală, nu ceasul local
  pin_cached        BOOLEAN NULL       -- semnarea a folosit un PIN deja deblocat?
  consent_given     BOOLEAN NULL       -- utilizatorul a confirmat explicit?
  duration_ms       INTEGER
  prev_hash         VARCHAR            -- amprenta intrării precedente
  entry_hash        VARCHAR            -- amprenta acestei intrări, incluzând prev_hash

audit_seal
  id                BIGINT PK
  sealed_at         TIMESTAMPTZ
  last_entry_id     BIGINT             -- până unde sigilează
  chain_hash        VARCHAR            -- capătul lanțului la momentul sigilării
  timestamp_token   BYTEA              -- marca temporală RFC 3161
  tsa_url           VARCHAR
```

**Reguli de acces**: nicio cale de cod nu execută `UPDATE` sau `DELETE` pe `audit_entry` sau
`audit_seal`. Pe PostgreSQL, utilizatorul aplicației **nu primește** aceste drepturi — restricția
trăiește în baza de date, nu doar în intenția programatorului.

### Restul schemei

```sql
signing_session   -- id, profile, principal, data_to_sign, cert_info, created_at, expires_at, state
signing_job       -- id, profile, principal, state, total, done, failed, created_at, finished_at,
                  --   webhook_url, idempotency_key
job_item          -- job_id, index, input_digest, state, error_code, output_ref
document_temp     -- id, principal, size, content_type, stored_at, expires_at, storage_ref
api_key           -- id, name, key_hash, permissions, created_at, expires_at, revoked_at, last_used_at
visual_template   -- name, image (BYTEA), image_dpi, layout (JSON), created_at
retained_document -- id, audit_entry_id, profile, storage_ref, retain_until, refresh_due_at
idempotency       -- key, principal, request_digest, response_ref, created_at, expires_at
```

---

## 7. Abstracțiunile cheie

### 7.1 `SignatureEngine` — adapterul peste DSS

```java
public interface SignatureEngine {
    PreparedSignature   prepare(Document doc, SigningParameters params, CertificateChain chain);
    SignedDocument      complete(PreparedSignature prepared, byte[] rawSignature);
    SignedDocument      extend(SignedDocument doc, SignatureLevel target);
    ValidationReport    validate(Document doc, ValidationParameters params);
    SignatureInventory  inspect(Document doc);
    TimestampedDocument timestamp(Document doc);
}
```

Singura implementare în MVP este `DssSignatureEngine`, în modulul `signer-engine-dss`. Niciun
tip DSS nu apare în semnăturile publice — `Document`, `SigningParameters`, `ValidationReport`
sunt tipuri proprii. Regula e verificată cu ArchUnit și pică la compilare.

### 7.2 `KeyProvider` — accesul la materialul criptografic

```java
public interface KeyProvider {
    String           name();
    KeyCapabilities  capabilities();   // ce legalLevel poate susține, maxConcurrency
    CertificateChain certificateChain(KeySelector selector);
    byte[]           sign(byte[] dataToSign, DigestAlgorithm alg, KeySelector selector,
                          UnlockContext unlock);
    HealthStatus     health();
}
```

| Implementare | Context | `legalLevel` maxim posibil |
|---|---|---|
| `Pkcs11KeyProvider` | Token USB sau HSM prin PKCS#11 | `QUALIFIED_*` (dacă dispozitivul e QSCD) |
| `MsCapiKeyProvider` | Windows Certificate Store / CNG | `QUALIFIED_*` (dacă certificatul e pe QSCD) |
| `Pkcs12KeyProvider` | Fișier keystore software | `ADVANCED_*` — **niciodată** calificat |
| `RemoteQtspKeyProvider` | Semnare la distanță la un QTSP | `QUALIFIED_SIGNATURE` |

`capabilities()` este ce alimentează validarea de la pornire (D05). Un `Pkcs12KeyProvider` care
raportează onest că nu poate susține `QUALIFIED_*` face ca un profil greșit configurat să
oprească aplicația, nu să producă documente înșelătoare.

**`UnlockContext`** e mecanismul prin care se obține deblocarea: pe `desktop` declanșează
dialogul sau folosește sesiunea deja deblocată; pe server rezolvă secretul prin `SecretProvider`;
la QTSP declanșează fluxul de autorizare al furnizorului.

### 7.3 `SecretProvider`

```java
public interface SecretProvider {
    Secret resolve(SecretRef ref);   // Secret se auto-maschează la toString()
    void   subscribe(SecretRef ref, Consumer<Secret> onRotate);
}
```

Referire uniformă în configurare, indiferent de sursă:

```yaml
pin: "${secret:vault:kv/signer/token-pin}"
pin: "${secret:env:SIGNER_TOKEN_PIN}"
pin: "${secret:file:/run/secrets/token-pin}"
```

Implementări în MVP/Etapa 2: `env`, `file`, `vault` (Spring Cloud Vault). Loc rezervat pentru
Windows DPAPI și pentru furnizorii cloud. **Pe `desktop` nu există secrete în configurare** —
`SecretProvider` servește acolo doar tokenul local de API.

### 7.4 `TrustServices`

```java
public interface TimestampService {      // listă ordonată, comutare la eșec sau latență
    TimestampToken timestamp(byte[] digest, DigestAlgorithm alg);
}
public interface TrustListService {      // LOTL/TSL, reîmprospătat în fundal
    TrustStatus status(CertificateChain chain);
    LotlHealth  health();
}
public interface RevocationService {     // OCSP/CRL cu cache pe nextUpdate
    RevocationData fetch(CertificateChain chain, RevocationPolicy policy);
}
```

---

## 8. PIN, sesiune și consimțământ (profilul `desktop`)

Zona cu cele mai multe capcane din tot produsul.

### 8.1 Strategia implicită: sesiune vie, PIN uitat

1. Utilizatorul introduce PIN-ul în dialogul nativ.
2. Se deschide sesiunea PKCS#11 și se face login.
3. **PIN-ul este suprascris în memorie imediat** (`char[]`, zeroizat — niciodată `String`).
4. Sesiunea rămâne deschisă până la expirarea TTL sau până la un eveniment de invalidare.

Modul alternativ, `pin-cache.mode=PIN`, păstrează PIN-ul în memorie pentru a putea reface
login-ul transparent după invalidarea sesiunii de către driver. Se activează explicit de
administrator și se înregistrează în audit (`pin_cached`).

### 8.2 Reguli, indiferent de mod

| Regulă | Valoare implicită | Motiv |
|---|---|---|
| TTL **absolut**, nu glisant | 1h | Glisant = token deblocat la nesfârșit dacă se semnează periodic |
| Plafon de semnături per deblocare | 100 | Limită independentă de timp |
| Invalidare la scoaterea tokenului | da | Evident, și detectabil |
| Invalidare la blocarea stației / suspendare | da | Utilizatorul nu mai e prezent |
| Invalidare la ieșirea din aplicație | da | Cache-ul moare cu procesul |
| Buton „Uită PIN-ul” în tray | da | Control explicit al utilizatorului |
| Persistare pe disc | **niciodată** | Nici criptat, nici prin DPAPI |
| PIN ca parametru REST | **niciodată** | Altfel orice proces local care găsește portul poate semna |
| Bifa „ține minte 1 oră” | **nebifată** din start | Consimțământ, nu comoditate impusă |
| Oprire după 2 PIN-uri greșite | da, cu avertisment | Al treilea blochează definitiv cardul |

### 8.3 Consimțământ separat de cache (`consent: REQUIRE_CONFIRMATION`)

Cu acest mod activat, PIN-ul rămâne cache-uit, dar **fiecare semnare cere un clic pe „Aprob”**,
cu metadatele cererii afișate: cine cere, ce fișier, ce amprentă, ce profil, ce certificat.

Excepția necesară: la **semnare în lot**, confirmarea este **una singură pentru tot lotul**, cu
lista documentelor afișată. Utilizatorul aprobă „aceste 40 de documente”, nu de 40 de ori.

Raționamentul: ce voia utilizatorul când a bifat „ține minte” a fost *să nu retasteze PIN-ul de
40 de ori*, nu *să semneze orbește orice cerere din ora următoare*.

### 8.4 Avertisment pentru previzualizare (Etapa 5)

Un PDF poate afișa conținut diferit în funcție de vizualizator, de fonturi lipsă, de conținut
opțional sau de JavaScript. **O previzualizare făcută superficial creează o falsă siguranță** —
mai onest fără previzualizare decât cu una care poate fi păcălită. De aceea e planificată ca
etapă separată, făcută atent, nu strecurată în MVP.

---

## 9. API REST

`/api/v1`, OpenAPI 3.1 scris întâi. Adăugarea de câmpuri nu rupe; eliminarea și schimbarea
semanticii da. Clienții **trebuie** să ignore câmpurile necunoscute.

### 9.1 Operații

| Metodă | Cale | Etapa | Descriere |
|---|---|---|---|
| `POST` | `/signatures` | 1 | Semnare dintr-o bucată; sincron sub prag, altfel `202` cu `jobId` |
| `POST` | `/signatures/prepare` | 1 | Întoarce `dataToSign` + `sessionId` + certificatul folosit |
| `POST` | `/signatures/complete` | 1 | Primește semnătura brută, întoarce documentul asamblat |
| `POST` | `/validations` | 1 | Verdict complet asupra unui document semnat |
| `GET` | `/capabilities` | 1 | Ce poate **această instalare**: formate, niveluri, profiluri, nivel juridic |
| `GET` | `/diagnostics` | 1 | Tokenuri, certificate, drivere, TSA-uri, LOTL, versiuni (`signer:admin`) |
| `POST` | `/signatures/batch` | 3 | Lot, cu o singură deblocare, succes parțial |
| `GET` | `/jobs/{id}` | 3 | Starea unui job asincron |
| `POST` | `/extensions` | 3 | Ridicarea unui document semnat la un nivel superior |
| `POST` | `/timestamps` | 3 | Marcă temporală pe un document nesemnat |
| `GET` | `/documents/{id}/signatures` | 3 | Ce semnături poartă documentul |
| `POST` | `/documents` | 3 | Încărcare temporară, referibilă în operații ulterioare |
| `GET`/`POST` | `/admin/api-keys` | 2 | Ciclu de viață al cheilor de API |
| `GET` | `/admin/audit` | 2 | Interogare și export verificabil al jurnalului |

### 9.2 Forma cererii

```jsonc
POST /api/v1/signatures
{
  "profile": "contracte-director",
  "documents": [
    { "name": "contract-1234.pdf", "content": "<base64>" },
    { "name": "anexa.pdf", "documentId": "tmp_01J..." },        // din POST /documents
    { "name": "raport.pdf", "url": "https://minio.intern/..." } // doar din listă albă
  ],
  "overrides": {                       // opțional; profilul rămâne plafonul
    "level": "B_LTA",
    "reason": "Aprobare contract",
    "location": "București",
    "visual": {
      "template": "director",
      "page": "LAST",
      "position": { "anchor": "BOTTOM_RIGHT", "marginPt": 20 },
      "text": "${signerName}\n${signingTime}\n${reason}"
    }
  },
  "async": false,
  "webhookUrl": "https://client.intern/hooks/semnare"
}
```

Suprascrierile **nu pot depăși** politica profilului: nu pot cere un `legalLevel` mai mare, nu
pot dezactiva verificarea revocării, nu pot schimba furnizorul de chei. Ce e plafonat rămâne
plafonat.

Transport alternativ: `multipart/form-data` pentru documente mari.

Antet `Idempotency-Key`: o reîncercare după un timeout de rețea întoarce rezultatul original, nu
produce a doua semnătură și nu consumă a doua marcă temporală.

### 9.3 Forma răspunsului

```jsonc
{
  "requestId": "req_01J...",
  "results": [
    {
      "name": "contract-1234.pdf",
      "status": "SIGNED",
      "format": "PADES",
      "level": "B_LT",                        // nivelul EFECTIV atins
      "inputDigest": "sha256:...",
      "outputDigest": "sha256:...",
      "signedAt": "2026-09-13T11:02:41Z",     // din marca temporală, nu din ceasul local
      "signer": { "subject": "CN=...", "issuer": "CN=...", "serial": "...",
                  "qualified": true, "notAfter": "2027-04-02T00:00:00Z" },
      "archiveRefreshDueBy": null,            // completat pentru B_LTA
      "auditEntryId": 148233,
      "content": "<base64>"
    },
    {
      "name": "anexa.pdf",
      "status": "FAILED",
      "error": { "code": "SIGNER.PDF_ENCRYPTED", "retryable": false,
                 "requiresHumanAction": false,
                 "message": "Documentul este protejat cu parolă și nu poate fi semnat." }
    }
  ],
  "summary": { "total": 2, "signed": 1, "failed": 1 }
}
```

Un lot **nu pică în întregime** din cauza unui document invalid.

### 9.4 Taxonomia erorilor

Format *Problem Details* (RFC 9457), îmbogățit cu cod stabil, categorie de reîncercare, mesaj
tehnic, mesaj afișabil în română și `correlationId` regăsibil identic în audit.

| Cod | `retryable` | `requiresHumanAction` | Situație |
|---|---|---|---|
| `SIGNER.PIN_INVALID` | **false** | true | PIN greșit — reîncercarea automată blochează cardul |
| `SIGNER.PIN_REQUIRED` | false | true | Sesiune expirată, e nevoie de deblocare |
| `SIGNER.TOKEN_ABSENT` | false | true | Tokenul nu e conectat |
| `SIGNER.TOKEN_LOCKED` | false | true | Card blocat — necesită furnizorul |
| `SIGNER.CERT_EXPIRED` | false | true | Certificat expirat |
| `SIGNER.CERT_REVOKED` | false | true | Certificat revocat |
| `SIGNER.PROFILE_FORBIDDEN` | false | false | Apelantul nu are dreptul pe acest profil |
| `SIGNER.PDF_ENCRYPTED` | false | false | PDF protejat cu parolă (D34) |
| `SIGNER.PDF_MALFORMED` | false | false | Document ilizibil |
| `SIGNER.DOCUMENT_TOO_LARGE` | false | false | Peste limitele profilului |
| `SIGNER.EXISTING_SIGNATURE_BROKEN` | false | false | Semnarea ar invalida semnături existente |
| `SIGNER.URL_NOT_ALLOWED` | false | false | Origine în afara listei albe |
| `SIGNER.TSA_UNAVAILABLE` | **true** | false | Toate TSA-urile configurate au eșuat |
| `SIGNER.REVOCATION_UNAVAILABLE` | true | false | OCSP/CRL inaccesibil, politica e `FAIL` |
| `SIGNER.CAPACITY_EXCEEDED` | true | false | Coadă plină sau timp de așteptare depășit |
| `SIGNER.AUDIT_UNAVAILABLE` | true | false | Jurnalul nu poate fi scris → nu se semnează (D20) |
| `SIGNER.SESSION_EXPIRED` | false | false | Sesiunea prepare/complete a expirat |

**Regulă**: fiecare cod apare în documentație cu ce înseamnă și ce trebuie făcut. Un cod
nedocumentat este un bug.

### 9.5 Reguli de robustețe

- Sesiunea prepare/complete expiră (implicit 5 min, configurabil).
- Documentele temporare expiră (implicit 1h), se curăță automat și **nu intră niciodată** în
  retenția profilului.
- Așteptarea la coadă nu blochează un fir HTTP — de aceea contează pragul sincron/asincron.
- Cotă și limitare de rată per cheie de API, configurabile.
- Webhook cu reîncercări exponențiale, semnat (HMAC) ca destinatarul să verifice originea.

---

## 10. Semnătură vizuală

Caseta vizuală **nu are valoare juridică** — e decor peste semnătura criptografică. Serviciul nu
trebuie să lase niciodată impresia că „fără imagine nu e semnat”. Această frază intră și în
documentația publică.

### Poziționare

| Mod | Etapa | Formă |
|---|---|---|
| Explicit | 3 | `{ "page": 3, "x": 380, "y": 60, "width": 160, "height": 60 }` |
| Simbolic | 3 | `{ "page": "LAST", "anchor": "BOTTOM_RIGHT", "marginPt": 20 }` |
| Câmp existent | 4 | `{ "fieldName": "SemnaturaDirector" }` |
| Ancorare pe text | 4 | `{ "textAnchor": "{{SEMNATURA_DIRECTOR}}", "offsetPt": [0, -10] }` |

Ancorarea pe text e ce transformă serviciul din „semnează acest PDF” în „semnează orice document
generat de sistemul meu” — singura care funcționează când documentul are lungime imprevizibilă.

### Conținut

`VisualTemplate` = imagine (opțional) + blocuri de text cu substituenți: `${signerName}`,
`${signingTime}`, `${reason}`, `${location}`, `${certificateIssuer}`, `${certificateSerial}`.
Definit o dată în profil, suprascriibil per apel. Integrarea simplă trimite
`"visual": "director"`; cea sofisticată trimite tot obiectul.

Imaginea poate fi trimisă inline sau stocată în profil și referită pe nume. **Validarea DPI la
încărcare**: sub un prag configurabil se emite avertisment explicit — o imagine de 200×80 px
întinsă pe 5 cm arată lamentabil la tipărire.

---

## 11. Audit

### Ce se înregistrează

Toate câmpurile din §6, pentru **fiecare** operație, **inclusiv eșecurile** — PIN greșit, profil
interzis, certificat expirat. Tiparele de atac se văd în eșecuri, nu în reușite.

**Nu se înregistrează niciodată**: conținutul documentului, PIN-uri, parole, material
criptografic. Numele fișierului e opțional per profil, fiindcă poate fi el însuși informație
personală (`demisie-Popescu-Ion.pdf`).

### Lanțul și sigilarea

```text
entry[n].prev_hash  = entry[n-1].entry_hash
entry[n].entry_hash = SHA-256( canonical(entry[n] fără entry_hash) )

la fiecare N intrări sau X minute:
  seal = TSA( chain_hash )   →  audit_seal
```

Sigilarea temporală transformă jurnalul din „consistent intern” în **probă cu dată certă**:
diferența dintre „pot verifica eu” și „poate verifica un terț”. Refolosește exact clientul de TSA
existent, deci costă foarte puțin peste simpla înlănțuire.

### Garanții operaționale

- Scriere **în aceeași tranzacție** cu operația, înainte ca răspunsul să plece.
- **Fără audit scris, nu se semnează** (D20). Baza de date devine dependență critică — asumat
  conștient: o semnătură neînregistrată e mai periculoasă decât una care n-a avut loc.
- Append-only la nivel de aplicație **și** de drepturi în baza de date.
- Fără ștergere automată. Ștergerea, dacă e vreodată necesară, se face prin comandă explicită de
  administrare, care **se auto-înregistrează** în jurnal.
- **Export verificabil**: un interval de jurnal + lanțul + mărcile temporale, plus un verificator
  autonom care confirmă integritatea **fără acces la aplicație sau la baza de date**. Un audit pe
  care nu-l poți da nimănui nu e audit, e log.
- **Copiile de siguranță se verifică prin restaurare reală**, periodic. O copie nerestaurată
  niciodată nu e o copie, e o speranță.
- Pe `desktop`: aceleași reguli, în H2 local. Avertisment de scris în documentație — *un audit
  care trăiește doar pe laptopul directorului este un audit cu o singură viață*.

---

## 12. Securitate

### 12.1 Autentificare și autorizare

Modelul intern de identitate (`principal` + permisiuni) este **al nostru**; OIDC este doar o
sursă de identitate care se mapează în el. Altfel ajungi cu două sisteme de autorizare paralele
care diverg.

| Mod | Context | Note |
|---|---|---|
| Chei de API | Autonom, fără dependențe | Stocate ca hash, cu expirare, revocabile, cu ultima folosire |
| OIDC / JWT | Cu Keycloak instalat | Activabil din configurare, opțional |
| mTLS | Server-to-server strict | Opțional |
| Token local | `desktop` | Generat la pornire, fișier accesibil doar utilizatorului curent |

Permisiuni: `signer:use:<profil>`, `signer:validate`, `signer:admin`.

Tokenul local de pe `desktop` nu e formalism: fără el, orice proces de pe stație care găsește
portul poate cere o semnătură — iar cu PIN-ul cache-uit, o obține.

### 12.2 Călirea intrărilor

| Amenințare | Măsură |
|---|---|
| PDF criptat | **Refuzat categoric** (D34) |
| PDF cu restricții care interzic semnarea | Refuzat — altfel încalci intenția emitentului |
| PDF deja semnat | Obligatoriu actualizare incrementală + **verificare după semnare** că semnăturile preexistente au rămas valide; dacă nu, operația eșuează și nu întoarce nimic |
| XXE în XML | Parsere cu DTD și entități externe dezactivate, **fără opțiune de activare nicăieri** |
| Bombe de decompresie | Plafon pe dimensiunea despachetată, raport maxim de compresie |
| PDF malformat intenționat | Timp maxim de procesare, memorie maximă, plafon de pagini |
| Document uriaș | Limită de dimensiune per profil, aplicată la intrare |
| Tip fals | Validare pe conținut, nu pe extensie sau pe `Content-Type` |
| SSRF prin `url` | Listă albă goală din start, validare **după** rezolvarea DNS, fără redirectări, plafon de descărcare |
| Scurgere prin `/diagnostics` | Cere `signer:admin`, dezactivabil complet |
| Secrete în jurnale | Tipuri auto-mascate la `toString()`, excluse din `/actuator/env` și din rapoarte |
| Fișiere temporare rămase | Curățare garantată **și pe calea de eroare** |

Îmbunătățire ulterioară (Etapa 5): procesarea documentelor într-un **proces separat**, ca o
cădere a parserului să nu ia cu ea serviciul și accesul la chei.

### 12.3 Recuperare

Cheile **nu se salvează** — tot rostul unui QSCD e că materialul criptografic nu poate fi extras.
Planul de recuperare pentru un token pierdut sau blocat nu este tehnic, ci procedural: certificat
nou de la furnizor. Se scrie explicit în documentația de operare, fiindcă e genul de lucru pe
care oamenii îl descoperă în ziua proastă.

---

## 13. Concurență și capacitate

Un token USB este un dispozitiv **serial**: o operație pe rând, câteva semnături pe secundă. Un
HSM execută zeci sau sute în paralel. API-ul primește, însă, oricâte cereri simultane.

- **Semafor per `KeyProvider`**, cu `maxConcurrency` declarat (implicit 1 pentru PKCS#11 pe
  token, configurabil pentru HSM).
- **Fir dedicat per token**: toate operațiile trec prin el, cu cereri puse în coadă. Multe
  drivere PKCS#11 nu sunt sigure sub concurență nici când pretind că sunt; costul e de
  microsecunde, beneficiul e imunitatea la o categorie întreagă de erori imposibil de reprodus.
- **Coadă mărginită**, cu lungime maximă și timp maxim de așteptare. La depășire:
  `SIGNER.CAPACITY_EXCEEDED`, temporar-reîncercabil. O coadă nemărginită nu e o soluție, e un mod
  mai lent de a cădea.
- **Benzi separate**: cererile sincrone au prioritate; joburile asincrone consumă capacitatea
  rămasă. Un lot de 500 de documente nu înfometează o cerere interactivă.
- **Fără reîncercare automată pe operația de card** — reîncercarea oarbă poate consuma o
  încercare de PIN. Se reîncearcă automat doar ce e demonstrabil fără efect asupra
  dispozitivului: apeluri TSA, OCSP, descărcări.

---

## 14. Cross-cutting

### Configurare

Ierarhie: valori implicite din cod → fișier de configurare → variabile de mediu → suprascrieri
per profil. Validată integral **la pornire**: profiluri, furnizori de chei, TSA-uri, secrete,
coerența `legalLevel` ↔ `keyProvider`, coerența `archiveRefresh` ↔ `retain`. Orice inconsistență
oprește pornirea, cu motivul scris explicit. **Nu** se pornește ca să se eșueze la prima semnare,
la trei zile după instalare.

### Observabilitate

**Sănătate cu semnificație reală**, nu „procesul trăiește”: bază de date accesibilă; fiecare TSA
răspunde; vechimea cache-ului LOTL sub prag; tokenul/HSM prezent și certificatul găsit;
**certificatul de semnare nu expiră în mai puțin de N zile** (implicit 30). Separate pe
`liveness` și `readiness`, ca un TSA căzut să nu provoace repornirea aplicației. Degradarea bazei
de date scoate serviciul din rotație **înainte** de a eșua cereri.

**Metrici** (Etapa 4, Micrometer/Prometheus): semnături pe format × nivel × profil × rezultat;
durata semnării **descompusă pe etape** (acces la cheie, TSA, revocare, asamblare); rata de eșec
pe TSA; vârsta LOTL; deblocări și rata de PIN greșit; lungimea cozii. Descompunerea pe etape e ce
spune *de ce* a durat 9 secunde.

**Jurnalizare structurată** (JSON) cu `correlationId` propagat din API până în audit, astfel încât
„semnarea mea de marți la 14:30 a eșuat” să fie o singură interogare.

Alertarea rămâne în seama infrastructurii clientului; produsul expune semnalele.

### Internaționalizare

Mesajele tehnice în engleză (coduri, jurnale), mesajele afișabile utilizatorului în **română și
engleză**. Dialogul de PIN și tray-ul respectă limba sistemului.

---

## 15. Testare

Cea mai subestimată problemă a produsului: nu poți pune un token calificat într-o rulare
automată, și nu poți cere o marcă temporală reală la fiecare test.

| Componentă | Soluție | De ce |
|---|---|---|
| PKI de test | **Generat la rulare**: CA rădăcină, intermediară, certificate valide / expirate / revocate / cu utilizări greșite | Certificatele comise în depozit expiră și pică testele peste un an fără ca nimeni să fi schimbat cod |
| Token PKCS#11 | **SoftHSM2** | Testezi *exact* aceeași cale de cod — login, sesiuni, PIN greșit, blocare |
| Autoritate de timp | **TSA locală** în teste | Altfel testele sunt lente, capricioase și dependente de internet |
| Bază de date | **Testcontainers** (PostgreSQL) + același set rulat pe H2 | Prinde divergențele de portabilitate din D22 |
| Corectitudinea semnăturii | **Semnează → validează**, la toate combinațiile format × nivel | Testul-far al întregului produs |
| Verificare externă | Adobe Acrobat, măcar manual, în fiecare etapă | Validatorul propriu și semnatarul propriu împart aceleași presupuneri, deci pot fi greșiți împreună |
| Hardware real | **Matrice manuală documentată**: ce token, ce driver, ce sistem, ce s-a verificat, când | Nu se automatizează, dar se face repetabil |
| Granițe de module | **ArchUnit** | `eu.europa.esig.*` doar în `signer-engine-dss`; `signer-core` fără dependențe |
| Contract API | Teste generate din OpenAPI | Prind rupturile înainte să ajungă la integratori |

Teste care trebuie să existe explicit, fiindcă acoperă exact promisiunile riscante:

- semnarea unui PDF deja semnat **nu invalidează** semnătura precedentă;
- un profil cu `legalLevel: QUALIFIED_*` și `keyProvider: PKCS12` **oprește pornirea**;
- un `archiveRefresh: true` cu `retain: HASH_ONLY` **oprește pornirea**;
- semnarea cu auditul indisponibil **eșuează**, nu produce document;
- lanțul de audit rupt artificial e **detectat** de verificatorul autonom;
- PIN greșit **nu** declanșează reîncercare automată;
- scoaterea tokenului în timpul unui lot produce eroare curată, nu corupție;
- un XML cu entitate externă **nu** declanșează nicio cerere de rețea sau de fișier.

---

## 16. Ambalare și distribuție

### Linux (server)

Formă principală: **imagine de container**, cu `docker compose` pentru instalare completă
(aplicație + PostgreSQL). Alternativă documentată: arhivă cu unitate `systemd`, pentru cazurile
cu HSM unde expunerea dispozitivului și a driverului PKCS#11 în container complică lucrurile. Nu
se oferă *doar* container: la un serviciu care atinge hardware criptografic, obligarea
containerului costă la primul client cu HSM.

### Windows (client)

**MSI prin `jpackage`**, cu JRE inclus — pe stații corporate nu ai voie să instalezi Java
separat. Instalat ca serviciu Windows pornit la logon, cu tray icon și dialog JavaFX pentru PIN.

**Actualizare**: verificare de versiune la pornire, cu notificare; descărcare declanșată de
utilizator; verificarea amprentei pachetului. **Fără auto-update silențios** — un produs care se
auto-actualizează pe stații unde se semnează documente calificate este un vector de aprovizionare
pe care nu-l construim.

**Semnarea instalatorului** cu certificat de semnare de cod (OV sau EV) este **obligatorie**
înainte de orice distribuție reală. Fără ea, SmartScreen avertizează la fiecare instalare că
aplicația e nesigură — ironie greu de explicat la un produs de semnare electronică. Cost real:
certificat cu plată anuală, iar EV cere token hardware. Se planifică, nu se descoperă.

**Driver PKCS#11**: autodetectare din căile cunoscute (SafeNet, Gemalto/Thales, Oberthur,
Bit4id etc.), cu suprascriere manuală în configurare, plus `/diagnostics` care listează ce
tokenuri, ce certificate și ce drivere vede efectiv. Din experiență, majoritatea suportului unui
astfel de produs este „nu-mi vede tokenul”; un diagnostic bun transformă ora de depanare în trei
minute.

---

## 17. Licențiere

Produsul este **închis**. DSS este **LGPL-2.1**. Combinația este permisă, cu condiții care în
Java au consecințe tehnice concrete. Esența LGPL: codul propriu rămâne propriu, dar utilizatorul
trebuie să poată **înlocui biblioteca** cu o versiune proprie.

Reguli de construcție, verificabile automat:

1. **Fără shading / relocare de pachete** pe dependențele LGPL. JAR-ul executabil Spring Boot e
   în regulă — păstrează dependențele ca JAR-uri separate în `BOOT-INF/lib`.
2. **Fără `native-image`** pe modulele care ating DSS. Legarea statică iese din zona confortabilă
   a LGPL. Dacă vreodată se dorește pornire instantanee pe desktop, se reanalizează separat.
3. **Fără modificarea DSS.** Dacă devine necesară, modificările se publică sub LGPL — doar ele,
   nu restul produsului.
4. **Obligații de însoțire**: menționarea bibliotecii și a licenței, textul LGPL, link către
   versiunea exactă a sursei.
5. **Raport de licențe generat la fiecare build**, cu **pică** pe licențe neagreate (GPL pur,
   AGPL, licențe necunoscute). Apără de scenariul în care o dependență tranzitivă aduce GPL în
   produsul închis fără ca nimeni să observe.

Izolarea DSS într-un singur modul (D03) devine astfel și o proprietate juridică: dacă apare
vreodată un client cu politică strictă de licențe sau o verificare de achiziție, se înlocuiește
**un modul**, nu produsul.

> Acest paragraf nu este consultanță juridică. Practica descrisă este cea larg acceptată în
> ecosistemul Java pentru biblioteci LGPL. Înainte de comercializare serioasă, o verificare
> juridică de o oră pe acest punct e bine cheltuită.

---

## 18. Plan pe etape

Principiul ordonării: **fiecare etapă se termină cu ceva verificabil de un singur dezvoltator,
cu hardware real, fără dependență de terți**.

### Etapa 0 — Spike-uri — *1–2 săpt.*

Fără cod de producție. Trei explorări cu răspuns clar:

1. **PKCS#11 cu tokenul real**: enumerarea slot-urilor și a certificatelor; login și semnare;
   comportamentul la scoaterea tokenului, la blocarea stației, la suspendare/revenire; cât
   trăiește efectiv o sesiune; ce erori întoarce driverul și cât de descriptive sunt; cum se
   comportă la a doua și a treia încercare de PIN greșit (**fără** a atinge a treia pe cardul de
   producție — cu card de test sau SoftHSM2 pentru acest caz).
2. **`jpackage` + serviciu Windows + tray + dialog JavaFX**: MSI care se instalează, pornește la
   logon, afișează tray, deschide un dialog modal deasupra altor ferestre, se dezinstalează
   curat.
3. **Compatibilitate DSS 6.x cu versiunea de Spring Boot aleasă** (jakarta, conflicte de
   dependențe, BouncyCastle tranzitiv).

*Ieșire:* notă scrisă cu ce s-a confirmat și ce s-a infirmat. Dacă ceva nu merge cum presupunem,
arhitectura se ajustează **înainte** de Etapa 1.

### Etapa 1 — MVP desktop: „semnez un PDF cu tokenul meu” — *4–6 săpt.*

- `signer-core`: model de domeniu, porturi, profiluri, validarea de pornire (D05).
- `signer-engine-dss`: `prepare` / `complete` / `validate` pentru PAdES; granița ArchUnit.
- `signer-keyprovider`: `Pkcs11KeyProvider`, autodetectare driver, semafor + fir dedicat.
- `signer-trust`: TSA cu listă și comutare; LOTL cu cache persistent și reîmprospătare în fundal.
- `signer-audit`: lanț de amprente, sigilare temporală, verificator autonom.
- `signer-persistence`: H2 + Flyway.
- `signer-api`: OpenAPI, `/signatures`, `/prepare`, `/complete`, `/validations`,
  `/capabilities`, `/diagnostics`; taxonomia de erori; token local.
- `signer-app-desktop`: tray, dialog PIN cu metadate, cache de sesiune cu toate regulile din
  §8.2.
- `signer-testkit`: PKI generat, SoftHSM2, TSA locală.
- MSI prin `jpackage`.

*Criteriu de terminare:* semnezi un PDF cu tokenul tău, Adobe Acrobat îl afișează ca valid și
calificat, validatorul propriu confirmă, iar jurnalul de audit poate dovedi operația — inclusiv
după restaurarea dintr-o copie de siguranță.

### Etapa 2 — Server Linux — *4–5 săpt.*

- `signer-app-server`, PostgreSQL, Flyway pe ambele baze, `signer-secrets` (env/file/vault).
- `Pkcs12KeyProvider` și `Pkcs11KeyProvider` către HSM.
- Chei de API cu ciclu de viață complet; OIDC/Keycloak opțional; autorizare pe profil.
- Sănătate, jurnalizare structurată, `/admin/audit`.
- Container + `compose`; alternativă `systemd` documentată.
- Mod izolat (LOTL din fișier, TSA intern) — schelet funcțional.

*Criteriu:* sigiliu electronic automat pe server, fără nicio intervenție umană, cu secretele în
Vault și audit complet. Semnarea eșuează curat când baza de date e indisponibilă.

### Etapa 3 — API bogat — *4–6 săpt.*

- XAdES (enveloped/detached), ASiC-E.
- Augmentare (`/extensions`), semnături multiple, `/documents/{id}/signatures`, `/timestamps`.
- Loturi cu o singură deblocare și succes parțial; joburi asincrone; webhook semnat.
- `POST /documents` cu TTL; listă albă pentru `url`.
- **Semnătură vizuală**: poziționare explicită și simbolică, `VisualTemplate` cu imagine și
  substituenți, validare DPI.
- Idempotență, cote, limitare de rată.
- Clienți generați din OpenAPI (Java, TypeScript, Python) + documentație de integrare.

*Criteriu:* un integrator poate construi peste serviciu citind doar specificația, fără să întrebe
nimic.

### Etapa 4 — Maturizare — *4–6 săpt.*

- Modul `signer-agent` (long-poll pe amprentă) + `signer-cli`.
- `RemoteQtspKeyProvider` pentru semnare la distanță.
- Mod `consent: REQUIRE_CONFIRMATION`, cu confirmare unică pe lot.
- Ancorare vizuală pe text; câmpuri de semnătură existente; certificare PDF.
- `MsCapiKeyProvider`.
- Metrici Prometheus, cu descompunerea duratei pe etape.

### Etapa 5 — Termen lung

- Previzualizare WYSIWYS reală, făcută atent (§8.4).
- Planificator de reîmprospătare a arhivei LTA, ca **componentă distinctă** cu versionare,
  gestiunea eșecurilor parțiale și lanț de custodie.
- Izolarea parsării documentelor în proces separat.
- Mod complet izolat, fără ieșire la internet.
- CAdES.

---

## 19. Scenariu de acceptanță („MVP gata”)

1. Instalezi MSI-ul pe o stație Windows curată, fără Java preinstalat. Serviciul pornește,
   tray-ul apare.
2. Conectezi tokenul. `/diagnostics` îl listează, cu certificatul și driverul detectat.
3. `POST /api/v1/signatures` cu un PDF și profilul `implicit`. Apare dialogul de PIN cu numele
   fișierului, amprenta și certificatul. Bifezi „ține minte 1 oră”.
4. Primești PDF-ul semnat. Adobe Acrobat afișează semnătura ca **validă** și **calificată**.
5. `POST /api/v1/validations` pe documentul rezultat confirmă: `B_LT`, lanț de încredere complet,
   certificat calificat, marcă temporală prezentă.
6. A doua semnare, în interiorul orei, **nu** mai cere PIN.
7. Scoți tokenul. Următoarea semnare eșuează cu `SIGNER.TOKEN_ABSENT`; cache-ul e invalidat.
8. Reconectezi tokenul: se cere din nou PIN-ul.
9. Introduci un PIN greșit: primești `SIGNER.PIN_INVALID` cu `retryable: false` și un avertisment
   despre numărul de încercări rămase. Nu se face nicio reîncercare automată.
10. Oprești baza de date. Semnarea eșuează cu `SIGNER.AUDIT_UNAVAILABLE`. **Niciun document
    semnat nu este returnat.**
11. Exporți jurnalul de audit. Verificatorul autonom confirmă integritatea lanțului și
    validitatea mărcilor temporale, fără acces la aplicație.
12. Modifici manual o intrare în baza de date. Verificatorul **detectează** ruptura și indică
    intrarea.
13. Configurezi un profil `QUALIFIED_SIGNATURE` cu `keyProvider` de tip `PKCS12`. Aplicația
    **refuză să pornească**, cu mesaj explicit.
14. Trimiți un PDF criptat: `SIGNER.PDF_ENCRYPTED`. Trimiți un PDF de 200 MB:
    `SIGNER.DOCUMENT_TOO_LARGE`. Ambele apar în audit ca eșecuri.

---

## 20. Riscuri și necunoscute

| Risc | Impact | Atenuare |
|---|---|---|
| Drivere PKCS#11 de calitate variabilă | Erori imposibil de reprodus, blocaje | Fir dedicat per token; Etapa 0 pe hardware real; matrice manuală de testare |
| Sesiunea PKCS#11 invalidată de evenimente de sistem | Eșecuri aparent aleatorii pe desktop | Detectare explicită + re-cerere de PIN; mod `PIN` ca alternativă asumată |
| DSS schimbă API-ul între versiuni majore | Efort de migrare | Adapter + ArchUnit: impactul e confinat într-un modul |
| LOTL mare și lent la prima descărcare | Pornire lentă, senzație de produs blocat | Cache persistent + reîmprospătare în fundal, niciodată pe calea critică |
| TSA public lent sau instabil | Semnări care expiră, loturi eșuate | Listă cu comutare, plafon de latență, metrici pe rata de eșec |
| Semnarea invalidează semnături preexistente | Distrugi dovada altcuiva | Actualizare incrementală + verificare obligatorie după semnare, testată automat |
| Confuzie „calificat vs. avansat” la client | Documente fără valoarea juridică așteptată | `legalLevel` declarat, validat la pornire, prezent în `/capabilities` și în audit |
| Certificat de semnare de cod pentru MSI | Blocaj la distribuție, cost neanticipat | Planificat explicit în §16, nu descoperit la lansare |
| Portabilitate SQL PostgreSQL ↔ H2 | Divergențe descoperite târziu | Același set de teste rulat pe ambele, din Etapa 2 |
| Construcția Windows cere Windows | CI pe două sisteme | Asumat: Linux pentru nucleu și server, Windows pentru ambalaj, MSCAPI și token |
| Produs speculativ, fără client | Se construiește ce nu trebuie | MVP tăiat ca **unealtă proprie utilizabilă**: chiar fără client, produce valoare pentru dezvoltator |
| Necunoscut: API-urile QTSP de semnare la distanță | Efort greu de estimat pentru Etapa 4 | Spike separat la începutul Etapei 4, după obținerea unui cont de test |

---

## 21. Stack tehnic (referință)

```text
Limbaj        Java 21
Backend       Spring Boot (versiune fixată în Etapa 0, după verificarea compatibilității DSS)
Semnare       DSS 6.x (eu.europa.ec.joinup.sd-dss) – dss-pades, dss-xades, dss-asic-xades,
              dss-token, dss-service, dss-tsl-validation, dss-validation
Cripto        BouncyCastle (tranzitiv prin DSS), PKCS#11 via SunPKCS11 / dss-token
Desktop UI    JavaFX (dialog PIN) + AWT SystemTray
Bază de date  PostgreSQL 16 (server) / H2 în mod fișier (desktop); migrări Flyway comune
Secrete       Spring Cloud Vault (implementare a SecretProvider)
API           OpenAPI 3.1 scris întâi; generare de interfețe și de clienți
Build         Gradle, multi-modul, version catalog, convention plugins în buildSrc,
              dependency verification cu amprente fixate
Test          JUnit 5, AssertJ, Testcontainers, SoftHSM2, ArchUnit, teste de contract OpenAPI
Observabil.   Spring Boot Actuator; Micrometer + Prometheus din Etapa 4
Ambalare      Docker + Compose (Linux); jpackage → MSI (Windows)
CI            Linux (nucleu, server) + Windows (ambalaj, MSCAPI, token)
```

### Structura modulelor (granițe verificate cu ArchUnit)

```text
signer-core          -- model de domeniu, porturi, profiluri, politici
                        ZERO Spring, ZERO DSS, ZERO persistență
signer-engine-dss    -- SINGURA zonă care atinge eu.europa.esig.*
signer-keyprovider   -- PKCS11 / PKCS12 / MSCAPI / REMOTE_QTSP; semafoare, fir dedicat,
                        cache de sesiune și de PIN
signer-trust         -- TSA cu comutare, LOTL/TSL, revocare, cache-uri
signer-audit         -- lanț de amprente, sigilare temporală, export, verificator autonom
signer-secrets       -- SecretProvider: env / file / vault
signer-persistence   -- JPA + Flyway, portabil PostgreSQL/H2
signer-api           -- REST generat din OpenAPI, autentificare, autorizare pe profil,
                        taxonomia de erori
signer-app-server    -- ambalaj Spring Boot pentru Linux
signer-app-desktop   -- ambalaj Windows: serviciu, tray, dialog PIN, jpackage
signer-agent         -- modul opțional: long-poll către un server central (Etapa 4)
signer-cli           -- client al API-ului local (Etapa 4)
signer-testkit       -- PKI generat, SoftHSM2, TSA locală, fixture-uri de documente
```

**Reguli ArchUnit obligatorii:**

1. `signer-core` nu depinde de niciun alt modul al proiectului și de niciun framework.
2. `eu.europa.esig.*` este importat **exclusiv** în `signer-engine-dss`.
3. `signer-app-desktop` nu depinde de driverul PostgreSQL.
4. Niciun modul în afară de `signer-persistence` nu importă `jakarta.persistence.*`.
5. Nicio clasă din afara `signer-audit` nu emite `UPDATE`/`DELETE` pe tabelele de audit.
6. `Secret` și tipurile derivate nu apar niciodată ca parametru de jurnalizare.
