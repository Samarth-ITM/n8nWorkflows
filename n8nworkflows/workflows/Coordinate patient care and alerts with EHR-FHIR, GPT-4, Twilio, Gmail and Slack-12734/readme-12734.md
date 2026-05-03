Coordinate patient care and alerts with EHR/FHIR, GPT-4, Twilio, Gmail and Slack

https://n8nworkflows.xyz/workflows/coordinate-patient-care-and-alerts-with-ehr-fhir--gpt-4--twilio--gmail-and-slack-12734


# Coordinate patient care and alerts with EHR/FHIR, GPT-4, Twilio, Gmail and Slack

## 1. Workflow Overview

Despite the provided title (“Coordinate patient care and alerts with EHR/FHIR, GPT-4, Twilio, Gmail and Slack”) and the sticky notes describing a healthcare/FHIR scenario, the actual executable nodes implement an **airline operations control (OCC) disruption monitoring and alerting system**. It ingests flight events on a schedule and via webhook, validates/normalizes them, deduplicates repeat events, runs GPT‑4o analysis for disruption prediction, applies an operational rules engine, routes by severity, stores results to PostgreSQL tables, and sends multi-channel alerts (Slack, email, SMS). It also aggregates metrics for a dashboard table.

### 1.1 Entry Points & Ingestion
- Scheduled polling every 2 minutes (API polling flow)
- Webhook for real-time ingestion (event-driven flow)

### 1.2 Data Validation, Normalization, and Rate Limiting
- Normalizes incoming fields, timestamps, and flight numbers
- Adds validation metadata
- Wait node acts as a basic rate limiter between calls and processing

### 1.3 Idempotency / Deduplication Gate
- SHA-256 hash idempotency key stored in workflow static data (24h sliding window)
- Duplicate events are stopped; only new events proceed

### 1.4 AI Disruption Prediction (GPT‑4o) + Structured Parsing
- LangChain Agent uses GPT‑4o with a “return JSON” structured output parser
- Produces disruption probability, delay impact, safety risk, severity, recommended actions, etc.

### 1.5 Operational Rules Engine
- Applies airline constraints (crew duty, turnaround, slots, maintenance, connections, passenger rights)
- Produces feasibility score/status and rule violations

### 1.6 Severity Routing, Persistence, and Notifications
- Switch routes critical/high/medium/low
- Writes to corresponding Postgres tables
- Critical path triggers Slack → Email → SMS → passenger notification preparation → rebooking check

### 1.7 Cross-Path Merge & Dashboard Metrics
- Merges outputs from several notification/storage branches
- Aggregates counts/averages and stores dashboard metrics

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled API Polling Ingestion
**Overview:** Polls an external flight schedule API every 2 minutes using configured endpoints, then hands results to a rate limiter and the validation pipeline.  
**Nodes involved:**  
- Schedule: Poll APIs Every 2 Minutes  
- Workflow Configuration  
- Fetch Flight & Weather Data  
- Rate Limit Handler  

#### Node: Schedule: Poll APIs Every 2 Minutes
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Config choices:** Runs every 2 minutes.
- **Inputs/outputs:** No input. Output → *Workflow Configuration*.
- **Failure/edge cases:** High frequency can create backlog if downstream is slow; consider concurrency/execution limits.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central config constants.
- **Config choices (interpreted):**
  - Stores placeholder API URLs: `flightApiUrl`, `weatherApiUrl`, `telemetryApiUrl`, `passengerApiUrl`
  - Stores `dbConnectionString` (unused directly by DB nodes—credentials are handled separately in n8n)
  - Alert routing vars: `slackOccChannel`, `groundStaffEmail`, `customerServicePhone`
  - Thresholds: `delayThresholdMinutes`=30, `criticalSeverityThreshold`=90 (not directly used elsewhere)
  - `includeOtherFields: true` so input fields would be preserved (here it’s a trigger, so mostly config only).
- **Key expressions:** Values referenced later, e.g. Slack channel and thresholds.
- **Inputs/outputs:** Input from Schedule. Output → *Fetch Flight & Weather Data*.
- **Failure/edge cases:** Placeholder values must be replaced or HTTP request will fail.

#### Node: Fetch Flight & Weather Data
- **Type / role:** `HTTP Request` — pulls flight data (despite node name mentioning weather, only flightApiUrl is used here).
- **Config choices:**
  - URL from expression: `{{$('Workflow Configuration').first().json.flightApiUrl}}`
  - Query param: `limit=100`
  - Response format: JSON
- **Inputs/outputs:** Input from *Workflow Configuration*. Output → *Rate Limit Handler*.
- **Failure/edge cases:** DNS/timeout/auth issues; API may paginate; schema differences are addressed in the next Code node.

