# Clinic Management & Electronic Medical Record Application

## Project Agent Guide (`AGENTS.md` / `agent.md`)

> **Purpose:** This document is the working source of truth for AI
> coding agents and developers building the clinic application. Read it
> before making architectural, database, API, UI, or workflow changes.
> Update it when an agreed product decision changes.

------------------------------------------------------------------------

## 1. Product vision

Build a configurable, multi-tenant clinic management and electronic
medical record (EMR) application for independent doctors and small
clinics.

The product should replace paper-based patient registration,
consultation notes, medical history lookup, and prescription writing. It
must also support clinics with different staff structures without
maintaining separate application codebases.

### Core product promise

A doctor should be able to: 1. Find an existing patient or register a
new patient. 2. Review relevant patient demographics, allergies,
conditions, and previous consultations. 3. Record the current
consultation. 4. Add diagnoses, clinical notes, and a prescription. 5.
Finalize the consultation and generate a printable/shareable
prescription. 6. Retrieve the patient's history during a later visit.

Clinics may optionally enable reception and pharmacy workflows. The
application must remain useful when those modules are absent.

### Product category and scope

Initially, this is a **clinic management and EMR application**, not a
full hospital information system (HIS). Prioritize independent doctors
and small clinics. Do not expand into hospital billing, laboratory,
radiology, insurance claims, or complex inventory unless separately
approved.

------------------------------------------------------------------------

## 2. Product operating modes

These are configurations of one product, not separate applications.

### A. Independent doctor / home consultation

Typical capabilities: - Doctor login and clinic/practice setup. -
Patient registration and search. - Patient medical profile, allergies,
and conditions. - Consultation creation without a receptionist or
queue. - Consultation notes, diagnoses, and prescriptions. - Previous
consultation history. - Prescription PDF generation and printing. -
Encounter type such as `CLINIC`, `HOME_VISIT`, or, in a later phase,
`TELECONSULTATION`.

The doctor must be able to register a patient and start a consultation
directly.

### B. Multi-user clinic

Optional capabilities: - Receptionist registers patients and adds
patients to a doctor's queue. - Doctor sees their queue and opens a
consultation. - Multiple doctors can work in the same clinic. - Pharmacy
staff can view authorized, finalized prescriptions. - Clinic
administrator manages clinic settings, staff membership, roles, and
permissions. - Queue and pharmacy features are enabled only when
configured.

A clinic may have any supported combination, for example: - Doctor
only. - Multiple doctors. - Doctor + pharmacy. - Doctor +
receptionist. - Doctor + receptionist + pharmacy. - Multiple doctors +
receptionist + pharmacy.

Do not assume every clinic has all roles.

------------------------------------------------------------------------

## 3. Technology decisions

These are the current preferred technologies. Do not replace them
without an explicit project decision.

  -----------------------------------------------------------------------
  Area                                Technology / approach
  ----------------------------------- -----------------------------------
  Desktop frontend                    Flutter

  Backend                             Java with Spring Boot

  API                                 REST over HTTPS, JSON

  Database                            PostgreSQL

  Database migrations                 Flyway

  Backend security                    Spring Security

  API documentation                   OpenAPI / Swagger

  Backend tests                       JUnit; Testcontainers for database
                                      integration tests

  Flutter architecture                VIPER (see section 11)

  Flutter state management            Riverpod (decided; do not mix with
                                      Bloc or other approaches)

  Backend architecture                DDD layered architecture with SOLID
                                      principles (see section 3)

  Deployment                          Dockerized backend and managed
                                      hosting

  PDF                                 A reliable server-side PDF library
                                      or a controlled Flutter print
                                      workflow
  -----------------------------------------------------------------------

Use a supported Java LTS version compatible with the chosen Spring Boot
release. Pin dependency versions. Avoid unnecessary dependencies.

### Architecture decision: modular monolith first

Build **one Spring Boot deployable application** with clear business
modules. Do not start with microservices.

Suggested backend modules: - `auth` - `clinic` - `user` - `patient` -
`allergy` - `appointment` - `queue` - `consultation` - `prescription` -
`pharmacy` - `audit` - `shared`

Each module should own its business rules and persistence logic. Keep
controllers thin. Put business rules in application/service/domain
layers. Avoid cross-module access to another module's repositories where
a service/API boundary is more appropriate.

Split services only when demonstrated scaling, team ownership, or
operational needs justify the added complexity.

### Backend architecture: DDD layers and SOLID principles

The Spring Boot backend **must** follow Domain-Driven Design (DDD)
layered architecture and SOLID principles. This applies inside every
business module listed above.

#### Layers (per module)

Each module is split into four layers:

  -----------------------------------------------------------------------
  Layer                               Responsibility
  ----------------------------------- -----------------------------------
  `api` (interface/presentation)      REST controllers, request/response
                                      DTOs, input validation, mapping to
                                      application commands/queries, HTTP
                                      error mapping. No business rules.

  `application`                       Use cases / application services,
                                      commands and queries, transaction
                                      boundaries, tenant and permission
                                      checks, orchestration of domain
                                      objects. No HTTP or persistence
                                      details.

  `domain`                            Aggregates, entities, value
                                      objects, domain services, domain
                                      events, domain exceptions, and
                                      repository interfaces. All core
                                      business rules and invariants live
                                      here. No Spring web, JPA, or other
                                      framework dependencies.

  `infrastructure`                    JPA persistence models, repository
                                      implementations, Flyway-related
                                      persistence code, PDF generation,
                                      security/token adapters, and other
                                      external integrations.
  -----------------------------------------------------------------------

