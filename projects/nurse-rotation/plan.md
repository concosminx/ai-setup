# Hospital Nurse Scheduling — Implementation Plan

**Status:** ready to build · **Supersedes:** `x-nurse-PLAN.md` (kept as the vision/backlog document)
**Target:** a working vertical slice first, then breadth.

---

## 0. Assumptions (correct these and I'll revise)

These four decisions were not confirmed; the plan below is written against them. Everything downstream is marked so you can see what changes if an assumption flips.

| # | Assumption | If wrong, this changes |
|---|---|---|
| A1 | **Portfolio / learning project**, not a system going live in a hospital. No GDPR/DPIA work, no legal review of labor rules, no penetration testing. | Phase 4 grows substantially (compliance, pen-test, data retention, real IdP). |
| A2 | **Solo developer, side-project pace**, no hard deadline. Optimize for "always have something that runs". | Milestone granularity and CI ambition. |
| A3 | **Separate git repository** named `hospital-scheduler`, not a subfolder of `ai-setup`. | Repo layout in §2 only. |
| A4 | **Romanian defaults** for scheduling policy (see §7), explicitly *not* a compliance guarantee. Romanian + English UI. | Default `SchedulingPolicy` row values and i18n bundles. |

---

## 1. Locked technical decisions

Environment verified on this machine: **JDK 17.0.11 (Corretto), Node 22.15, Docker 28.4, Gradle 8.8**. No blockers.

| Concern | Decision | Rationale |
|---|---|---|
| Language / JVM | **Java 17** | Already installed; the baseline for both Spring Boot 3.x and OptaPlanner 10.x. Don't chase 21 until there's a reason. |
| Build | **Gradle Wrapper, Groovy DSL** (`build.gradle`) | Mandatory per constraint. Groovy DSL over Kotlin DSL: every Spring/OptaPlanner example you'll copy from is Groovy or Maven. |
| Spring Boot | **3.5.x** (latest 3.5 patch) | Verify at init that the OptaPlanner starter you pick supports it; if not, drop to the highest 3.x the starter's BOM declares. **Do not** jump to Spring Boot 4 / Framework 7 before OptaPlanner support is confirmed. |
| Solver | **`org.optaplanner:optaplanner-spring-boot-starter`, 10.x** (10.1/10.2 line, Apache KIE) | Confirmed live and published to Maven Central under `org.optaplanner`. The starter auto-configures `SolverManager` and `SolverFactory` from `application.yml` — do not hand-roll solver config. |
| Constraint API | **Constraint Streams** (`ConstraintProvider`), never Drools DRL | Type-safe, debuggable, and unit-testable with `ConstraintVerifier`. |
| Score | **`HardSoftScore`** for MVP | Upgrade path to `HardMediumSoftScore` documented in §6.4 — do not pre-emptively adopt it. |
| DB | **PostgreSQL 16**, Docker only | No local install. |
| Migrations | **Flyway**, `ddl-auto: validate` always | Hibernate never generates schema, not even in dev. |
| API | REST/JSON, `/api/**`, **springdoc-openapi** for docs | |
| Auth | **Spring Security form login + session cookie**, BCrypt, users in DB | Simplest thing that works with a UI5 SPA on the same origin. JWT/OIDC is Phase 4 and probably never. |
| Frontend | **OpenUI5 + UI5 Tooling** (`@ui5/cli`), dev server proxying `/api` → `:8080` | Prod: `ui5 build` output copied into the boot jar's static resources → one deployable artifact. |
| Testing | JUnit 5, `ConstraintVerifier` (per constraint), **Testcontainers** PostgreSQL for integration | No H2. H2 lies about Postgres behaviour and you'll pay for it in Flyway and timestamptz. |
| Packaging | Single Spring Boot fat jar + Postgres container | Modular monolith. No Kubernetes. |
| Package root | `com.hospital.scheduler` | |

---

## 2. Repository layout

