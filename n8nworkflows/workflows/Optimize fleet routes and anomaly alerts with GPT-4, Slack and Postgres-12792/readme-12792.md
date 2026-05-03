Optimize fleet routes and anomaly alerts with GPT-4, Slack and Postgres

https://n8nworkflows.xyz/workflows/optimize-fleet-routes-and-anomaly-alerts-with-gpt-4--slack-and-postgres-12792


# Optimize fleet routes and anomaly alerts with GPT-4, Slack and Postgres

## 1. Workflow Overview

**Workflow name (in JSON):** Real-Time Fleet Telemetry Intelligence and Dispatch Automation  
**Provided title:** Optimize fleet routes and anomaly alerts with GPT-4, Slack and Postgres

**Purpose:**  
Processes incoming fleet telemetry (real-time via webhook and periodically via schedule), normalizes and validates the data, enriches it with external traffic/weather information, uses GPT‑4o agents to (1) optimize routes and predict ETA and (2) detect operational/safety/compliance anomalies, then applies business rules to route notifications/actions (Slack, email, manual review wait), stores telemetry/incidents in Postgres, and finally returns a JSON response.

**Major logical blocks**
- **1.1 Trigger & Configuration Layer:** Webhook + schedule triggers merged; configuration variables set.
- **1.2 Telemetry Normalization & Data Quality:** Normalize schema → validate values/types → deduplicate.
- **1.3 External Context Enrichment:** Fetch traffic/weather → merge into telemetry.
- **1.4 AI Processing (Parallel Intelligence):** Route optimization agent + structured output; anomaly detection agent + structured output.
- **1.5 Business Rules & Severity Routing:** Compute priority/rule violations → switch by severity.
- **1.6 Actions & Persistence:** Slack notifications, email to ops, wait-for-review, rerouting simulation, Postgres storage, reporting query.
- **1.7 Webhook Response:** Final JSON response (response mode “lastNode”).

> Note: Several sticky notes describe a **veterinary inventory** scenario, which conflicts with the fleet/telemetry nodes. They are documented as-is, but they do not match the implemented logic.

---

## 2. Block-by-Block Analysis

### 1.1 Trigger & Configuration Layer

**Overview:**  
Accepts telemetry either via authenticated webhook (real-time) or via a scheduled trigger (batch/polling scenario). A Set node centralizes workflow constants (API URLs, thresholds, channel IDs, email).

**Nodes involved:**
- Telemetry Webhook Trigger
- Scheduled Data Sync
- Workflow Configuration
- Merge Trigger Sources

#### Telemetry Webhook Trigger
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — real-time entry point.
- **Configuration (interpreted):**
  - **POST** endpoint path: `/fleet-telemetry`
  - **Authentication:** Header Auth (requires configured header credential)
  - **Response mode:** `lastNode` (the final node’s output becomes the HTTP response)
- **Key variables/expressions:** none; passes request body as item JSON.
- **Connections:**
  - Output → **Workflow Configuration**
- **Edge cases / failures:**
  - Missing/invalid header auth → 401/403
  - Large payloads or unexpected body structure can break downstream normalization assumptions.

#### Scheduled Data Sync
- **Type / role:** `Schedule Trigger` — periodic execution entry point.
- **Configuration:**
  - Runs every **N minutes** (interval field is “minutes”; actual value not specified in JSON beyond structure).
- **Connections:**
  - Output → **Merge Trigger Sources** (input 2 / index 1)
- **Edge cases / failures:**
  - If this trigger produces no telemetry payload, downstream normalization may generate null `vehicleId`.

#### Workflow Configuration
- **Type / role:** `Set` — defines constants and parameters used throughout.
- **Configuration:**
  - Adds fields (keeps other incoming fields too):
    - `trafficApiUrl`, `weatherApiUrl` (placeholders)
    - thresholds: `maxSpeedThreshold` (80), `fuelEfficiencyThreshold` (15), `idleTimeThreshold` (300), `complianceHoursLimit` (11)
    - `slackChannelNormal`, `slackChannelCritical`, `opsTeamEmail` (placeholders)
- **Key expressions:** none (static values)
- **Connections:**
  - Output → **Merge Trigger Sources** (input 1 / index 0)
- **Edge cases / failures:**
  - Placeholders not replaced will cause HTTP request URL to be invalid or Slack channel resolution to fail.

#### Merge Trigger Sources
- **Type / role:** `Merge` — combines webhook-driven runs and scheduled runs into one pipeline.
- **Configuration:**
  - Default merge behavior (no explicit mode set in parameters).
- **Connections:**
  - Output → **Normalize Telemetry Data**
- **Edge cases / failures:**
  - With default merge settings, behavior depends on arrival of inputs; in some configurations, Merge can “wait” for both inputs. Here, since both triggers connect, verify the merge mode in UI to ensure it doesn’t block execution when only one trigger fires.

---

### 1.2 Telemetry Normalization & Data Quality

