IoT sensor monitoring with GPT-4o anomaly detection, MQTT & multi-channel alerts

https://n8nworkflows.xyz/workflows/iot-sensor-monitoring-with-gpt-4o-anomaly-detection--mqtt---multi-channel-alerts-11909


# IoT sensor monitoring with GPT-4o anomaly detection, MQTT & multi-channel alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (JSON):** IoT Sensor Data Aggregation with AI-Powered Anomaly Detection  
**User title:** IoT sensor monitoring with GPT-4o anomaly detection, MQTT & multi-channel alerts

This workflow ingests IoT sensor readings (real-time via MQTT and periodic via schedule), normalizes them, deduplicates them with a SHA-256 fingerprint, uses an AI Agent (GPT-4o-mini) to detect anomalies and classify severity, then routes alerts (Slack + Gmail) and archives all processed events to Google Sheets.

### 1.1 Input Reception (multi-entry)
- Real-time: MQTT topic wildcard subscription
- Periodic: Schedule every 15 minutes  
Both feed into a “choose branch” merge so a single downstream pipeline is reused.

### 1.2 Normalization + Metadata Enrichment
Defines threshold/config constants, then parses raw payloads into a consistent schema with timestamps, sensorId, readings, and embedded threshold metadata.

### 1.3 Fingerprinting + Deduplication
Computes a SHA256 hash over sensorId, timestamp, and readings; removes duplicates by hash.

### 1.4 AI Anomaly Detection + Parsing
Sends a structured prompt to an n8n LangChain AI Agent using an OpenAI chat model; parses the agent output into strict JSON and computes alert routing fields.

### 1.5 Alert Routing + Archival
Switches by severity:
- **critical** → Gmail email + Slack critical channel
- **warning** → Slack warning channel
- fallback/other → no alert (just merges)
All alert branches are merged and appended to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (multi-entry)

**Overview:** Accepts sensor data from MQTT in real-time or triggers periodic processing every 15 minutes. Both entry points converge into a single processing path.

**Nodes involved:**
- MQTT Sensor Trigger
- Batch Process Schedule
- Merge Triggers

#### Node: MQTT Sensor Trigger
- **Type/role:** `n8n-nodes-base.mqttTrigger` — listens to MQTT messages.
- **Config choices:**
  - Subscribes to `sensors/+/data` (single-level wildcard `+` captures sensor identifier).
- **Key data expectations:**
  - Typical MQTT trigger items include fields like `topic` and `message` (often string).
- **Connections:**
  - Output → Merge Triggers (input index 0).
- **Failure/edge cases:**
  - Broker auth/TLS errors, connection drops.
  - Message payload not JSON (handled later by parsing node, but may affect downstream fields like temperature/humidity).
- **Version notes:** typeVersion 1.

#### Node: Batch Process Schedule
- **Type/role:** `n8n-nodes-base.scheduleTrigger` — periodic trigger.
- **Config choices:**
  - Runs every 15 minutes.
- **Connections:**
  - Output → Merge Triggers (input index 1).
- **Failure/edge cases:**
  - This trigger does not fetch sensor data by itself; as built, it simply “ticks” the pipeline. Without an upstream data pull, downstream parsing will receive an item with schedule metadata rather than sensor readings.
- **Version notes:** typeVersion 1.2.

#### Node: Merge Triggers
- **Type/role:** `n8n-nodes-base.merge` — consolidates multiple trigger branches.
- **Config choices:**
  - `mode: chooseBranch` (passes through one of the incoming branches depending on which triggered).
- **Connections:**
  - Inputs: MQTT Sensor Trigger (0), Batch Process Schedule (1)
  - Output → Define Sensor Thresholds
- **Failure/edge cases:**
  - If schedule triggers without actual sensor payload retrieval, downstream parsing will likely produce null readings and `sensorId: unknown` (because it tries to infer from `topic` or `sensorId`).
- **Version notes:** typeVersion 3.

---

### Block 2 — Normalization + Metadata Enrichment

**Overview:** Establishes threshold and alert routing configuration, then parses incoming items into a uniform reading schema, including timestamps and embedding thresholds for later AI evaluation.

