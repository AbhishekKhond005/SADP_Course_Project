# Patient card and connected care: an architecture study

This README is my proposed architecture for the system described in [the architecture drivers PDF](CSE752-2026-17-ArchDrivers.pdf). It is a learning exercise, not an implementation. I have used patterns we study in software architecture to ask a practical question: **where should each responsibility live, and how can the modules cooperate without losing control of patient data?**

The PDF identifies ten features in six modules. I keep its module boundaries and use the same IDs throughout. Each diagram's `M1`–`M6` boxes are **connection points** to the diagrams with those headings; arrow labels name the information or request crossing the boundary. Components inside a module are proposed design choices, not requirements stated verbatim in the PDF.

## The system at a glance

I would use a **federated architecture**: facilities keep their detailed clinical and billing records, while shared services coordinate identity, consent, discovery and audit. Applications call shared services for decisions, then retrieve allowed data from the facility that owns it. Events help with follow-up work, but a consent decision cannot be postponed to an event consumer.

```mermaid
flowchart LR
    patient[Patient] --> M4[M4 Access to Care]
    staff[Clinical staff] --> M5[M5 Clinical Care]
    patient --> M6[M6 Engagement & Operations]
    admin[Facility admin / insurer] --> M6

    M4 -- "identity and card commands" --> M1[M1 Identity & Access]
    M5 -- "authentication" --> M1
    M6 -- "authentication" --> M1
    M2[M2 Trust & Data Exchange] -- "resolve UPID / validate caller" --> M1
    M5 -- "consented history request; summary publication" --> M2
    M6 -- "consented dashboard request" --> M2
    M2 -- "authorized record request" --> records[(Facility clinical records)]
    M5 --> records
    M6 --> billing[(Facility billing and configuration)]
    M3[M3 Open Ecosystem] -. "API contracts, SDKs, adapters; embedded sync agent" .-> M1
    M3 -. "API contracts and schemas" .-> M2
    M3 -. "SDKs and embedded sync agent" .-> M4
    M3 -. "SDKs and embedded sync agent" .-> M5
    M3 -. "SDKs and embedded sync agent" .-> M6
```

The solid arrows are runtime interactions. The dotted arrows are mostly build-time dependencies; the exception is M3's sync agent, which runs inside facility applications. The record store is shown beside M5 because M5 owns clinical workflows, but each facility operates its own store. M2 keeps record **pointers**, not copies of those records.

| Module | PDF features | Main architectural idea | Owns or governs |
| --- | --- | --- | --- |
| M1 Identity & Access | F1, F2 | Layered service with reader adapters | UPIDs, identity matching, credentials, card status |
| M2 Trust & Data Exchange | F3, F4, F5 | Policy enforcement plus federated gateway | Consent, audit, record pointers, registries, exchange rules |
| M3 Open Ecosystem | F6 | Ports and adapters, versioned contracts, offline queue | Specifications, SDKs, reference stack, sync agent |
| M4 Access to Care | F7 | Transactional booking workflow | Facility appointments, registration and check-in |
| M5 Clinical Care | F8 | Local clinical workflow with federated reads | Facility clinical records and care actions |
| M6 Engagement & Operations | F9, F10 | Event-driven follow-up plus local billing | Patient views, reminders, facility billing and administration |

## M1 — Identity & Access

I would separate identity rules from card hardware using **ports and adapters**. A reader translates QR, RFID, NFC or manual input into the same credential lookup. The UPID is an opaque identifier; the card carries a revocable credential, not a clinical record. Duplicate matching produces a *review candidate*, not an automatic merge.

```mermaid
flowchart LR
    M4[M4: register / issue / replace / check in] --> api[Identity API]
    M5[M5: staff or patient authentication] --> api
    M6[M6: patient or admin authentication] --> api
    M2[M2: resolve UPID / validate caller] --> api
    readers[QR / RFID / NFC / manual] --> adapter[Vendor-neutral reader adapter]
    adapter --> api
    api --> auth[Authenticate; issue short-lived token]
    api --> matching[Find identity / create opaque UPID]
    matching --> review{Possible duplicate?}
    review -- Yes --> human[Registry human review before merge]
    review -- No --> identity[(Identity and UPID store)]
    human --> identity
    api --> cards[Bind / replace / revoke credential]
    cards --> status[(Authoritative card-status store)]
    auth --> status
    auth --> identity
    M3[M3: reader contract and SDK] -.-> adapter
```

**Connection contract:** M4 asks M1 to identify a patient and change card status. M5, M6 and M2 use M1 to authenticate or resolve identity. Every card use must check current revocation status; a long-lived offline acceptance of a lost card would contradict the PDF's immediate-revocation driver. If the shared service is unreachable, the application must state that this check is unavailable rather than silently treating a cached card as valid.