**Overview:**  
Unifies varied telemetry formats into a single schema, validates required fields and ranges, then deduplicates by `(vehicleId, timestamp)`.

**Nodes involved:**
- Normalize Telemetry Data
- Validate Data Quality
- Remove Duplicate Events

#### Normalize Telemetry Data
- **Type / role:** `Code` — schema normalization.
- **Configuration:**
  - Runs once per item.
  - Produces a normalized object with:
    - `vehicleId`, `timestamp`, `location.lat/lng`, `speed`, `fuelLevel`, `engineStatus`, `driverId`, `odometer`, `temperature`, `eventType`
  - Accepts multiple possible source field names (`vehicle_id`, `VehicleID`, etc.).
- **Key variables:**
  - Uses `$input.item.json` as `inputData`.
- **Connections:**
  - Output → **Validate Data Quality**
- **Edge cases / failures:**
  - If upstream item is configuration-only (no telemetry fields), `vehicleId` becomes `null` → invalid downstream.
  - `location.lat/lng` may be `null`; later HTTP request builds URL from them.

#### Validate Data Quality
- **Type / role:** `Code` — validation & annotation.
- **Configuration:**
  - Adds:
    - `isValid` boolean
    - `validationErrors` array (or `undefined` when none)
  - Validates:
    - Required: `vehicleId`, `timestamp`, `location`
    - Types: `vehicleId` string; `timestamp` parseable date; `location` object
    - Ranges: `speed` 0–200; `fuelLevel` 0–100
- **Connections:**
  - Output → **Remove Duplicate Events**
- **Edge cases / failures:**
  - This node does **not** stop invalid records; it only flags them. Downstream nodes do not check `isValid`, so invalid records can still trigger API calls/AI/DB inserts.

#### Remove Duplicate Events
- **Type / role:** `Remove Duplicates` — idempotency guard.
- **Configuration:**
  - Compare selected fields: `vehicleId, timestamp`
- **Connections:**
  - Output → **Fetch Traffic & Weather Data**
  - Output → **Merge Telemetry with External Data**
- **Edge cases / failures:**
  - If `vehicleId` or `timestamp` is null, deduping may behave unexpectedly (collisions or ineffective dedupe).

---

### 1.3 External Context Enrichment

**Overview:**  
Calls an external endpoint (named “Traffic & Weather Data” but appears to use only `trafficApiUrl`) and merges its response into telemetry.

**Nodes involved:**
- Fetch Traffic & Weather Data
- Merge Telemetry with External Data

#### Fetch Traffic & Weather Data
- **Type / role:** `HTTP Request` — external enrichment.
- **Configuration:**
  - URL expression:  
    `{{ trafficApiUrl }}?lat={{ $json.location.lat }}&lng={{ $json.location.lng }}`
  - Query parameter: `apiKey` placeholder
  - Timeout: 10s
  - Response format: JSON
  - Batching enabled (batchSize 10)
- **Key expressions/variables:**
  - `$('Workflow Configuration').first().json.trafficApiUrl`
  - `$json.location.lat` / `$json.location.lng`
- **Connections:**
  - Output → **Merge Telemetry with External Data** (input 2 / index 1)
- **Edge cases / failures:**
  - `trafficApiUrl` placeholder or invalid URL → request failure
  - Null lat/lng → malformed URL / API error
  - Rate limits/timeouts → execution errors unless error handling is added.

#### Merge Telemetry with External Data
- **Type / role:** `Merge` — enrich input1 (telemetry) with input2 (external API).
- **Configuration:**
  - Mode: `combine`
  - Join mode: `enrichInput1`
  - Match field: `vehicleId`
- **Connections:**
  - Output → **Route Optimization Agent**
- **Edge cases / failures:**
  - External API response may not contain `vehicleId`, so matching may fail; then telemetry may not be enriched.
  - If multiple telemetry items share a vehicleId, merge behavior can be ambiguous.

---

### 1.4 AI Processing (Route Optimization + Anomaly Detection)

**Overview:**  
Two LangChain “Agent” nodes call GPT‑4o with system instructions and enforce structured JSON outputs via schema parsers.

**Nodes involved:**
- Route Optimization Agent
- OpenAI GPT-4 Model
- Route Optimization Output Parser
- Anomaly Detection Agent
- OpenAI GPT-4 Model for Anomaly
- Anomaly Detection Output Parser

#### OpenAI GPT-4 Model
- **Type / role:** `lmChatOpenAi` — provides LLM backend to the route agent.
- **Configuration:**
  - Model: `gpt-4o`
  - maxTokens: 2000, temperature: 0.3
- **Credentials:** OpenAI API credential (“OpenAi account”)
- **Connections:**
  - AI languageModel output → **Route Optimization Agent**
- **Edge cases / failures:**
  - Missing/invalid OpenAI credential
  - Token limits exceeded if telemetry/context is large
  - Model availability / rate limiting.

