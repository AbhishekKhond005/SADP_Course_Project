# Patient Card and Connected Care

Patient Card and Connected Care is a proposed architecture for participating healthcare facilities. A patient uses one unified patient ID (UPID) and card across facilities. Shared services coordinate identity, consent, and record sharing. Each facility keeps its own detailed clinical records, appointments, and billing data.

## Modules and responsibilities

### M1 — Identity & Access

- Match registrations to an existing UPID or create one for a new patient. Send uncertain matches for human review before merging identities.
- Authenticate patients, staff, and administrators and provide the identity and role information other modules need.
- Issue, link, replace, suspend, and revoke card credentials. The card identifies the patient through a credential, while clinical records stay at their source.
- Resolve QR, RFID, NFC, or manually entered credentials and check current card status whenever a card is presented.

### M2 — Trust & Data Exchange

- Record patient consent, including when it is granted and withdrawn.
- Maintain the facility and provider directory and record pointers that identify where records can be requested.
- Verify the caller through M1 and check the patient's current consent, the purpose, and the requested record category before forwarding a request to the owning facility.
- Return only the records the source facility is authorized to disclose, with information about where they came from. Record access decisions and disclosures in an append-only audit trail.

### M3 — Open Ecosystem

- Publish versioned APIs, data formats, and card-reader specifications so facilities and vendors can integrate independently.
- Provide development kits, adapters, a reference implementation, and compatibility tests.
- Supply the sync agent embedded in M4, M5, and M6. It maintains a protected minimal cache and queues eligible operations during outages.
- Replay queued operations with unique IDs to prevent duplicate work and flag conflicts for review.

### M4 — Access to Care

- Provide patient registration and front-desk workflows. Call M1 for identity matching and card issuance, replacement, or revocation.
- Show appointment availability and handle booking, cancellation, and rescheduling. Use the facility's appointment store to confirm a slot without double-booking it.
- Check in patients after M1 verifies the presented credential. Pass the checked-in visit to M5 and send scheduling events to M6.

### M5 — Clinical Care

- Use M1 to authenticate clinical staff and open an encounter from a visit checked in through M4.
- Record consultations, observations, diagnoses, laboratory orders and results, and signed prescriptions in the treating facility's clinical store. Track who recorded each action and when.
- Request permitted external history through M2 and publish approved summaries or record pointers.
- Send signed prescriptions to the pharmacy, record dispensing outcomes, and send visit completion and follow-up events to M6.

### M6 — Engagement & Operations

- Provide a patient dashboard through M1 sign-in and M2 access checks for protected records.
- Use appointment and care events from M4 and M5 to schedule follow-ups and send reminders with minimal detail. Keep pending messages in a durable queue so delivery can be retried safely.
- Manage facility-owned itemized bills and insurance claims, submit claims to insurers, and track their responses.
- Let facility administrators configure staff roles, facility settings, and operational policies.

## User stories and system behavior

### 1. First-time patient registers and attends a visit

**User story.** As a first-time patient, I want to register and receive a card so I can book a visit and use the same identity for future care.

1. The patient gives their registration details to the front desk. M4 asks M1 to look for an existing identity. If there is no match, M1 creates a UPID. If the match is uncertain, it flags the case for human review.
2. Once the identity is established, M4 requests a card credential from M1 and helps the patient choose an available appointment. M4 confirms the booking when the facility successfully reserves the slot. M6 can then send a reminder.
3. At arrival, the patient presents the card. M4 asks M1 to check its current status, then sends the checked-in visit to the clinician's worklist in M5.
4. The clinician signs in and records the encounter at the treating facility. After the visit, M5 sends M6 the information needed for billing and any follow-up. The patient can sign in to M6 to view their own permitted information.

### 2. Returning patient books or changes a visit

**User story.** As a returning patient, I want to book, reschedule, or cancel a visit using my existing identity so I can manage my care without registering again.

1. The patient uses M4 to book or change an appointment. The facility updates its appointment store, and M6 uses the scheduling event to handle reminders.
2. At check-in, M1 resolves the card to the existing UPID and verifies its current status. M4 sends the visit to M5, where the clinician can work with the patient's local history under facility rules.
3. If the card is lost, the front desk verifies the patient's identity through M4 and asks M1 to revoke the old credential and issue a replacement under the same UPID. The next live check at any participating facility will reject the revoked card.

### 3. Existing patient visits another facility

**User story.** As a patient visiting a new facility, I want staff to use my existing identity and see only the outside history I have permitted them to access.

1. The new facility resolves the patient's card through M4 and M1. The patient uses the same UPID, and the new facility manages its own appointment and encounter.
2. If the clinician needs prior results, M5 sends M2 the patient ID, caller identity, purpose, and record category. The patient can grant or withdraw permission through the consent workflow.
3. M2 verifies the caller, checks current consent, finds the facility that holds the record, and audits the decision. The source facility applies its own disclosure rules and returns only the permitted data. M2 records the disclosure and returns the result with its source.
4. The clinician sees whether the request succeeded, was denied, or could not be completed. If consent is withdrawn, later requests use the updated decision. The new facility keeps its own encounter locally and can publish an approved pointer or summary for future discovery.

### 4. Care team orders tests and prescribes treatment

**User story.** As a member of the care team, I want orders, results, and prescriptions linked to the correct encounter so I can act on them and trace their source.

1. A clinician opens the checked-in visit in M5, records findings, and places a lab order. Lab staff link the sample and result to that order and encounter.
2. The clinician signs a prescription in M5. The pharmacy returns a dispensing status, which M5 records with the encounter.
3. When the visit is complete or further care is needed, M5 sends M6 a minimal event. M6 queues any required reminder, and the patient can sign in to view permitted follow-up information.

### 5. Facility administrator manages operations

**User story.** As a facility administrator, I want to manage access and monitor billing so staff can do their work and the facility can track claims.

1. The administrator signs in through M1 and uses M6 to assign facility roles and configure local policies. M4 manages the appointment slots and resources that patients can book.
2. M6 uses scheduling and clinical events to start the relevant billing work. It produces an itemized bill from local billing data and can submit a claim to the insurer.
3. The insurer returns the claim's status or a rejection reason. M6 records the result locally for the facility to review.

### 6. Facility staff work during a connectivity outage

**User story.** As a staff member, I want to continue eligible local work and know which actions are still pending so I can set accurate expectations with patients.

1. Staff can continue eligible local clinical work. The sync agent supplied by M3 queues eligible operations, including booking requests that the facility cannot yet confirm.
2. A queued booking stays pending until the facility confirms the slot. Card validation and protected record access require live checks. When those checks are unavailable, the application reports that outcome and withholds the protected action or data.
3. After reconnection, M3 replays queued operations using their original IDs. The owning module confirms, rejects, or flags each operation as a conflict. Reusing the IDs prevents duplicate bookings, orders, or charges, and staff review conflicts that need a decision.

### 7. Facility IT team connects a new hospital

**User story.** As a facility integration developer, I want a published interface and test tools so the hospital can join the network using its existing systems.

1. The team uses M3's specifications, adapters, and reference implementation to connect its applications and card readers. It runs compatibility tests against the supported contract version.
2. The facility is registered in M2's directory with the record categories it supports. Its own API handles authorized requests for its records, and its applications keep their local data and use M3's sync agent where applicable.

## Project references

- [Project overview](project-overview.md)
- [Functional module specification](functional-modules.md)
- [Module integration and failure handling](02%20Internal%20Integration/Internal%20Integration.md)
- [Architecture drivers](CSE752-2026-17-ArchDrivers.pdf)