#### Node: Rate Limit Handler
- **Type / role:** `Wait` — throttling/rate limiting.
- **Config choices:** Wait `amount=0.5` (seconds in practice for Wait node “time interval” style).
- **Inputs/outputs:** Input from *Fetch Flight & Weather Data*. Output → *Validate & Normalize Data*.
- **Failure/edge cases:** Adds latency; does not guarantee upstream API rate compliance if multiple parallel executions happen.

---

### Block 2 — Real-Time Webhook Ingestion
**Overview:** Accepts real-time airline events via HTTP POST and feeds them into the same validation pipeline as scheduled polling.  
**Nodes involved:**  
- Webhook: Real-Time Event Ingestion  
- Validate & Normalize Data  

#### Node: Webhook: Real-Time Event Ingestion
- **Type / role:** `Webhook` — event-driven entry point.
- **Config choices:**
  - Path: `/airline-events`
  - Method: POST
  - Auth: `headerAuth` (requires configured header credential)
  - Response: `lastNode`
- **Credentials:** `httpHeaderAuth` credential named “Chart-IMG” (must exist and match expected header).
- **Inputs/outputs:** No input. Output → *Validate & Normalize Data*.
- **Failure/edge cases:** Invalid/missing auth header → 401/403; large payloads; schema mismatch handled by normalization.

---

### Block 3 — Validation & Normalization
**Overview:** Ensures events have required fields, normalizes naming conventions, sanitizes strings, validates flight number format, converts times to ISO, and emits validation metadata.  
**Nodes involved:**  
- Validate & Normalize Data  

#### Node: Validate & Normalize Data
- **Type / role:** `Code` — schema normalization and validation.
- **Config choices:**
  - Mode: run once per item.
  - Normalizes fields: `flightNumber`, `departureTime`, `arrivalTime`, `status`, `origin`, `destination`, etc.
  - Sanitizes strings by trimming and removing `< > " '`.
  - Validates IATA flight number regex: `/^[A-Z]{2,3}\d{1,4}$/i`.
  - Converts timestamps to ISO via `new Date(timestamp).toISOString()`.
  - Emits `validation: { isValid, timestamp, errors[], warnings[] }`.
- **Inputs/outputs:** Input from *Rate Limit Handler* OR *Webhook*. Output → *Check Idempotency & Deduplication*.
- **Failure/edge cases:**
  - `isValid` may be false but the workflow does not branch on it; invalid data continues unless later steps break.
  - Some downstream nodes expect fields not produced here (e.g., `weatherConditions`, `telemetryData` in the AI prompt) — may become `undefined`.

---

### Block 4 — Idempotency & Deduplication Gate
**Overview:** Prevents repeated processing of the same flight event by hashing key fields and storing seen hashes in workflow static data for 24 hours.  
**Nodes involved:**  
- Check Idempotency & Deduplication  
- Is Duplicate Event?  

#### Node: Check Idempotency & Deduplication
- **Type / role:** `Code` — idempotency key generation and cache.
- **Config choices:**
  - Uses Node.js `crypto` SHA‑256 over: `flightNumber|departureTime|eventType`
  - Uses `$workflow.staticData` keys:
    - `eventCache` (boolean map)
    - `eventCacheTimestamps`
  - Cleans entries older than 24 hours (sliding window)
  - Outputs: `idempotencyKey`, `isDuplicate`, `eventStatus`, `cacheSize`
- **Inputs/outputs:** Input from *Validate & Normalize Data*. Output → *Is Duplicate Event?*
- **Failure/edge cases:**
  - **eventType is often missing** (normalization doesn’t set it), so many different events might hash similarly if `eventType` stays empty.
  - Static data is per-workflow; in clustered n8n or stateless setups, behavior may differ.
  - Cache growth: bounded by 24h cleanup, but still could be large under high throughput.

#### Node: Is Duplicate Event?
- **Type / role:** `IF` — routing gate.
- **Config choices:** Condition checks `isDuplicate === true`.
- **Connections:**  
  - **True branch:** goes to an empty output (no node connected) → duplicates are dropped.  
  - **False branch:** to *AI Agent: Disruption Prediction & Analysis*.
- **Failure/edge cases:** None specific; but “true branch is unhandled” means duplicates are silently ignored (by design).

---

### Block 5 — AI Disruption Prediction + Structured Output
**Overview:** Uses a LangChain Agent backed by GPT‑4o to analyze disruption risk and return structured JSON, with an output parser enforcing schema correctness.  
**Nodes involved:**  
- AI Agent: Disruption Prediction & Analysis  
- OpenAI GPT-4  
- Structured Output: Disruption Analysis  

#### Node: OpenAI GPT-4
- **Type / role:** `lmChatOpenAi` — LLM provider.
- **Config choices:** Model `gpt-4o`, temperature 0.3, maxTokens 2000.
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Connections:** Provides `ai_languageModel` to:
  - *AI Agent: Disruption Prediction & Analysis*
  - *Structured Output: Disruption Analysis*
