# Functional Modules

This document defines the functional modules for the connected healthcare
solution described in [README.md](README.md). It identifies the internal
modules, external modules, module functions, layered responsibilities, and
the way modules invoke one another.

The design is intentionally federated. Each facility retains detailed
clinical and billing information, while shared services coordinate identity,
consent, discovery, and exchange. The module boundaries are logical
boundaries; they can be deployed as separate services or grouped into a
facility application.

## 1. Internal functional modules

| ID | Module | Description | Main owner |
| --- | --- | --- | --- |
| M1 | Patient Identity & Login | Creates a common patient ID, verifies users, and manages patient cards and login details. | Shared identity service |
| M2 | Consent & Record Sharing | Manages patient permission, finds which hospital has the medical records, allows secure sharing, and keeps track of record access. | Shared trust service |
| M3 | Platform & Integration | Provides common rules, data formats, tools, and guidelines to help hospitals connect and exchange information. Also supports offline data synchronization. | Platform/ecosystem team |
| M4 | Registration & Appointments | Handles patient registration, card-related services, appointment booking, and hospital check-in. | Facility |
| M5 | Medical Services | Handles patient visits, medical records, test reports, diagnoses, prescriptions, and treatment details. | Facility |
| M6 | Patient Services & Hospital Management | Handles patient information, reminders, follow-ups, billing, insurance claims, and hospital administration. | Facility |

### M1 — Patient Identity & Login

**Functions**

- Register a patient and create an opaque unified patient ID (UPID).
- Match a registration to an existing UPID and create a duplicate-review
  candidate when matching is uncertain.
- Authenticate patients, staff, and administrators.
- Resolve a presented card or credential to a UPID.
- Issue, bind, replace, suspend, and revoke card credentials.
- Check current credential status for every card use.
- Return the caller identity and authorization claims needed by M2, M4, M5,
  and M6.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Interface | Identity, authentication, and card APIs; reader adapter port |
| Application | Registration, authentication, credential lifecycle, and duplicate-review workflows |
| Domain | UPID, identity matching, credential status, and revocation rules |
| Infrastructure | Identity store, card-status store, token provider, and QR/RFID/NFC/manual reader adapters |

### M2 — Consent & Record Sharing

**Functions**

- Register facilities, providers, supported record categories, and trust
  metadata.
- Capture, grant, withdraw, and evaluate patient consent.
- Validate the caller and resolve the UPID through M1.
- Locate the facility that owns a requested record.
- Authorize a record request using caller, patient, purpose, category, and
  consent.
- Forward an authorized request to the originating facility API.
- Return permitted records with provenance and source information.
- Publish and update record pointers and allowed summaries.
- Write an append-only audit entry for every decision and disclosure.
- Apply rate limits and reject invalid or unauthorized exchange requests.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Interface | Versioned exchange API and facility-to-facility API |
| Application | Consent evaluation, record discovery, authorization, forwarding, and audit orchestration |
| Domain | Consent policy, disclosure rules, trust relationships, record-locator rules |
| Infrastructure | Consent store, provider directory, record locator, audit log, certificates, and gateway transport |

### M3 — Platform & Integration

**Functions**

- Define versioned API contracts, schemas, error formats, and code sets.
- Provide SDKs and reference implementations for the module interfaces.
- Provide vendor-neutral card-reader and facility-integration adapters.
- Run conformance and compatibility tests for implementations.
- Provide the embedded offline cache and durable operation queue.
- Attach idempotency keys and schema versions to queued operations.
- Replay eligible operations after connectivity is restored.
- Detect synchronization conflicts and route them for human review.
- Publish compatibility and deprecation information.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Contract | Open API specifications, schemas, code sets, and version rules |
| Adapter | SDKs, reader adapters, facility adapters, and reference implementations |
| Synchronization | Protected minimal cache, durable queue, replay, acknowledgement, and conflict handling |
| Validation | Conformance, compatibility, and contract-test tooling |