**Nodes involved:**
- Define Sensor Thresholds
- Parse Sensor Payload

#### Node: Define Sensor Thresholds
- **Type/role:** `n8n-nodes-base.set` — injects constants/config into the execution.
- **Config choices:**
  - Uses **raw JSON output** to define:
    - `thresholds`: min/max + units for temperature, humidity, pressure, co2
    - `alertConfig`: Slack channels + email recipients
- **Key variables:**
  - Downstream nodes reference:
    - `$('Define Sensor Thresholds').first().json.thresholds`
    - `$('Define Sensor Thresholds').first().json.alertConfig.emailRecipients`
- **Connections:**
  - Input ← Merge Triggers
  - Output → Parse Sensor Payload
- **Failure/edge cases:**
  - Invalid JSON in the Set node would break execution.
  - Hard-coded channels and email recipients must match real Slack workspace/channel names and desired recipients.
- **Version notes:** typeVersion 3.4.

#### Node: Parse Sensor Payload
- **Type/role:** `n8n-nodes-base.code` — transforms inbound message(s) into a normalized object.
- **Config choices (interpreted):**
  - Reads all incoming items: `$input.all()`.
  - Pulls thresholds from the Set node: `$('Define Sensor Thresholds').first().json.thresholds`.
  - Tries to parse `item.json.message` as JSON if it’s a string; otherwise uses `item.json`.
  - Produces a normalized `reading` object:
    - `sensorId`: from `sensorData.sensorId` or from `sensorData.topic?.split('/')[1]` or `'unknown'`
    - `location`: default `'Main Facility'`
    - `timestamp` and `metadata.receivedAt`: current time ISO
    - `readings`: temperature/humidity/pressure/co2 with `?? null`
    - `metadata.source`: `item.json.topic` or `'batch'`
    - `metadata.thresholds`: embedded thresholds
- **Connections:**
  - Input ← Define Sensor Thresholds
  - Output → Generate Data Fingerprint
- **Failure/edge cases:**
  - If MQTT message includes `topic` but not inside `message`, `sensorData.topic` might be missing; sensorId inference might fail.
  - If schedule trigger fires, `item.json.message` likely doesn’t exist; the node will fall back to `item.json` (schedule payload), producing null readings.
  - Timestamp is generated at processing time, not device time—this affects deduplication and anomaly context.
- **Version notes:** typeVersion 2.

---

### Block 3 — Fingerprinting + Deduplication

**Overview:** Creates a deterministic hash fingerprint from key fields and removes duplicate readings to reduce repeated alerts and redundant logs.

**Nodes involved:**
- Generate Data Fingerprint
- Remove Duplicate Readings

#### Node: Generate Data Fingerprint
- **Type/role:** `n8n-nodes-base.crypto` — hashes a string to SHA256.
- **Config choices:**
  - Hash type: SHA256
  - Input value expression:
    - `{{ $json.sensorId + '-' + $json.timestamp + '-' + JSON.stringify($json.readings) }}`
  - Stores result in `dataHash`.
- **Connections:**
  - Input ← Parse Sensor Payload
  - Output → Remove Duplicate Readings
- **Failure/edge cases:**
  - Deduplication is weakened by using `timestamp` generated “now”; two identical sensor readings at different processing times will not dedupe.
  - If `readings` contains non-deterministic ordering (unlikely in JS object as created here), stringify would be stable enough in this controlled structure.
- **Version notes:** typeVersion 1.

#### Node: Remove Duplicate Readings
- **Type/role:** `n8n-nodes-base.removeDuplicates` — filters repeated items.
- **Config choices:**
  - `compare: selectedFields`
  - `fieldsToCompare: dataHash`
- **Connections:**
  - Input ← Generate Data Fingerprint
  - Output → AI Anomaly Detector
- **Failure/edge cases:**
  - If all items have distinct timestamps, almost nothing is deduped.
- **Version notes:** typeVersion 1.

---

### Block 4 — AI Anomaly Detection + Parsing

**Overview:** Uses an AI Agent powered by an OpenAI chat model to assess sensor readings against thresholds, then parses the AI response into structured fields used for routing.