Suggested package layout:

```
com.<org>.clinic.<module>
├── api              # controllers, DTOs, API mappers
├── application      # use cases, commands, queries
├── domain           # aggregates, entities, value objects,
│                    # domain services, repository interfaces, events
└── infrastructure   # JPA entities, repository impls, adapters
```

#### Dependency rules

1.  Dependencies point inward: `api` → `application` → `domain`.
    `infrastructure` depends on `domain` (and `application` ports) by
    implementing its interfaces.
2.  The `domain` layer must not depend on `api`, `application`,
    `infrastructure`, Spring MVC, or JPA.
3.  Controllers call application services/use cases only. Controllers
    must never call repositories or contain business logic.
4.  Repository **interfaces** are declared in `domain`; their
    implementations live in `infrastructure`.
5.  JPA entities are persistence models in `infrastructure` and are
    mapped to/from domain objects. They are never returned from the API
    and never used as the domain model unless an exception is
    explicitly approved and recorded in the change log.
6.  Cross-module access goes through the other module's application
    service/API or through domain events. Reference other modules'
    aggregates by ID, not by object reference, and never use another
    module's repositories directly.

#### Domain modelling guidance

-   Identify aggregates and keep aggregate boundaries small. Likely
    aggregate roots: `Clinic`, `ClinicUser`, `Patient` / `ClinicPatient`,
    `QueueEntry`, `Consultation`, `Prescription` (with its items).
-   Enforce invariants inside aggregates, for example: a finalized
    prescription cannot be edited, only amended; valid queue and
    consultation state transitions; a prescription must have at least
    one valid item before finalization.
-   Use value objects for concepts such as `ClinicId`, `PatientId`,
    `PatientNumber`, dosage, frequency, and duration instead of raw
    strings and primitives where it improves correctness.
-   Use domain events (for example, `PrescriptionFinalized`) for audit
    and for cross-module reactions such as pharmacy worklists.
-   Use ubiquitous language: class, method, and API names must match the
    clinical/business terms used in this document.
-   Every aggregate carries its `clinic_id` (tenant scope), and
    repository methods require the clinic context.
-   Transactions are opened in the `application` layer, one use case per
    transaction, and one aggregate modified per transaction where
    practical.

#### SOLID principles (how they apply here)

-   **Single Responsibility:** one use case per application service
    class where practical (for example, `FinalizePrescriptionUseCase`);
    classes have one reason to change.
-   **Open/Closed:** extend behavior through new implementations or
    strategies (for example, new PDF templates or notification
    channels) rather than modifying stable code with growing
    conditionals.
-   **Liskov Substitution:** implementations of a domain interface must
    be substitutable without changing correctness (including tenant
    scoping behavior).
-   **Interface Segregation:** prefer small, focused interfaces (for
    example, separate read and write repository ports) over large
    "god" interfaces.
-   **Dependency Inversion:** the `application` and `domain` layers
    depend on abstractions; infrastructure provides the
    implementations. Use constructor injection; avoid field injection
    and static/global state.

#### Enforcement

-   Add architecture tests (for example, ArchUnit, as a test-only
    dependency) that fail the build if layer dependency rules are
    violated.
-   Unit-test domain logic without Spring or a database. Use
    integration tests (Testcontainers) for infrastructure and API
    behavior.
-   Reject changes in review that place business rules in controllers
    or repositories, or that leak JPA entities into the API.

### Deployment model

-   Each clinic installs the same Flutter Desktop application on its
    computers.
-   All installations connect securely to the centrally hosted Spring
    Boot API.
-   One backend deployment serves multiple clinic tenants.
-   PostgreSQL is accessed only by the backend.
-   Flutter must never connect directly to PostgreSQL.
-   Backend updates are deployed centrally; desktop application updates
    use a controlled, signed distribution/update process.

The desktop client must not contain database credentials or privileged
server secrets.

------------------------------------------------------------------------

## 4. Multi-tenancy and clinic data isolation

### Default isolation rule

**Patient records are clinic-scoped by default.** A clinic must not be
able to read or modify another clinic's patient records.

Do not automatically share patient records across clinics. Any future
cross-clinic sharing must be a separately designed, explicit,
consent-aware feature with appropriate authorization and auditability.

### Tenant model

-   `clinics` represents a tenant/practice.
-   `users` represents a login identity.
-   `clinic_users` links users to clinics and holds clinic-specific
    membership/status/role relationships.
-   A user may potentially belong to more than one clinic, but every
    request must operate within an explicitly authorized clinic context.
-   Clinic-owned records must carry a `clinic_id` directly or be safely
    scoped through a parent relationship.