```text
hospital-scheduler/                 ← own git repo (A3)
├── settings.gradle                 ← root: single Gradle build
├── build.gradle
├── gradle.properties
├── gradlew / gradlew.bat / gradle/wrapper/
├── src/
│   ├── main/java/com/hospital/scheduler/
│   ├── main/resources/
│   │   ├── application.yml
│   │   ├── db/migration/
│   │   └── static/               ← UI5 build output lands here (gitignored)
│   └── test/
├── frontend/
│   ├── package.json
│   ├── ui5.yaml
│   └── webapp/
├── docker/
│   └── docker-compose.yml          ← postgres only for dev
├── .github/workflows/ci.yml
├── README.md
└── plan.md
```

**Deviation from the vision doc:** no `backend/` subfolder. The Gradle build sits at the repo root and `frontend/` is a sibling source folder, not a second build. One `./gradlew build` produces the whole application; a Gradle task shells out to `npm run build` in `frontend/` and copies `dist/` into `src/main/resources/static/`. Simpler for a solo dev than two independent builds.

### Package structure

```text
com.hospital.scheduler
├── nurse/          Nurse, NurseRepository, NurseService, NurseController, dto/
├── department/     Department, …
├── skill/          Skill, …
├── shift/          Shift, ShiftType (enum), ShiftRequirement, …
├── availability/   Availability, AvailabilityType (enum), …
├── preference/     NursePreference, PreferenceType (enum), …
├── schedule/       Schedule, ScheduleStatus (enum), ShiftAssignment, ScheduleService, …
├── planning/
│   ├── domain/     NurseSchedule, PlanningShiftAssignment, PlanningNurse, PlanningShift  ← POJOs, no JPA
│   ├── solver/     NurseScheduleConstraintProvider, ScheduleSolverService
│   └── mapper/     PlanningDataLoader (JPA→POJO), PlanningResultWriter (POJO→JPA)
├── policy/         SchedulingPolicy, PolicyService
├── security/       SecurityConfig, AppUser, Role (enum), UserDetailsServiceImpl
├── audit/          AuditLog, AuditService, @Auditable
└── common/         exceptions, error handling, time (HospitalClock), config
```

**The `planning/` boundary is the single most important architectural rule here.** OptaPlanner annotations appear *only* in `planning/domain`. JPA annotations never appear there. `PlanningDataLoader` and `PlanningResultWriter` are the only classes that know both worlds. Everything outside `planning/` talks to `ScheduleSolverService`, which exposes four methods and no OptaPlanner types.

---

## 3. Data model — concrete

### 3.1 Schema (Flyway `V1__baseline.sql`)

One baseline migration for the MVP, not ten. Split into numbered migrations only once the app has run against data you care about.