#### Route Optimization Output Parser
- **Type / role:** `outputParserStructured` — enforces schema for route results.
- **Schema fields:**
  - `optimizedRoute` (string)
  - `estimatedETA` (string)
  - `delayMinutes` (number)
  - `fuelSavings` (number)
  - `routeConfidence` (number 0–100)
  - `recommendations` (string)
- **Connections:**
  - AI outputParser → **Route Optimization Agent**
- **Edge cases / failures:**
  - If the model returns non-conforming JSON, the parser will fail and the workflow errors.

#### Route Optimization Agent
- **Type / role:** LangChain Agent — generates route/ETA recommendations.
- **Configuration:**
  - Prompt text expression includes: `vehicleId`, `location`, `destination`, `trafficConditions`, `weatherConditions`
  - System message instructs optimization, ETA, delays, fuel efficiency, and hours-of-service compliance.
  - Output parser enabled (structured).
- **Inputs/Outputs:**
  - Main input from **Merge Telemetry with External Data**
  - Uses **OpenAI GPT-4 Model** as language model
  - Uses **Route Optimization Output Parser**
  - Main output → **Anomaly Detection Agent**
- **Edge cases / failures:**
  - `destination`, `trafficConditions`, `weatherConditions` are not guaranteed to exist in upstream data (may lead to weak outputs).
  - If the upstream merge didn’t enrich correctly, the agent context is incomplete.

#### OpenAI GPT-4 Model for Anomaly
- **Type / role:** `lmChatOpenAi` — provides LLM backend to anomaly agent.
- **Configuration:**
  - Model: `gpt-4o`
  - maxTokens: 1500, temperature: 0.2
- **Credentials:** same OpenAI credential
- **Connections:**
  - AI languageModel output → **Anomaly Detection Agent**

#### Anomaly Detection Output Parser
- **Type / role:** `outputParserStructured` — enforces schema for anomaly results.
- **Schema fields:**
  - `anomalyDetected` (boolean)
  - `anomalyType` (string; examples in description: speed/mechanical/compliance/route/safety)
  - `severityLevel` (string: NORMAL/WARNING/CRITICAL)
  - `description` (string)
  - `recommendedAction` (string)
  - `requiresManualReview` (boolean)
- **Connections:**
  - AI outputParser → **Anomaly Detection Agent**
- **Edge cases / failures:**
  - Parser failure if model output deviates from schema.

#### Anomaly Detection Agent
- **Type / role:** LangChain Agent — classifies anomalies and severity.
- **Configuration:**
  - Prompt text includes: `vehicleId`, `speed`, `fuelLevel`, `engineStatus`, `driverHours`, `temperature`
  - System message mandates severity classification and action recommendation.
  - Output parser enabled (structured).
- **Connections:**
  - Main input from **Route Optimization Agent**
  - Main output → **Apply Business Rules**
- **Edge cases / failures:**
  - `driverHours` isn’t produced earlier (normalization outputs `driverId`, not `driverHours`), so prompt may include undefined.
  - If `severityLevel` missing due to parser failure, downstream switch will route to fallback.

---

### 1.5 Business Rules & Severity Routing

**Overview:**  
Computes operational priority and rule violations, then routes flow by AI-detected `severityLevel`.

**Nodes involved:**
- Apply Business Rules
- Route by Severity

#### Apply Business Rules
- **Type / role:** `Code` — enrich item with business constraints and priority score.
- **Configuration highlights:**
  - Reads config via: `$('Workflow Configuration').first()?.json || {}`
  - Computes:
    - `businessRulesPassed` (boolean)
    - `priority` (0–100)
    - `ruleViolations` (array)
    - timestamps (`businessRulesAppliedAt`, `lastApiCallTimestamp`)
  - Checks:
    - Dispatch capacity (uses `config.maxDispatchCapacity` but **not defined** in config Set; defaults to 100)
    - DOT compliance flags (`dotCompliant`, `vehicleInspectionCurrent`)
    - Driver hours (`config.maxDriverHoursOfService` but **not defined**; defaults to 11)
    - Delivery deadline urgency (`deliveryDeadline`)
    - API rate-limiting (`minApiCallIntervalMs`)
    - Adds priority based on `item.anomalyDetected` and `item.severity` (note: anomaly schema uses `severityLevel`, not `severity`)
- **Connections:**
  - Output → **Route by Severity**
- **Edge cases / failures:**
  - Uses fields that may not exist (`currentDispatchCount`, `driverHoursWorked`, `driverRestHours`, `deliveryDeadline`, `item.severity`), so logic may not behave as intended.
  - Does not block processing even if `businessRulesPassed` is false.

#### Route by Severity
- **Type / role:** `Switch` — routes by `severityLevel`.
- **Rules:**
  - NORMAL → output “Normal Operations”
  - WARNING → output “Needs Review”
  - CRITICAL → output “Critical Alert”
  - Fallback output renamed to “Unclassified”