-   Use database foreign keys and, where appropriate, composite
    uniqueness constraints that include `clinic_id`.

### Mandatory tenant security rules

1.  Derive the active clinic from an authenticated and authorized
    membership/context. Never trust a client-supplied `clinic_id` by
    itself.
2.  Enforce tenant authorization in the backend for every read and
    write.
3.  Never rely on Flutter screen visibility or client-side filtering for
    security.
4.  Scope repository queries by clinic.
5.  Prevent cross-tenant references, including patient, consultation,
    prescription, doctor, and pharmacy references.
6.  Add automated negative tests proving Clinic A cannot access Clinic B
    data by guessing IDs or changing request parameters.
7.  Audit sensitive access and clinical record changes.
8.  Do not leak patient details in exception messages, logs, analytics,
    or URLs.

PostgreSQL Row-Level Security may be evaluated as an additional defense,
but it does not replace correct application authorization.

------------------------------------------------------------------------

## 5. Roles and permissions

Use role-based access control (RBAC), with clinic-level role assignments
and explicit permissions. Roles are bundles of permissions, not
hard-coded UI identities.

### Initial roles

#### Clinic administrator

-   Configure clinic profile and enabled modules.
-   Invite/deactivate clinic users.
-   Assign roles and permissions within allowed limits.
-   View operational configuration and reports.
-   Clinical-record access must be separately authorized; administrator
    status must not automatically mean unrestricted clinical access.

#### Receptionist

-   Register patients.
-   Search patients within their clinic.
-   Update permitted demographic/contact details.
-   Create/manage appointments or queue entries when enabled.
-   View queue status.
-   Must not edit or finalize clinical notes or prescriptions unless
    explicitly granted an additional permission.

#### Doctor

-   Search and view authorized patient records.
-   Register a patient if permitted.
-   View allergies, conditions, and consultation history.
-   Create and edit their permitted draft consultations.
-   Record vitals, clinical notes, diagnoses, and treatment plans.
-   Create, review, and finalize prescriptions.
-   Amend/correct finalized records only through a controlled, auditable
    process.

#### Pharmacy staff

-   View prescriptions they are authorized to access, normally finalized
    prescriptions.
-   View prescribing doctor and medication instructions.
-   Record dispensing status/details if the pharmacy/dispensing feature
    is enabled.
-   Must not alter the original prescription or clinical notes.

### Permission examples

Use stable permission identifiers such as: - `PATIENT_CREATE` -
`PATIENT_VIEW` - `PATIENT_UPDATE` - `PATIENT_SEARCH` - `QUEUE_VIEW` -
`QUEUE_MANAGE` - `CONSULTATION_CREATE` - `CONSULTATION_VIEW` -
`CONSULTATION_EDIT_DRAFT` - `CONSULTATION_FINALIZE` -
`PRESCRIPTION_CREATE` - `PRESCRIPTION_VIEW` - `PRESCRIPTION_FINALIZE` -
`PRESCRIPTION_AMEND` - `PHARMACY_DISPENSE` - `CLINIC_USER_MANAGE` -
`CLINIC_SETTINGS_MANAGE` - `AUDIT_LOG_VIEW`

Check both permission and tenant scope. Where necessary, also check
record ownership/assignment and clinical workflow state.

------------------------------------------------------------------------

## 6. Clinic configuration and feature flags

Keep feature configuration separate from user authorization.

Potential clinic-level settings: - `reception_enabled` -
`pharmacy_enabled` - `queue_enabled` - `home_visit_enabled` -
`multiple_doctors_enabled`

The exact storage design may use a `clinic_modules` table, typed columns
on `clinic_settings`, or a feature configuration model. Choose a
maintainable design and validate configuration on the backend.

Rules: - A disabled module must not be usable through the API, even if a
client calls its endpoint directly. - Enabling pharmacy does not grant
every user pharmacy permissions. - Queue behavior must be optional. - A
doctor-only clinic must be able to create consultations without queue
entries. - Home visits should not require reception or queue
functionality. - Flutter may hide unavailable features, but backend
enforcement is mandatory. - Do not scatter hard-coded clinic-specific
conditions throughout the code. Centralize feature and permission
checks.

------------------------------------------------------------------------

## 7. Core domain and database model

The following is a conceptual starting model, not a mandate to create
every table in the first migration. Refine it with constraints and
relationships before implementation.

### Tenant and access management

-   `clinics`
-   `users`
-   `clinic_users`
-   `roles`
-   `permissions`
-   `role_permissions`
-   `clinic_settings` and/or `clinic_modules`
-   Optional `user_sessions` / refresh-token storage, depending on
    authentication design

### Patient management

-   `patients`
-   `clinic_patients`
-   `patient_allergies`
-   `patient_conditions`

Keep stable patient identity separate from clinic-specific registration.
`clinic_patients` can hold the clinic-specific patient number and
relationship. Do not assume a global patient record is visible across
tenants.

Patient profile fields should be selected deliberately and kept to the
minimum needed. Typical fields may include: - Clinic patient number -
Full name - Date of birth or age representation, as appropriate -
Sex/gender fields appropriate to clinical and legal needs -
Phone/contact details - Address, if needed - Emergency contact, if
needed - Relevant demographic metadata