- **Failure/edge cases:** Rate limits, quota exhaustion, model availability, token overflow if inputs become large.

#### Node: Structured Output: Disruption Analysis
- **Type / role:** `outputParserStructured` — enforces structured JSON from the agent.
- **Config choices:** `autoFix: true` (attempts to correct malformed JSON).
- **Connections:** Connected to the agent via `ai_outputParser`.
- **Failure/edge cases:** If the model output is too malformed, parsing may still fail; “autoFix” can introduce subtle data changes.

#### Node: AI Agent: Disruption Prediction & Analysis
- **Type / role:** `LangChain Agent` — orchestration of prompt + toolchain.
- **Config choices:**
  - Prompt text includes variables: flight number, departure time, destination, status, **weatherConditions**, **telemetryData**.
  - System message requests:
    - disruption probability (0–100%), delay impact minutes
    - safety risk level
    - severity (low/medium/high/critical)
    - recommended actions
    - OCC review flag, plus multiple operational considerations
  - Has output parser enabled (structured JSON expected).
- **Inputs/outputs:** Input from *Is Duplicate Event? (false)*. Output → *Apply Operational Rules Engine*.
- **Failure/edge cases:**
  - Input fields `weatherConditions` and `telemetryData` are not produced upstream; agent may hallucinate or return partials unless data is supplied.
  - Must ensure the expected JSON keys match downstream expectations (they currently don’t consistently match; see DB nodes).

---

### Block 6 — Operational Rules Engine + Severity Routing
**Overview:** Applies deterministic airline operational constraints and computes feasibility, then routes the event by severity to persistence and alerting branches.  
**Nodes involved:**  
- Apply Operational Rules Engine  
- Route by Severity & Type  

#### Node: Apply Operational Rules Engine
- **Type / role:** `Code` — business rules.
- **Config choices (interpreted):**
  - Crew duty time limit >14h → critical violation, -30 score
  - Turnaround <45 min → high violation, -20 score
  - Slot constraints for a list of congested airports; if `slotAvailable === false` → critical, -40
  - Maintenance required if `maintenanceStatus==='required'` or `hoursSinceLastMaintenance>100` → critical, -50
  - Connection time checks (min 60 intl, 45 domestic) → high violation, -10 each
  - Passenger rights flags based on delay thresholds (care >120, compensation >180)
  - Feasibility score → status mapping:
    - <30 NOT_FEASIBLE
    - <60 REQUIRES_INTERVENTION
    - <80 FEASIBLE_WITH_CONSTRAINTS
    - else FEASIBLE
  - Adds `operationalRules`, `operationalFeasibility`, `iataCompliance`, `operationalContext`
- **Inputs/outputs:** Input from AI Agent. Output → *Route by Severity & Type*.
- **Failure/edge cases:**
  - Many referenced fields may not exist unless the AI returns them (`crewDutyTime`, `turnaroundTime`, etc.), leading to default assumptions.
  - Returns `enrichedData` as a plain object (not `{json: ...}`); n8n Code node typically expects items. In “runOnceForEachItem”, returning an object is accepted as the new `json` for that item in many n8n versions, but this can be version-sensitive.

#### Node: Route by Severity & Type
- **Type / role:** `Switch` — routes based on `$json.severity`.
- **Config choices:**
  - Outputs: Critical, High, Medium, Low (renamed outputs)
  - Fallback output index 3 (routes unknown severity to Low path)
- **Inputs/outputs:** Input from rules engine. Outputs to the four Postgres storage nodes.
- **Failure/edge cases:** If the AI outputs severity with different casing (e.g., “CRITICAL”), it will not match; normalization is not applied here.

---

### Block 7 — Persistence (PostgreSQL) by Severity
**Overview:** Writes events into severity-specific tables for auditability and downstream reporting.  
**Nodes involved:**  
- Store: Critical Events DB  
- Store: High Priority Events DB  
- Store: Medium Priority Events DB  
- Store: Low Priority Events DB  

> Note: The mappings are inconsistent across severity levels (some expect `eventId`, others `event_id`; some expect `raw_data` fields). This is a major integration risk unless upstream standardizes the schema.

#### Node: Store: Critical Events DB
- **Type / role:** `Postgres` — insert critical event row.
- **Config choices:**
  - Table: `public.critical_events`
  - Columns map uses expressions like:
    - `event_id: {{$json.eventId}}` (**likely missing**; upstream uses `idempotencyKey` not `eventId`)
    - `raw_data: {{$json}}`
    - `flight_number: {{$json.flightNumber}}`
    - `recommended_actions: {{$json.recommendedActions}}` (**naming mismatch vs earlier “proposedActions”**)