```sql
create table department (
  id          bigserial primary key,
  code        varchar(32)  not null unique,
  name        varchar(128) not null,
  active      boolean      not null default true
);

create table skill (
  id          bigserial primary key,
  code        varchar(32)  not null unique,
  name        varchar(128) not null,
  description text
);

create table nurse (
  id               bigserial primary key,
  employee_number  varchar(32)  not null unique,
  first_name       varchar(64)  not null,
  last_name        varchar(64)  not null,
  email            varchar(255) not null unique,
  department_id    bigint       not null references department(id),
  contract_hours   numeric(5,2) not null,     -- contracted hours per week
  max_weekly_hours numeric(5,2),              -- null → fall back to policy
  active           boolean      not null default true
);

create table nurse_skill (
  nurse_id bigint not null references nurse(id) on delete cascade,
  skill_id bigint not null references skill(id) on delete cascade,
  primary key (nurse_id, skill_id)
);

create table shift (
  id            bigserial primary key,
  department_id bigint      not null references department(id),
  shift_type    varchar(16) not null,          -- MORNING | EVENING | NIGHT
  start_at      timestamptz not null,
  end_at        timestamptz not null,
  constraint shift_time_order check (end_at > start_at),
  unique (department_id, shift_type, start_at)
);
create index idx_shift_window on shift (department_id, start_at, end_at);

create table shift_requirement (
  id             bigserial primary key,
  shift_id       bigint  not null references shift(id) on delete cascade,
  skill_id       bigint  references skill(id),    -- null = "any qualified nurse"
  required_count int     not null check (required_count > 0)
);

create table availability (
  id       bigserial   primary key,
  nurse_id bigint      not null references nurse(id) on delete cascade,
  day      date        not null,
  type     varchar(24) not null,     -- VACATION | SICK_LEAVE | UNAVAILABLE
  note     text,
  unique (nurse_id, day)
);

create table nurse_preference (
  id        bigserial   primary key,
  nurse_id  bigint      not null references nurse(id) on delete cascade,
  type      varchar(32) not null,    -- PREFERS_SHIFT_TYPE | AVOIDS_SHIFT_TYPE
                                     -- | PREFERS_DAY_OFF | AVOIDS_WEEKDAY_SHIFT_TYPE
  shift_type varchar(16),            -- nullable, depends on type
  day_of_week int,                   -- 1..7 ISO, nullable
  day         date,                  -- nullable, for one-off requests
  weight      int not null default 1 -- 1..3, multiplies the soft penalty
);

create table scheduling_policy (
  id                          bigserial primary key,
  department_id               bigint references department(id),  -- null = hospital default
  min_rest_hours              int not null,
  max_weekly_hours            numeric(5,2) not null,
  max_consecutive_days        int not null,
  max_consecutive_night_shifts int not null,
  max_weekend_shifts_per_month int not null
);

create table schedule (
  id            bigserial   primary key,
  name          varchar(128) not null,
  department_id bigint      not null references department(id),
  start_date    date        not null,
  end_date      date        not null,
  status        varchar(16) not null,   -- §5
  hard_score    int,
  soft_score    int,
  created_at    timestamptz not null default now(),
  created_by    varchar(64),
  approved_at   timestamptz,
  approved_by   varchar(64),
  published_at  timestamptz,
  version       int not null default 0   -- JPA optimistic locking
);

create table shift_assignment (
  id           bigserial primary key,
  schedule_id  bigint not null references schedule(id) on delete cascade,
  shift_id     bigint not null references shift(id),
  skill_id     bigint references skill(id),  -- the requirement slot this row fills
  nurse_id     bigint references nurse(id),  -- NULL = unfilled slot
  pinned       boolean not null default false,
  unique (schedule_id, shift_id, id)
);
create index idx_assignment_schedule on shift_assignment (schedule_id);
create index idx_assignment_nurse on shift_assignment (schedule_id, nurse_id);

create table app_user (
  id            bigserial primary key,
  username      varchar(64) not null unique,
  password_hash varchar(255) not null,
  nurse_id      bigint references nurse(id),  -- set for NURSE role users
  enabled       boolean not null default true
);

create table app_user_role (
  user_id bigint not null references app_user(id) on delete cascade,
  role    varchar(32) not null,   -- ADMIN | SCHEDULER | DEPARTMENT_MANAGER | NURSE
  primary key (user_id, role)
);

create table audit_log (
  id          bigserial   primary key,
  username    varchar(64) not null,
  at          timestamptz not null default now(),
  action      varchar(64) not null,
  entity_type varchar(64) not null,
  entity_id   bigint,
  old_value   jsonb,
  new_value   jsonb
);
create index idx_audit_entity on audit_log (entity_type, entity_id);
```

### 3.2 Decisions embedded above, and why

- **`timestamptz` everywhere for instants; `date` for calendar concepts.** Shifts are real points in time (`Instant` in Java). Availability is a calendar day in the hospital's zone. Mixing the two is the classic bug in this domain, so the types differ on purpose.
- **Hospital time zone is configuration, not data**: `app.hospital.zone: Europe/Bucharest` in `application.yml`, wrapped in a `HospitalClock` bean. Nothing else in the codebase calls `ZoneId.systemDefault()` — enforce that with an ArchUnit test or a grep in CI.
- **`shift_assignment` is one row per required headcount slot, not one row per shift.** This is the single most consequential modeling decision (see §6.1).
- **`nurse_id` is nullable.** An unfilled slot is a representable, penalized state — not a crash. Without this the solver can report "no solution found" and tell you nothing.
- **`pinned`** exists from day one because manual edits (Phase 3) need it, and retrofitting it means a migration plus a solver-config change.
- **Contract is a column on `nurse`, not a table.** The vision doc had a `contract` entity; three numbers on the nurse row carries the MVP. Promote to a table when contract *types* need to be shared and named.