## M2 — Trust & Data Exchange

This is a **federated gateway** with a **policy decision point**. The gateway checks the caller and patient consent for *each* request, locates the owning facility, then forwards an authorized, standards-based request. A separate audit service records the decision and access in an append-only, tamper-evident log. A successful audit write should be part of the access path so an unlogged disclosure is not treated as success.

```mermaid
flowchart LR
    M5[M5: permitted history / publish summary] --> gateway[Versioned exchange gateway]
    M6[M6: patient dashboard request] --> gateway
    gateway --> M1[M1: resolve UPID and validate caller]
    gateway --> policy[Consent policy decision]
    patient[Patient: grant / withdraw consent] --> policy
    policy --> consent[(Consent policies)]
    gateway --> directory[(Facility/provider directory; code sets)]
    gateway --> locator[(Record locator: UPID + category + facility pointer)]
    policy -- "allow / deny" --> gateway
    gateway -- "record decision and access" --> audit[(Independent append-only audit)]
    gateway -- "authorized request, open schema" --> facility[Originating facility API]
    facility --> M5store[(M5: facility clinical records)]
    M3[M3: versioned APIs and schemas] -.-> gateway
```

**Connection contract:** M5 and M6 submit purpose, patient UPID, caller identity and requested record category to M2. M2 returns only data that the origin facility is authorized to disclose. M1 supplies identity, while the directory and certificate-based facility trust identify the destination. The locator contains no clinical content. Rate limiting at the gateway protects many facilities from bursts; published code sets and FHIR-aligned schemas make the exchange interpretable.

## M3 — Open Ecosystem

Here I would use **ports and adapters** to make standards more than documentation. A versioned API contract defines requests and data shapes; SDKs, a reference implementation and conformance tests help independent teams implement them. The offline sync agent is embedded in applications at the facility, where it stores a protected minimal cache and a durable operation queue.

```mermaid
flowchart LR
    specs[Open, versioned API and reader specifications] --> sdk[SDKs / reference implementation]
    specs --> tests[Conformance and compatibility tests]
    sdk -. "build-time" .-> M1[M1: card-reader adapter]
    sdk -. "build-time" .-> M2[M2: gateway and exchange schemas]
    sdk -. "build-time" .-> apps[M4 / M5 / M6 applications]
    tests -. "verify contracts" .-> apps
    apps --> cache[Embedded sync agent: protected minimal cache]
    cache --> queue[(Durable local operation queue)]
    queue -- "network restored: replay with IDs and versions" --> remote[M4 / M5 / M6 owning service]
    remote -- "acknowledge / reject / conflict" --> reconcile{Reconcile}
    reconcile -- Conflict --> review[Flag for human review]
    reconcile -- Accepted --> cache
```

**Connection contract:** M3 defines what M1, M2, M4, M5 and M6 must agree on; it is not a central runtime database. The embedded agent synchronizes only operations permitted for that application. Each queued operation needs an idempotency key so retrying cannot create duplicate appointments, orders or charges. Conflicts in clinical facts or identity merges need review rather than last-write-wins. Offline mode can support local work, but new cross-facility disclosures and card validity checks still require live trust decisions.

## M4 — Access to Care

For booking I would use a **transactional workflow** at the facility: reserve a slot with a uniqueness rule, then confirm the appointment. This is preferable to an asynchronous event for preventing two people from taking the same slot. Registration and card replacement call M1, while check-in connects the front desk to M5's care workflow.

```mermaid
flowchart LR
    patient[Patient / front desk] --> app[Registration and booking UI]
    app --> M1[M1: resolve UPID; issue / replace / revoke card]
    app --> slots[Slot and resource service]
    slots --> db[(Facility appointments: unique slot + resource)]
    slots --> booked[Confirmed appointment]
    app --> scan[Check-in via card-reader adapter]
    M3[M3: SDK and embedded sync agent] -.-> app
    M3 -.-> scan
    scan -- "validate credential" --> M1
    scan --> checkin[Visit check-in]
    booked --> M6[M6: appointment notification / billing context]
    checkin --> M5[M5: visit worklist and clinical encounter]
```

**Connection contract:** M4 receives identity and card decisions from M1; it sends a confirmed booking or check-in to M5 and a minimal scheduling event to M6. The facility appointment store owns slot availability. A disconnected front desk may queue a *request*, but it cannot promise a globally confirmed slot until the authoritative booking transaction succeeds.

## M5 — Clinical Care