- **Connections:**
  - Normal Operations → **Store Telemetry Records**
  - Needs Review → **Wait for Manual Review**
  - Critical Alert → **Alert Critical Incidents**
- **Edge cases / failures:**
  - If `severityLevel` missing/invalid (e.g., lowercase), routes to fallback “Unclassified” (but no node is connected to fallback, so execution may end early depending on n8n behavior).

---

### 1.6 Actions & Persistence

**Overview:**  
Stores telemetry, alerts dispatchers for normal operations, alerts Slack + prepares customer notification for critical events, pauses for manual review on warnings, stores incident records, optionally triggers rerouting, and generates a daily-style report query.

**Nodes involved:**
- Store Telemetry Records
- Notify Dispatchers via Slack
- Alert Critical Incidents
- Prepare Customer Notification
- Check if Re-routing Required
- Trigger Re-routing Action
- Wait for Manual Review
- Store Incident Records
- Email Operations Team
- Generate Performance Report

#### Store Telemetry Records
- **Type / role:** `Postgres` — insert telemetry into `public.fleet_telemetry`.
- **Configuration:**
  - Columns mapped from item:
    - `vehicleId`, `timestamp`, `location`, `speed`, `fuelLevel`, `engineStatus`
    - AI outputs: `optimizedRoute`, `estimatedETA`, `anomalyType`, `severityLevel`
  - Outputs selected columns back.
- **Connections:**
  - Output → **Notify Dispatchers via Slack**
- **Edge cases / failures:**
  - `location` is an object; DB column must be `json/jsonb` (or compatible). If it’s text, insert fails.
  - Timestamp type mismatch if not ISO or DB expects timestamptz.

#### Notify Dispatchers via Slack
- **Type / role:** `Slack` — normal operations notification.
- **Configuration:**
  - OAuth2 Slack auth
  - Channel ID from config: `slackChannelNormal`
  - Message uses `vehicleId`, `estimatedETA`, `fuelSavings`
- **Connections:**
  - Output → **Generate Performance Report**
- **Edge cases / failures:**
  - Invalid Slack OAuth / missing scopes
  - `slackChannelNormal` placeholder not replaced
  - `fuelSavings` missing if LLM output omitted/failed.

#### Alert Critical Incidents
- **Type / role:** `Slack` — critical alert notification.
- **Configuration:**
  - Channel ID from config: `slackChannelCritical`
  - Message includes `vehicleId`, `anomalyType`, `description`, `recommendedAction`
- **Connections:**
  - Output → **Prepare Customer Notification**
- **Edge cases / failures:**
  - Same Slack auth/channel issues as above.

#### Prepare Customer Notification
- **Type / role:** `Set` — prepares outbound customer comms metadata.
- **Configuration:**
  - `customerMessage` template
  - `notificationChannel`: `"sms"` (placeholder, no SMS node present)
  - `requiresRerouting`: boolean expression:  
    `severityLevel === 'CRITICAL' && (anomalyType === 'mechanical' || anomalyType === 'route')`
- **Connections:**
  - Output → **Check if Re-routing Required**
- **Edge cases / failures:**
  - This prepares SMS content but no downstream SMS integration exists; only rerouting path is implemented.

#### Check if Re-routing Required
- **Type / role:** `IF` — gates rerouting.
- **Configuration:**
  - True when `$json.requiresRerouting` is true.
- **Connections:**
  - True output → **Trigger Re-routing Action**
  - False output: not connected (workflow may stop for that branch).
- **Edge cases / failures:**
  - If false branch is unconnected, critical incidents that don’t require reroute won’t proceed to reporting/response unless another path continues.

#### Trigger Re-routing Action
- **Type / role:** `Code` — simulated rerouting and dispatch reassignment.
- **Configuration:**
  - Generates `incidentId`, selects a best alternative vehicle from a hardcoded list, creates `newRoute`, returns:
    - `reroutingConfirmation`, `reroutingLog`, `newVehicleId`, `updatedETA`, timestamps
- **Connections:**
  - Output → **Generate Performance Report**
- **Edge cases / failures:**
  - Uses `location.lon` in some places but earlier normalization uses `lng`. Potential coordinate mismatch.
  - This is a simulation; real implementation would need DB queries and dispatch system API calls.

#### Wait for Manual Review
- **Type / role:** `Wait` — pauses workflow for human review (WARNING severity).
- **Configuration:**
  - Resume via webhook
  - Wait limit enabled; `resumeAmount: 24` (units depend on UI; typically hours when configured with “limit wait time”, but verify in node UI)
- **Connections:**
  - Output (after resume) → **Store Incident Records**
- **Edge cases / failures:**
  - If resume webhook is never called, execution will time out/stop after limit.
  - Requires operational process to call the resume URL with review data.