**Nodes involved:**
- OpenAI Chat Model
- AI Anomaly Detector
- Parse AI Analysis

#### Node: OpenAI Chat Model
- **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides an LLM to LangChain nodes.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.3` (more deterministic/consistent classification).
- **Connections:**
  - AI output (port `ai_languageModel`) → AI Anomaly Detector
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials.
  - Model availability or quota/rate limits.
- **Version notes:** typeVersion 1.2.

#### Node: AI Anomaly Detector
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent reasoning + response generation.
- **Config choices:**
  - Prompt includes sensor metadata + thresholds and requires **exact JSON** response with fields:
    - `hasAnomaly` (boolean)
    - `severity` (critical/warning/normal)
    - `anomalies` (array of strings)
    - `reasoning` (string)
    - `recommendation` (string)
  - System message enforces role (“IoT monitoring expert”) and “Always respond in valid JSON format.”
- **Key expressions:**
  - Uses many `{{ $json... }}` fields from normalized record and embedded thresholds.
- **Connections:**
  - Main output → Parse AI Analysis
  - Receives model via `ai_languageModel` from OpenAI Chat Model.
- **Failure/edge cases:**
  - Agent may return non-JSON or include extra text; downstream parser tries to extract `{...}` via regex.
  - If readings are null (e.g., schedule-triggered path), the model may produce ambiguous results or misclassify.
- **Version notes:** typeVersion 1.7.

#### Node: Parse AI Analysis
- **Type/role:** `n8n-nodes-base.code` — extracts JSON from AI output and merges with original sensor data.
- **Config choices (interpreted):**
  - Takes first agent output item: `$input.first()`.
  - Retrieves original data from `$('Remove Duplicate Readings').first().json` (note: this means it always uses the *first* deduped item, even if multiple items are processed).
  - Extracts JSON block using regex `/\{[\s\S]*\}/` and `JSON.parse`.
  - On parse failure, sets defaults:
    - `hasAnomaly: false`, `severity: normal`, plus error reasoning.
  - Outputs:
    - `analysis` object
    - `alertLevel: aiAnalysis.severity`
    - `requiresAlert: aiAnalysis.hasAnomaly && aiAnalysis.severity !== 'normal'`
- **Connections:**
  - Output → Route by Severity
- **Failure/edge cases:**
  - **Multi-item bug risk:** using `.first()` from “Remove Duplicate Readings” can mismatch AI output to the wrong sensor reading if multiple items are processed in one execution.
  - Regex extraction can accidentally grab too much if the agent outputs multiple JSON-like blocks.
- **Version notes:** typeVersion 2.

---

### Block 5 — Alert Routing + Archival

**Overview:** Routes events by severity into appropriate notification channels, merges alert outcomes, and archives the event to Google Sheets.

**Nodes involved:**
- Route by Severity
- Send Critical Email
- Slack Critical Alert
- Slack Warning Alert
- Merge Alert Outputs
- Archive to Google Sheets

#### Node: Route by Severity
- **Type/role:** `n8n-nodes-base.switch` — conditional branching.
- **Config choices:**
  - Rule 1 (renamed output “Critical”): `$json.alertLevel == "critical"`
  - Rule 2 (renamed output “Warning”): `$json.alertLevel == "warning"`
  - Fallback output: `extra` (used for “normal” or unexpected severity strings)
- **Connections:**
  - Output 0 (Critical) → Send Critical Email, Slack Critical Alert
  - Output 1 (Warning) → Slack Warning Alert
  - Output 2 (Fallback/extra) → Merge Alert Outputs
- **Failure/edge cases:**
  - Severity strings must exactly match (case-sensitive). “Critical” or “CRITICAL” would go to fallback.
- **Version notes:** typeVersion 3.2.

#### Node: Send Critical Email
- **Type/role:** `n8n-nodes-base.gmail` — sends email for critical alerts.
- **Config choices:**
  - Recipient: from thresholds config (`alertConfig.emailRecipients`)
  - Subject includes sensorId and first anomaly (or default).
  - Body includes readings, reasoning, anomaly list (joined), and recommendation.
- **Connections:**
  - Input ← Route by Severity (Critical)
  - Output → Merge Alert Outputs
- **Failure/edge cases:**
  - Gmail OAuth not configured or insufficient scopes.
  - `anomalies.join(...)` will work if anomalies is array; if AI returns wrong type, expression can fail.
- **Version notes:** typeVersion 2.1.

#### Node: Slack Critical Alert
- **Type/role:** `n8n-nodes-base.slack` — posts critical message to Slack.
- **Config choices:**
  - Channel selected by name: `#iot-critical` (hard-coded in node, not using config object here).
  - Message includes readings, reasoning, recommendation.