---

## 4. Scheduling policy defaults (A4)

Seeded into `scheduling_policy` as the hospital-wide row (`department_id = null`):

| Field | Default | Source |
|---|---|---|
| `min_rest_hours` | 12 | Romanian Codul Muncii art. 135 (12h between working days) |
| `max_weekly_hours` | 48 | art. 114 — the legal ceiling including overtime; contracted norm is 40 |
| `max_consecutive_days` | 6 | art. 137 — one rest day per 7 |
| `max_consecutive_night_shifts` | 3 | Common hospital practice, not statute |
| `max_weekend_shifts_per_month` | 2 | Fairness policy, not statute |

Department-level rows override the hospital row field-by-field. **This is a plausible starting configuration, not legal advice** (A1) — the README says so explicitly.

---

## 5. Schedule lifecycle

```text
DRAFT ──solve──► SOLVING ──► SOLVED ──► APPROVED ──► PUBLISHED
  ▲                 │           │
  └────stop/fail────┘      manual edits
                                │
                          (stays SOLVED, may re-solve)
```

MVP implements five states. **`REVIEW` is dropped** — it was indistinguishable from `SOLVED` in the vision doc; `SOLVED` *is* the review state. `CANCELLED`, `REVISED`, and `ARCHIVED` are Phase 4+.

Transition rules, enforced in `ScheduleService`, not in the controller:

| From → To | Trigger | Guard |
|---|---|---|
| DRAFT → SOLVING | `POST /solve` | Schedule has ≥1 shift with requirements; no other SOLVING schedule for that department |
| SOLVING → SOLVED | solver terminates | Best solution persisted first, status flipped in the same transaction |
| SOLVING → DRAFT | `POST /stop`, or solver error | Partial solution discarded |
| SOLVED → SOLVING | re-solve | Pinned assignments preserved |
| SOLVED → APPROVED | `POST /approve` | `hard_score = 0`, role SCHEDULER or ADMIN |
| APPROVED → PUBLISHED | `POST /publish` | role SCHEDULER or ADMIN |
| PUBLISHED → * | — | Immutable in MVP |

---

## 6. The OptaPlanner model — the part worth getting right

### 6.1 Slot-per-requirement, nullable planning variable

A shift needing "4 nurses, of whom 2 ICU-qualified and 1 charge nurse" expands into **four** `ShiftAssignment` rows before solving begins:

```text
ICU / Sep 1 / NIGHT  requires 4  (2×ICU, 1×CHARGE_NURSE)
   ↓ expansion at schedule creation
slot 1  skill=ICU            nurse=?
slot 2  skill=ICU            nurse=?
slot 3  skill=CHARGE_NURSE   nurse=?
slot 4  skill=null (any)     nurse=?
```

Consequences, all of them good:

- **Minimum staffing stops being a counting constraint** and becomes "no slot may be left unassigned" — a one-line constraint over `nurse == null`.
- **The skill constraint is per-slot**, so "which nurse satisfies which requirement" is decided by the solver rather than by a matching algorithm you'd otherwise have to write.
- Move selectors work out of the box; no custom moves needed for the MVP.

The planning variable is declared `@PlanningVariable(allowsUnassigned = true)` — verify the exact attribute name against the 10.x API at implementation time, it was `nullable = true` in the 8.x line.

### 6.2 Planning classes

```java
@PlanningSolution
class NurseSchedule {
  @ProblemFactCollectionProperty @ValueRangeProvider List<PlanningNurse> nurses;
  @ProblemFactCollectionProperty List<PlanningShift> shifts;
  @ProblemFactCollectionProperty List<UnavailableDay> unavailabilities;
  @ProblemFactCollectionProperty List<PlanningPreference> preferences;
  @ProblemFactCollectionProperty List<SchedulingPolicySnapshot> policies;   // single-element list
  @PlanningEntityCollectionProperty List<PlanningShiftAssignment> assignments;
  @PlanningScore HardSoftScore score;
}

@PlanningEntity
class PlanningShiftAssignment {
  @PlanningId Long id;
  PlanningShift shift;          // problem fact
  Long requiredSkillId;         // nullable
  @PlanningVariable(allowsUnassigned = true) @PlanningPin-aware
  PlanningNurse nurse;
  @PlanningPin boolean pinned;
}
```