#### Store Incident Records
- **Type / role:** `Postgres` — insert review/incident outcomes into `public.incident_records`.
- **Configuration:**
  - Stores: `timestamp`, `vehicleId`, `description`, `severityLevel`, plus review metadata (`reviewedAt`, `reviewedBy`, `reviewStatus`, `incidentType`)
  - Note: mappings reference `$json.incidentType` and `$json.reviewStatus`, but anomaly output uses `anomalyType` and `requiresManualReview`. Manual review payload must provide the expected fields.
- **Connections:**
  - Output → **Email Operations Team**
- **Edge cases / failures:**
  - Missing fields may insert nulls or violate NOT NULL constraints.

#### Email Operations Team
- **Type / role:** `Email Send` — notifies ops for incident review.
- **Configuration:**
  - To: config `opsTeamEmail`
  - From: placeholder
  - HTML body uses: `vehicleId`, `anomalyType`, `severityLevel`, `description`, `recommendedAction`
- **Connections:**
  - Output → **Generate Performance Report**
- **Edge cases / failures:**
  - SMTP credentials not shown in JSON; node will fail without configured email transport.
  - Placeholder from-address must be valid for the SMTP provider.

#### Generate Performance Report
- **Type / role:** `Postgres` — executes an analytics query over last day of telemetry.
- **Configuration:**
  - SQL (as provided) aggregates by `vehicle_id`, counts events, avg speed, critical count, avg fuel.
  - Potential schema mismatch: earlier inserts use column `vehicleId` (camelCase) but query uses `vehicle_id` (snake_case).
- **Connections:**
  - Output → **Respond to Webhook**
- **Edge cases / failures:**
  - Column naming mismatch can cause SQL error (`column vehicle_id does not exist`).
  - If telemetry table is large, query may be slow without indexes on timestamp/vehicle.

---

### 1.7 Webhook Response

**Overview:**  
Returns a JSON response to the webhook caller.

**Nodes involved:**
- Respond to Webhook

#### Respond to Webhook
- **Type / role:** `Respond to Webhook` — finalizes HTTP response.
- **Configuration:**
  - HTTP 200
  - JSON body:
    - `status`, `message`
    - `recordsProcessed`: `{{ $json.total_events }}`
    - `timestamp`: `{{ $now }}`
- **Connections:** none (terminal)
- **Edge cases / failures:**
  - If the path that reaches this node doesn’t have `total_events` (e.g., from rerouting output), `recordsProcessed` may be null/undefined.
  - If execution started from Schedule Trigger (not webhook), this node is unnecessary and may behave unexpectedly depending on n8n context.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telemetry Webhook Trigger | Webhook | Real-time ingestion endpoint (POST) | — | Workflow Configuration | ## Scheduled Inventory Assessment<br>**Why:** Automated data sync triggers AI-powered quality checks and stock level analysis, identifying critical supply shortages before they impact patient care.<br><br>## How It Works<br>This workflow automates veterinary clinic operations and client communications for animal hospitals and veterinary practices managing appointments, inventory, and patient care. It solves the dual challenge of maintaining medical supply levels while delivering personalized pet care updates and appointment coordination. The system processes scheduled inventory data through AI-powered quality validation and restocking recommendations, then branches into two intelligent pathways: supplier coordination via email for replenishment, and client engagement through personalized appointment reminders, follow-up care instructions, and satisfaction surveys distributed via email and messaging platforms. This eliminates manual inventory tracking, reduces appointment no-shows, and ensures consistent post-visit care communication. |