- **Connections:**
  - Input ← Route by Severity (Critical)
  - Output → Merge Alert Outputs
- **Failure/edge cases:**
  - Slack credential missing; channel name may not resolve (private channels require bot membership).
- **Version notes:** typeVersion 2.2.

#### Node: Slack Warning Alert
- **Type/role:** `n8n-nodes-base.slack` — posts warning message to Slack.
- **Config choices:**
  - Channel selected by name: `#iot-alerts` (hard-coded in node).
  - Uses first anomaly or fallback string.
- **Connections:**
  - Input ← Route by Severity (Warning)
  - Output → Merge Alert Outputs
- **Failure/edge cases:**
  - Same Slack channel resolution/auth constraints as above.
- **Version notes:** typeVersion 2.2.

#### Node: Merge Alert Outputs
- **Type/role:** `n8n-nodes-base.merge` — consolidates different alert branches before archiving.
- **Config choices:**
  - Default merge settings (no explicit mode shown). In n8n, default is typically “append” behavior for multiple inputs depending on node version/UI.
- **Connections:**
  - Inputs: from Send Critical Email, Slack Critical Alert, Slack Warning Alert, and the switch fallback
  - Output → Archive to Google Sheets
- **Failure/edge cases:**
  - If multiple alert actions fire (critical triggers both email and Slack), this merge can produce multiple items to archive (potentially duplicating sheet rows unless designed).
- **Version notes:** typeVersion 3.

#### Node: Archive to Google Sheets
- **Type/role:** `n8n-nodes-base.googleSheets` — appends a row for logging/history.
- **Config choices:**
  - Operation: Append
  - **documentId and sheetName are empty** in the JSON and must be configured.
- **Connections:**
  - Input ← Merge Alert Outputs
  - Output: none (terminal)
- **Failure/edge cases:**
  - Missing Google Sheets OAuth credentials.
  - Missing documentId/sheetName prevents execution.
  - Column mapping not defined here; depending on n8n UI, it may append JSON fields unpredictably unless explicitly mapped.