M3 is primarily a build-time/platform dependency. Its synchronization agent is
also a runtime component embedded in M4, M5, and M6 applications. It is not a
central clinical or billing database.

### M4 — Registration & Appointments

**Functions**

- Register a patient at a facility through M1.
- Issue, replace, suspend, or revoke a patient card through M1.
- Search availability by facility, provider, resource, and time.
- Reserve and confirm an appointment using an authoritative facility
  transaction.
- Cancel or reschedule an appointment.
- Check in a patient using a current M1 credential decision.
- Create a visit/check-in context for M5.
- Publish minimal appointment and check-in events for M6.
- Queue eligible local work through the M3 synchronization agent when
  disconnected; do not claim a globally confirmed appointment until the
  authoritative transaction succeeds.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Interface | Patient/front-desk UI and appointment/check-in API |
| Application | Registration, booking, cancellation, rescheduling, and check-in workflows |
| Domain | Slot uniqueness, appointment lifecycle, facility resources, and visit context |
| Infrastructure | Facility appointment store, M1/M5/M6 clients, reader adapter, and M3 sync agent |

### M5 — Medical Services

**Functions**

- Authenticate clinical staff and establish role context through M1.
- Consume a checked-in visit from M4 and create a clinical encounter.
- Record observations, diagnoses, notes, orders, results, and prescriptions.
- Manage laboratory order, sample, and result attribution workflows.
- Create signed prescriptions and record pharmacy responses.
- Request a patient's permitted external history from M2.
- Publish an allowed clinical summary or record pointer to M2.
- Emit minimal `VisitCompleted` and `FollowUpNeeded` events to M6.
- Continue eligible local clinical work during an outage and synchronize it
  through M3 without silently bypassing trust decisions.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Interface | Clinician UI and clinical service API |
| Application | Encounter, history retrieval, order, result, prescription, and summary workflows |
| Domain | Clinical record lifecycle, provenance, signing, and care-workflow rules |
| Infrastructure | Facility clinical store, M1/M2/M4/M6 clients, pharmacy adapter, and M3 sync agent |

### M6 — Patient Services & Hospital Management

**Functions**

- Authenticate patients, facility administrators, and staff through M1.
- Present a patient dashboard using consented access through M2.
- Consume appointment and clinical events from M4 and M5.
- Schedule follow-up actions and reminders.
- Send minimal reminders through notification providers.
- Create itemized bills and claims from facility-owned billing state.
- Submit claims to an insurer and process responses.
- Manage facility roles, configuration, and operational policies.
- Maintain an idempotent notification outbox and retry delivery safely.

**Layers**

| Layer | Responsibility |
| --- | --- |
| Interface | Patient dashboard, administration UI, billing API, and notification API |
| Application | Dashboard, event intake, follow-up, billing, claims, and administration workflows |
| Domain | Reminder rules, bill/claim lifecycle, role policy, and operational configuration |
| Infrastructure | Facility billing/configuration stores, event intake, outbox, notification adapters, M1/M2 clients, and M3 sync agent |

## 2. External functional modules

External modules are systems outside the six internal solution modules. Only
the functions required by this solution are included.

| External module | Description | Required functions |
| --- | --- | --- |
| Patient | Person receiving care and controlling consent. | Register, present credential, book/cancel appointments, grant/withdraw consent, view permitted records, receive reminders, pay or review bills. |
| Clinical staff | Doctors, nurses, lab staff, and pharmacists operating clinical workflows. | Authenticate, check worklists, record care, order tests, sign prescriptions, and review permitted history. |
| Facility administrator | Manages a participating facility. | Configure roles/resources, manage facility metadata, review duplicate candidates, and administer billing policies. |
| Facility identity/card reader | Hardware or manual entry mechanism at a facility. | Read QR/RFID/NFC/manual credentials and return a normalized credential to M1. |
| Originating facility API | API owned by the facility that stores the requested record. | Authenticate the M2 request, authorize local disclosure, return the permitted record, and provide provenance. |
| Pharmacy | External dispensing system. | Receive a signed prescription, return dispense/reject status, and provide dispensing provenance. |
| Insurer | External payer/claims system. | Receive an itemized claim, return acceptance/adjudication status, and provide rejection reasons. |
| Notification provider | SMS, email, or push delivery service. | Accept a minimal notification, return delivery status, and expose retry-safe request handling. |