| Scheduled Data Sync | Schedule Trigger | Periodic trigger for sync runs | — | Merge Trigger Sources | ## Scheduled Inventory Assessment<br>**Why:** Automated data sync triggers AI-powered quality checks and stock level analysis, identifying critical supply shortages before they impact patient care.<br><br>## How It Works<br>This workflow automates veterinary clinic operations and client communications for animal hospitals and veterinary practices managing appointments, inventory, and patient care. It solves the dual challenge of maintaining medical supply levels while delivering personalized pet care updates and appointment coordination. The system processes scheduled inventory data through AI-powered quality validation and restocking recommendations, then branches into two intelligent pathways: supplier coordination via email for replenishment, and client engagement through personalized appointment reminders, follow-up care instructions, and satisfaction surveys distributed via email and messaging platforms. This eliminates manual inventory tracking, reduces appointment no-shows, and ensures consistent post-visit care communication. |
| Workflow Configuration | Set | Central configuration variables | Telemetry Webhook Trigger | Merge Trigger Sources | ## Scheduled Inventory Assessment<br>**Why:** Automated data sync triggers AI-powered quality checks and stock level analysis, identifying critical supply shortages before they impact patient care.<br><br>## How It Works<br>This workflow automates veterinary clinic operations and client communications for animal hospitals and veterinary practices managing appointments, inventory, and patient care. It solves the dual challenge of maintaining medical supply levels while delivering personalized pet care updates and appointment coordination. The system processes scheduled inventory data through AI-powered quality validation and restocking recommendations, then branches into two intelligent pathways: supplier coordination via email for replenishment, and client engagement through personalized appointment reminders, follow-up care instructions, and satisfaction surveys distributed via email and messaging platforms. This eliminates manual inventory tracking, reduces appointment no-shows, and ensures consistent post-visit care communication. |
| Merge Trigger Sources | Merge | Consolidate trigger sources | Workflow Configuration; Scheduled Data Sync | Normalize Telemetry Data | ## Scheduled Inventory Assessment<br>**Why:** Automated data sync triggers AI-powered quality checks and stock level analysis, identifying critical supply shortages before they impact patient care.<br><br>## How It Works<br>This workflow automates veterinary clinic operations and client communications for animal hospitals and veterinary practices managing appointments, inventory, and patient care. It solves the dual challenge of maintaining medical supply levels while delivering personalized pet care updates and appointment coordination. The system processes scheduled inventory data through AI-powered quality validation and restocking recommendations, then branches into two intelligent pathways: supplier coordination via email for replenishment, and client engagement through personalized appointment reminders, follow-up care instructions, and satisfaction surveys distributed via email and messaging platforms. This eliminates manual inventory tracking, reduces appointment no-shows, and ensures consistent post-visit care communication. |
| Normalize Telemetry Data | Code | Normalize telemetry schema | Merge Trigger Sources | Validate Data Quality | ## Setup Steps<br>1. Configure webhook or schedule trigger for veterinary management system inventory data sync<br>2. Add AI model API keys for inventory quality validation<br>3. Connect supplier email system with template configurations for automated purchase orders<br>4. Set up client communication channels with appointment and care instruction templates<br>5. Integrate customer database for pet records and appointment history |
| Validate Data Quality | Code | Validate required fields and ranges | Normalize Telemetry Data | Remove Duplicate Events | ## Setup Steps<br>1. Configure webhook or schedule trigger for veterinary management system inventory data sync<br>2. Add AI model API keys for inventory quality validation<br>3. Connect supplier email system with template configurations for automated purchase orders<br>4. Set up client communication channels with appointment and care instruction templates<br>5. Integrate customer database for pet records and appointment history |
| Remove Duplicate Events | Remove Duplicates | Deduplicate by vehicleId+timestamp | Validate Data Quality | Fetch Traffic & Weather Data; Merge Telemetry with External Data | ## Setup Steps<br>1. Configure webhook or schedule trigger for veterinary management system inventory data sync<br>2. Add AI model API keys for inventory quality validation<br>3. Connect supplier email system with template configurations for automated purchase orders<br>4. Set up client communication channels with appointment and care instruction templates<br>5. Integrate customer database for pet records and appointment history |
| Fetch Traffic & Weather Data | HTTP Request | External traffic/weather enrichment | Remove Duplicate Events | Merge Telemetry with External Data | ## Prerequisites<br>Veterinary practice management software with API/webhook capabilities, AI service API access<br>## Use Cases<br>Multi-location veterinary hospitals coordinating inventory across sites<br>## Customization<br>Modify AI prompts for species-specific care instructions<br>## Benefits<br>Reduces supply management time by 75%, prevents critical medication stockouts |
| Merge Telemetry with External Data | Merge | Join external data into telemetry | Remove Duplicate Events; Fetch Traffic & Weather Data | Route Optimization Agent | ## Prerequisites<br>Veterinary practice management software with API/webhook capabilities, AI service API access<br>## Use Cases<br>Multi-location veterinary hospitals coordinating inventory across sites<br>## Customization<br>Modify AI prompts for species-specific care instructions<br>## Benefits<br>Reduces supply management time by 75%, prevents critical medication stockouts |
| Route Optimization Agent | LangChain Agent | Route optimization + ETA (structured) | Merge Telemetry with External Data | Anomaly Detection Agent | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| OpenAI GPT-4 Model | OpenAI Chat Model | LLM backend for route agent | — | Route Optimization Agent (AI languageModel) | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| Route Optimization Output Parser | Structured Output Parser | Enforce route JSON schema | — | Route Optimization Agent (AI outputParser) | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| Anomaly Detection Agent | LangChain Agent | Detect anomalies + severity (structured) | Route Optimization Agent | Apply Business Rules | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| OpenAI GPT-4 Model for Anomaly | OpenAI Chat Model | LLM backend for anomaly agent | — | Anomaly Detection Agent (AI languageModel) | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| Anomaly Detection Output Parser | Structured Output Parser | Enforce anomaly JSON schema | — | Anomaly Detection Agent (AI outputParser) | ## Dual AI Processing Streams<br>**Why:** Parallel processing generates supplier-specific restocking recommendations and client-personalized appointment communications, optimizing both supply chain and patient experience simultaneously. |
| Apply Business Rules | Code | Priority scoring & rule violations | Anomaly Detection Agent | Route by Severity | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Route by Severity | Switch | Branch by severityLevel | Apply Business Rules | Store Telemetry Records; Wait for Manual Review; Alert Critical Incidents | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Store Telemetry Records | Postgres | Persist telemetry to fleet_telemetry | Route by Severity (Normal Operations) | Notify Dispatchers via Slack | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Notify Dispatchers via Slack | Slack | Normal ops Slack message | Store Telemetry Records | Generate Performance Report | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Alert Critical Incidents | Slack | Critical Slack alert | Route by Severity (Critical Alert) | Prepare Customer Notification | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Prepare Customer Notification | Set | Compose customer message + reroute flag | Alert Critical Incidents | Check if Re-routing Required | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Check if Re-routing Required | IF | Gate rerouting action | Prepare Customer Notification | Trigger Re-routing Action (true) | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Trigger Re-routing Action | Code | Simulated dispatch reassignment | Check if Re-routing Required (true) | Generate Performance Report | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Wait for Manual Review | Wait | Pause WARNING cases for human review | Route by Severity (Needs Review) | Store Incident Records | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Store Incident Records | Postgres | Persist review/incident data | Wait for Manual Review | Email Operations Team | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Email Operations Team | Email Send | Notify ops by email | Store Incident Records | Generate Performance Report | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Generate Performance Report | Postgres | Aggregated daily analytics query | Notify Dispatchers via Slack; Trigger Re-routing Action; Email Operations Team | Respond to Webhook | ## Multi-Channel Distribution<br>**Why:** Coordinated delivery through supplier emails, client appointment reminders, follow-up care instructions, and automated satisfaction surveys ensures comprehensive practice management coverage. |
| Respond to Webhook | Respond to Webhook | Return final JSON HTTP response | Generate Performance Report | — |  |
| Data Ingestion Layer | Sticky Note | Comment block | — | — |  |
| AI Intelligence Layer | Sticky Note | Comment block | — | — |  |
| Notification & Action Layer | Sticky Note | Comment block | — | — |  |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Workflow**
   - Name it as desired (e.g., “Real-Time Fleet Telemetry Intelligence and Dispatch Automation”).
   - (Optional) Keep workflow inactive until credentials and placeholders are configured.

