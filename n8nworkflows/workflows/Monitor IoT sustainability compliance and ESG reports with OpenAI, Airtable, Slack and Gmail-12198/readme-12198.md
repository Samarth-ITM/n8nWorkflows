Monitor IoT sustainability compliance and ESG reports with OpenAI, Airtable, Slack and Gmail

https://n8nworkflows.xyz/workflows/monitor-iot-sustainability-compliance-and-esg-reports-with-openai--airtable--slack-and-gmail-12198


# Monitor IoT sustainability compliance and ESG reports with OpenAI, Airtable, Slack and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *IoT-Integrated Energy & Sustainability Tracker with ESG Reporting*  
**Stated title:** *Monitor IoT sustainability compliance and ESG reports with OpenAI, Airtable, Slack and Gmail*

**Purpose:** Poll IoT sustainability measurements every 15 minutes (energy, water, waste), normalize them, run two AI analyses (compliance + anomaly/predictive risk), send real-time Slack alerts when issues are detected, and aggregate data for an AI-generated ESG report emailed via Gmail.

**Target use cases:** industrial/facility sustainability monitoring, regulatory compliance monitoring, operations maintenance signals, periodic ESG reporting for stakeholders.

### 1.1 Scheduling & Configuration
Runs on a 15-minute schedule, sets workflow-level constants (thresholds, recipients, Slack channel).

### 1.2 Data Acquisition & Normalization
Fetches readings (implemented with an Airtable node, representing the IoT data source) and converts records into a consistent schema.

### 1.3 AI Compliance Evaluation (OpenAI + structured JSON)
Compares readings to thresholds and produces a strict JSON compliance assessment.

### 1.4 AI Anomaly & Predictive Risk (OpenAI + structured JSON)
Detects anomalies and outputs risk score + maintenance recommendations in structured JSON.

### 1.5 Issue Routing (IF + Slack)
If non-compliance or anomaly is detected, prepares alert payload and posts to Slack.

### 1.6 Aggregation & ESG Reporting (Aggregate + OpenAI + Gmail)
Aggregates item data and generates an HTML ESG report, then emails it via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:** Triggers the workflow every 15 minutes and defines key parameters (thresholds, recipients, Slack channel). These values are referenced later via expressions.

**Nodes involved:**
- Every 15 Minutes
- Workflow Configuration

#### Node: Every 15 Minutes
- **Type / role:** `Schedule Trigger` — entry point that starts executions periodically.
- **Configuration:** Runs every **15 minutes**.
- **Connections:** Outputs to **Workflow Configuration**.
- **Type version:** 1.3
- **Failure / edge cases:** n8n instance downtime will skip runs unless you add catch-up logic externally.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central configuration object for thresholds and destinations.
- **Key fields set:**
  - `iotApiUrl` (placeholder; not actually used by subsequent nodes in this workflow as provided)
  - `complianceThresholdEnergy` = 1000
  - `complianceThresholdWater` = 500
  - `complianceThresholdWaste` = 100
  - `alertEmail` (placeholder; not used downstream)
  - `esgReportEmail` (placeholder; not used downstream)
  - `slackChannel` (placeholder; used by Slack node)
- **Notable choice:** `includeOtherFields: true` (keeps incoming fields if any—here mainly irrelevant because it’s triggered by schedule).
- **Connections:** Outputs to **Get IoT Data**.
- **Type version:** 3.4
- **Failure / edge cases:** Placeholder values must be replaced; otherwise Slack channel expression may resolve to invalid/empty.

---

### Block 2 — Data Acquisition & Normalization
**Overview:** Pulls IoT-like data (implemented via Airtable) and normalizes it into consistent keys so downstream AI prompts don’t break.

**Nodes involved:**
- Get IoT Data
- Parse and Structure IoT Data

#### Node: Get IoT Data
- **Type / role:** `Airtable` — used as the data source for sensor readings.
- **Configuration choices (interpreted):**
  - Base and Table are **not set** (empty values). You must select:
    - Airtable Base (where IoT readings are stored)
    - Airtable Table (e.g., `SensorReadings`)
  - Operation is not explicitly shown in JSON; by default this node typically lists/reads records depending on UI selection.