- **Inputs/outputs:** Input from Switch “Critical”. Output → *Alert: OCC Slack Channel*.
- **Failure/edge cases:** DB credential missing; schema mismatch; JSON column types; undefined fields cause null insert; mapping mismatch may violate NOT NULL constraints.

#### Node: Store: High Priority Events DB
- **Type / role:** `Postgres` — insert high priority row.
- **Config choices:**
  - Table: `public.high_priority_events`
  - Column mapping uses snake_case keys: `$json.event_id`, `$json.raw_data`, etc. (**likely missing**)
- **Inputs/outputs:** Input from Switch “High”. Output → *Merge All Notification Paths* (input index 2).
- **Failure/edge cases:** Same as above; additionally, schema mapping list is defined but still relies on snake_case inputs.

#### Node: Store: Medium Priority Events DB
- **Type / role:** `Postgres` — insert medium priority row.
- **Config choices:** Table `public.medium_priority_events`; expects `$json.event_id`, `$json.raw_data`, etc. (**likely missing**)
- **Inputs/outputs:** Input from Switch “Medium”. Output → *Merge All Notification Paths* (input index 3).

#### Node: Store: Low Priority Events DB
- **Type / role:** `Postgres` — insert low priority row.
- **Config choices:** Table `public.low_priority_events`; expects `$json.event_id` etc. (**likely missing**)
- **Inputs/outputs:** Input from Switch “Low”. No outgoing connection shown (it’s routed but not merged afterward).
- **Failure/edge cases:** Low path is not merged into dashboard metrics, so low events may be undercounted unless Switch output 3 is actually connected elsewhere (it is connected here, but not merged).

---

### Block 8 — Critical Event Alerting (Slack → Email → SMS) + Passenger Notification Prep
**Overview:** For critical events, notifies OCC in Slack, emails ground staff, sends SMS to customer service, prepares passenger notification data, and checks whether rebooking workflow should be triggered.  
**Nodes involved:**  
- Alert: OCC Slack Channel  
- Email: Ground Staff Notification  
- SMS: Customer Service Alert  
- Prepare Passenger Notification Data  
- Requires Rebooking?  
- Trigger Rebooking Workflow  

#### Node: Alert: OCC Slack Channel
- **Type / role:** `Slack` — OCC channel alert.
- **Config choices:**
  - OAuth2 authentication
  - Channel ID from config: `{{$('Workflow Configuration').first().json.slackOccChannel}}`
  - Message uses: flightNumber, disruptionProbability, delayImpactMinutes, safetyRisk, affectedPassengers, and `recommendedActions.join(", ")`
- **Inputs/outputs:** Input from *Store: Critical Events DB*. Output → *Email: Ground Staff Notification*.
- **Failure/edge cases:**
  - `recommendedActions` must be an array; if missing or string, `.join()` will error.
  - Slack credential scope/channel permission issues.

#### Node: Email: Ground Staff Notification
- **Type / role:** `Email Send` — ground staff email.
- **Config choices:**
  - To: config `groundStaffEmail`
  - HTML template uses `recommendedActions.map(...)` (expects array)
  - From: `user@example.com` (must be allowed by SMTP/Gmail setup)
- **Inputs/outputs:** Input from Slack node. Output → *SMS: Customer Service Alert*.
- **Failure/edge cases:** SMTP/Gmail auth; spam/DKIM; `recommendedActions` not an array will break the expression.

#### Node: SMS: Customer Service Alert
- **Type / role:** `Twilio` — SMS outbound.
- **Config choices:**
  - To: config `customerServicePhone`
  - From: placeholder must be replaced with Twilio number
  - Message interpolates flightNumber, probability, delay, affectedPassengers
- **Inputs/outputs:** Input from Email node. Output → *Prepare Passenger Notification Data*.
- **Failure/edge cases:** Twilio credentials, invalid from/to numbers, region restrictions, rate limiting.

#### Node: Prepare Passenger Notification Data
- **Type / role:** `Set` — prepares downstream “passenger outreach” payload.
- **Config choices:**
  - Adds: `notificationType=passenger_alert`, `notificationChannel=email_sms`, `messageTemplate=flight_disruption`
  - `passengerList` set to `{{$json.affectedPassengerIds}}` (often missing unless AI supplies it)
  - `urgency={{$json.severity}}`
  - `includeOtherFields: true`
- **Inputs/outputs:** Input from Twilio. Output → *Requires Rebooking?*
- **Failure/edge cases:** `affectedPassengerIds` may not exist; stored as string type in Set config.

#### Node: Requires Rebooking?
- **Type / role:** `IF` — decides to trigger rebooking.
- **Config choices:** OR condition:
  - `delayImpactMinutes > delayThresholdMinutes (30)`
  - OR `severity == 'critical'`
