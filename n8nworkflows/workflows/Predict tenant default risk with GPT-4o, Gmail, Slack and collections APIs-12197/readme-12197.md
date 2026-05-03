Predict tenant default risk with GPT-4o, Gmail, Slack and collections APIs

https://n8nworkflows.xyz/workflows/predict-tenant-default-risk-with-gpt-4o--gmail--slack-and-collections-apis-12197


# Predict tenant default risk with GPT-4o, Gmail, Slack and collections APIs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Predict tenant default risk with GPT-4o, Gmail, Slack and collections APIs  
**Workflow name (in JSON):** Tenant Risk & Credit Exposure Prediction with Automated Collection Strategy

**Purpose:**  
Runs a daily tenant risk assessment by aggregating tenant-related data (rent payment history, credit bureau information, employment records), sending the merged dataset to a GPT-4o-powered AI agent to compute a structured risk evaluation, then routing notifications and automations based on risk level. High risk triggers Gmail + automated collections; medium risk triggers Slack; outcomes are recorded in Airtable.

**Target use cases:**
- Ongoing monitoring of existing tenants for early default signals
- Objective risk scoring to reduce manual bias
- Automating escalation (notifications + collections initiation) and logging outcomes

### 1.1 Scheduling & Runtime Configuration
Daily trigger and centralized configuration variables (API endpoints, recipients, channel IDs).

### 1.2 Multi-Source Data Aggregation
Fetches credit bureau data, rent payment item/history, and employment records from separate systems.

### 1.3 Data Normalization / Merge
Combines all fetched datasets into a single payload for AI scoring.

### 1.4 AI Risk Scoring (GPT-4o) + Structured Output
LangChain Agent uses GPT-4o with a strict structured output parser to produce consistent fields: riskScore, riskLevel, probability, action, factors, tenantId.

### 1.5 Decisioning & Automated Response
Switch node routes by risk level:
- HIGH → Gmail alert → Collections API (POST)
- MEDIUM → Slack notification
(LOW is configured as a route but not actually connected to a handling path.)

### 1.6 Persistence / Logging
Upserts a record into Airtable for both High and Medium outcomes (and would also for Low if connected).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration

**Overview:**  
Starts the workflow daily at a specific hour and sets all environment-like variables (API URLs, recipients) in one place for reuse via expressions.

**Nodes involved:**
- Daily Risk Assessment Schedule
- Workflow Configuration

#### Node: Daily Risk Assessment Schedule
- **Type / role:** `Schedule Trigger` — workflow entry point, time-based execution.
- **Configuration (interpreted):** Runs daily at **02:00** (triggerAtHour: 2).
- **Connections:**  
  - Output → **Workflow Configuration**
- **Failure / edge cases:**
  - n8n instance timezone affects actual run time.
  - If n8n is down at scheduled time, execution may be skipped depending on instance settings.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes runtime constants.
- **Configuration (interpreted):**
  - Creates these fields (placeholders must be replaced):
    - `rentPaymentApiUrl` (present but not used by current rent node)
    - `creditBureauApiUrl`
    - `employmentApiUrl` (present but not used by BambooHR node)
    - `collectionApiUrl`
    - `tenantDatabaseApiUrl` (present but not used)
    - `alertEmailRecipient`
    - `slackChannel`
  - **Include other fields:** enabled (passes through existing data too).
- **Key expressions used by downstream nodes:**
  - `$('Workflow Configuration').first().json.creditBureauApiUrl`
  - `$('Workflow Configuration').first().json.collectionApiUrl`
  - `$('Workflow Configuration').first().json.alertEmailRecipient`
  - `$('Workflow Configuration').first().json.slackChannel`
- **Connections:**
  - Output → **Fetch Credit Bureau Data**
  - Output → **Get a rent payment item**
  - Output → **Fetch Employment Records**
- **Failure / edge cases:**
  - Placeholder values will cause HTTP failures, invalid channel IDs, or misrouted notifications.
  - If you later run multiple items through the workflow, `first()` can unintentionally lock to the first item’s config; consider using `$json` config fields directly in a single-item workflow or ensuring config is static.

---

### Block 2 — Multi-Source Data Aggregation

**Overview:**  
Collects tenant risk inputs from three sources: credit bureau, rent payments, and employment records. These are later merged.