Avoid collecting unnecessary sensitive information.

### Appointment and queue

-   `appointments`
-   `consultation_queue`
-   `consultation_status_history` or a generalized workflow history
    table

Appointments and queue entries are not the same thing. A walk-in may
have a queue entry without a prior appointment. A booked appointment may
later create or join a queue.

### Consultation / EMR

-   `consultations`
-   `clinical_notes`
-   `vitals`
-   `diagnoses`
-   Optional structured treatment-plan records
-   `consultation_status_history`

Each consultation represents one encounter. It should record: - Clinic -
Patient - Treating doctor - Encounter type (`CLINIC`, `HOME_VISIT`,
etc.) - Start and completion timestamps - Status - Relevant clinical
content and author metadata

A patient with five visits should have one clinic-patient relationship
and five separate consultation records, not one overwritten history
field.

### Prescription

-   `prescriptions`
-   `prescription_items`
-   `medications` or a curated medication catalogue
-   Optional `dispensing_records`

Prescription header may include: - Clinic, patient, consultation,
prescribing doctor - Prescription number - Status (`DRAFT`, `FINALIZED`,
`AMENDED`, `CANCELLED` as carefully defined) - Created/finalized
timestamps - Version or amendment reference - Relevant instructions

Prescription item may include: - Medication/catalogue reference or
entered medication name - Strength - Dose - Dose unit - Route -
Frequency - Duration - Quantity - Instructions - Substitution/other
relevant fields if required

Do not assume all prescriptions are simple tablets. Design dosage fields
carefully and avoid ambiguous free-text-only representations where
structured data is feasible.

### Audit

-   `audit_logs`
-   Optional dedicated immutable clinical-record version/amendment
    tables

Record actor, clinic, action, target record, timestamp, and relevant
before/after metadata where appropriate. Avoid putting unnecessary
patient clinical content or secrets in logs.

### Database implementation rules

-   Use Flyway for every schema change.
-   Use UUIDs or another consistently selected identifier strategy.
-   Use foreign keys, check constraints, and unique constraints.
-   Add indexes for common tenant-scoped access patterns.
-   Use UTC timestamps in storage; convert to the clinic/user timezone
    in the UI.
-   Use pagination for patient search/history and other potentially
    large result sets.
-   Avoid unbounded fetches and N+1 queries.
-   Use optimistic locking/version columns where concurrent editing is
    possible.
-   Use transactions for consultation finalization and prescription
    finalization.
-   Do not delete finalized clinical records as an ordinary CRUD
    operation. Use controlled correction, cancellation, retention, and
    archival policies.

Useful initial indexes include: - `(clinic_id, patient_id, started_at)`
for consultation history. - `(clinic_id, patient_number)` for clinic
patient lookup. - `(clinic_id, status, created_at)` for
queue/prescription worklists, adjusted to actual query patterns.

------------------------------------------------------------------------

## 8. Main clinical workflow

### Standard clinic workflow

1.  Patient arrives.
2.  Receptionist searches for the patient within the clinic.
3.  If no match exists, receptionist registers the patient.
4.  Receptionist creates an appointment or queue entry, if those
    features are enabled.
5.  Doctor opens the patient and current encounter.
6.  Doctor reviews demographics, allergies, conditions, and relevant
    previous consultations.
7.  Doctor records vitals where applicable.
8.  Doctor records clinical notes and diagnoses.
9.  Doctor creates and reviews the prescription.
10. Doctor finalizes the consultation and prescription.
11. Authorized pharmacy staff can view the finalized prescription when
    pharmacy is enabled.
12. Prescription can be printed/shared through an approved workflow.
13. Consultation remains available in the patient's clinic-scoped
    history.

### Doctor-only / home consultation workflow

1.  Doctor searches for or registers a patient.
2.  Doctor creates an encounter directly, without a queue.
3.  Doctor records encounter type, for example `HOME_VISIT`.
4.  Doctor reviews available patient history and allergies.
5.  Doctor records notes, diagnoses, and prescription.
6.  Doctor finalizes and prints/shares the prescription.
7.  Encounter is stored in the patient's clinic-scoped history.

### Queue states

Define a controlled state machine rather than arbitrary status strings.
Candidate states: - `WAITING` - `CALLED` - `IN_CONSULTATION` -
`COMPLETED` - `CANCELLED` - `NO_SHOW`

Validate allowed transitions on the backend and record status history.
Do not use queue status as the consultation status; they are related but
separate concepts.

### Consultation states

Candidate states: - `DRAFT` - `IN_PROGRESS` - `COMPLETED` - `CANCELLED`

Finalize the state model before implementing APIs. Define which roles
may transition each state and whether a completed consultation can be
amended.

------------------------------------------------------------------------

## 9. Prescription integrity and clinical safety

This is a high-priority domain rule.

-   Draft prescriptions may be edited by an authorized doctor.
-   Finalized prescriptions must not be silently overwritten.
-   Corrections must create an auditable amendment, replacement, or
    version according to the approved workflow.