- **Connections:**
  - True → *Trigger Rebooking Workflow*
  - False → *Merge All Notification Paths* (input index 1)
- **Failure/edge cases:** If `delayImpactMinutes` is missing or non-numeric, comparison may misbehave (“loose” validation).

#### Node: Trigger Rebooking Workflow
- **Type / role:** `Code` — simulates initiating a rebooking process.
- **Config choices (interpreted):**
  - Loops through items; expects `item.json.passengers`, `flightDetails`, `alternativeFlights`, `priorityLevel`
  - Generates request ID `RBK-${Date.now()}-${random}`
  - Builds `apiPayload` with endpoint `/api/rebooking/initiate` but **does not actually call HTTP**
- **Inputs/outputs:** Input from IF (true). Output → *Merge All Notification Paths* (input index 0).
- **Failure/edge cases:** Upstream data fields (`passengers`, `flightDetails`) are not created anywhere else; will default to empty arrays/objects, causing “successful initiation” with 0 passengers.

---

### Block 9 — Merge + Dashboard Metrics Storage
**Overview:** Consolidates outputs from multiple branches and writes aggregated KPIs to a dashboard metrics table.  
**Nodes involved:**  
- Merge All Notification Paths  
- Generate Dashboard Metrics  
- Store: Dashboard Metrics DB  

#### Node: Merge All Notification Paths
- **Type / role:** `Merge` — combines multiple streams for unified aggregation.
- **Config choices:** `numberInputs=4`
- **Inputs/outputs (as wired):**
  - Input 0: from *Trigger Rebooking Workflow*
  - Input 1: from *Requires Rebooking?* (false)
  - Input 2: from *Store: High Priority Events DB*
  - Input 3: from *Store: Medium Priority Events DB*
  - Output → *Generate Dashboard Metrics*
- **Failure/edge cases:** Low severity path is not connected here; metrics will omit low events unless separately handled.

#### Node: Generate Dashboard Metrics
- **Type / role:** `Aggregate` — aggregates all item data.
- **Config choices:** `aggregateAllItemData` (expects consistent fields across items).
- **Outputs/assumptions:** Downstream expects fields like `lowCount`, `highCount`, `avgDelayImpact`, etc., but no upstream node explicitly creates these. This node may not produce those keys automatically unless configured with specific aggregations (not shown here).
- **Inputs/outputs:** Input from Merge. Output → *Store: Dashboard Metrics DB*.
- **Failure/edge cases:** Misconfiguration can yield missing metric fields, causing DB insert failures.

#### Node: Store: Dashboard Metrics DB
- **Type / role:** `Postgres` — stores KPIs in `public.dashboard_metrics`.
- **Config choices:** Inserts:
  - counts: low/high/medium/critical/total
  - averages: delay impact, disruption probability
  - total affected passengers
  - timestamp `$now`
- **Inputs/outputs:** Input from Aggregate. End of flow.
- **Failure/edge cases:** If aggregate output lacks these fields, inserts may be null/invalid.

---