**Nodes involved:**
- Fetch Credit Bureau Data
- Get a rent payment item
- Fetch Employment Records

#### Node: Fetch Credit Bureau Data
- **Type / role:** `HTTP Request` — fetch credit bureau report data.
- **Configuration (interpreted):**
  - **URL:** from config: `={{ $('Workflow Configuration').first().json.creditBureauApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Default method is **GET** (not explicitly set).
- **Connections:**
  - Input ← Workflow Configuration
  - Output → Merge Tenant Data
- **Failure / edge cases:**
  - Auth is not configured (no auth headers/token shown). Real credit bureau APIs typically require OAuth2/API keys.
  - Missing/invalid URL placeholder will throw request error.
  - Response format differences (array vs object) can break later `.first().json` assumptions in merge.

#### Node: Get a rent payment item
- **Type / role:** `PayPal` node — fetches a payout item (used here as “rent payment history”, though it is not a typical rent ledger).
- **Configuration (interpreted):**
  - Resource: `payoutItem`
  - Operation not specified in the snippet beyond resource; likely defaults to a “get” style operation requiring IDs.
- **Connections:**
  - Input ← Workflow Configuration
  - Output → Merge Tenant Data
- **Failure / edge cases:**
  - PayPal credentials and required identifiers (payout batch/item ID) are not shown; without them, node will fail or return empty.
  - This is not truly “history”; it’s a single payout item unless configured to iterate. For actual rent payment history you may need a different system/API or additional pagination logic.

#### Node: Fetch Employment Records
- **Type / role:** `BambooHR` node — pulls employment/person records.
- **Configuration (interpreted):**
  - Operation: `getAll` (fetch all records visible to the API key/user).
- **Connections:**
  - Input ← Workflow Configuration
  - Output → Merge Tenant Data
- **Failure / edge cases:**
  - Pulling “all” employees can be large; may time out or exceed limits.
  - Tenant ↔ employee matching is not defined; without filtering by tenantId/person identifier, risk scoring may be based on irrelevant records.

---

### Block 3 — Data Merge / Normalization

**Overview:**  
Builds a single JSON structure containing the three upstream datasets so the AI agent can reason over them together.

**Nodes involved:**
- Merge Tenant Data

#### Node: Merge Tenant Data
- **Type / role:** `Set` — constructs a consolidated payload.
- **Configuration (interpreted):**
  - Creates:
    - `rentPaymentData` = `{{ $('Fetch Rent Payment History').first().json }}`
    - `creditBureauData` = `{{ $('Fetch Credit Bureau Data').first().json }}`
    - `employmentData` = `{{ $('Fetch Employment Records').first().json }}`
  - **Include other fields:** enabled.
- **Critical issue (expression mismatch):**
  - The workflow has **no node named “Fetch Rent Payment History”**. The rent node is named **“Get a rent payment item”**.
  - As written, `$('Fetch Rent Payment History')...` will throw an expression error at runtime.
  - Fix: change to `$('Get a rent payment item').first().json` (or rename the PayPal node to match).
- **Connections:**
  - Input ← (three branches) Fetch Credit Bureau Data, Get a rent payment item, Fetch Employment Records
  - Output → Risk Analysis AI Agent
- **Failure / edge cases:**
  - `.first()` hides multi-item scenarios; if upstream returns multiple items, only the first is used.
  - If any upstream node returns no items, `.first()` can be undefined depending on n8n behavior; expect runtime expression errors or empty objects.

---

### Block 4 — AI Risk Scoring (GPT-4o) + Structured Output

**Overview:**  
A LangChain AI agent prompts GPT-4o with merged tenant data, enforces a structured JSON output schema, and produces deterministic fields for routing and downstream automation.

**Nodes involved:**
- Risk Analysis AI Agent
- OpenAI Chat Model
- Risk Score Output Parser

#### Node: Risk Analysis AI Agent
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + output parsing.
- **Configuration (interpreted):**
  - Prompt (user text) injects merged fields:
    - `Rent Payment History: {{ $json.rentPaymentData }}`
    - `Credit Bureau Data: {{ $json.creditBureauData }}`
    - `Employment Records: {{ $json.employmentData }}`
  - System message defines:
    - Risk scoring rubric (0–100), mapping to LOW/MEDIUM/HIGH
    - Default probability (0–100%)
    - Recommended actions: MONITOR / CONTACT / COLLECTION
    - Requires key risk factors list
  - `hasOutputParser: true` (expects a parser node attached)
- **Connections:**
  - Main input ← Merge Tenant Data
  - Main output → Route by Risk Level
  - AI language model input ← OpenAI Chat Model
  - AI output parser input ← Risk Score Output Parser
- **Failure / edge cases:**
  - If merged data is very large, token limits may be reached; consider summarizing upstream data first.
  - If the model returns fields that don’t match schema, output parser will fail.
  - Ensure the model includes `tenantId`; current prompt does not explicitly supply a tenantId unless embedded in upstream JSON.

#### Node: OpenAI Chat Model
- **Type / role:** `OpenAI Chat` language model for the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - Credentials: OpenAI account (API key stored in n8n credentials).
- **Connections:**
  - Output (AI languageModel) → Risk Analysis AI Agent
- **Failure / edge cases:**
  - Invalid API key / quota exceeded.
  - Model availability / org policy restrictions.

#### Node: Risk Score Output Parser
- **Type / role:** `Structured Output Parser` — validates/coerces LLM output into a strict schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `riskScore` (number)
    - `riskLevel` (string: expected LOW/MEDIUM/HIGH)
    - `defaultProbability` (number)
    - `recommendedAction` (string)
    - `keyRiskFactors` (array of strings)
    - `tenantId` (string)
- **Connections:**
  - Output (ai_outputParser) → Risk Analysis AI Agent
- **Failure / edge cases:**
  - If LLM returns riskScore as a string, parsing may fail (depends on parser strictness).
  - Consider adding `required` fields in schema for stricter enforcement (currently schema lists properties but does not explicitly mark them required).

---

### Block 5 — Risk Routing & Response Actions

**Overview:**  
Routes the structured AI result by risk level. High-risk tenants trigger email + collections initiation; medium-risk triggers Slack notification. Low-risk route exists but has no connected handling.

**Nodes involved:**
- Route by Risk Level
- Send High Risk Alert Email
- Trigger Automated Collection
- Send Medium Risk Notification

#### Node: Route by Risk Level
- **Type / role:** `Switch` — conditional branching.
- **Configuration (interpreted):**
  - Three named outputs:
    - “High Risk” if `{{$json.riskLevel}} == "HIGH"`
    - “Medium Risk” if `{{$json.riskLevel}} == "MEDIUM"`
    - “Low Risk” if `{{$json.riskLevel}} == "LOW"`
- **Connections (as wired):**
  - Output 0 (“High Risk”) → Send High Risk Alert Email
  - Output 1 (“Medium Risk”) → Send Medium Risk Notification
  - **No connection for “Low Risk” output** (currently dropped).
- **Failure / edge cases:**
  - If `riskLevel` is missing or misspelled (e.g., “High”), none of the rules match and no downstream nodes run.
  - Consider adding a default/fallback route to log unexpected outputs.

#### Node: Send High Risk Alert Email
- **Type / role:** `Gmail` — sends urgent notification email.
- **Configuration (interpreted):**
  - To: `={{ $('Workflow Configuration').first().json.alertEmailRecipient }}`
  - Subject: `URGENT: High Risk Tenant Alert - {{ $json.tenantId }}`
  - HTML body includes:
    - tenantId, riskScore, defaultProbability, recommendedAction
    - factors rendered via:  
      `{{ $json.keyRiskFactors.map(factor => `<li>${factor}</li>`).join("") }}`
- **Connections:**
  - Input ← Route by Risk Level (High Risk)
  - Output → Trigger Automated Collection
- **Failure / edge cases:**
  - Gmail OAuth2 credential expired or missing scopes.
  - If `keyRiskFactors` is not an array, `.map` will fail.

#### Node: Trigger Automated Collection
- **Type / role:** `HTTP Request` — calls collections system to initiate process.
- **Configuration (interpreted):**
  - POST to `collectionApiUrl` from config
  - JSON body includes:
    - tenantId, riskScore, defaultProbability
    - action: INITIATE_COLLECTION
    - priority: HIGH
    - keyRiskFactors: `{{ JSON.stringify($json.keyRiskFactors) }}`
  - Sends `Content-Type: application/json`
- **Potential payload issue:**
  - `keyRiskFactors` is injected as a **stringified JSON**, not a real JSON array, because the body already is JSON. Many APIs expect an array, not a string.
  - Prefer: `"keyRiskFactors": {{ $json.keyRiskFactors }}` (without `JSON.stringify`) if n8n allows direct array insertion in JSON body.
- **Connections:**
  - Input ← Send High Risk Alert Email
  - Output → Create or update a record
- **Failure / edge cases:**
  - Collections API auth not defined (no token/header).
  - Non-2xx response handling not specified; consider “Continue On Fail” if you still want logging.

#### Node: Send Medium Risk Notification
- **Type / role:** `Slack` — posts message for proactive review.
- **Configuration (interpreted):**
  - OAuth2 authentication
  - Channel: `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message uses markdown and lists key factors:
    - `{{ $json.keyRiskFactors.map(factor => `• ${factor}`).join("\n") }}`