-   Preserve who prescribed, who amended, and when.
-   Pharmacy users may view/dispense authorized finalized prescriptions
    but may not modify the original prescription.
-   Printing/exporting must not create another consultation or duplicate
    prescription.
-   Clearly display patient identity, prescribing clinician, clinic
    identity, date, medication, strength, dose, route, frequency,
    duration, and instructions as applicable.
-   Validate required fields and sensible values; do not invent clinical
    defaults.
-   Do not generate medical advice or choose medications autonomously.
-   Clinical decision support, interaction checking, and medication
    catalogues require validated sources and a separately approved
    safety design.

The application is a record-keeping and workflow tool. It must not imply
that software validation replaces a clinician's judgment.

------------------------------------------------------------------------

## 10. API design principles

Use versioned REST endpoints, for example `/api/v1/...`.

Possible endpoint groups (final names should follow one consistent
convention):

### Authentication and clinic context

-   `POST /api/v1/auth/login`
-   `POST /api/v1/auth/refresh` if refresh tokens are used
-   `POST /api/v1/auth/logout`
-   `GET /api/v1/me`
-   `GET /api/v1/me/clinics`
-   `POST /api/v1/me/active-clinic` only if active clinic switching is
    part of the design

### Clinic and configuration

-   `GET /api/v1/clinics/current`
-   `GET /api/v1/clinics/current/configuration`
-   `PATCH /api/v1/clinics/current/configuration`
-   Clinic staff and role endpoints guarded by permissions

### Patients

-   `POST /api/v1/patients`
-   `GET /api/v1/patients`
-   `GET /api/v1/patients/{patientId}`
-   `PATCH /api/v1/patients/{patientId}`
-   `GET /api/v1/patients/{patientId}/allergies`
-   `GET /api/v1/patients/{patientId}/conditions`
-   `GET /api/v1/patients/{patientId}/consultations`

### Queue

-   `POST /api/v1/queue`
-   `GET /api/v1/queue`
-   `PATCH /api/v1/queue/{queueEntryId}/status`

### Consultations

-   `POST /api/v1/consultations`
-   `GET /api/v1/consultations/{consultationId}`
-   `PATCH /api/v1/consultations/{consultationId}` for permitted
    drafts/in-progress records
-   `POST /api/v1/consultations/{consultationId}/finalize`

### Prescriptions

-   `POST /api/v1/consultations/{consultationId}/prescriptions`
-   `GET /api/v1/prescriptions/{prescriptionId}`
-   `PATCH /api/v1/prescriptions/{prescriptionId}` for drafts only
-   `POST /api/v1/prescriptions/{prescriptionId}/finalize`
-   `POST /api/v1/prescriptions/{prescriptionId}/amendments`
-   Pharmacy worklist/dispensing endpoints only when that module is
    enabled

API rules: - Use DTOs; never serialize JPA entities directly. - Validate
all input on the backend. - Use consistent error response structures and
meaningful HTTP status codes. - Apply pagination, sorting allowlists,
and bounded search. - Never accept an arbitrary tenant ID as proof of
authorization. - Verify every referenced record belongs to the
authorized clinic. - Make finalization operations idempotent or protect
against duplicate submissions. - Use optimistic locking or equivalent
conflict handling for concurrent edits. - Document APIs with OpenAPI.

------------------------------------------------------------------------

## 11. Flutter application guidance

### Suggested high-level screens

#### Shared

-   Login
-   Clinic selection (if a user belongs to multiple clinics)
-   Main dashboard
-   Profile/settings
-   Notifications or task indicators, if needed

#### Doctor

-   Doctor dashboard
-   Patient search/list
-   Patient registration/edit
-   Patient profile
-   Patient history
-   New consultation
-   Consultation workspace
-   Prescription editor
-   Prescription preview/print

#### Receptionist (only when enabled and authorized)

-   Reception dashboard
-   Patient registration/search
-   Appointment/queue screen
-   Queue status management

#### Pharmacy (only when enabled and authorized)

-   Prescription worklist
-   Prescription detail
-   Dispensing workflow, if implemented

#### Clinic administrator

-   Clinic settings
-   Feature/module settings
-   Staff management
-   Role/permission management
-   Audit view, if authorized

### UI principles

-   Desktop-first layouts with keyboard-friendly workflows.
-   Fast patient search and minimal clicks for common doctor tasks.
-   Clear patient identity and clinic context on clinical screens.
-   Prominent allergy visibility without relying on color alone.
-   Distinguish draft, in-progress, and finalized records clearly.
-   Do not hide critical clinical warnings in secondary tabs.
-   Use responsive layouts so the UI can later support tablet/mobile
    where practical.
-   Keep API, state, and UI layers separate.
-   Do not put business authorization rules only in Flutter.
-   Handle loading, empty, validation, network, and permission-denied
    states explicitly.
-   Avoid retaining patient details after logout or clinic switching.

### Flutter architecture: VIPER with Riverpod

The Flutter application **must** follow the VIPER architecture, using
**Riverpod** for state management and dependency injection. Do not use
Bloc, Provider, GetX, or other state-management approaches alongside it.