## 3. Module invocation and dependencies

### Invocation rules

1. M4 invokes M1 for registration, UPID resolution, card lifecycle actions,
   and current credential validation.
2. M5 invokes M1 for clinical-staff authentication and M2 for external
   history and summary publication.
3. M6 invokes M1 for dashboard/administrator authentication and M2 for
   consented patient record access.
4. M2 invokes M1 to validate the caller and resolve the UPID. M2 invokes an
   originating facility API only after its own policy and consent decision
   succeeds and the audit record has been written.
5. M4 invokes M5 after check-in and invokes M6 with minimal scheduling events.
6. M5 invokes M6 with minimal care events such as `VisitCompleted` and
   `FollowUpNeeded`.
7. M3 supplies contracts and adapters to M1, M2, M4, M5, and M6. Its
   embedded sync agent invokes the owning module when queued work is replayed.
8. M5 invokes the pharmacy; M6 invokes the insurer and notification provider.
   These external calls must use idempotency keys and explicit failure
   statuses.

### Dependency table

| Module 1 (invoker) | Module 2 (receiver) | High-level input | High-level output |
| --- | --- | --- | --- |
| M4 | M1 | Registration data, credential command, presented card | UPID, identity decision, credential status |
| M5 | M1 | Staff credentials and role context | Authentication and authorization claims |
| M6 | M1 | Patient/admin credentials | Authenticated identity and claims |
| M2 | M1 | Caller identity and patient reference | Validated caller and UPID resolution |
| M5 | M2 | UPID, purpose, caller, requested record category | Consented history or explicit deny/unavailable result |
| M6 | M2 | Authenticated patient, purpose, requested dashboard data | Authorized patient-facing records with provenance |
| M5 | M2 | Allowed summary and originating-facility pointer | Recorded locator/update acknowledgement |
| M2 | Originating facility API | Authorized record request and provenance context | Permitted source record or disclosure refusal |
| M4 | M5 | Confirmed check-in and visit context | Clinical worklist/encounter context |
| M4 | M6 | Minimal appointment/check-in event | Reminder and billing context |
| M5 | M6 | Minimal visit-completed/follow-up event | Follow-up task and billing trigger |
| M3 | M1, M2, M4, M5, M6 | Contracts, SDK calls, queued operation with idempotency key | Conformant response, acknowledgement, or conflict |
| M5 | Pharmacy | Signed prescription | Dispense, reject, or pending status |
| M6 | Insurer | Itemized claim | Claim acceptance, adjudication, or rejection |
| M6 | Notification provider | Minimal reminder and delivery key | Delivery status |

Solid runtime dependencies are request/response or event interactions. M3's
contract and SDK dependencies are normally build-time; only its embedded
offline synchronization agent is a runtime dependency.

## 4. Layered architecture

Each runtime business module follows the same conceptual layers:

1. **Interface layer** — UI, public API, event consumer, or external adapter.
2. **Application layer** — coordinates a use case and invokes ports; it does
   not own persistence details.
3. **Domain layer** — owns business rules, state transitions, and invariants.
4. **Infrastructure layer** — databases, queues, tokens, network clients,
   hardware, and third-party integrations.

Dependencies point inward from interface and infrastructure adapters through
application ports to the domain. A module does not read another module's
database directly. Cross-module communication uses versioned contracts,
explicit outputs, and the dependency table above.

## 5. Mapping to architecture design principles

The image supplied with the requirement describes service-oriented principles.
They are applied where they improve this solution; they are design guidance,
not a requirement to force every principle onto every interaction.