2. **Add Trigger: “Telemetry Webhook Trigger”**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `fleet-telemetry`
   - Authentication: **Header Auth**
     - Create/attach an **HTTP Header Auth** credential (define expected header name/value in credential).
   - Response mode: **Last Node**

3. **Add Trigger: “Scheduled Data Sync”**
   - Node: **Schedule Trigger**
   - Interval: minutes (set the minutes value you want, e.g., every 5 minutes)

4. **Add “Workflow Configuration”**
   - Node: **Set**
   - Add fields (and enable “Include Other Fields”):
     - `trafficApiUrl` (string): your traffic API endpoint base URL
     - `weatherApiUrl` (string): your weather API endpoint base URL (note: not used by current HTTP node unless you extend it)
     - `maxSpeedThreshold` (number): 80
     - `fuelEfficiencyThreshold` (number): 15
     - `idleTimeThreshold` (number): 300
     - `complianceHoursLimit` (number): 11
     - `slackChannelNormal` (string): Slack channel ID for normal notifications
     - `slackChannelCritical` (string): Slack channel ID for critical alerts
     - `opsTeamEmail` (string): operations email

5. **Add “Merge Trigger Sources”**
   - Node: **Merge**
   - Connect:
     - Webhook Trigger → Workflow Configuration → Merge (Input 1)
     - Scheduled Data Sync → Merge (Input 2)
   - In the node UI, ensure the merge mode won’t block when only one trigger fires (commonly “Pass-through” behavior is desired).

6. **Add “Normalize Telemetry Data”**
   - Node: **Code** (Run once for each item)
   - Paste normalization logic that maps varied input keys into:
     - `vehicleId`, `timestamp`, `location.lat/lng`, `speed`, `fuelLevel`, `engineStatus`, `driverId`, `odometer`, `temperature`, `eventType`

7. **Add “Validate Data Quality”**
   - Node: **Code** (per item)
   - Implement checks for required fields + ranges; output `isValid` and `validationErrors`.

8. **Add “Remove Duplicate Events”**
   - Node: **Remove Duplicates**
   - Compare: **Selected Fields**
   - Fields: `vehicleId, timestamp`

9. **Add “Fetch Traffic & Weather Data”**
   - Node: **HTTP Request**
   - Response: JSON
   - Timeout: 10000ms
   - URL expression:  
     `{{ $('Workflow Configuration').first().json.trafficApiUrl }}?lat={{ $json.location.lat }}&lng={{ $json.location.lng }}`
   - Add query parameter `apiKey` with your provider key
   - (Optional) Configure batching (batch size 10)

10. **Add “Merge Telemetry with External Data”**
    - Node: **Merge**
    - Mode: **Combine**
    - Join mode: **Enrich Input 1**
    - Match field: `vehicleId`
    - Connect:
      - Remove Duplicate Events → Merge (Input 1)
      - Fetch Traffic & Weather Data → Merge (Input 2)