- **Connections:**
  - Input ← Route by Risk Level (Medium Risk)
  - Output → Create or update a record
- **Failure / edge cases:**
  - Slack OAuth token scope missing (e.g., `chat:write`).
  - `slackChannel` must be a valid channel ID, not a name (depending on node settings).

---

### Block 6 — Persistence (Airtable Upsert)

**Overview:**  
Logs the outcome of High and Medium risk processing to Airtable via an upsert operation (intended for tracking, auditing, and follow-up workflows).

**Nodes involved:**
- Create or update a record

#### Node: Create or update a record
- **Type / role:** `Airtable` — upsert a row.
- **Configuration (interpreted):**
  - Operation: `upsert`
  - Base and Table are **not selected** (values empty).
  - Column mapping: auto-map input data (but schema is empty in the JSON snippet).
- **Connections:**
  - Input ← Trigger Automated Collection
  - Input ← Send Medium Risk Notification
  - No outputs connected further.
- **Failure / edge cases:**
  - Will fail until Base/Table are selected and credentials are configured.
  - Upsert requires a defined unique key/matching columns; currently `matchingColumns` is empty, so behavior may not match intent.
  - If you want separate tables for “risk assessments” vs “collections”, split into multiple Airtable nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Risk Assessment Schedule | Schedule Trigger | Daily workflow entry point | — | Workflow Configuration | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Workflow Configuration | Set | Central config variables (URLs, recipients, channel) | Daily Risk Assessment Schedule | Fetch Credit Bureau Data; Get a rent payment item; Fetch Employment Records | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Fetch Credit Bureau Data | HTTP Request | Pull credit bureau dataset | Workflow Configuration | Merge Tenant Data | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Get a rent payment item | PayPal | Pull rent payment-related data (as configured) | Workflow Configuration | Merge Tenant Data | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Fetch Employment Records | BambooHR | Pull employment records | Workflow Configuration | Merge Tenant Data | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Merge Tenant Data | Set | Merge 3 datasets into one payload | Fetch Credit Bureau Data; Get a rent payment item; Fetch Employment Records | Risk Analysis AI Agent | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |
| Risk Analysis AI Agent | LangChain Agent | Risk scoring, classification, recommendations | Merge Tenant Data | Route by Risk Level | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | GPT-4o model provider | — (agent tool connection) | Risk Analysis AI Agent | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |
| Risk Score Output Parser | Structured Output Parser (LangChain) | Enforce structured JSON output | — (agent tool connection) | Risk Analysis AI Agent | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |
| Route by Risk Level | Switch | Branch by HIGH/MEDIUM/LOW | Risk Analysis AI Agent | Send High Risk Alert Email; Send Medium Risk Notification | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |
| Send High Risk Alert Email | Gmail | Email escalation for HIGH risk | Route by Risk Level | Trigger Automated Collection |  |
| Trigger Automated Collection | HTTP Request | Initiate collections process for HIGH risk | Send High Risk Alert Email | Create or update a record |  |
| Send Medium Risk Notification | Slack | Post MEDIUM risk notification to Slack | Route by Risk Level | Create or update a record |  |
| Create or update a record | Airtable | Persist assessment outcome (upsert) | Trigger Automated Collection; Send Medium Risk Notification | — |  |
| Sticky Note | Sticky Note | Comment block | — | — | ## Multi-Source Data Aggregation<br><br>**What:** Fetches rent payment history, credit bureau data, and employment records<br><br>**Why:** Fragmented data creates blind spots in risk assessment. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Prerequisites<br>Payment system API, credit bureau access, employment verification API<br>## Use Cases<br>Rental application screening, existing tenant monitoring<br>## Customization<br>Modify risk scoring criteria, adjust alert thresholds<br>## Benefits<br>Reduces defaults through early detection, eliminates screening bias |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Setup Steps<br>1. Configure payment history, credit bureau, and employment credentials in fetch nodes<br>2. Add OpenAI API key for risk analysis and set Gmail/Slack credentials for alerts<br>3. Customize risk score thresholds |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## How It Works<br>This workflow automates tenant screening by analyzing payment history, credit, and employment data to predict rental risks. Designed for property managers, landlords, and real estate agencies, it solves the challenge of objectively evaluating tenant reliability and preventing payment defaults.The system runs daily assessments, fetching rent payment history, credit bureau reports, and employment records. An AI agent merges this data, calculates risk scores, and routes alerts based on severity. High-risk tenants trigger immediate email notifications for intervention, medium-risk cases post to Slack for monitoring, while low-risk updates save quietly to databases. Automated collection workflows initiate for high-risk cases. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## AI Risk Scoring & Automated Response<br><br>**What:** AI agent analyzes merged data, calculates risk scores, routes alerts by severity<br><br>**Why:** Manual risk evaluation is subjective and slow. AI delivers consistent |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it appropriately (e.g., “Tenant Risk & Credit Exposure Prediction…”).