### Block 10 — Documentation-Only Sticky Notes (Non-Executable)
**Overview:** These are comments describing a *healthcare patient coordination* solution (EHR/FHIR, reminders, HIPAA), which does not match the airline nodes. They are still part of the workflow JSON and should be treated as documentation artifacts.  
**Nodes involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule: Poll APIs Every 2 Minutes | scheduleTrigger | Scheduled entry point | — | Workflow Configuration | ## Patient Data Ingestion  \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |
| Webhook: Real-Time Event Ingestion | webhook | Real-time entry point | — | Validate & Normalize Data | ## Patient Data Ingestion  \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |
| Workflow Configuration | set | Central config values | Schedule: Poll APIs Every 2 Minutes | Fetch Flight & Weather Data | ## Patient Data Ingestion  \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |
| Fetch Flight & Weather Data | httpRequest | Pull flight data from external API | Workflow Configuration | Rate Limit Handler | ## Patient Data Ingestion  \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |
| Rate Limit Handler | wait | Throttle downstream processing | Fetch Flight & Weather Data | Validate & Normalize Data | ## Patient Data Ingestion  \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |
| Validate & Normalize Data | code | Validate/normalize schema and timestamps | Rate Limit Handler; Webhook: Real-Time Event Ingestion | Check Idempotency & Deduplication | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| Check Idempotency & Deduplication | code | Deduplicate events (24h cache) | Validate & Normalize Data | Is Duplicate Event? | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| Is Duplicate Event? | if | Drop duplicates / allow new | Check Idempotency & Deduplication | (duplicates dropped); AI Agent: Disruption Prediction & Analysis | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| AI Agent: Disruption Prediction & Analysis | langchain agent | AI analysis + recommendations | Is Duplicate Event? (false) | Apply Operational Rules Engine | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| OpenAI GPT-4 | lmChatOpenAi | LLM model provider (gpt-4o) | — (AI connection) | AI Agent: Disruption Prediction & Analysis; Structured Output: Disruption Analysis | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| Structured Output: Disruption Analysis | outputParserStructured | Parse/repair structured JSON | — (AI connection) | AI Agent: Disruption Prediction & Analysis (as output parser) | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| Apply Operational Rules Engine | code | Deterministic constraint checks & feasibility | AI Agent: Disruption Prediction & Analysis | Route by Severity & Type | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Route by Severity & Type | switch | Route by severity | Apply Operational Rules Engine | Store: Critical Events DB; Store: High Priority Events DB; Store: Medium Priority Events DB; Store: Low Priority Events DB | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Store: Critical Events DB | postgres | Persist critical events | Route by Severity & Type (Critical) | Alert: OCC Slack Channel | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Alert: OCC Slack Channel | slack | Slack alert to OCC | Store: Critical Events DB | Email: Ground Staff Notification | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Email: Ground Staff Notification | emailSend | Email alert | Alert: OCC Slack Channel | SMS: Customer Service Alert | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| SMS: Customer Service Alert | twilio | SMS alert | Email: Ground Staff Notification | Prepare Passenger Notification Data | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Prepare Passenger Notification Data | set | Build passenger notification payload | SMS: Customer Service Alert | Requires Rebooking? | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Requires Rebooking? | if | Decide rebooking trigger | Prepare Passenger Notification Data | Trigger Rebooking Workflow; Merge All Notification Paths | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Trigger Rebooking Workflow | code | Initiate rebooking (simulated) | Requires Rebooking? (true) | Merge All Notification Paths | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Store: High Priority Events DB | postgres | Persist high events | Route by Severity & Type (High) | Merge All Notification Paths | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Store: Medium Priority Events DB | postgres | Persist medium events | Route by Severity & Type (Medium) | Merge All Notification Paths | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Store: Low Priority Events DB | postgres | Persist low events | Route by Severity & Type (Low) | — | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Merge All Notification Paths | merge | Consolidate multi-branch outputs | Trigger Rebooking Workflow; Requires Rebooking? (false); Store: High Priority Events DB; Store: Medium Priority Events DB | Generate Dashboard Metrics |  |
| Generate Dashboard Metrics | aggregate | Compute KPI rollups | Merge All Notification Paths | Store: Dashboard Metrics DB |  |
| Store: Dashboard Metrics DB | postgres | Persist KPI rollups | Generate Dashboard Metrics | — |  |
| Sticky Note | stickyNote | Comment | — | — | ## Prerequisites\nActive EHR system with FHIR API access or HL7 integration capability.  \n## Use Cases\nAutomated appointment reminder campaigns reducing no-shows.\n## Customization\nModify risk scoring models for specialty-specific patient populations. \n## Benefits\nReduces patient no-show rates by 40% through timely, personalized reminders. |
| Sticky Note1 | stickyNote | Comment | — | — | ## Setup Steps\n1. Configure EHR/FHIR API credentialsfor patient data access\n2. Set up webhook endpoints for real-time clinical event notifications\n3. Add OpenAI API key for patient risk stratification and communication personalization\n4. Configure Twilio credentials for SMS and voice call delivery\n5. Set Gmail OAuth or SMTP credentials for email appointment reminders\n6. Connect Slack workspace and define care coordination alert channels |
| Sticky Note2 | stickyNote | Comment | — | — | ## How It Works\nThis workflow automates end-to-end patient care coordination by monitoring appointment schedules, clinical events, and care milestones while orchestrating personalized communications across multiple channels. Designed for healthcare operations teams, care coordinators, and patient engagement specialists, it solves the challenge of manual patient follow-up, missed appointments, and fragmented communication across care teams. The system triggers on scheduled intervals and real-time clinical events, ingesting data from EHR systems, appointment schedulers, and lab result feeds. Patient records flow through validation and risk stratification layers using AI models that identify high-risk patients, predict no-show probability, and recommend intervention timing. The workflow applies clinical protocols for appointment reminders, medication adherence checks, and post-discharge follow-ups. Critical cases automatically route to care coordinators via Slack alerts, while routine communications deploy via SMS, email, and patient portal notifications. All interactions log to secure databases for compliance documentation. This eliminates manual outreach coordination, reduces no-shows by 40%, and ensures HIPAA-compliant patient engagement at scale. |
| Sticky Note3 | stickyNote | Comment | — | — | ## Care Team Alerting \nRoutes high-risk patient cases to care coordinators via Slack with clinical context and recommended interventions.\n**Why** - Ensures timely human oversight for complex cases while automating routine coordination tasks. |
| Sticky Note4 | stickyNote | Comment | — | — | ## Clinical Protocol Application \nApplies evidence-based care pathways for appointment reminders, medication adherence, post-discharge follow-ups based on patient conditions.\n**Why** - Ensures consistent, guideline-compliant patient communications while reducing manual protocol interpretation by care teams. |
| Sticky Note5 | stickyNote | Comment | — | — | ## Risk Stratification\nRoutes patient records through AI models calculating no-show probability, readmission risk, and care urgency scores.\n**Why** - Identifies patients requiring proactive intervention before adverse events occur, shifting from reactive to predictive care management. |
| Sticky Note6 | stickyNote | Comment | — | — | ## Patient Data Ingestion \nFetches appointment schedules, clinical events, and patient demographics from EHR/FHIR APIs using scheduled triggers and webhooks.\n**Why** - Centralizes patient data from fragmented healthcare systems, enabling unified care coordination without manual data compilation. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger (Scheduled)**
   1. Add node: **Schedule Trigger**
   2. Set interval: every **2 minutes**
   3. Name: “Schedule: Poll APIs Every 2 Minutes”