Plain POJOs. No JPA. Built by `PlanningDataLoader` in one transaction with explicit fetch joins — **fetch every collection eagerly there**; lazy-loading inside the solver thread is a guaranteed `LazyInitializationException` and the failure mode is confusing.

### 6.3 Constraints — concrete, with weights

Hard constraints (each worth 1 hard point per violation unless noted):

| ID | Constraint | Implementation sketch |
|---|---|---|
| H1 | No overlapping shifts per nurse | `forEachUniquePair(assignment, equal(nurse), overlapping(start, end))` |
| H2 | No assignment on VACATION / SICK_LEAVE / UNAVAILABLE | join assignment × unavailability on nurse + shift's local date(s); overnight shifts touch **two** days — check both |
| H3 | Slot's required skill is held by the assigned nurse | filter `requiredSkillId != null && !nurse.skills.contains(...)` |
| H4 | Every slot is filled | `forEach(assignment).filter(a -> a.nurse == null)` |
| H5 | Weekly hours ≤ policy max | group by nurse + ISO week, sum duration, penalize excess **hours** (weight = hours over) |
| H6 | Rest between consecutive shifts ≥ policy min | pair per nurse, penalize gap shortfall in hours |
| H7 | Nurse's department matches shift's department | trivially true in MVP (single-department schedules) — implement anyway, it's the guard for cross-department scheduling later |

Soft constraints (weights are a starting point; tune against real data in Phase 3):

| ID | Constraint | Weight |
|---|---|---|
| S1 | Honor `PREFERS_*` / penalize `AVOIDS_*` preferences | 10 × preference weight (1–3) |
| S2 | Honor requested days off | 20 |
| S3 | Consecutive working days over policy max | 30 per day over |
| S4 | Consecutive night shifts over policy max | 30 per shift over |
| S5 | Weekend fairness — load-balance weekend shifts across nurses | 5 × squared deviation from mean |
| S6 | Night fairness — same, for nights | 5 × squared deviation |
| S7 | Distance from contracted hours | 2 per hour of deviation |

**Weight discipline:** fairness (S5/S6) uses squared deviation so that one badly-treated nurse costs more than diffuse mild unfairness — linear fairness constraints produce schedules that look "fair on average" and feel unfair to individuals. Keep every weight in a single `ConstraintWeights` class so tuning is one file, not a scavenger hunt.

### 6.4 When to move to HardMediumSoftScore

Move when — and only when — you want unfilled slots (H4) to be *tolerable but worse than everything else*, i.e. "give me the best partial roster if full coverage is impossible". Today H4 is hard, so an under-staffed period yields a negative hard score and no usable answer. Symptom to watch for: `hardScore < 0` on realistic data with no way to fix it by adding nurses. Then H4 moves to medium and stays above all soft constraints.

### 6.5 Solver configuration

`application.yml`:

```yaml
optaplanner:
  solver:
    termination:
      spent-limit: 60s
      unimproved-spent-limit: 15s
    move-thread-count: AUTO
```

Async execution uses the starter's `SolverManager` with `solveAndListen`, so best-solution updates arrive as the solve runs; the listener updates an in-memory `SolverStatusRegistry` (score + elapsed) that `GET /status` polls, and the final callback persists assignments. **Never** hold a DB transaction open across the solve.

`ScheduleSolverService` — the entire surface OptaPlanner is allowed to have:

```java
void solve(Long scheduleId);
SolverStatusView getStatus(Long scheduleId);   // status, hardScore, softScore, elapsed
void stop(Long scheduleId);
List<ConstraintViolationView> explain(Long scheduleId);  // ScoreExplanation, Phase 3
```