I would keep the **clinical workflow and its data together at each facility**. Clinicians write vitals, notes, orders, results and prescriptions locally. For a patient's history elsewhere, M5 asks M2 to obtain consented records from the originating facility. This is a federated read, not a single shared clinical database.

```mermaid
flowchart LR
    M4[M4: checked-in visit] --> worklist[Clinical worklist]
    doctor[Doctor / nurse] --> worklist
    worklist --> M1[M1: authenticate staff and role]
    worklist --> encounter[Encounter: vitals, notes, diagnosis]
    worklist -- "request permitted history" --> M2[M2: consent + locator + gateway + audit]
    M2 -- "authorized history from source facility" --> worklist
    encounter --> local[(Facility clinical record)]
    encounter --> order[Lab order and sample workflow]
    order --> result[Attributed test result]
    result --> local
    encounter --> rx[Signed e-prescription]
    rx --> dispense[Pharmacy dispensing]
    dispense --> local
    local -- "publish allowed summary / locator update" --> M2
    encounter -- "visit completed / follow-up needed" --> M6[M6: follow-up and billing triggers]
    M3[M3: schemas and embedded sync agent] -.-> worklist
```

**Connection contract:** M4 starts the visit; M1 establishes the actor; M2 supplies external history only after consent and audit. M5 stores authoritative detailed records at the facility and sends M2 only an allowed summary and location pointer. M6 receives minimal events such as `VisitCompleted` or `FollowUpNeeded`, with identifiers rather than sensitive note text. Orders, prescriptions and results must record who issued them and when. During a shared-network outage, local work can continue, while external history is explicitly shown as unavailable and queued changes sync later.

## M6 — Engagement & Operations

This module combines two kinds of work. **Event-driven processing** fits follow-up reminders because sending a message may happen later and be retried. Billing and staff permissions need **facility-owned transactional state** because a charge, claim or role change needs an authoritative result. The patient dashboard reads through M2 so it cannot bypass consent and identity rules.

```mermaid
flowchart LR
    M4[M4: booking / check-in event] --> events[Facility event intake]
    M5[M5: visit completed / follow-up needed] --> events
    events --> rules[Rule-based follow-up]
    rules --> outbox[(Notification outbox)]
    outbox --> sender[SMS / email / push worker]
    sender --> patient[Patient: minimal reminder]
    events --> bill[Facility billing and claim workflow]
    bill --> ledger[(Facility itemised bills / claims)]
    ledger --> insurer[Insurer interface]
    patient --> dashboard[Patient dashboard]
    dashboard --> M1[M1: patient authentication]
    dashboard --> M2[M2: consented record access and audit]
    admin[Facility admin] --> roles[Roles and configuration]
    roles --> ledger
    roles --> M1
    M3[M3: SDK and embedded sync agent] -.-> dashboard
```

**Connection contract:** M4 and M5 provide only the event data needed to schedule a reminder or start a bill. The notification worker uses an outbox and idempotent delivery attempts to handle spikes without losing events; messages avoid sensitive clinical detail. The insurer interface exposes itemised claims under facility policy, while billing records remain at that facility. M1 authenticates dashboard and admin users, M2 mediates patient record views, and per-facility role configuration controls staff actions.

## Following one visit across the diagrams

The same interface labels above can be followed in time. This sequence makes the synchronous decisions and asynchronous work easier to distinguish.

```mermaid
sequenceDiagram
    actor Patient
    participant M4 as M4 Access to Care
    participant M1 as M1 Identity & Access
    participant M5 as M5 Clinical Care
    participant M2 as M2 Trust & Data Exchange
    participant Origin as Originating facility
    participant M6 as M6 Engagement & Operations
    Patient->>M4: Book and arrive with card
    M4->>M1: Resolve UPID and validate current card status
    M1-->>M4: Identity / credential decision
    M4->>M5: Checked-in visit
    M5->>M1: Authenticate clinician
    M5->>M2: Request external history for this purpose
    M2->>M1: Validate caller and UPID
    M2->>M2: Check consent, locate record, audit decision
    M2->>Origin: Request authorized record
    Origin-->>M2: Permitted history
    M2-->>M5: History and provenance
    M5->>M5: Record care in facility store
    M5->>M2: Publish allowed summary / pointer
    M5-->>M6: VisitCompleted / FollowUpNeeded event
    M6->>M6: Produce local bill and queue minimal reminder
    M6-->>Patient: Reminder or dashboard update
```

M3 does not appear as a runtime participant in this sequence because its main contribution is the contract each participant implements. When connectivity is poor, its embedded sync agent queues eligible local operations; it does not replace M1's revocation check or M2's live consent decision.