- **Connections:** Input from **Workflow Configuration**; output to **Parse and Structure IoT Data**.
- **Type version:** 2.1
- **Failure / edge cases:**
  - Missing Airtable credentials or missing Base/Table selection.
  - Large tables may require pagination / “Return All” tuning.
  - Data shape may differ from what the Code node expects (`timestamp`, `energyUsage`, etc.).

#### Node: Parse and Structure IoT Data
- **Type / role:** `Code` — transforms raw records into a normalized schema per item.
- **Logic (key behavior):**
  - For each input item:
    - `timestamp`: `rawData.timestamp` else current ISO time
    - `energyUsage`: `rawData.energyUsage` or `rawData.energy` else 0
    - `waterUsage`: `rawData.waterUsage` or `rawData.water` else 0
    - `wasteGenerated`: `rawData.wasteGenerated` or `rawData.waste` else 0
    - `location`: `rawData.location` else `'Unknown'`
    - `sensorId`: `rawData.sensorId` or `rawData.id` else `'Unknown'`
- **Connections:** Input from **Get IoT Data**; output to **Compliance Monitor Agent**.
- **Type version:** 2
- **Failure / edge cases:**
  - If Airtable outputs nested fields (common: `fields` object), this code may not find expected keys. You may need `rawData.fields.timestamp`, etc.
  - Non-numeric values (strings like `"1000 kWh"`) will flow downstream; consider sanitizing.

---

### Block 3 — AI Compliance Evaluation (OpenAI + structured output)
**Overview:** Uses an AI agent to compare each reading against thresholds and emits a strict JSON object describing compliance, violations, severity, and recommendations.

**Nodes involved:**
- Compliance Monitor Agent
- OpenAI Model - Compliance
- Compliance Output Parser

#### Node: OpenAI Model - Compliance
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the LLM for the compliance agent.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI account (`openAiApi`)
- **Connections:** Connected to **Compliance Monitor Agent** via `ai_languageModel`.
- **Type version:** 1.3
- **Failure / edge cases:** API quota, invalid key, model availability changes, network timeouts.

#### Node: Compliance Output Parser
- **Type / role:** `Structured Output Parser` — enforces a JSON schema on the agent’s response.
- **Schema (manual):**
  - `isCompliant` (boolean)
  - `violations` (array of strings)
  - `severity` (string)
  - `recommendations` (array of strings)
- **Connections:** Connected to **Compliance Monitor Agent** via `ai_outputParser`.
- **Type version:** 1.3
- **Failure / edge cases:**
  - Model may output invalid JSON or omit fields; parser will fail the node.
  - Severity values are not enumerated—agent may output unexpected strings; downstream should tolerate.

#### Node: Compliance Monitor Agent
- **Type / role:** `LangChain Agent` — composes the compliance prompt and requests structured output.
- **Prompt content (key expressions):**
  - Uses normalized item fields: `{{$json.energyUsage}}`, `{{$json.waterUsage}}`, `{{$json.wasteGenerated}}`, `{{$json.location}}`, `{{$json.timestamp}}`
  - Pulls thresholds from config node:
    - `{{ $('Workflow Configuration').first().json.complianceThresholdEnergy }}`
    - `{{ $('Workflow Configuration').first().json.complianceThresholdWater }}`
    - `{{ $('Workflow Configuration').first().json.complianceThresholdWaste }}`
- **System message:** Instructs compliance checking, severity classification, recommendations, and to return structured JSON defined by parser.
- **Connections:**
  - Main input from **Parse and Structure IoT Data**
  - `ai_languageModel` from **OpenAI Model - Compliance**
  - `ai_outputParser` from **Compliance Output Parser**
  - Main output to **Anomaly Detection Agent**
