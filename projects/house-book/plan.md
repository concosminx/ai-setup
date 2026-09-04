# HouseBook — Implementation Plan

**Status:** ready to build · **Target:** an offline-first Android home-inventory app with an encrypted cloud backup service.
**Reference product:** [HouseBook – Home Inventory](https://play.google.com/store/apps/details?id=chenige.chkchk.wairz) (Android/iOS/web, cloud-synced). This plan replicates its *functionality*, not its *architecture* — see §0.

---

## 0. The one decision that shapes everything else

The reference app is **cloud-synced with live multi-user sharing**. This plan is **offline-only with backup/restore**.

That is a deliberate divergence, taken after the sync design was worked out and rejected. The consequences, stated once so they are never re-litigated mid-build:

| Reference app | This app | Why |
|---|---|---|
| Server is the source of truth, phone is a cache | **SQLite on the phone is the source of truth**, server holds encrypted blobs | Deletes the entire sync subsystem: outbox, change log, cursors, conflict resolution |
| Invite housemates to a live inventory | **No live sharing.** Later: publish a read-only rendered snapshot | Live sharing *is* sync. It cannot be bolted onto a backup service |
| Server can read your data | **Server can read nothing** — E2E encrypted | The data is photographs of your home, serial numbers and prices. See §6 |
| Works on Android, iOS, web | **Android only** | No Mac on the dev machine. Code never compiled for iOS does not compile for iOS |

**If live sharing ever becomes a requirement, this plan does not stretch to cover it.** That would be a different system, and the honest response is to reopen the design rather than grow this one into it.

### Assumptions

| # | Assumption | If wrong, this changes |
|---|---|---|
| A1 | **Personal use now, Play Store possible later.** No billing, no GDPR tooling, no crash reporting yet — but nothing in the design blocks them. | §6 auth grows (password reset, Google sign-in), plus privacy policy and store listing (P8). |
| A2 | **Solo developer, side-project pace.** Optimize for "always have something that runs". | Phase granularity only. |
| A3 | **Own git repository** `house-book` at `D:\Projects\git-others\house-book\`, not a subfolder of `ai-setup`. | §2 only. |
| A4 | **Romanian + English UI**, RON as the default currency. | ARB bundles and the default currency setting. |

---

## 1. Locked technical decisions

Environment verified on this machine: **Flutter 3.44.6 stable · Dart 3.12.2 · JDK 17.0.11 (Corretto) · Gradle 8.8 · Docker 28.4.0 · Node 22.15**. No blockers.

| Concern | Decision | Rationale |
|---|---|---|
| Client | **Flutter 3.44.x, Android only** | Windows dev machine; Play is Android; iOS/web are later implementations behind existing interfaces |
| Local DB | **SQLite via `drift`** | Typed queries, reactive streams, first-class FTS5 and migration support |
| State | **Riverpod 3** over `drift` streams | The database *is* the state in an offline app. BLoC would manage in-memory state that doesn't exist here |
| Navigation | **`go_router`** | Deep-linkable `/node/<id>` for QR scans; arbitrary-depth tree push/pop |
| Client IDs | **Client-minted UUIDv7**, stored as TEXT | Offline creation needs permanent identity from birth; v7 keeps index locality |
| Backend | **Java 17 · Gradle 8.8 Groovy DSL · Spring Boot 3.5.x** | Already installed and verified; nothing here pins a lower version |
| Server DB | **PostgreSQL 16**, Docker only | ~5 small tables, but `pg_dump` and headroom are free |
| Migrations | **Flyway**, `ddl-auto: validate` always | Hibernate never generates schema, not even in dev |
| Object storage | **S3 API via AWS SDK v2 `S3Client`**, path-style. **RustFS** in dev (pinned tag), **Cloudflare R2** in prod | MinIO was archived April 2026; RustFS is Apache-2.0. Behind a `PhotoStorage` interface, so the dev/prod split costs nothing |
| Crypto | **Argon2id → AES-256-GCM**, client-side only | §6 |
| Auth | **Self-issued JWT**: 15 min access, 60-day rotating refresh with reuse detection | Cookies fight a native background worker; Keycloak outweighs the whole app |
| API | REST/JSON under `/api/**`, springdoc-openapi, RFC 7807 errors | |
| Testing | JUnit 5 + Testcontainers Postgres (server); `drift` migration tests + `flutter_test` (client). **No H2** | H2 lies about Postgres and you pay for it in Flyway |
| Deployment | Small VPS · Docker Compose · **Caddy** for automatic TLS · Postgres container · R2 · nightly `pg_dump` → R2 | ~€4/month for something reachable from anywhere with no VPN on the client |
| Android app ID | **`eu.pakithecat.housebook`** | Owned domain. **Cannot be changed after a Play release** |
| Server package root | **`eu.pakithecat.housebook.api`** | |

---

## 2. Repository layout

```text
house-book/                          ← own git repo (A3)
├── app/                             ← Flutter, the bulk of the project
│   ├── pubspec.yaml
│   ├── lib/
│   │   ├── data/                    drift tables, DAOs, photo store, backup client
│   │   ├── domain/                  models + logic, NO flutter imports
│   │   └── ui/                      screens, widgets, routing
│   ├── android/
│   ├── l10n/                        app_en.arb, app_ro.arb
│   └── test/
├── backend/                         ← Spring Boot, a few hundred lines
│   ├── build.gradle
│   ├── gradlew / gradle/wrapper/
│   └── src/main/java/eu/pakithecat/housebook/api/
├── docker/
│   ├── docker-compose.yml           postgres + rustfs (dev)
│   └── Caddyfile                    prod
├── docs/
├── README.md
└── plan.md
```

**No root Gradle build.** `nurse-rotation` put Gradle at the root because UI5 build output could be folded into the boot jar. An APK cannot be — it is a separate artifact with its own release cycle — and here the backend is the *smaller* half. Two independent builds, one repo, one history.

### Client package structure

```text
lib/
├── data/
│   ├── db/            AppDatabase, tables/, daos/, migrations/
│   ├── photos/        PhotoStore (content-addressed files), capture, thumbnails
│   ├── backup/        BackupClient, Crypto, Manifest, ZipExporter
│   └── api/           generated REST client for the backend
├── domain/
│   ├── model/         Location, Item, Photo, Label, ConditionReport  (plain Dart)
│   ├── tree/          path maintenance, move/reparent, cycle + depth guards
│   ├── money/         Money (minor units), formatting-free arithmetic
│   ├── search/        query parsing, scope construction
│   └── render/        PdfRenderer, CsvRenderer, LabelSheetRenderer
└── ui/
    ├── router.dart
    ├── locations/     tree browser, node detail, breadcrumbs
    ├── items/         capture loop, item detail, edit
    ├── search/
    ├── backup/        onboarding, passphrase, status, restore
    ├── labels/        scan, bind, print
    └── reports/       condition reports, diff view
```

**`domain/` must not import `package:flutter`.** Enforce with a lint rule or a grep in CI. Everything interesting — path maintenance, money arithmetic, report diffing — is then unit-testable with no widget harness, which is the difference between tests you write and tests you mean to write.

---

## 3. Client data model (SQLite / `drift`)

### 3.1 Schema v1

```sql
-- Locations: one self-referencing tree. Depth capped at 6 in the service layer.
create table location (
  id          text primary key,             -- UUIDv7
  parent_id   text references location(id),
  kind        text not null,                -- PROPERTY | ROOM | CONTAINER
  name        text not null,
  path        text not null,                -- '/<root>/<...>/<id>/' materialized
  note        text,
  created_at  integer not null,             -- epoch millis
  updated_at  integer not null
);
create index idx_location_parent on location(parent_id);
create index idx_location_path   on location(path);

create table item (
  id              text primary key,         -- UUIDv7
  location_id     text not null references location(id),
  name            text not null,
  description     text,
  brand           text,
  model           text,
  serial_number   text,
  quantity        integer not null default 1,
  purchase_price  integer,                  -- minor units (bani). NEVER a float
  estimated_value integer,                  -- minor units, replacement value
  currency        text,                     -- null = app default (RON)
  purchase_date   integer,                  -- epoch millis, date-only semantics
  warranty_until  integer,
  created_at      integer not null,
  updated_at      integer not null
);
create index idx_item_location on item(location_id);
create index idx_item_warranty on item(warranty_until);

create table tag (
  id   text primary key,
  name text not null unique collate nocase
);
create table item_tag (
  item_id text not null references item(id) on delete cascade,
  tag_id  text not null references tag(id)  on delete cascade,
  primary key (item_id, tag_id)
);

-- Photos are content-addressed: the file on disk is named <hash>.jpg
create table photo (
  hash        text not null,                -- SHA-256 of the plaintext JPEG
  item_id     text references item(id) on delete cascade,
  entry_id    text references condition_entry(id) on delete cascade,
  taken_at    integer not null,
  is_cover    integer not null default 0,
  width       integer not null,
  height      integer not null,
  byte_size   integer not null,
  primary key (hash, item_id, entry_id)
);
create index idx_photo_item  on photo(item_id);
create index idx_photo_entry on photo(entry_id);

create table label (
  code      text primary key,               -- whatever the QR encodes, opaque
  node_id   text references location(id) on delete set null,
  bound_at  integer
);

create table condition_report (
  id            text primary key,
  root_node_id  text not null references location(id),
  kind          text not null,              -- CHECK_IN | CHECK_OUT | PERIODIC
  title         text not null,
  created_at    integer not null,
  finalised_at  integer,                    -- null = still editable
  content_hash  text                        -- SHA-256 over canonical content at finalisation
);
create table condition_entry (
  id         text primary key,
  report_id  text not null references condition_report(id) on delete cascade,
  node_id    text not null references location(id),
  rating     text not null,                 -- GOOD | FAIR | DAMAGED
  notes      text,
  unique (report_id, node_id)
);

create table app_setting (
  key   text primary key,
  value text not null
);                                          -- default currency, locale, last_backup_at, cursor state
```

### 3.2 Decisions embedded above

- **`path` is materialized**, maintained on every insert and reparent. Breadcrumbs are a string split; "everything under the garage" is `path LIKE '/…/<garage>/%'`; search scoping is free. The price is that moving a subtree rewrites every descendant's `path` — in one transaction, in `domain/tree/`, and nowhere else. People move a shelf roughly never.
- **Depth cap of 6 and cycle prevention live in `domain/tree/`,** because SQLite will not enforce either. A reparent that would make a node its own ancestor must throw, not corrupt.
- **Money is `integer` minor units, always.** A `REAL` price column is a bug you ship and discover during an insurance claim. `purchase_price` is what you paid; `estimated_value` is what a replacement costs — the export totals the latter and falls back to the former.
- **`currency` is nullable**, meaning "use the app default". One column now; an ugly retrofit through every total later.
- **Photos are keyed by content hash**, so the same photo attached to two items is one file on disk and one uploaded object. The composite primary key allows a photo to belong to an item *or* a condition entry (exactly one is non-null — enforce in the DAO).
- **`label.node_id` is nullable and `on delete set null`.** A sticker outlives the box it was on; scanning an unbound code is a normal state that opens the bind screen, not an error.
- **No custom-field table.** It becomes a junk drawer of unindexed strings. Add a column when you actually miss one.

### 3.3 Full-text search

```sql
create virtual table item_fts using fts5(
  name, description, brand, model, tags, location_path,
  content = 'item', content_rowid = 'rowid',
  tokenize = "unicode61 remove_diacritics 2"
);
```

- **`remove_diacritics 2` is not optional.** Without it `cutie` does not match *cuție* and you will lose an hour finding out why.
- Kept current by triggers on `item`, `item_tag` and `location` (a rename must reindex the subtree's `location_path`).
- Ranked with **bm25**, so best match sorts first rather than oldest.
- The index lives in the same database file, so it rides along in the `VACUUM INTO` snapshot and needs no rebuild after a restore.
- **Serial-number fragments fall back to `LIKE '%…%'`** on `item.serial_number` — a table scan, acceptable at a few thousand rows. If it ever hurts, add a second FTS5 table with `tokenize = 'trigram'` over serial and model; it is additive and needs no migration of existing data.
- **Scope** is a `location.path` prefix predicate joined onto the FTS query: "search the garage only" costs nothing.

### 3.4 Migrations

`drift`'s `MigrationStrategy` with a step-by-step migration per schema version, and **schema files committed** (`drift_dev` schema dumps) so `verifySelfMigration` can run every historical version forward in tests. This is not ceremony: a restored backup from an old app version *must* open. A broken migration means an unopenable backup, which is the same as no backup.

---

## 4. Server data model (PostgreSQL / Flyway `V1__baseline.sql`)

The server stores identity, wrapped keys, and pointers to opaque blobs. It stores **no inventory data**, because it cannot read any.

```sql
create table app_user (
  id            bigserial primary key,
  email         varchar(255) not null unique,
  password_hash varchar(255),                 -- null when only federated identities exist
  created_at    timestamptz not null default now(),
  enabled       boolean not null default true
);

create table auth_identity (
  id        bigserial primary key,
  user_id   bigint not null references app_user(id) on delete cascade,
  provider  varchar(32) not null,             -- PASSWORD | GOOGLE
  subject   varchar(255) not null,
  unique (provider, subject)
);

create table refresh_token (
  id          bigserial primary key,
  user_id     bigint not null references app_user(id) on delete cascade,
  token_hash  varchar(64) not null unique,    -- SHA-256, never the token itself
  family_id   uuid not null,                  -- rotation family, for reuse detection
  issued_at   timestamptz not null default now(),
  expires_at  timestamptz not null,
  revoked_at  timestamptz
);
create index idx_refresh_family on refresh_token(family_id);

-- Wrapped master key material. Server sees only ciphertext and KDF parameters.
create table user_key (
  user_id            bigint primary key references app_user(id) on delete cascade,
  kdf_salt           bytea not null,
  kdf_params         jsonb not null,          -- argon2id m, t, p
  mk_wrapped_pass    bytea not null,          -- MK encrypted under passphrase-derived KEK
  mk_wrapped_recover bytea not null,          -- MK encrypted under recovery-code-derived KEK
  updated_at         timestamptz not null default now()
);

create table backup_manifest (
  id             uuid primary key,            -- client-minted
  user_id        bigint not null references app_user(id) on delete cascade,
  created_at     timestamptz not null default now(),
  finalised_at   timestamptz,
  app_schema_ver int not null,
  db_object_key  varchar(128),                -- blinded hash of the encrypted db snapshot
  manifest_blob  bytea not null,              -- encrypted; contains the photo hash list
  byte_size      bigint
);
create index idx_manifest_user on backup_manifest(user_id, created_at desc);

-- One row per encrypted object the user has stored. Enables "which do you already have".
create table backup_object (
  user_id      bigint not null references app_user(id) on delete cascade,
  blinded_hash varchar(64) not null,          -- HMAC(K_dedup, sha256(plaintext))
  byte_size    bigint not null,
  created_at   timestamptz not null default now(),
  primary key (user_id, blinded_hash)
);

create table manifest_object (
  manifest_id  uuid not null references backup_manifest(id) on delete cascade,
  blinded_hash varchar(64) not null,
  primary key (manifest_id, blinded_hash)
);
```

**Why `blinded_hash` rather than the plain SHA-256:** deduplication needs the server to answer "do you already have this object?". If the client sent raw content hashes, the server could test whether a user possesses any *known* image — a real privacy leak in an app whose whole premise is that the server learns nothing. `HMAC-SHA256(K_dedup, sha256(plaintext))` with a per-user key preserves dedup and reveals nothing. Objects in R2 are keyed `u/<user_id>/<blinded_hash>`.

**Retention** is a scheduled job: keep the newest 10 finalised manifests per user, delete the rest, then delete any `backup_object` no surviving manifest references and remove it from R2. Orphan sweeping on the server mirrors the one on the client.

---

## 5. Backup and restore

### 5.1 Key hierarchy

```text
passphrase ──Argon2id(salt, m=64MB, t=3, p=1)──► KEK_pass ─┐
                                                            ├─► unwraps MK (random 256-bit)
recovery code ──Argon2id(salt', same params)──► KEK_recov ─┘
                                                             │
                                          HKDF-SHA256(MK) ───┼──► K_content   (AES-256-GCM)
                                                             └──► K_dedup     (HMAC-SHA256)
```

- **MK is random and never derived from the passphrase.** Changing the passphrase re-wraps MK; it does not re-encrypt a single byte of backup data.
- **The recovery code is 128 bits, base32, shown once at setup**, with the flow refusing to continue until the user confirms they have stored it. It is the only defence against a forgotten passphrase, and there is no reset — that is the point.
- **MK is cached in `flutter_secure_storage`** (Android Keystore-backed) so the background backup worker can encrypt without a prompt. The passphrase is typed exactly twice in the app's life: at setup, and on a new device during restore.

### 5.2 Backup procedure

1. `VACUUM INTO` a temp file — **never copy a live SQLite file**; gzip it.
2. Enumerate local photos; each already has its plaintext SHA-256 (it is the filename).
3. Compute `blinded_hash = HMAC(K_dedup, sha256)` for every photo plus the gzipped snapshot.
4. `POST /api/backups` → manifest id. `POST /api/backups/{id}/objects/check` with the blinded hashes → the server returns the missing subset.
5. For each missing object: `POST …/objects/presign` → **presigned PUT straight to R2**, encrypted with `K_content` under AES-256-GCM, random 96-bit nonce prepended, streamed in chunks so a 5 MB photo never lands in memory whole. The JVM never touches image bytes.
6. Upload the encrypted manifest (the plaintext-hash list, item count, app schema version) and `POST …/finalize`.
7. Record `last_backup_at` locally.

The first backup uploads everything; a backup after an afternoon of cataloguing uploads the twenty new photos and a 5 MB database. That asymmetry is the entire reason for content addressing.

### 5.3 Restore

Sign in → enter passphrase → unwrap MK → list manifests by date → pick one → download and decrypt the database snapshot (seconds) → **`drift` runs its migration chain forward, and the app is usable immediately** → photos stream in lazily in the background, most-recently-used first, with an unobtrusive progress indicator. Any design that shows a 2 GB progress bar before showing data is the wrong one.

### 5.4 Scheduling and the truth on screen

- `workmanager` periodic task, constraints **unmetered network + charging**, roughly daily.
- **Android background execution is not reliable.** OEM battery management (Samsung, Xiaomi, Huawei) will silently skip it. Do not pretend otherwise.
- The home screen therefore always shows **"Last backed up: 3 days ago"** — neutral under a week, amber past a week, red past a month — beside a "Back up now" button. One row of UI that converts an untrustworthy OS guarantee into information the user can act on.

### 5.5 The P2 escape hatch

Before any of §5 exists, ship one button: **"Export everything to a zip"** — the `VACUUM INTO` snapshot plus the photo directory, unencrypted, written to Downloads and handed to the share sheet. Roughly 40 lines, no server, no crypto, no account. It removes the only irreversible risk in this plan (see §11) and it is a strict subset of the work §5.2 needs anyway.

---

## 6. Authentication

- **Registration and login** with email + password, BCrypt hashes, Bean Validation on the request records.
- **Access token** JWT, 15 minutes, signed HS256 with a secret from the environment; carries `sub` and nothing sensitive.
- **Refresh token** 60 days, opaque random, stored **hashed** server-side, **rotated on every use**. Presenting an already-rotated token revokes the whole `family_id` — that is reuse detection, and it is the difference between a stolen token being useful for a minute and for two months.
- **Client:** refresh in `flutter_secure_storage`, and a **single refresh mutex** in the HTTP client so ten queued requests hitting a 401 together trigger one refresh, not ten.
- **`auth_identity` exists from V1** so "Sign in with Google" is a provider row rather than a user-table migration on the day you publish.
- **Password reset stays unbuilt** until a Play release is real (A1). It needs an email provider and it is the one auth flow that is easy to get subtly wrong.
- **Share tokens (P8+)** are a separate table and a separate public endpoint — never a JWT, never tied to a user session.

### API surface

```text
POST   /api/auth/register
POST   /api/auth/login                 → { access, refresh }
POST   /api/auth/refresh               → rotated pair
POST   /api/auth/logout
GET    /api/me

GET    /api/keys                       → { salt, kdfParams, mkWrappedPass, mkWrappedRecovery }
PUT    /api/keys                       → re-wrap after passphrase change

POST   /api/backups                    → { manifestId }
POST   /api/backups/{id}/objects/check → { missing: [blindedHash] }
POST   /api/backups/{id}/objects/presign
POST   /api/backups/{id}/finalize
GET    /api/backups                    → manifest list
GET    /api/backups/{id}               → manifest + presigned GET URLs
```

RFC 7807 `application/problem+json` for every 4xx/5xx via `@RestControllerAdvice`. DTOs as Java records; entities never cross the controller boundary.

---

## 7. Photos

- **Capture:** `camera` or `image_picker`, then downscale to **~1600px longest edge, JPEG q85** before anything else touches it. A 12 MP phone photo is 5 MB and adds nothing to an inventory.
- **Thumbnail** generated client-side at capture (~256px) and stored alongside, so grid views are instant and work offline with no decode cost.
- **Storage:** app-private directory, file named `<sha256>.jpg`. The hash is computed once, at capture, and is thereafter the photo's identity everywhere — DB row, dedup, backup.
- **Orphan sweep:** a periodic job deletes files no `photo` row references. Skipping this is how a photo app quietly eats 4 GB.
- **The capture loop is the primary flow of the entire app.** Camera opens directly from a location screen → shoot → a single name field with **"Save & next"** as the primary action → camera again. Everything else (brand, price, serial) is optional and editable later. If adding an item costs a full-screen form and four taps, cataloguing stops at twenty items and the project dies half-finished.

---

## 8. QR labels

- The QR encodes a **short opaque code**; the app maps codes to nodes in the local `label` table. It does not encode a UUID or a URL — the server could not resolve a URL anyway, since it cannot read your data.
- **Bind-first workflow:** buy a roll of pre-printed unique QR stickers, stick them on boxes, then scan. An **unrecognised code always lands on "bind this label to…?"** — never an error. That single rule is what makes the feature feel intentional. It also means the app works with any sticker you can buy, with no printer.
- **Printing comes second:** generate codes locally and lay them out on an A4 sticker grid, reusing the PDF renderer from §9.
- Scanning via `mobile_scanner`; a bound code routes to `/node/<id>` through `go_router`.

---

## 9. Export

Rendered **on-device** — forced by §5's encryption, and better for it: no server round trip, works offline, no templating service to operate.

- **PDF** (`pdf` + `printing`): grouped by location, one **~800px photo per item inline**, an explicit **"include all photos"** toggle for the heavy version, and **per-room and grand totals on the cover page**. That total is the reason the document exists.
- **CSV:** one flat row per item, every field. The machine artifact for an insurer's spreadsheet, and your proof that the data is not trapped in this app.
- **Scope:** any subtree (free, via `path`), any tag, or any saved search. Manual multi-select is a later refinement.
- **Delivery:** write to app storage, hand to the **Android share sheet**. No custom upload, and it sidesteps scoped-storage headaches entirely.
- Built once as `domain/render/`, then reused by label sheets (§8) and condition reports (§10). Three one-off renderers would be the wrong shape.

---

## 10. Condition reports

A report **annotates the existing location tree**; it does not build a parallel one. The garage is one node; in the March report it is `GOOD`, in the September report `DAMAGED` with three photos. Belongings come along automatically because they hang under those nodes.

- **Immutability is the whole point.** On finalisation, compute a SHA-256 over the canonical serialization of the report and its entries, store it in `content_hash`, and refuse every subsequent edit. A report you can silently amend is worthless in the dispute it exists for.
- **Photos are report-scoped** (`photo.entry_id`). The same wall in March and September is two photos, both permanent. Content addressing dedupes identical ones and keeps different ones.
- **Diffing** two finalised reports over the same `root_node_id`: per node, rating change and note change; per item, present/absent/changed. Side-by-side photos.
- Renders to PDF through §9.
- Finalised reports are the highest-value data in the app — losing a check-in report loses a deposit — so they ride in the encrypted backup like everything else.

---

## 11. Phases

Each phase ends with something runnable. Do not start the next until the deliverable is true.

### P1 — Skeleton
Flutter project, `drift` schema v1 with the location tree, `path` maintenance in `domain/tree/` with cycle and depth guards, `go_router`, create/rename/move/delete nodes, breadcrumbs.
**Done when:** you can build your house's structure and navigate it.

### P2 — Capture (+ escape hatch)
Items, camera capture, downscale, thumbnails, content-addressed photo store, cover photos, the photograph → name → save-and-next loop. Tags. **Plus §5.5's local zip export.**
**Done when:** you catalogue a real room, don't hate the process, and can copy a zip of everything to your PC. **Start dogfooding here** — the capture loop is the one thing in this design that cannot be validated on paper.

### P3 — Find
FTS5 with triggers, bm25 ranking, diacritic folding, subtree scoping, tag filter, warranty filter, serial `LIKE` fallback.
**Done when:** you find something you'd forgotten you owned.

### P4 — Backup
Spring Boot skeleton, Flyway V1, auth (§6), R2/RustFS behind `PhotoStorage`, Argon2id + AES-GCM, blinded dedup, manifests, `workmanager` scheduling, staleness indicator, restore.
**Done when:** you wipe the emulator, sign in, enter the passphrase, and everything comes back.

### P5 — Export
PDF + CSV, totals, scoping, share sheet.
**Done when:** you produce a document you would actually send an insurer.

### P6 — Labels
Code binding, `mobile_scanner`, unbound-code flow, then printable label sheets.
**Done when:** you scan a box and it opens.

### P7 — Condition reports
Reports, entries, report-scoped photos, finalisation and hashing, diff view, PDF.
**Done when:** two reports over one property compare cleanly.

### P8 — Publish (only if you decide to)
Password reset with an email provider, Sign in with Google via `auth_identity`, privacy policy, crash reporting, Play listing and signing, share-as-published-snapshot.

**Explicitly not in this plan:** live sync, multi-user sharing, iOS, web, notifications, custom fields, barcode/UPC product lookup, receipt OCR, insurance-company integrations, billing.

---

## 12. Testing

| Layer | Tool | Bar |
|---|---|---|
| `domain/` | `flutter_test` (pure Dart) | Tree operations (reparent, path rewrite, cycle rejection, depth cap), money arithmetic, report diffing. **No Flutter imports means no widget harness** |
| Migrations | `drift_dev` schema dumps + `verifySelfMigration` | **Every historical version migrates forward.** A broken migration means an unopenable backup, which equals no backup |
| Search | Integration test on a real SQLite file | Diacritic folding (`cutie` → *cuție*), bm25 ordering, subtree scoping |
| Crypto | Round-trip test | encrypt → upload → download → decrypt → **byte-identical**. Plus: wrong passphrase fails cleanly, recovery code unwraps the same MK |
| Backup | End-to-end against Testcontainers + RustFS | Full backup → wipe → restore → database compares equal |
| Backend | JUnit 5, MockMvc, **Testcontainers Postgres** | Auth happy path, refresh rotation, **reuse detection revokes the family**, presign auth checks |
| Widgets | `flutter_test` | Only the capture loop. It is the flow that must not regress |

---

## 13. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| **Hours of cataloguing lost before backup exists (P4)** | **High** | §5.5's zip export in P2. This is the only irreversible risk in the plan |
| Forgotten passphrase → permanently unreadable backup | Medium | Recovery code at setup, with a confirmation gate. Say plainly in the UI that there is no reset |
| Android background work silently never runs | **High** | Assume it. The staleness indicator (§5.4) makes failure visible rather than silent |
| Capture loop too slow → app abandoned half-full | **High** | Dogfood from end of P2, before building anything on top of it |
| Photo storage grows without bound | Medium | Downscale at capture, orphan sweep both sides, R2 free tier is generous but not infinite |
| Subtree move corrupts `path` | Medium | One implementation in `domain/tree/`, transactional, with tests for reparent and cycle rejection |
| RustFS is early-stage | Low | Dev-only; prod is R2; everything behind `PhotoStorage` and AWS SDK v2 |
| Scope creep toward sync/sharing | **High** | §0. Live sharing is a different system; the answer is to reopen the design, not to grow this one |
| Play release assumed away | Low (A1) | `auth_identity`, `owner`-scoped server data and E2E encryption already carry the load-bearing parts |

---

## 14. Definition of done — first coding session

```text
✓ git repo initialized at D:\Projects\git-others\house-book\, .gitignore covers build/, .dart_tool/, *.g.dart if generated
✓ app/ created with applicationId eu.pakithecat.housebook, flutter run reaches a device
✓ drift AppDatabase with the location table, schema v1 dumped and committed
✓ Riverpod + go_router wired: a root screen listing top-level locations
✓ domain/tree/ path maintenance with unit tests for insert, reparent, cycle rejection, depth cap
✓ l10n set up with app_en.arb + app_ro.arb; the first screen has zero string literals
✓ Material 3 light + dark theme from a seed colour
```

The backend does not exist yet, and should not. It earns its place in P4.
