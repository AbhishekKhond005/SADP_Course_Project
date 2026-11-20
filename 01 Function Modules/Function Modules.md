# Function Modules

An architecture driver is a need or constraint that shapes the design. Each module below lists its drivers, the design response, and the other parts it needs.

## M1 Identity & Access

**Features and users.** F1 and F2 cover the universal patient ID, login, and card credentials. Patients, front-desk staff, clinicians, and the registry authority use this shared service. It owns identity and card-status records and issues short-lived login tokens.

**Architecture drivers.** Every patient needs one persistent unified patient ID (UPID), with human review before any duplicate merge. The ID carries no personal information. A common reader interface supports QR, RFID, NFC, and manual input. When a card is revoked, the next check at any facility must see that change. Current status checks support this rule, and identity matching and login must handle demand from many facilities.

**Dependencies.** M1 uses M3's contracts and reader specifications, plus the identity store, card-status store, and card-reader adapters. M2, M4, M5, and M6 call M1 for identity, login, or card decisions.

## M2 Trust & Data Exchange

**Features and users.** F3 to F5 cover consent, audit, record discovery, secure exchange, and shared registries. Patients grant or withdraw consent, and the registry authority maintains the shared service. M2 owns consent policies, the provider and facility directory, and a record locator that stores pointers. The audit trail is independently controlled.

**Architecture drivers.** Apply current patient consent consistently to each protected request. Keep detailed clinical data at the originating facility. Use an append-only audit trail that makes tampering detectable. Exchange data through open, FHIR-aligned healthcare formats and published codes. Use rate limits to protect the gateway when many facilities make requests at once.

**Dependencies.** M2 uses M1 to validate callers and resolve UPIDs, M3 for exchange contracts, and the originating facility's API for permitted source data. M5 and M6 call M2 for protected record access. M5 also registers approved locator information with M2.

## M3 Open Ecosystem

**Features and users.** F6 covers open APIs, documentation, developer tools, reader specifications, a working reference example, and offline sync. Facility IT teams and vendors use these resources to connect their applications.

**Architecture drivers.** Make the specifications and tools openly available. Keep API versions compatible so facilities can upgrade on different schedules. Help facilities continue eligible work during poor connectivity using a protected, minimal local cache and a durable queue. Replay requests when the network returns, using a unique operation ID to prevent duplicate work, and flag conflicts for review.

**Dependencies.** M3 publishes reusable contracts without a live dependency on another business module. M1, M2, M4, M5, and M6 use its specifications or tools. The sync agent runs inside M4, M5, and M6 and sends eligible queued work back to the service that owns it.

## M4 Access to Care

**Features and users.** F7 covers registration, cards, appointments, cancellation, rescheduling, and check-in. Patients and front-desk staff use this facility application. M4 owns appointment slots, bookings, and check-in status.

**Architecture drivers.** Verify identity against the UPID before issuing or replacing a card. Reserve and confirm each slot and resource in one facility database transaction so it cannot be double-booked. Keep peak-hour check-in fast with a focused identity check and visit update. An offline booking request remains pending until the booking service confirms it.

**Dependencies.** M4 uses M1 for identity and card status and M3 for its toolkit and embedded sync. It hands checked-in visits to M5 and publishes minimal scheduling events for M6.

## M5 Clinical Care

**Features and users.** F8 covers consultations, vitals, diagnoses, lab orders and results, signed prescriptions, and dispensing. Clinicians, lab staff, and pharmacists use this facility application. The treating facility owns the detailed clinical record.

**Architecture drivers.** Give clinicians permitted history during a consultation. Keep detailed records at their home facility and limit routine outward publication to an approved summary or locator update. Record who issued each order or prescription and when. Support eligible local care work when the link to shared services fails. Show external history as unavailable when the live consent check cannot run.

**Dependencies.** M5 uses M1 for staff sign-in, M2 for record requests and publication, M3 for standards and offline sync, and M4 for checked-in visit details. It sends minimal clinical events for M6 and can use an external pharmacy interface for dispensing.

## M6 Engagement & Operations

**Features and users.** F9 and F10 cover patient dashboards, rule-based follow-ups, reminders, itemised bills, insurance claims, staff roles, and facility settings. Patients, administrators, clinicians, and insurers use its outputs.

**Architecture drivers.** A dashboard must show each patient only their own permitted information. SMS, email, and push reminders carry only essential details. Each facility keeps its own billing data and sets its own staff permissions. A durable notification queue and retry identifiers help handle demand spikes and avoid duplicate reminders.

**Dependencies.** M6 uses M1 for sign-in, M2 for permitted dashboard records, M3 for contracts and eligible offline work, and events from M4 and M5 for reminders and billing. It connects to insurers and notification providers and owns local bills, claims, role settings, and delivery status.