| Principle | Application in this solution |
| --- | --- |
| Service loose coupling | M4, M5, and M6 call M1/M2 through stable APIs; no module shares another module's database. Events carry minimal data and can be consumed independently. |
| Service abstraction | M1 hides card technology and identity-store details. M2 hides consent evaluation, record location, and facility routing behind an exchange API. |
| Service composability | M4 composes identity, booking, and check-in services; M5 composes authentication, consented history, clinical records, and pharmacy; M6 composes events, billing, and notifications. |
| Standardized service contract | M3 publishes versioned APIs, schemas, error formats, code sets, idempotency rules, and provenance requirements. |
| Service reusability | M1 authentication is reused by M2, M4, M5, and M6. M2's authorization gateway is reused by clinical history and patient dashboard flows. |
| Service autonomy | Each facility controls its appointments, clinical records, billing, and local operational state. M2 stores pointers and policy decisions rather than copying all clinical data. |
| Service statelessness | API handlers remain stateless where practical; identity and consent state are persisted in authoritative stores. Long-running work is represented by durable events, queues, or workflow records rather than process memory. |
| Service discoverability | M3 publishes service descriptions, schemas, versions, and conformance information. M2 maintains the facility/provider directory needed to discover the owner of a record. |

## 6. Important boundary and failure rules

- A card is not a clinical record. Card credentials are revocable and every
  use requires a current M1 status decision.
- M2 must not disclose a record without a successful consent/policy decision
  and an audit entry.
- M2 stores record pointers and provenance, not a second authoritative copy of
  detailed facility records.
- A queued offline operation must have an idempotency key and an explicit
  acknowledgement or conflict result.
- Offline mode may support local work, but it cannot silently approve a new
  cross-facility disclosure or an unavailable card-revocation check.
- External failures are returned as explicit unavailable, rejected, pending,
  or conflict outcomes; they are not converted into successful responses.

## 7. Detailed explanation of individual modules

This section explains the role of each module in the complete solution and
how its responsibilities are separated from the other modules.

### 7.1 M1 — Patient Identity & Login

M1 is the common identity boundary for patients, clinical staff, facility
administrators, and trusted services. Its most important responsibility is to
provide one stable, opaque UPID without exposing clinical information in the
identity itself. A patient may interact with multiple facilities, but each
facility uses the same UPID when a valid identity match exists.

The module separates identity from the physical card. A card, QR code, RFID
token, NFC token, or manually entered credential is only a way to present a
credential. Reader adapters normalize these different technologies before the
identity service evaluates them. M1 owns credential issuance, replacement,
status, and revocation, so a lost or compromised card can be invalidated
without changing the patient's clinical records.

During registration, M1 searches for a possible existing identity. A
high-confidence match can be linked to the existing UPID, while an uncertain
match becomes a duplicate-review case for an authorized human. M1 never
automatically merges identities based only on an uncertain match. Other
modules receive an identity or authorization decision, not direct access to
the identity database.

### 7.2 M2 — Consent & Record Sharing

M2 is the trust boundary for sharing information between facilities. It
ensures that a request is made by an authenticated caller, concerns the
correct patient, has a valid purpose, and is permitted by the patient's
consent and applicable facility policy. M2 therefore protects the system from
modules directly bypassing consent to read another facility's records.

For a history request, M2 first validates the caller and resolves the UPID
through M1. It then evaluates consent, finds the facility that owns the
requested category of record, writes the authorization and audit decision,
and forwards the request to the originating facility. The originating
facility remains authoritative for the detailed record. M2 returns only the
permitted result together with provenance and does not create an uncontrolled
central copy of the clinical data.

M2 also supports the reverse direction: M5 can publish an allowed summary and
record pointer so that the record can be discovered later. A pointer contains
location and classification information rather than the detailed clinical
content. Every denial, unavailable source, and successful disclosure is
returned explicitly and is recorded for accountability.