11. **Add AI nodes for route optimization**
    1. Node: **OpenAI Chat Model** (`lmChatOpenAi`)
       - Model: `gpt-4o`
       - Temperature: 0.3, Max tokens: 2000
       - Credential: configure **OpenAI API** credential
    2. Node: **Structured Output Parser**
       - Schema includes: optimizedRoute, estimatedETA, delayMinutes, fuelSavings, routeConfidence, recommendations
    3. Node: **Agent**
       - System message: route optimization instructions
       - User text template referencing telemetry + traffic/weather fields
       - Connect AI model and output parser to the agent via AI ports
       - Main input: from Merge Telemetry with External Data

12. **Add AI nodes for anomaly detection**
    1. Node: **OpenAI Chat Model** (second instance)
       - Model: `gpt-4o`, Temperature: 0.2, Max tokens: 1500
       - Same OpenAI credential
    2. Node: **Structured Output Parser**
       - Schema: anomalyDetected, anomalyType, severityLevel, description, recommendedAction, requiresManualReview
    3. Node: **Agent**
       - System message: anomaly detection instructions
       - User text template referencing speed/fuel/engine/driver hours/temp
       - Connect AI model + output parser via AI ports
       - Main input: from Route Optimization Agent

13. **Add “Apply Business Rules”**
    - Node: **Code**
    - Read configuration with `$('Workflow Configuration').first().json`
    - Compute `priority`, `ruleViolations`, `businessRulesPassed`
    - Output enriched item

14. **Add “Route by Severity”**
    - Node: **Switch**
    - Create outputs:
      - NORMAL → “Normal Operations”
      - WARNING → “Needs Review”
      - CRITICAL → “Critical Alert”
      - Fallback → “Unclassified” (optional)
    - Expression field: `$json.severityLevel`

15. **Normal path (NORMAL)**
    1. Add “Store Telemetry Records” (Postgres)
       - Credential: Postgres connection
       - Operation: Insert into `public.fleet_telemetry`
       - Map columns from item (ensure DB types, especially `location` as JSONB)
    2. Add “Notify Dispatchers via Slack”
       - Credential: Slack OAuth2
       - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannelNormal }}`
       - Message template (normal ops)
    3. Connect to “Generate Performance Report”

16. **Warning path (WARNING)**
    1. Add “Wait for Manual Review”
       - Resume: webhook
       - Set wait time limit as desired (and ensure reviewers can call the resume URL)
    2. Add “Store Incident Records” (Postgres)
       - Insert into `public.incident_records`
       - Ensure manual review webhook provides fields like `reviewStatus`, `reviewedBy`, `reviewedAt`, `incidentType` (or adjust mapping to match anomaly fields)
    3. Add “Email Operations Team”
       - Configure SMTP/Email credential (depends on n8n setup)
       - To: `{{ $('Workflow Configuration').first().json.opsTeamEmail }}`
       - From: valid sender address
    4. Connect to “Generate Performance Report”

17. **Critical path (CRITICAL)**
    1. Add “Alert Critical Incidents” (Slack)
       - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannelCritical }}`
    2. Add “Prepare Customer Notification” (Set)
       - Add `customerMessage`, `notificationChannel`, `requiresRerouting` expression
    3. Add “Check if Re-routing Required” (IF)
       - True when `requiresRerouting` is true
    4. Add “Trigger Re-routing Action” (Code)
       - Implement rerouting logic (or keep the simulation)
    5. Connect rerouting output to “Generate Performance Report”
    - (Optional but recommended) Connect the **false** branch (no reroute) to reporting/response so critical non-reroute cases also complete.

18. **Add “Generate Performance Report”**
    - Node: **Postgres → Execute Query**
    - Use your analytics SQL.
    - Ensure column names match your `fleet_telemetry` schema (camelCase vs snake_case).

19. **Add “Respond to Webhook”**
    - Node: **Respond to Webhook**
    - Response: JSON, HTTP 200
    - Body references `total_events`; ensure the preceding node actually outputs it, or adjust.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Several sticky notes describe veterinary clinic inventory/communications use cases (prerequisites, setup steps, distribution rationale). These notes do not match the fleet telemetry implementation and should be updated or removed to avoid confusion. | Applies to: “Data Ingestion Layer”, “AI Intelligence Layer”, “Notification & Action Layer”, “Sticky Note”, “Sticky Note1”, “Sticky Note2”. |
| DB schema consistency issue: inserts map `vehicleId` but report query uses `vehicle_id`. Align naming (either change DB columns or update query/mappings). | Postgres nodes: Store Telemetry Records, Generate Performance Report. |
| Validation (`isValid`) is not enforced. Consider adding an IF node to stop or quarantine invalid telemetry before HTTP/AI/DB operations. | Between Validate Data Quality and Remove Duplicate Events (or right after validation). |
| Merge trigger behavior can block if configured to wait for both inputs. Ensure Merge node is set to a non-blocking strategy for multi-trigger workflows. | Merge Trigger Sources. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.