---

## 7. REST API

Unchanged in shape from the vision doc, with these specifics nailed down:

- **Errors:** RFC 7807 `application/problem+json` for every 4xx/5xx via `@RestControllerAdvice`.
- **Lists:** Spring Data `Pageable` (`?page=&size=&sort=`) on all collection endpoints. Not optional — the nurse list is 100+ rows on day one.
- **DTOs, never entities, cross the controller boundary.** Java records in `<domain>/dto/`.
- **Validation:** `@Valid` + Bean Validation on request records; violations map to 400 with per-field detail.

Endpoints beyond the vision doc's list:

```text
GET  /api/schedules/{id}/violations      → explained hard/soft violations (Phase 3)
POST /api/schedules/{id}/assignments/{assignmentId}/nurse   → manual assign, body {nurseId, force}
POST /api/schedules/{id}/approve
POST /api/schedules/{id}/publish
GET  /api/policies?departmentId=
GET  /api/me                              → current user, roles, linked nurse
```

Manual-assignment response carries a validation verdict so the UI can render §9's warnings:

```json
{
  "verdict": "ALLOWED_WITH_WARNINGS",
  "violations": [
    {"severity": "WARNING", "code": "REST_PERIOD", "message": "Only 8h rest before this shift (policy: 12h)"},
    {"severity": "BLOCKING", "code": "ON_VACATION", "message": "Elena is on vacation on 2026-09-05"}
  ]
}
```

`verdict ∈ {ALLOWED, ALLOWED_WITH_WARNINGS, BLOCKED}`. BLOCKING severity comes from H2/H3 (availability, skills) — the rules that make a roster *invalid*. Everything else warns. Requests with `force: true` accept warnings, never blocks.

---

## 8. Frontend

UI5 app under `frontend/webapp`, routes: `Dashboard`, `Nurses`, `NurseDetail`, `Shifts`, `Availability`, `Schedules`, `ScheduleDetail`.

Decisions the vision doc left open:

- **Models:** `JSONModel` against the plain REST API. **Not OData** — the backend is REST/JSON, and forcing OData semantics onto it would be the tail wagging the dog.
- **The schedule grid** (nurse × day matrix) is the hard piece. Use `sap.m.Table` with dynamically generated columns, one per day, cells rendered from an assignment lookup map. Evaluate `sap.ui.table.Table` only if row count becomes a performance issue. A 30-day × 100-nurse grid is 3,000 cells — virtualize or paginate by week from the start.
- **Solver progress** is a `setInterval` poll of `GET /status` every 2 s while status is `SOLVING`. SSE/WebSocket is a Phase 4 nicety; polling is 15 lines and correct.
- **i18n from commit one** (A4): `i18n.properties` (English, the fallback bundle), `i18n_ro.properties`. No string literals in views — enforce by review.

---

## 9. Manual changes and re-optimization

Phase 3 scope. The rule set is in §7's verdict model. Additionally:

- Every manual assignment sets `pinned = true` on that row, so a subsequent re-solve preserves it (`@PlanningPin`).
- **MVP re-optimization is full regeneration** with pins retained. Real-time `ProblemChange`-based repair is deliberately out of scope — it's the single largest complexity jump in the OptaPlanner API and buys nothing until schedules are live.

---

## 10. Testing strategy

| Layer | Tool | Bar |
|---|---|---|
| Constraints | `ConstraintVerifier` | **Every constraint in §6.3 has ≥2 tests: one violating, one satisfying.** Non-negotiable — this is the only way to know the solver does what you think. |
| Domain services | JUnit 5 + Mockito | Lifecycle transitions, manual-assignment verdicts |
| Repositories / migrations | `@DataJpaTest` + Testcontainers Postgres | Flyway runs clean on an empty DB, every time |
| API | `@SpringBootTest` + MockMvc + Testcontainers | Happy path + auth denial per role |
| End-to-end solve | One integration test: seed → solve → assert `hardScore == 0` | This is the regression net for the whole system |
| DST | Dedicated test class | Shifts spanning the March/October Europe/Bucharest transitions — a 23h and a 25h day |
| UI | OPA5, ~4 journeys | Phase 4. Do not invest before the UI stops changing. |