- **Type version:** 3.1
- **Failure / edge cases:**
  - If config node doesn’t run or thresholds missing, expressions may resolve null and degrade analysis.
  - If `energyUsage/waterUsage/wasteGenerated` are missing/strings, analysis quality drops.
  - Parser mismatch errors stop the chain.

---

### Block 4 — AI Anomaly & Predictive Risk
**Overview:** A second AI agent evaluates whether the reading looks anomalous, estimates risk, and suggests maintenance actions. Output is expected to be structured JSON.

**Nodes involved:**
- Anomaly Detection Agent
- OpenAI Model - Anomaly
- Anomaly Output Parser

#### Node: OpenAI Model - Anomaly
- **Type / role:** `LangChain Chat Model (OpenAI)` — model powering anomaly agent.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI account (`openAiApi`)
- **Connections:** To **Anomaly Detection Agent** via `ai_languageModel`.
- **Type version:** 1.3
- **Failure / edge cases:** Same as other OpenAI node (quota/timeouts/model deprecation).

#### Node: Anomaly Output Parser
- **Type / role:** `Structured Output Parser` — here configured with an example JSON rather than an explicit schema.
- **Expected shape (from example):**
  - `hasAnomaly` (boolean)
  - `anomalyType` (string)
  - `riskScore` (number)
  - `predictedIssues` (array of strings)
  - `maintenanceRequired` (boolean)
  - `maintenanceRecommendations` (array of strings)
- **Connections:** To **Anomaly Detection Agent** via `ai_outputParser`.
- **Type version:** 1.3
- **Failure / edge cases:**
  - Because it uses an example, strictness depends on node behavior/version; outputs may still vary.
  - If the agent returns different keys, downstream expressions may become undefined.

#### Node: Anomaly Detection Agent
- **Type / role:** `LangChain Agent` — anomaly detection prompt and structured output.
- **Prompt content (key expressions):**
  - Uses current reading fields from item: `{{$json.energyUsage}}`, `{{$json.waterUsage}}`, `{{$json.wasteGenerated}}`, `{{$json.location}}`, `{{$json.timestamp}}`
  - Includes compliance context: `{{$json.isCompliant}}` and `{{$json.violations}}`
- **Important integration detail:** This node receives the output of **Compliance Monitor Agent** in the main chain; therefore it must have access to both the reading and compliance fields. If the compliance agent replaces the item JSON entirely, you may lose original reading fields unless merged (behavior depends on agent node). Validate item shape in executions.
- **Connections:**
  - Main input from **Compliance Monitor Agent**
  - `ai_languageModel` from **OpenAI Model - Anomaly**
  - `ai_outputParser` from **Anomaly Output Parser**
  - Main output to **Check for Issues**
- **Type version:** 3.1
- **Failure / edge cases:**
  - Missing `isCompliant` if compliance parser failed or returned unexpected.
  - Model may not have historical baseline; “anomaly” is heuristic unless you provide prior data.
  - Parser failures stop workflow.

---

### Block 5 — Issue Detection & Slack Alerting
**Overview:** Checks if compliance failed or an anomaly was detected. If yes, prepares a Slack-friendly payload and sends a Slack message.

**Nodes involved:**
- Check for Issues
- Prepare Alert Data
- Send Slack Alert

#### Node: Check for Issues
- **Type / role:** `IF` — routes items based on issue conditions.
- **Condition logic (OR):**
  1) `{{ $('Anomaly Detection Agent').item.json.isCompliant }} == false`
  2) `{{ $('Anomaly Detection Agent').item.json.hasAnomaly }} == true`
- **Connections:**
  - Input from **Anomaly Detection Agent**
  - **True** output → **Prepare Alert Data**
  - **False** output → **Aggregate Daily Data** (note: this means only “no-issue” items are aggregated as built)
- **Type version:** 2.3
- **Failure / edge cases:**
  - If the Anomaly agent output doesn’t contain `isCompliant` or `hasAnomaly`, expressions may evaluate to null/false or error (depending on n8n expression evaluation tolerance).
  - The current design excludes “issue” records from the ESG aggregation path.