2) **Add trigger: “Schedule Trigger”**
   - Node name: `Daily Risk Assessment Schedule`
   - Set schedule to run **daily at 02:00** (or your preferred hour/timezone).

3) **Add a “Set” node for configuration**
   - Node name: `Workflow Configuration`
   - Add string fields:
     - `creditBureauApiUrl`
     - `collectionApiUrl`
     - `alertEmailRecipient`
     - `slackChannel`
   - (Optional fields as in the workflow: `rentPaymentApiUrl`, `employmentApiUrl`, `tenantDatabaseApiUrl`)
   - Enable **Include Other Fields**.
   - Connect: **Schedule Trigger → Workflow Configuration**.

4) **Add data fetch nodes (3 branches from config)**

   4.1) **HTTP Request** for credit bureau  
   - Name: `Fetch Credit Bureau Data`  
   - URL: `{{ $('Workflow Configuration').first().json.creditBureauApiUrl }}`
   - Method: GET (or what your API requires)
   - Headers: `Content-Type: application/json`
   - Add authentication (API key/OAuth2) as required by your provider.
   - Connect: **Workflow Configuration → Fetch Credit Bureau Data**

   4.2) **PayPal node** for rent/payment data  
   - Name: `Get a rent payment item`
   - Resource: `Payout Item` (as in JSON)  
   - Configure PayPal credentials and required identifiers (batch/item id) so it returns the intended payment data.
   - Connect: **Workflow Configuration → Get a rent payment item**
   - If you actually need *rent payment history*, replace this with:
     - an HTTP Request to your property management/payment system, or
     - a database query node, plus pagination/filtering by tenantId.

   4.3) **BambooHR node** for employment  
   - Name: `Fetch Employment Records`
   - Operation: `Get All`
   - Configure BambooHR credentials (API key) and any filters if needed.
   - Connect: **Workflow Configuration → Fetch Employment Records**
   - Recommended: filter to the relevant tenant/person to avoid pulling the entire directory.

5) **Add a Set node to merge all datasets**
   - Name: `Merge Tenant Data`
   - Enable **Include Other Fields**
   - Add fields:
     - `creditBureauData` = `{{ $('Fetch Credit Bureau Data').first().json }}`
     - `employmentData` = `{{ $('Fetch Employment Records').first().json }}`
     - `rentPaymentData` = **FIX THIS** to match your node name:
       - `{{ $('Get a rent payment item').first().json }}`
   - Connect each fetch node into `Merge Tenant Data` (3 incoming connections).