#### VIPER components and their Flutter/Riverpod mapping

  -----------------------------------------------------------------------
  VIPER component                     Implementation in this project
  ----------------------------------- -----------------------------------
  **View**                            Flutter widgets (`ConsumerWidget` /
                                      `ConsumerStatefulWidget`). Renders
                                      state and forwards user actions to
                                      the Presenter. No business logic,
                                      no API calls.

  **Interactor**                      Use-case classes holding the
                                      application/business logic for one
                                      feature action (for example,
                                      `SearchPatientsInteractor`). Calls
                                      repositories. No Flutter imports.

  **Presenter**                       Riverpod `Notifier` /
                                      `AsyncNotifier` that holds the
                                      screen state, calls Interactors,
                                      and maps results into view state
                                      (loading, data, empty, error,
                                      permission-denied).

  **Entity**                          Immutable domain models (for
                                      example, `Patient`,
                                      `Consultation`, `Prescription`)
                                      independent of Flutter and of API
                                      JSON shapes.

  **Router**                          Navigation layer (for example,
                                      `go_router`) exposed through a
                                      Riverpod provider. Presenters
                                      request navigation through the
                                      Router; Views do not hard-code
                                      cross-feature navigation logic.
  -----------------------------------------------------------------------

Data flow: **View → Presenter → Interactor → Repository → API client**,
and back as immutable state: **API DTO → Entity → Presenter state →
View**.

#### Suggested feature-first folder structure

```
lib/
├── core/                    # theme, error types, network client,
│                            # secure storage, shared widgets, utilities
├── app/                     # app bootstrap, root router, global providers
└── features/
    └── <feature>/           # e.g. auth, patient, consultation, prescription
        ├── view/            # screens and widgets
        ├── presenter/       # Riverpod notifiers + view state classes
        ├── interactor/      # use cases
        ├── entity/          # domain entities
        ├── router/          # feature routes / navigation contracts
        └── data/            # repository interface + implementation,
                             # API data sources, DTOs, mappers
```

#### Rules

1.  Views depend only on Presenters (through Riverpod providers). Views
    must not call Interactors, repositories, or API clients directly.
2.  Presenters depend on Interactors, not on repositories or API
    clients directly.
3.  Interactors depend on repository **interfaces**; repository
    implementations are provided through Riverpod providers so they can
    be overridden in tests.
4.  Interactors and Entities must not import Flutter (`package:flutter`)
    libraries.
5.  Entities are separate from API DTOs; map DTOs to Entities in the
    `data` layer.
6.  Keep state immutable. Model view state explicitly (loading, data,
    empty, error, permission-denied) rather than with loose booleans.
7.  Use Riverpod providers for all dependency injection; avoid
    singletons, service locators, and global mutable state.
8.  Clear patient-related state when the user logs out or switches
    clinics (for example, by scoping or invalidating providers).
9.  Do not put authorization or tenant rules only in Flutter; the
    backend remains the source of truth.
10. Do not put business logic in widgets or Presenters; it belongs in
    Interactors (and ultimately the backend).
11. Do not add shortcuts that skip layers, even for small screens.
    Consistency matters more than brevity.

#### Testing

-   Unit-test Interactors with mocked repositories.
-   Unit-test Presenters with Riverpod `ProviderContainer` overrides.
-   Widget-test Views with mocked Presenters/providers.
-   Add integration tests for the first vertical slice.

------------------------------------------------------------------------

## 12. Security, privacy, and operations

Medical data is sensitive. Security is a release requirement, not a
later enhancement.

Minimum requirements: - HTTPS for all client-server communication. -
Secure password hashing using a modern password encoder. - Short-lived
access credentials and a considered refresh/session strategy. - Secure
token storage on desktop; never store secrets in plain text. -
Authorization checks on every protected backend operation. - Clinic
tenant isolation tests. - Encryption at rest and encrypted backups. -
Backup restoration tests, not just backup creation. - Audit trails for
important clinical changes and sensitive access. - Rate limiting and
abuse protection for authentication and sensitive endpoints. - Safe
error messages and redacted logs. - Dependency and container image
updates. - Secrets managed outside source control. - Controlled desktop
application distribution and updates. - Session revocation when staff
access is removed. - Defined retention, export, correction, and deletion
policies consistent with applicable law.

For deployment in India, assess the applicability of the Digital
Personal Data Protection Act and rules in force at launch, along with
applicable medical-record, professional, and other regulatory
requirements. Obtain qualified legal/security advice before production
use. If ABDM integration is pursued, treat it as a separate standards
and consent project.

### Offline behavior

MVP default: online-only access is simpler and safer.

Do not casually cache complete patient records on desktop devices. If
offline use becomes a requirement, design it as a separate feature
with: - Encrypted local storage. - Device/user authorization. - Minimal
offline data. - Expiry and revocation behavior. - Conflict resolution
and synchronization. - Auditability. - Protection against data remaining
after logout or device loss.

------------------------------------------------------------------------

## 13. Delivery roadmap

### Phase 1 --- Core doctor application (MVP)

