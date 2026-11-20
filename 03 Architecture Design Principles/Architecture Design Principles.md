# Architecture Design Principles

## Seven principles in the SOA lecture sequence

### 1 Standardized Service Contract
Every service follows the same rules for its published interface. M3 publishes versioned request formats, error responses, and compatibility rules. For example, M5 and M6 use the same record-request pattern when calling M2.

### 2 Loose Coupling
Modules know as little as possible about each other's implementation. M4 uses the M1 interface for card validation. M6 receives small booking and visit events and can process them independently. Each module accesses its own database.

### 3 Service Abstraction
A service exposes what callers need to use it and keeps the mechanics inside. M1 hides the differences between card readers. M2 presents a single record-request interface and handles consent checks, record location, and routing internally.

### 4 Service Reusability
A capability can support several workflows. M1 provides the same sign-in and identity functions to multiple modules. M2 handles both the clinicians' history requests and the patient dashboard's protected record requests.

### 5 Service Autonomy
Each service controls its own rules, data, and day-to-day work. The facility owns appointment availability in M4, clinical records in M5, and billing in M6. M2 controls its own consent and routing decisions.

### 6 Service Statelessness
A service can process a request without remembering a previous request in its running program. Each request carries the context it needs. Durable stores keep identity, consent, bookings, and queued work so the application can restart without losing that information.

### 7 Service Discoverability
Teams can find a service and learn how to use it. M3 publishes service descriptions, supported versions, and usage examples. M2 uses its facility and provider directory to find where a record request should go.

## Service Composability

Services can be combined to complete a larger task. One visit brings together identity and card checks in M1, check-in in M4, consented history in M2, clinical work in M5, and follow-up in M6. Each part contributes through its published interface.

Source: CSE 752 Service-Oriented Architecture lecture (18 SOA), slides 23–32. Slide 23 uses the word “SEVEN” and its diagram names eight concepts. Slides 24–31 cover the seven listed above, with Service Abstraction appearing twice. Slide 32 covers Service Composability.