Test data: a `dev` Flyway migration or `CommandLineRunner` seeding **2 departments, 20 nurses, 6 skills, 14 days** — small enough to solve in seconds during development. The 100-nurse / 30-day set is a benchmark fixture, not a dev fixture.

---

## 11. Phases

Each phase ends with something runnable. Do not start the next until the deliverable is true.

### Phase 1 — Vertical slice (the only phase with a fixed shape)

The goal is one thin path through every layer, not breadth.

1. Repo, Gradle wrapper, Spring Boot skeleton, `docker compose up` → Postgres
2. `V1__baseline.sql`, JPA entities, `ddl-auto: validate` passes
3. Seed data (2 departments, 20 nurses, 14 days of shifts + requirements)
4. `planning/` POJOs, `PlanningDataLoader`, slot expansion
5. **Constraints H1–H4 only**, each with `ConstraintVerifier` tests
6. `ScheduleSolverService.solve()`, synchronous, called from a test
7. `POST /api/schedules/{id}/solve` async + `GET /status` + `GET /assignments`
8. UI5 app that renders the nurse × day grid for one schedule

**Done when:** `./gradlew build` is green, `docker compose up` + `bootRun` starts, a seeded schedule solves to `hardScore = 0`, and the grid renders it in the browser.

### Phase 2 — Domain breadth
CRUD (REST + UI5) for nurses, departments, skills, shifts, requirements, availability, preferences. Constraints H5–H7 and S1–S2. Scheduling policy entity and resolution (department → hospital fallback).
**Done when:** a schedule can be produced end-to-end without touching SQL.

### Phase 3 — Quality and control
Soft constraints S3–S7 and weight tuning against the 100-nurse fixture. Manual assignment with the verdict model. Pinning and re-solve. Score explanation / violations view. Solver progress UI.
**Done when:** a human scheduler could plausibly prefer the output to a spreadsheet.

### Phase 4 — Governance and operations
Spring Security with the four roles and per-endpoint authorization. Audit log. Approve/publish. GitHub Actions CI (`./gradlew build` + Testcontainers). Dockerfile for the app. Structured logging. Benchmarks at 20/50/100/300 nurses.
**Done when:** it deploys from CI and you know its performance characteristics.

**Explicitly not in this plan:** microservices, Kubernetes, distributed solving, multi-hospital tenancy, mobile app, notifications, payroll/HR integration, shift bidding, forecasting. They live in `x-nurse-PLAN.md` §44 as the backlog.

---

## 12. Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| OptaPlanner 10.x ↔ Spring Boot 3.5 version friction | Medium | Resolve at Phase 1 step 1, before any code. Let the starter's BOM pick the Spring Boot version if there's conflict. |
| API drift from 8.x-era tutorials (`nullable` → `allowsUnassigned`, package renames post-Apache donation) | **High** | Trust the 10.x Javadoc and the `incubator-kie-optaplanner-quickstarts` repo over blog posts. Most search results for "OptaPlanner nurse rostering" are 8.x. |
| Solver finds no feasible solution and it's unclear why | High | `ScoreExplanation` from Phase 1, not Phase 3, the moment you hit it. Also: seed enough nurses that a solution exists. |
| Timezone/DST bugs discovered late | Medium | `HospitalClock`, `timestamptz`, DST tests in Phase 1. |
| Schedule grid performance in UI5 at 100×30 | Medium | Paginate by week from the start. |
| Scope creep from the vision doc | **High** (A2) | Phase gates above. The backlog is a different file on purpose. |

---

## 13. Definition of done — first coding session

```text
✓ git repo initialized, .gitignore covers build/, node_modules/, src/main/resources/static/
✓ ./gradlew build succeeds (empty app, one context-load test)
✓ docker compose up starts Postgres 16
✓ Flyway V1 applies to an empty database
✓ ddl-auto: validate passes against the JPA entities
✓ Spring Boot starts and /actuator/health returns UP
```

Everything else is Phase 1 step 2 onward.