Deliver: - Clinic setup and doctor login. - Patient registration and
search. - Clinic-scoped patient identity. - Patient allergies and
conditions. - Consultation creation and history. - Clinical notes and
diagnoses. - Prescription builder and finalization. - Prescription
PDF/print. - Basic clinic/user authorization. - Audit events for
important clinical actions. - Flyway migrations and automated tests.

**Acceptance outcome:** A doctor can complete and retrieve a patient
consultation without using a paper prescription pad, subject to clinical
and regulatory validation.

### Phase 2 --- Multi-user clinic

Deliver: - Clinic staff membership. - RBAC and permission management. -
Receptionist workflow. - Appointment/queue workflow. - Multiple doctor
support. - Pharmacy prescription worklist. - Optional dispensing
tracking. - Configurable clinic modules. - Expanded audit and
operational reporting.

**Acceptance outcome:** One backend and one desktop application support
clinics with different role/module combinations.

### Phase 3 --- Commercial readiness

Deliver: - Production hosting and monitoring. - Automated encrypted
backups and tested restoration. - Tenant-isolation/security testing. -
Desktop installer and update process. - Data export and operational
recovery procedures. - Support/onboarding documentation. - Subscription
and billing only after the product and pricing are validated. - Pilot
rollout and feedback process.

### Phase 4 --- Expansion

Consider based on customer demand: - Android/mobile access. - Follow-up
reminders. - Advanced appointment scheduling. - Inventory and
dispensing. - Reporting and analytics. - Integrations, including ABDM
where appropriate. - Carefully designed offline functionality.

Do not implement all future features in the MVP.

------------------------------------------------------------------------

## 14. Business model assumptions

A possible subscription model may be tested, but the following prices
are **illustrative assumptions, not researched market prices or approved
commitments**:

  Plan concept            Example monthly price Target
  --------------------- ----------------------- ------------------------------------
  Solo Doctor                              ₹799 Independent doctor
  Clinic                                 ₹1,999 Small clinic
  Multi-doctor Clinic                    ₹3,999 Clinic with multiple staff/doctors

Validate willingness to pay, competitor pricing, support costs, hosting,
taxes, sales costs, and retention before adopting prices.

Do not build billing complexity before confirming that doctors and
clinics use and value the core workflow.

------------------------------------------------------------------------

## 15. Engineering standards for AI coding agents

When working in this repository:

1.  Read this file and inspect the existing repository before proposing
    or changing code.
2.  Preserve existing project conventions unless they conflict with a
    documented requirement.
3.  Do not assume files, modules, dependencies, endpoints, or database
    tables exist; inspect first.
4.  Before implementation, summarize the change, affected modules,
    migration/API implications, and tests.
5.  Implement in small, reviewable increments.
6.  Do not introduce microservices or new infrastructure without an
    explicit reason and approval.
7.  Do not hard-code clinic IDs, user IDs, role combinations, secrets,
    or environment-specific URLs.
8.  Do not bypass tenant authorization for convenience.
9.  Do not expose JPA entities directly through API responses.
10. Do not silently change finalized clinical records.
11. Use Flyway migrations for database changes; never rely on automatic
    schema generation in production.
12. Add unit and integration tests for business rules and
    security-sensitive behavior.
13. Add negative tests for cross-tenant access and insufficient
    permissions.
14. Keep API documentation and examples updated.
15. Handle errors safely without leaking medical data.
16. Avoid unnecessary abstractions, speculative features, and premature
    optimization.
17. If requirements conflict or a clinical/legal decision is unclear,
    stop and ask for clarification rather than inventing behavior.
18. Never fabricate clinical advice, medicine instructions, regulatory
    compliance, or test results.
19. Do not claim a feature is complete unless it has been implemented
    and relevant tests have passed.
20. At the end of a task, report files changed, migrations added,
    behavior implemented, tests run/results, known limitations, and
    recommended next step.
21. Follow the mandated architecture: VIPER + Riverpod in Flutter
    (section 11) and DDD layers + SOLID in the Spring Boot backend
    (section 3). Do not bypass layers or mix architectural styles.
22. If a requirement seems to need an exception to these architectural
    rules, stop and ask before proceeding, then record the decision in
    the change log.

### Definition of done

A task is complete only when: - The requested behavior is implemented. -
Input validation and authorization are present. - Tenant scope is
enforced. - Database changes have migrations where applicable. -
Relevant tests are added and pass. - Errors and edge cases are
handled. - API/UI documentation is updated where needed. - No secrets or
patient data are accidentally exposed. - The implementation does not
break existing workflows. - The code follows the mandated
architecture (VIPER + Riverpod in Flutter; DDD layers + SOLID in the
backend) and architecture tests pass.

------------------------------------------------------------------------

## 16. MVP acceptance criteria

Use these as initial end-to-end release checks.

### Patient

-   A doctor can register a patient.
-   Duplicate-search behavior is clear and does not silently merge
    records.
-   A patient can be searched within the active clinic.
-   Patient history is clinic-scoped and paginated.
-   Allergies and conditions are visible to an authorized doctor.

### Consultation