- **Version notes:** typeVersion 4.5.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## IoT Sensor Data Aggregation with AI-Powered Anomaly Detection… (Overview, Key Features, Required Credentials, Trigger Options) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / block annotation | — | — | ### Step 1: Data Ingestion… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / block annotation | — | — | ### Step 2: Data Processing… |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / block annotation | — | — | ### Step 3: AI Analysis… |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / block annotation | — | — | ### Step 4: Alert & Archive… |
| MQTT Sensor Trigger | n8n-nodes-base.mqttTrigger | Real-time ingestion via MQTT | — | Merge Triggers | ### Step 1: Data Ingestion… |
| Batch Process Schedule | n8n-nodes-base.scheduleTrigger | Periodic trigger | — | Merge Triggers | ### Step 1: Data Ingestion… |
| Merge Triggers | n8n-nodes-base.merge | Converge trigger branches | MQTT Sensor Trigger; Batch Process Schedule | Define Sensor Thresholds | ### Step 1: Data Ingestion… |
| Define Sensor Thresholds | n8n-nodes-base.set | Inject thresholds + alert config | Merge Triggers | Parse Sensor Payload | ### Step 2: Data Processing… |
| Parse Sensor Payload | n8n-nodes-base.code | Normalize payload, add timestamps/metadata | Define Sensor Thresholds | Generate Data Fingerprint | ### Step 2: Data Processing… |
| Generate Data Fingerprint | n8n-nodes-base.crypto | SHA256 hash for deduplication | Parse Sensor Payload | Remove Duplicate Readings | ### Step 2: Data Processing… |
| Remove Duplicate Readings | n8n-nodes-base.removeDuplicates | Filter repeated items by hash | Generate Data Fingerprint | AI Anomaly Detector | ### Step 2: Data Processing… |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | AI Anomaly Detector (ai_languageModel) | ### Step 3: AI Analysis… |
| AI Anomaly Detector | @n8n/n8n-nodes-langchain.agent | AI reasoning + anomaly classification | Remove Duplicate Readings; OpenAI Chat Model (ai_languageModel) | Parse AI Analysis | ### Step 3: AI Analysis… |
| Parse AI Analysis | n8n-nodes-base.code | Parse/validate agent JSON, compute routing fields | AI Anomaly Detector | Route by Severity | ### Step 3: AI Analysis… |
| Route by Severity | n8n-nodes-base.switch | Branch by critical/warning/other | Parse AI Analysis | Send Critical Email; Slack Critical Alert; Slack Warning Alert; Merge Alert Outputs | ### Step 4: Alert & Archive… |
| Send Critical Email | n8n-nodes-base.gmail | Email notification for critical events | Route by Severity (Critical) | Merge Alert Outputs | ### Step 4: Alert & Archive… |
| Slack Critical Alert | n8n-nodes-base.slack | Slack critical channel post | Route by Severity (Critical) | Merge Alert Outputs | ### Step 4: Alert & Archive… |
| Slack Warning Alert | n8n-nodes-base.slack | Slack warning channel post | Route by Severity (Warning) | Merge Alert Outputs | ### Step 4: Alert & Archive… |
| Merge Alert Outputs | n8n-nodes-base.merge | Consolidate alert/no-alert paths | Send Critical Email; Slack Critical Alert; Slack Warning Alert; Route by Severity (fallback) | Archive to Google Sheets | ### Step 4: Alert & Archive… |
| Archive to Google Sheets | n8n-nodes-base.googleSheets | Append event to sheet (history) | Merge Alert Outputs | — | ### Step 4: Alert & Archive… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: “IoT Sensor Data Aggregation with AI-Powered Anomaly Detection” (or your preferred title).

2) **Add triggers**
- Add **MQTT Trigger** node:
  - Topic: `sensors/+/data`
  - Configure MQTT credentials (broker URL/port, username/password and TLS if needed).
- Add **Schedule Trigger** node:
  - Interval: every **15 minutes**.

3) **Merge trigger paths**
- Add **Merge** node named “Merge Triggers”:
  - Mode: **Choose Branch**
- Connect:
  - MQTT Trigger → Merge Triggers (input 0)
  - Schedule Trigger → Merge Triggers (input 1)

4) **Add configuration constants**
- Add **Set** node named “Define Sensor Thresholds”:
  - Mode: **Raw JSON**
  - Paste/create JSON with:
    - `thresholds.temperature/humidity/pressure/co2` min/max/unit
    - `alertConfig.criticalChannel`, `alertConfig.warningChannel`, `alertConfig.emailRecipients`
- Connect: Merge Triggers → Define Sensor Thresholds

5) **Parse and normalize incoming payload**
- Add **Code** node named “Parse Sensor Payload” with JS that:
  - Tries to parse `item.json.message` if it’s a JSON string
  - Builds a normalized object:
    - `sensorId`, `location`, `timestamp`
    - `readings` object (temperature/humidity/pressure/co2)
    - `metadata.receivedAt`, `metadata.source`, `metadata.thresholds`
- Connect: Define Sensor Thresholds → Parse Sensor Payload

6) **Generate fingerprint**
- Add **Crypto** node named “Generate Data Fingerprint”:
  - Operation/type: SHA256
  - Value expression: combine sensorId, timestamp, and JSON.stringify(readings)
  - Output field: `dataHash`
- Connect: Parse Sensor Payload → Generate Data Fingerprint

7) **Remove duplicates**
- Add **Remove Duplicates** node named “Remove Duplicate Readings”:
  - Compare: Selected fields
  - Field: `dataHash`