6) **Add AI components (LangChain)**

   6.1) **OpenAI Chat Model**
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4o`
   - Configure OpenAI credentials (API key) in n8n.

   6.2) **Structured Output Parser**
   - Name: `Risk Score Output Parser`
   - Schema: object with properties:
     - riskScore (number), riskLevel (string), defaultProbability (number),
       recommendedAction (string), keyRiskFactors (array of string), tenantId (string)

   6.3) **AI Agent**
   - Name: `Risk Analysis AI Agent`
   - Prompt includes the merged fields:
     - rentPaymentData, creditBureauData, employmentData
   - Add the provided system instructions for consistent scoring and actions.
   - Connect AI Agent tool connections:
     - **OpenAI Chat Model → Agent** (language model connection)
     - **Output Parser → Agent** (output parser connection)
   - Connect main flow:
     - **Merge Tenant Data → Risk Analysis AI Agent**

7) **Add routing logic**
   - Add a **Switch** node named `Route by Risk Level`
   - Create 3 rules:
     - `riskLevel equals HIGH` (rename output “High Risk”)
     - `riskLevel equals MEDIUM` (rename output “Medium Risk”)
     - `riskLevel equals LOW` (rename output “Low Risk”)
   - Connect: **Risk Analysis AI Agent → Route by Risk Level**

8) **Add notification + automation nodes**

   8.1) **Gmail node** for HIGH risk  
   - Name: `Send High Risk Alert Email`
   - To: `{{ $('Workflow Configuration').first().json.alertEmailRecipient }}`
   - Subject: `URGENT: High Risk Tenant Alert - {{ $json.tenantId }}`
   - Body: include riskScore, defaultProbability, recommendedAction, keyRiskFactors list
   - Configure Gmail OAuth2 credentials (scopes to send email).
   - Connect: **Route by Risk Level (High Risk) → Send High Risk Alert Email**

   8.2) **HTTP Request** to collections for HIGH risk  
   - Name: `Trigger Automated Collection`
   - Method: POST
   - URL: `{{ $('Workflow Configuration').first().json.collectionApiUrl }}`
   - JSON body: tenantId, riskScore, defaultProbability, action, priority, keyRiskFactors
   - Prefer sending `keyRiskFactors` as an array (not stringified) if the API expects JSON arrays.
   - Configure auth headers/tokens for the collections API.
   - Connect: **Send High Risk Alert Email → Trigger Automated Collection**

   8.3) **Slack node** for MEDIUM risk  
   - Name: `Send Medium Risk Notification`
   - Auth: OAuth2
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Text: include tenantId, score, probability, recommendedAction, and bullet list of factors
   - Connect: **Route by Risk Level (Medium Risk) → Send Medium Risk Notification**

   8.4) (Optional but recommended) Add a handler for **LOW risk**  
   - For parity with the description, connect “Low Risk” to logging (Airtable) or a “No-op”/metrics node.

9) **Add Airtable logging**
   - Node name: `Create or update a record`
   - Operation: `Upsert`
   - Select **Base** and **Table**
   - Configure a unique key for upsert (e.g., `tenantId` + assessment date, or a dedicated `assessmentId`)
   - Map fields from the AI output (riskScore, riskLevel, defaultProbability, recommendedAction, keyRiskFactors, tenantId, timestamp)
   - Configure Airtable credentials (personal access token).
   - Connect:
     - **Trigger Automated Collection → Airtable**
     - **Send Medium Risk Notification → Airtable**
     - (Optional) **Low Risk handler → Airtable**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works:** daily assessments; fetch payment/credit/employment; AI merges and scores; routes alerts; initiates collections for high risk; logs outcomes. | From “How It Works” sticky note |
| **Prerequisites:** payment system API, credit bureau access, employment verification API. | From “Prerequisites” sticky note |
| **Use cases:** rental application screening; existing tenant monitoring. | From “Prerequisites” sticky note |
| **Customization:** modify risk scoring criteria; adjust alert thresholds. | From “Prerequisites” + “Setup Steps” sticky notes |
| **Known build issues to resolve:** Merge node references a non-existent node name (“Fetch Rent Payment History”); Airtable base/table not set; LOW risk path not connected. | Derived from workflow wiring/config |
| **Collections payload nuance:** keyRiskFactors is stringified in JSON body; many APIs expect an array. | Derived from collections HTTP node |