-   A doctor can create a consultation without a queue.
-   A receptionist can create a queue entry when the queue module is
    enabled and permission is granted.
-   A doctor can open the correct patient's encounter.
-   Clinical notes and diagnoses are saved against the correct
    encounter.
-   Completed encounters remain available as history.
-   A later consultation does not overwrite an earlier one.

### Prescription

-   A doctor can create and edit a draft prescription.
-   Required prescription fields are validated.
-   A doctor can finalize a prescription.
-   A finalized prescription cannot be silently edited.
-   Amendments preserve the original and record the actor/time.
-   Printing/exporting does not duplicate the encounter or prescription.
-   Pharmacy can see finalized prescriptions only when enabled and
    authorized.

### Configuration and security

-   Doctor-only configuration works without a receptionist or queue.
-   Pharmacy functionality is unavailable when disabled.
-   Enabled modules still require user permissions.
-   Clinic A cannot access Clinic B patients, consultations,
    prescriptions, or queue entries.
-   A removed/deactivated clinic user cannot continue using that
    clinic's access.
-   Sensitive actions are auditable.

### Reliability

-   Database migrations run from a clean database.
-   Automated tests cover the principal workflows.
-   Backup restoration is tested before production.
-   API failures and network interruptions do not silently lose a draft.
-   Patient information is not exposed in ordinary logs.

------------------------------------------------------------------------

## 17. Decisions still requiring product-owner confirmation

Do not silently invent answers to these questions. Use safe, explicit
defaults for development and record the final decision here.

1.  **Clinic tenancy:** Confirm clinic-scoped patient records as the
    default. Current recommendation: yes.
2.  **Deployment geography:** Confirm initial hosting region and data
    residency requirements. India is the likely initial market, but
    confirm.
3.  **Desktop operating systems:** Confirm Windows-only MVP or Windows +
    Linux. Do not promise macOS until tested.
4.  **Authentication:** Decide whether email, phone, username, or
    clinic-specific login identifiers are required.
5.  **Doctor identity:** Determine required clinician registration,
    qualifications, registration number, and prescription-signature
    details.
6.  **Patient identifiers:** Decide clinic-generated patient number
    format and duplicate detection rules.
7.  **Prescription format:** Confirm required fields, clinic branding,
    doctor details, signature/stamp, language, and print size.
8.  **Medication catalogue:** Decide whether MVP uses free-text
    medication entry, a curated catalogue, or both.
9.  **Consultation finalization:** Confirm whether consultation and
    prescription finalize together or independently.
10. **Amendments:** Define who may amend a finalized record and how the
    patient/pharmacy sees corrections.
11. **Queue:** Confirm whether appointments are included in Phase 2 or
    whether walk-in queue is enough initially.
12. **Pharmacy:** Confirm whether Phase 2 only displays prescriptions or
    also tracks dispensing.
13. **Patient access:** Confirm whether patients will have a portal/app
    in a later phase.
14. **Offline use:** Current recommendation: not in MVP.
15. **Regulatory/privacy:** Obtain qualified review of applicable Indian
    requirements before production.

------------------------------------------------------------------------

## 18. Recommended first implementation sequence

For a new or existing repository, proceed in this order:

1.  Inspect repository and document current state.
2.  Establish backend build, configuration profiles, PostgreSQL
    connection, and Flyway.
3.  Create clinic, user, clinic membership, role, and permission
    foundations.
4.  Implement authentication and clinic-scoped authorization.
5.  Add patient and clinic-patient registration/search.
6.  Add allergies and conditions.
7.  Add consultation creation, notes, diagnoses, and history.
8.  Add prescription drafts, items, finalization, and amendments.
9.  Add prescription PDF/print workflow.
10. Build the Flutter login, patient search/profile, consultation, and
    prescription screens.
11. Add end-to-end tests for doctor-only workflows.
12. Pilot with a doctor and refine before beginning
    receptionist/pharmacy modules.

### First vertical slice

The first end-to-end slice should be:

**Login → select clinic → register/search patient → open patient profile
→ create consultation → record notes → add prescription → finalize →
print → view consultation history.**

Do not begin by building every screen independently. Build this complete
workflow through Flutter, API, service layer, and PostgreSQL so the
architecture is validated early.

------------------------------------------------------------------------

## 19. Change log

Maintain this section when project decisions change.

  -----------------------------------------------------------------------
  Date                    Decision/change         Reason
  ----------------------- ----------------------- -----------------------
  2026-11-11              Flutter Desktop +       Initial project
                          Spring Boot +           direction consolidated
                          PostgreSQL; modular     from planning
                          monolith; clinic-scoped discussion
                          data by default;        
                          configurable modules    
                          and RBAC                

  2026-11-13              Flutter must use VIPER  Product-owner
                          with Riverpod; Spring   architecture
                          Boot backend must       requirement
                          follow SOLID and DDD    
                          layered architecture    

  -----------------------------------------------------------------------

------------------------------------------------------------------------

**Instruction to the coding agent:** Treat this document as the project
baseline. Prefer a secure, testable, maintainable clinic workflow over
adding features quickly. Ask before changing foundational product
decisions.