- Connect: Generate Data Fingerprint → Remove Duplicate Readings

8) **Configure OpenAI model (LangChain)**
- Add **OpenAI Chat Model** node:
  - Model: `gpt-4o-mini`
  - Temperature: 0.3
  - Configure OpenAI credentials (API key) in n8n.
- This node will connect to the agent via the **ai_languageModel** connection type (not standard main).

9) **Add AI Agent**
- Add **AI Agent** node named “AI Anomaly Detector”:
  - Provide prompt that includes:
    - Sensor ID, location, timestamp
    - Readings plus threshold min/max
  - System message: enforce “Always respond in valid JSON”
  - Require output JSON with keys: `hasAnomaly`, `severity`, `anomalies`, `reasoning`, `recommendation`
- Connect:
  - Remove Duplicate Readings → AI Anomaly Detector (main)
  - OpenAI Chat Model → AI Anomaly Detector (ai_languageModel)

10) **Parse AI output**
- Add **Code** node named “Parse AI Analysis”:
  - Extract JSON from agent text (regex `{...}` then JSON.parse)
  - On failure, set defaults (severity normal, hasAnomaly false, etc.)
  - Output merged object with:
    - `analysis`
    - `alertLevel`
    - `requiresAlert`
- Connect: AI Anomaly Detector → Parse AI Analysis

11) **Route by severity**
- Add **Switch** node named “Route by Severity”:
  - Rule “Critical”: `{{$json.alertLevel}} equals "critical"` (case-sensitive)
  - Rule “Warning”: `{{$json.alertLevel}} equals "warning"`
  - Fallback enabled (for “normal”/other)
- Connect: Parse AI Analysis → Route by Severity

12) **Create alert nodes**
- Add **Gmail** node named “Send Critical Email”:
  - To: expression from config (`Define Sensor Thresholds` → `alertConfig.emailRecipients`)
  - Subject/body: include sensor details + AI analysis fields
  - Configure Gmail OAuth2 credentials in n8n.
- Add **Slack** node named “Slack Critical Alert”:
  - Post to channel `#iot-critical` (or use your config).
  - Configure Slack bot token credentials in n8n; ensure bot is in channel.
- Add **Slack** node named “Slack Warning Alert”:
  - Post to channel `#iot-alerts`.

13) **Connect alert routing**
- Route by Severity (Critical output) → Send Critical Email
- Route by Severity (Critical output) → Slack Critical Alert
- Route by Severity (Warning output) → Slack Warning Alert

14) **Merge alert outputs**
- Add **Merge** node named “Merge Alert Outputs”
- Connect:
  - Send Critical Email → Merge Alert Outputs
  - Slack Critical Alert → Merge Alert Outputs
  - Slack Warning Alert → Merge Alert Outputs
  - Route by Severity fallback output → Merge Alert Outputs (so “normal” still gets archived)

15) **Archive to Google Sheets**
- Add **Google Sheets** node named “Archive to Google Sheets”:
  - Operation: Append
  - Select **Document** (Spreadsheet) and **Sheet**
  - Map columns explicitly (recommended) to fields like:
    - timestamp, sensorId, location, temperature, humidity, co2, alertLevel, anomalies, reasoning, recommendation, dataHash
  - Configure Google Sheets OAuth2 credentials.
- Connect: Merge Alert Outputs → Archive to Google Sheets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Required credentials listed in canvas notes: MQTT Broker, OpenAI API key, Slack Bot Token, Google Sheets OAuth, Gmail OAuth | From workflow sticky overview note |
| Trigger options listed: MQTT real-time, Scheduled batch, Manual webhook trigger | The workflow notes mention webhook, but **no webhook trigger node exists** in the provided JSON (would need to be added if required). |
| Data deduplication uses SHA256 of `sensorId-timestamp-readings` | Because timestamp is generated at parse time, deduplication may not remove repeats across runs unless device timestamp is used. |
| Potential multi-item mismatch in Parse AI Analysis | It uses `$('Remove Duplicate Readings').first()` which can associate AI output with the wrong item when processing multiple readings in a single execution.