2) **Add Configuration Node**
   1. Add node: **Set** named “Workflow Configuration”
   2. Add fields (examples):
      - `flightApiUrl` (string) → your flight schedule endpoint
      - `weatherApiUrl`, `telemetryApiUrl`, `passengerApiUrl` (strings) as needed
      - `slackOccChannel` (string) → Slack channel ID
      - `groundStaffEmail` (string)
      - `customerServicePhone` (string)
      - `delayThresholdMinutes` (number) = 30
      - `criticalSeverityThreshold` (number) = 90 (optional; not used downstream)
   3. Enable “Include Other Fields”
   4. Connect: **Schedule Trigger → Workflow Configuration**

3) **Fetch Data from API**
   1. Add node: **HTTP Request** named “Fetch Flight & Weather Data”
   2. URL expression: `{{$('Workflow Configuration').first().json.flightApiUrl}}`
   3. Response format: **JSON**
   4. Add query parameter `limit=100`
   5. Connect: **Workflow Configuration → Fetch Flight & Weather Data**

4) **Rate Limiting**
   1. Add node: **Wait** named “Rate Limit Handler”
   2. Set wait time to **0.5 seconds**
   3. Connect: **Fetch Flight & Weather Data → Rate Limit Handler**

5) **Create Webhook Entry Point**
   1. Add node: **Webhook** named “Webhook: Real-Time Event Ingestion”
   2. Path: `airline-events`
   3. Method: POST
   4. Authentication: **Header Auth**
   5. Response mode: **Last Node**
   6. Create/attach **HTTP Header Auth** credential (define expected header key/value)
   7. Connect: **Webhook → Validate & Normalize Data** (next step)

6) **Validate & Normalize**
   1. Add node: **Code** named “Validate & Normalize Data”
   2. Mode: **Run once for each item**
   3. Paste logic equivalent to:
      - normalize field names, sanitize strings
      - validate flight number with IATA-like regex
      - convert departure/arrival timestamps to ISO
      - output `validation` metadata
   4. Connect:
      - **Rate Limit Handler → Validate & Normalize Data**
      - **Webhook → Validate & Normalize Data**

7) **Idempotency/Deduplication**
   1. Add node: **Code** named “Check Idempotency & Deduplication”
   2. Mode: **Run once for each item**
   3. Implement:
      - compute SHA‑256 of `flightNumber|departureTime|eventType`
      - store in `$workflow.staticData` with timestamps
      - prune entries older than 24 hours
      - output `isDuplicate` and `idempotencyKey`
   4. Connect: **Validate & Normalize Data → Check Idempotency & Deduplication**

8) **Duplicate Gate**
   1. Add node: **IF** named “Is Duplicate Event?”
   2. Condition: boolean `{{$('Check Idempotency & Deduplication').item.json.isDuplicate}}` is **true**
   3. Connect: **Check Idempotency & Deduplication → Is Duplicate Event?**
   4. Leave **true** branch unconnected (to drop duplicates)
   5. Connect **false** branch to AI Agent (next step)

9) **OpenAI Model**
   1. Add node: **OpenAI Chat Model** (LangChain) named “OpenAI GPT-4”
   2. Model: **gpt-4o**
   3. Temperature: **0.3**, Max tokens: **2000**
   4. Attach **OpenAI API credential**

10) **Structured Output Parser**
   1. Add node: **Structured Output Parser** named “Structured Output: Disruption Analysis”
   2. Enable **Auto-fix**

11) **AI Agent**
   1. Add node: **AI Agent** (LangChain) named “AI Agent: Disruption Prediction & Analysis”
   2. Configure:
      - System message: airline OCC analyst instructions (severity, probability, delay impact, actions, etc.)
      - User text template: include flight fields (flightNumber, departureTime, destination, status, weather, telemetry)
      - Enable structured output (connect parser)
   3. Connect AI wiring:
      - In the Agent, set **Language Model** to “OpenAI GPT-4”
      - Set **Output Parser** to “Structured Output: Disruption Analysis”
   4. Connect main flow: **Is Duplicate Event? (false) → AI Agent**