### 7.3 M3 — Platform & Integration

M3 makes the solution implementable by multiple facilities and vendors. It
defines the common language between modules: API contracts, message schemas,
code sets, versioning rules, error formats, provenance fields, and
idempotency requirements. This prevents each facility from creating a
different interpretation of identity, consent, appointments, or clinical
events.

The SDKs and reference implementations help application teams call M1 and M2
consistently and help facilities integrate M4, M5, and M6. Reader adapters
allow a facility to change card hardware without changing the identity
workflow. Conformance tests verify that an implementation follows the
published contract before it joins the ecosystem.

M3 also provides the embedded synchronization agent for temporary
connectivity loss. The agent stores only a protected minimal cache and a
durable queue of eligible local operations. When the connection returns, it
replays operations using idempotency keys and schema versions. Accepted,
rejected, and conflicting operations are distinguished; clinical conflicts
and identity conflicts are sent for review instead of being resolved by
blind last-write-wins behavior. M3 does not make offline data-sharing or
card-revocation decisions on behalf of M1 or M2.

### 7.4 M4 — Registration & Appointments

M4 is the facility-facing entry point for receiving a patient and arranging a
visit. It coordinates front-desk activities such as registration, card
operations, appointment scheduling, cancellation, rescheduling, and
check-in. It uses M1 for identity and credential decisions but does not own
the identity or card-status data.

The appointment workflow is transactional because a slot must not be
confirmed for two patients at the same time. M4 checks availability, reserves
the facility resource, and commits the appointment using the facility's
authoritative appointment store. It emits a minimal event after the
transaction succeeds. The event may give M6 notification and billing context
and may give M5 a visit context, but it does not contain unnecessary clinical
information.

At check-in, M4 validates the current credential through M1 and creates the
visit context consumed by M5. If the facility is temporarily disconnected,
M4 may queue a local request through M3, but it must clearly distinguish a
queued request from a confirmed appointment. This prevents an offline user
interface from promising a booking that the authoritative service has not
accepted.

### 7.5 M5 — Medical Services

M5 owns the clinical workflow at each facility. It creates and manages
encounters, worklists, observations, diagnoses, clinical notes, laboratory
orders, results, prescriptions, and related provenance. Detailed records
remain in the facility clinical store so that the facility can maintain
clinical authority, local retention rules, and operational availability.

Before a clinician performs a protected action, M5 authenticates the
clinician and establishes the required role through M1. When the clinician
needs history from another facility, M5 sends M2 the patient, purpose,
requested category, and caller context. M5 receives only the data that M2
authorizes and should display the source and provenance to the clinician.

After care is completed, M5 can publish an allowed summary or pointer through
M2. It also emits minimal events to M6, such as a completed visit or a
follow-up requirement. These events support billing and engagement without
sharing the full clinical note. Local clinical work may continue during a
network outage, but external history and new cross-facility disclosure remain
explicitly unavailable until a live trust decision is possible.

### 7.6 M6 — Patient Services & Hospital Management

M6 connects the clinical system to patient communication and facility
operations. It contains the patient dashboard, follow-up rules, reminder
processing, billing, claims, facility roles, and operational configuration.
It does not bypass M2 when a patient requests protected records; the dashboard
uses M1 for authentication and M2 for consented access.

M6 consumes the minimal appointment and clinical events produced by M4 and
M5. Follow-up rules turn those events into durable tasks, and the notification
outbox ensures that a temporary provider failure does not lose a reminder.
Notification messages should contain only the minimum information needed by
the patient and should be safe to retry using a delivery key.

Billing and claims remain facility-owned transactional workflows. M6 creates
itemized bills from local operational data, submits claims to the insurer, and
records acceptance, adjudication, or rejection. Facility administrators use
M6 to manage roles and configuration, while M1 remains responsible for
authentication and M2 remains responsible for consented record access. This
separation prevents patient engagement concerns, billing state, and clinical
records from becoming one tightly coupled data store.