#### Node: Prepare Alert Data
- **Type / role:** `Set` — builds a clean alert object for Slack.
- **Fields created (key expressions):**
  - `alertType`: `"Issues Detected"`
  - `location`: `{{$json.location}}`
  - `timestamp`: `{{$json.timestamp}}`
  - `complianceStatus`: `{{$json.isCompliant ? 'Compliant' : 'Non-Compliant'}}`
  - `violations`: `{{$json.violations}}` (stored as string type; may become `a,b` or `[object Object]` depending on source)
  - `anomalyDetected`: `{{$json.hasAnomaly}}`
  - `riskScore`: `{{$json.riskScore}}`
  - `maintenanceNeeded`: `{{$json.maintenanceRequired}}`
  - `recommendations`: `{{ JSON.stringify($json.maintenanceRecommendations) }}`
- **Connections:** Input from **Check for Issues (true path)**; output to **Send Slack Alert**.
- **Type version:** 3.4
- **Failure / edge cases:**
  - If `maintenanceRecommendations` is undefined, `JSON.stringify(undefined)` returns undefined (blank). You may want a fallback.

#### Node: Send Slack Alert
- **Type / role:** `Slack` — posts alert message to a channel.
- **Authentication:** OAuth2 (Slack credential).
- **Channel selection:** `channelId` from config: `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Message body:** Formatted text including location, timestamp, compliance, anomaly, risk, violations, maintenance, recommendations. (Contains a siren emoji in the message text.)
- **Connections:** Input from **Prepare Alert Data**; terminal for alert branch.
- **Type version:** 2.4
- **Failure / edge cases:**
  - Invalid Slack OAuth scopes (needs chat:write and channel access).
  - Wrong channel ID format, empty channel ID, or bot not in channel.
  - Message formatting: arrays/objects in `violations` may render poorly unless stringified.

---

### Block 6 — Aggregation & ESG Report Emailing
**Overview:** Aggregates items and generates an HTML ESG report through AI, then emails it via Gmail. In the provided connections, this happens only for items routed to the “false” branch of the IF.

**Nodes involved:**
- Aggregate Daily Data
- ESG Report Generator
- OpenAI Model - ESG Report
- 📧 Send Email

#### Node: Aggregate Daily Data
- **Type / role:** `Aggregate` — aggregates all item data together.
- **Mode:** `aggregateAllItemData` (combines data across all incoming items).
- **Connections:** Input from **Check for Issues (false path)**; output to **ESG Report Generator**.
- **Type version:** 1
- **Failure / edge cases:**
  - As wired, it does not actually enforce “daily”; it aggregates whatever items arrive in that execution.
  - If only one item arrives, aggregation adds limited value.
  - If you intended daily reporting, a separate daily trigger (or a data store) is needed.

#### Node: OpenAI Model - ESG Report
- **Type / role:** `LangChain Chat Model (OpenAI)` — model for ESG agent.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI account (`openAiApi`)
- **Connections:** To **ESG Report Generator** via `ai_languageModel`.
- **Type version:** 1.3
- **Failure / edge cases:** API quota/timeouts; HTML may be large.

#### Node: ESG Report Generator
- **Type / role:** `LangChain Agent` — converts aggregated sustainability dataset into a professionally formatted ESG report in HTML.
- **Prompt content:**
  - Injects aggregated JSON: `{{ JSON.stringify($json) }}`
- **System message:** Requires sections: Executive Summary, Environmental Impact Metrics, Compliance Status, Trend Analysis, Risk Assessment, Recommendations, Regulatory Compliance Statement; output HTML.
- **Connections:**
  - Main input from **Aggregate Daily Data**
  - `ai_languageModel` from **OpenAI Model - ESG Report**
  - Main output to **📧 Send Email**
- **Type version:** 3.1
- **Failure / edge cases:**
  - Without an output parser, HTML structure may vary and may include markdown or mixed formatting.
  - Very large aggregated JSON may exceed model context limit; consider summarizing or batching.

#### Node: 📧 Send Email
- **Type / role:** `Gmail` — sends the ESG report by email.
- **Configured fields:**
  - Subject: `"Send Report"`
  - Message: `" Report is attached"` (but no attachment is actually configured in this node as provided)
- **Missing configuration:** recipient (`to`) is not set in parameters; also the HTML report is not inserted into the email body.
- **Connections:** Input from **ESG Report Generator**; terminal for report branch.
- **Type version:** 2.1
- **Failure / edge cases:**
  - Gmail OAuth permissions and token expiry.
  - Missing `to` address will prevent sending.
  - If you intend to send HTML, you must map agent output into the email body (and set content type).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 15 Minutes | scheduleTrigger | Periodic workflow trigger | — | Workflow Configuration | ## How It Works … runs every 15 minutes… Dual AI agents… alerts… Daily data aggregates… emailed to stakeholders. / ## Real-Time Compliance & Anomaly Detection **What:** Fetches IoT sensor data every 15 minutes… **Why:** Manual monitoring misses violations… |
| Workflow Configuration | set | Defines thresholds, recipients, Slack channel | Every 15 Minutes | Get IoT Data | ## Setup Steps 1. Configure AirTable credentials… 2. Add OpenAI API keys… configure thresholds 3. Set Gmail/Slack credentials… / ## Real-Time Compliance & Anomaly Detection … |
| Get IoT Data | airtable | Fetch IoT/sensor records (via Airtable) | Workflow Configuration | Parse and Structure IoT Data | ## Setup Steps … / ## Real-Time Compliance & Anomaly Detection … |
| Parse and Structure IoT Data | code | Normalize/clean sensor fields | Get IoT Data | Compliance Monitor Agent | ## Real-Time Compliance & Anomaly Detection … |
| Compliance Monitor Agent | @n8n/n8n-nodes-langchain.agent | AI compliance evaluation vs thresholds | Parse and Structure IoT Data | Anomaly Detection Agent | ## Prerequisites IoT sensor platform API access, OpenAI API key, Gmail/Slack accounts… / ## Real-Time Compliance & Anomaly Detection … |
| OpenAI Model - Compliance | lmChatOpenAi | LLM for compliance agent | — (AI connection) | Compliance Monitor Agent | ## Prerequisites … |
| Compliance Output Parser | outputParserStructured | Enforce JSON output schema for compliance | — (AI connection) | Compliance Monitor Agent | ## Prerequisites … |
| Anomaly Detection Agent | @n8n/n8n-nodes-langchain.agent | AI anomaly detection and risk scoring | Compliance Monitor Agent | Check for Issues | ## Prerequisites … / ## Intelligent Alert Routing & ESG Reporting **What:** Issues trigger email and Slack alerts… **Why:** Alert fatigue… |
| OpenAI Model - Anomaly | lmChatOpenAi | LLM for anomaly agent | — (AI connection) | Anomaly Detection Agent | ## Prerequisites … |
| Anomaly Output Parser | outputParserStructured | Enforce structured anomaly output (example-based) | — (AI connection) | Anomaly Detection Agent | ## Prerequisites … |
| Check for Issues | if | Route based on non-compliance or anomaly | Anomaly Detection Agent | Prepare Alert Data (true), Aggregate Daily Data (false) | ## Intelligent Alert Routing & ESG Reporting … |
| Prepare Alert Data | set | Build Slack alert payload | Check for Issues (true) | Send Slack Alert | ## Intelligent Alert Routing & ESG Reporting … |
| Send Slack Alert | slack | Post alert to Slack channel | Prepare Alert Data | — | ## Intelligent Alert Routing & ESG Reporting … |
| Aggregate Daily Data | aggregate | Combine items for reporting | Check for Issues (false) | ESG Report Generator | ## Intelligent Alert Routing & ESG Reporting … |
| ESG Report Generator | @n8n/n8n-nodes-langchain.agent | Generate HTML ESG report from aggregated data | Aggregate Daily Data | 📧 Send Email | ## Intelligent Alert Routing & ESG Reporting … |
| OpenAI Model - ESG Report | lmChatOpenAi | LLM for ESG report agent | — (AI connection) | ESG Report Generator | ## Intelligent Alert Routing & ESG Reporting … |
| 📧 Send Email | gmail | Email ESG report | ESG Report Generator | — | ## Intelligent Alert Routing & ESG Reporting … |
| Sticky Note | stickyNote | Comment / metadata | — | — | (content is itself the note) ## Prerequisites… Use Cases… Customization… Benefits… |
| Sticky Note1 | stickyNote | Comment / setup steps | — | — | (content is itself the note) ## Setup Steps… |
| Sticky Note2 | stickyNote | Comment / how it works | — | — | (content is itself the note) ## How It Works… |
| Sticky Note4 | stickyNote | Comment / alert routing & ESG | — | — | (content is itself the note) ## Intelligent Alert Routing & ESG Reporting… |
| Sticky Note5 | stickyNote | Comment / realtime detection | — | — | (content is itself the note) ## Real-Time Compliance & Anomaly Detection… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node **Schedule Trigger** named **Every 15 Minutes**.
   2. Set interval to **Every 15 minutes**.

2) **Add configuration node**
   1. Add node **Set** named **Workflow Configuration**.
   2. Add fields:
      - `iotApiUrl` (string) – your endpoint (optional unless you replace Airtable)
      - `complianceThresholdEnergy` (number) = 1000
      - `complianceThresholdWater` (number) = 500
      - `complianceThresholdWaste` (number) = 100
      - `alertEmail` (string) – optional
      - `esgReportEmail` (string) – recommended (you’ll use it in Gmail “To”)
      - `slackChannel` (string) – Slack channel ID (e.g., `C0123...`)
   3. Enable **Include Other Fields**.
   4. Connect **Every 15 Minutes → Workflow Configuration**.

3) **Fetch IoT data (Airtable implementation)**
   1. Add node **Airtable** named **Get IoT Data**.
   2. Configure **Airtable credentials** (Personal Access Token or OAuth, depending on your n8n setup).
   3. Choose **Base** and **Table** containing readings.
   4. Select an operation appropriate to your table (typically **List** records) and consider filtering to “last 15 minutes”.
   5. Connect **Workflow Configuration → Get IoT Data**.

4) **Normalize records**
   1. Add node **Code** named **Parse and Structure IoT Data**.
   2. Paste logic equivalent to:
      - map `timestamp`, `energyUsage`, `waterUsage`, `wasteGenerated`, `location`, `sensorId`
      - default missing values to 0 / Unknown
   3. If Airtable returns `fields`, adjust the code to read from `item.json.fields`.
   4. Connect **Get IoT Data → Parse and Structure IoT Data**.

5) **Compliance AI agent + model + parser**
   1. Add node **OpenAI Chat Model** (LangChain) named **OpenAI Model - Compliance**.
      - Select model: **gpt-4.1-mini**
      - Configure **OpenAI credentials** (API key).
   2. Add node **Structured Output Parser** named **Compliance Output Parser**.
      - Set schema to include: `isCompliant` boolean, `violations` string[], `severity` string, `recommendations` string[].
   3. Add node **AI Agent** named **Compliance Monitor Agent**.
      - Set **Prompt** with the reading fields and thresholds (referencing **Workflow Configuration**).
      - Set **System Message** to instruct compliance check and strict JSON output.
      - Enable/attach the output parser.
   4. Wire:
      - **Parse and Structure IoT Data → Compliance Monitor Agent**
      - **OpenAI Model - Compliance → Compliance Monitor Agent** (AI language model connection)
      - **Compliance Output Parser → Compliance Monitor Agent** (AI output parser connection)

6) **Anomaly AI agent + model + parser**
   1. Add node **OpenAI Chat Model** named **OpenAI Model - Anomaly** (model **gpt-4.1-mini**, same credentials).
   2. Add node **Structured Output Parser** named **Anomaly Output Parser**.
      - Provide an explicit schema (recommended) or at least an example matching your intended keys.
   3. Add node **AI Agent** named **Anomaly Detection Agent**.
      - Prompt includes current reading + compliance status/violations.
      - System message instructs anomalies, risk score, maintenance actions, return structured JSON.
   4. Wire:
      - **Compliance Monitor Agent → Anomaly Detection Agent**
      - **OpenAI Model - Anomaly → Anomaly Detection Agent** (AI model connection)
      - **Anomaly Output Parser → Anomaly Detection Agent** (AI parser connection)

7) **Route issues**
   1. Add node **IF** named **Check for Issues**.
   2. Add OR conditions:
      - `isCompliant` equals `false`
      - `hasAnomaly` equals `true`
      (Prefer referencing `$json.isCompliant` / `$json.hasAnomaly` directly from incoming item to reduce brittle cross-node references.)
   3. Connect **Anomaly Detection Agent → Check for Issues**.

8) **Slack alert path (true branch)**
   1. Add node **Set** named **Prepare Alert Data** to format alert fields (location/timestamp/status/risk/etc.).
   2. Add node **Slack** named **Send Slack Alert**.
      - Configure **Slack OAuth2 credentials** (bot token with `chat:write` and channel access).
      - Set **Channel ID** to an expression referencing `Workflow Configuration.slackChannel`.
      - Compose message text using fields from **Prepare Alert Data**.
   3. Wire **Check for Issues (true) → Prepare Alert Data → Send Slack Alert**.

9) **Aggregation + ESG report path (false branch as currently designed)**
   1. Add node **Aggregate** named **Aggregate Daily Data**.
      - Set mode to **Aggregate All Item Data**.
   2. Add node **OpenAI Chat Model** named **OpenAI Model - ESG Report** (gpt-4.1-mini).
   3. Add node **AI Agent** named **ESG Report Generator**.
      - Prompt: inject aggregated JSON.
      - System message: request HTML ESG report with required sections.
   4. Add node **Gmail** named **📧 Send Email**.
      - Configure **Gmail OAuth2 credentials**.
      - Set **To** to your ESG recipients (e.g., `{{ $('Workflow Configuration').first().json.esgReportEmail }}`).
      - Set subject (e.g., “Daily ESG Report”).
      - Put the agent output into the email body (HTML), and enable HTML if available in node options.
   5. Wire **Check for Issues (false) → Aggregate Daily Data → ESG Report Generator → 📧 Send Email**, and connect the **OpenAI Model - ESG Report** to **ESG Report Generator** via AI model connection.

10) **(Recommended) Make reporting truly daily**
   - Add a separate **daily schedule trigger** for ESG reporting, and query/aggregate a full day’s records (from Airtable or a database), instead of aggregating only per 15-minute execution.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** IoT sensor platform API access, OpenAI API key, Gmail/Slack accounts. **Use Cases:** Manufacturing quality control, environmental compliance monitoring. **Customization:** Modify polling frequency, adjust compliance rules, customize anomaly thresholds. **Benefits:** Continuous compliance assurance, instant anomaly detection. | Sticky note “Prerequisites” |
| **Setup Steps:** 1) Configure AirTable credentials and set 15-minute schedule interval 2) Add OpenAI API keys for compliance and anomaly detection agents, configure regulatory thresholds 3) Set Gmail/Slack credentials for alerts and ESG report distribution | Sticky note “Setup Steps” |
| **How It Works:** Runs every 15 minutes; fetches IoT data; dual AI agents for compliance + anomalies; issues trigger alerts; daily aggregation produces ESG report emailed to stakeholders. | Sticky note “How It Works” |
| **Intelligent Alert Routing & ESG Reporting:** Issues trigger email and Slack alerts for immediate action. **Why:** Alert fatigue from undifferentiated notifications delays responses. | Sticky note “Intelligent Alert Routing & ESG Reporting” |
| **Real-Time Compliance & Anomaly Detection:** Fetches IoT sensor data every 15 minutes; parses/structures readings. **Why:** Manual monitoring misses critical violations and anomalies. | Sticky note “Real-Time Compliance & Anomaly Detection” |