12) **Operational Rules Engine**
   1. Add node: **Code** named “Apply Operational Rules Engine”
   2. Mode: run once per item
   3. Implement rules for crew, turnaround, slots, maintenance, connections, passenger rights; output feasibility score/status and violations.
   4. Connect: **AI Agent → Apply Operational Rules Engine**

13) **Severity Routing**
   1. Add node: **Switch** named “Route by Severity & Type”
   2. Add rules:
      - if `{{$json.severity}} == 'critical'` → output “Critical”
      - if equals `high` → “High”
      - `medium` → “Medium”
      - `low` → “Low”
   3. Set fallback to “Low”
   4. Connect: **Apply Operational Rules Engine → Route by Severity & Type**

14) **PostgreSQL Storage Nodes**
   1. Add **Postgres** node “Store: Critical Events DB” → table `public.critical_events`
   2. Add **Postgres** node “Store: High Priority Events DB” → `public.high_priority_events`
   3. Add **Postgres** node “Store: Medium Priority Events DB” → `public.medium_priority_events`
   4. Add **Postgres** node “Store: Low Priority Events DB” → `public.low_priority_events`
   5. Create/attach **Postgres credential** (host/db/user/password/SSL).
   6. Map columns consistently (recommended):
      - Use one naming convention (camelCase or snake_case), and ensure AI/rules output matches it.
      - Consider using `idempotencyKey` as `event_id` everywhere.
   7. Connect Switch outputs to each Postgres node.

15) **Critical Notifications**
   1. Add **Slack** node “Alert: OCC Slack Channel”
      - OAuth2 Slack credential
      - Channel ID from config
      - Message referencing fields (ensure arrays are arrays before `.join`)
   2. Add **Email Send** node “Email: Ground Staff Notification”
      - SMTP/Gmail credential
      - To from config, HTML body
   3. Add **Twilio** node “SMS: Customer Service Alert”
      - Twilio credential
      - From = your Twilio number; To from config
   4. Connect:  
      - **Store: Critical Events DB → Slack → Email → Twilio**

16) **Passenger Notification Prep + Rebooking Decision**
   1. Add **Set** node “Prepare Passenger Notification Data” with notification fields
   2. Add **IF** node “Requires Rebooking?”
      - `delayImpactMinutes > delayThresholdMinutes` OR severity is critical
   3. Add **Code** node “Trigger Rebooking Workflow”
      - If you want real triggering, replace the simulated `apiPayload` with an **HTTP Request** to an internal rebooking service or an **Execute Workflow** node.
   4. Connect: **Twilio → Prepare Passenger Notification Data → Requires Rebooking?**
      - True → Trigger Rebooking Workflow
      - False → Merge node (next step)

17) **Merge Paths**
   1. Add **Merge** node “Merge All Notification Paths” set to **4 inputs**
   2. Connect:
      - Input 0: **Trigger Rebooking Workflow**
      - Input 1: **Requires Rebooking? (false)**
      - Input 2: **Store: High Priority Events DB**
      - Input 3: **Store: Medium Priority Events DB**

18) **Aggregate Metrics**
   1. Add **Aggregate** node “Generate Dashboard Metrics”
      - Configure explicit aggregations to produce:
        - counts by severity
        - averages (delay impact, disruption probability)
        - total affected passengers
      - (As-is, the provided node configuration is likely insufficient unless further configured.)
   2. Connect: **Merge → Aggregate**

19) **Store Dashboard Metrics**
   1. Add **Postgres** node “Store: Dashboard Metrics DB” → table `public.dashboard_metrics`
   2. Map columns to aggregated fields and insert `$now` timestamp
   3. Connect: **Aggregate → Store: Dashboard Metrics DB**

20) **Optional: Add Sticky Notes**
   - Add sticky notes as comments (non-executable). Ensure they match the actual airline use case or update nodes to match the healthcare narrative.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The sticky notes describe a healthcare patient coordination workflow (EHR/FHIR, reminders, HIPAA), but the executable nodes implement an airline OCC disruption management workflow. | Ensure the documentation narrative and the node implementation are aligned before production use. |
| **Prerequisites**: “Active EHR system with FHIR API access or HL7 integration capability.” | Sticky Note (comment). |
| **Setup Steps**: EHR/FHIR credentials, webhook endpoints, OpenAI key, Twilio, Gmail/SMTP, Slack channels. | Sticky Note1 (comment). |
| “Reduces patient no-show rates by 40%…” | Sticky Note and Sticky Note2 (comment). This benefit statement does not match the airline logic as implemented. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.