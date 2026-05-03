Monitor revenue tax compliance and auto-correct anomalies with Anthropic, MagicCSV, Gmail and WhatsApp

https://n8nworkflows.xyz/workflows/monitor-revenue-tax-compliance-and-auto-correct-anomalies-with-anthropic--magiccsv--gmail-and-whatsapp-12790


# Monitor revenue tax compliance and auto-correct anomalies with Anthropic, MagicCSV, Gmail and WhatsApp

## 1. Workflow Overview

**Workflow name:** Revenue Tax Compliance Monitoring and Automated Correction System  
**Provided title:** Monitor revenue tax compliance and auto-correct anomalies with Anthropic, MagicCSV, Gmail and WhatsApp

This workflow runs on a schedule to retrieve revenue transactions, classify them for tax purposes, detect compliance anomalies, propose corrections, sync corrections back to an accounting system, and distribute a compliance summary via Gmail and WhatsApp.

### 1.1 Scheduled Data Retrieval
Runs weekly (configurable) and sets key configuration variables used downstream, then fetches revenue data.

### 1.2 AI-Powered Triple Validation (Categorize → Detect anomalies → Propose corrections)
Uses three LangChain Agent nodes (powered by Anthropic chat models) with structured output parsers to:
- categorize transactions for tax,
- identify anomalies and severity,
- generate correction instructions.

### 1.3 Sync + Automated Distribution
Posts the corrections to the accounting software API, formats a reporting summary, emails it to a tax agent, then sends a WhatsApp alert.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduled Data Retrieval

**Overview:** Triggers the workflow on a weekly schedule, injects configuration (API URL/key and notification targets), and retrieves revenue data for analysis.

**Nodes involved:**
- Weekly/Monthly Schedule
- Workflow Configuration
- Fetch Revenue Data

#### Node: **Weekly/Monthly Schedule**
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entry point trigger.
- **Configuration (interpreted):** Weekly schedule: every week on **day 1** (Monday in many locales) at **09:00**.
- **Connections:**
  - **Output →** Workflow Configuration
- **Edge cases / failures:**
  - Timezone differences (instance timezone) can shift the effective run time.
  - If you intended monthly runs, the current rule is **weekly** (despite the node name).

#### Node: **Workflow Configuration**
- **Type / role:** `Set` (`n8n-nodes-base.set`) — centralizes runtime configuration.
- **Configuration (interpreted):**
  - Adds fields (and keeps incoming fields due to `includeOtherFields: true`):
    - `accountingApiUrl` (placeholder)
    - `accountingApiKey` (placeholder)
    - `taxAgentEmail` (placeholder)
    - `whatsappPhoneNumber` (placeholder)
- **Key expressions/variables:** None; static placeholders meant to be replaced.
- **Connections:**
  - **Input ←** Weekly/Monthly Schedule
  - **Output →** Fetch Revenue Data
- **Edge cases / failures:**
  - Leaving placeholders unchanged will cause downstream failures (HTTP auth, invalid email/phone).
  - Storing API keys in node parameters is convenient but less secure than using n8n credentials/environment variables.

#### Node: **Fetch Revenue Data**
- **Type / role:** `n8n` node (`n8n-nodes-base.n8n`) — configured as a data fetcher.
- **Configuration (interpreted):**
  - **Resource:** `execution`
  - **Return all:** enabled
  - No filters set.
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Tax Categorization Agent
- **Important note (integration reality):**
  - Despite the sticky notes mentioning “MagicCSV”, this workflow JSON does **not** contain a MagicCSV node. This node is currently configured to query **executions**, not an accounting platform.
- **Edge cases / failures:**
  - If this node is expected to fetch accounting revenue but instead returns execution data, AI steps will receive unexpected structures.
  - Large result sets (`returnAll: true`) can increase memory usage and slow runs.

---

### Block 1.2 — AI-Powered Triple Validation

**Overview:** Three AI agents process the dataset sequentially, each with a structured output schema. Anthropic chat models supply the language model for each agent.

**Nodes involved:**
- Tax Categorization Agent
- Categorization Output Parser
- Anthropic Model - Categorization
- Anomaly Detection Agent
- Anomaly Detection Output Parser
- Anthropic Model - Anomaly Detection
- Correction Agent
- Correction Output Parser
- Anthropic Model - Correction
- MCP Client Tool

#### Node: **Tax Categorization Agent**
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — tax classification.
- **Configuration (interpreted):**
  - **Input text:** entire incoming JSON: `{{ $json }}`
  - **System message:** instructs model to identify revenue type, tax category, tax rate, special treatment flags, and return structured JSON.
  - **Output parser:** enabled (`hasOutputParser: true`)
- **Key expressions/variables:** `={{ $json }}`
- **Connections:**
  - **Input ←** Fetch Revenue Data
  - **Output →** Anomaly Detection Agent
  - **AI language model input ←** Anthropic Model - Categorization
  - **AI output parser input ←** Categorization Output Parser
- **Edge cases / failures:**
  - If upstream data is an array/list, `{{ $json }}` per item may not represent “each transaction” unless data is properly itemized.
  - Structured output can fail if the model returns invalid JSON or mismatched schema fields.

#### Node: **Categorization Output Parser**
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `transactionId` (string)
    - `revenueType` (string)
    - `taxCategory` (string)
    - `taxRate` (number)
    - `amount` (number)
    - `taxAmount` (number)
    - `specialTreatment` (boolean)
- **Connections:**
  - **Output parser →** Tax Categorization Agent (as its parser)
- **Edge cases / failures:**
  - Missing required properties will cause parse/validation errors.
  - If your real data has multiple transactions, a schema of a *single object* (not an array) may be too limiting.

#### Node: **Anthropic Model - Categorization**
- **Type / role:** Anthropic Chat Model (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — LLM backend.
- **Configuration (interpreted):**
  - **Model:** `claude-sonnet-4-5-20250929` (cached display name “Claude Sonnet 4.5”)
- **Credentials:** Anthropic API account credential.
- **Connections:**
  - **AI language model →** Tax Categorization Agent
- **Edge cases / failures:**
  - Anthropic credential missing/invalid → authentication error.
  - Model availability/renaming can break selection in some environments.

---

#### Node: **Anomaly Detection Agent**
- **Type / role:** LangChain Agent — anomaly detection over categorized output.
- **Configuration (interpreted):**
  - **Input text:** `{{ $json }}`
  - **System message:** detect missing invoices, incorrect categories, unusual rates/calculations, duplicates, missing docs, invoice/payment mismatches; assign severity.
  - **Output parser:** enabled
- **Connections:**
  - **Input ←** Tax Categorization Agent
  - **Output →** Correction Agent
  - **AI language model input ←** Anthropic Model - Anomaly Detection
  - **AI output parser input ←** Anomaly Detection Output Parser
- **Edge cases / failures:**
  - If categorization step outputs single objects rather than a coherent set, anomalies may be incomplete (e.g., duplicate detection needs a collection).
  - Schema requires `anomalies` array; if none found, model should return `anomalies: []` to avoid downstream `.length` issues.

#### Node: **Anomaly Detection Output Parser**
- **Type / role:** Structured Output Parser — validates anomalies format.
- **Configuration (interpreted):**
  - Schema: object with `anomalies` array of objects:
    - `transactionId` (string)
    - `anomalyType` (string)
    - `severity` (string: critical/high/medium/low by instruction)
    - `description` (string)
    - `affectedAmount` (number)
- **Connections:**
  - **Output parser →** Anomaly Detection Agent
- **Edge cases / failures:**
  - Severity is not formally enum-enforced in schema, but downstream humans may rely on the four values.

#### Node: **Anthropic Model - Anomaly Detection**
- **Type / role:** Anthropic Chat Model — LLM backend for anomaly detection agent.
- **Configuration:** same model `claude-sonnet-4-5-20250929`.
- **Credentials:** same Anthropic credential.
- **Connections:**
  - **AI language model →** Anomaly Detection Agent

---

#### Node: **Correction Agent**
- **Type / role:** LangChain Agent — proposes corrections.
- **Configuration (interpreted):**
  - **Input text:** `{{ $json }}`
  - **System message:** produce correction instructions including original transaction ID, corrected values, recalculated tax, notes.
  - **Output parser:** enabled
- **Connections:**
  - **Input ←** Anomaly Detection Agent
  - **Output →** Sync to Accounting Software
  - **AI language model input ←** Anthropic Model - Correction
  - **AI output parser input ←** Correction Output Parser
  - **AI tool input ←** MCP Client Tool (available as a tool to the agent)
- **Edge cases / failures:**
  - If anomalies are empty, the model should return `corrections: []`; otherwise summary counts and sync payload may misbehave.
  - Tooling: MCP tool is connected but not configured with a concrete server/action here; agent may attempt to call it and fail depending on environment.

#### Node: **Correction Output Parser**
- **Type / role:** Structured Output Parser — validates corrections format.
- **Configuration (interpreted):**
  - Schema: object with `corrections` array of objects:
    - `transactionId` (string)
    - `correctionType` (string)
    - `originalValue` (string)
    - `correctedValue` (string)
    - `notes` (string)
- **Connections:**
  - **Output parser →** Correction Agent
- **Edge cases / failures:**
  - `originalValue`/`correctedValue` are strings; if you need numeric corrections, you may want to allow numbers in schema.

#### Node: **Anthropic Model - Correction**
- **Type / role:** Anthropic Chat Model — LLM backend for correction agent.
- **Configuration:** same model selection.
- **Credentials:** Anthropic credential.
- **Connections:**
  - **AI language model →** Correction Agent

#### Node: **MCP Client Tool**
- **Type / role:** MCP Client Tool (`@n8n/n8n-nodes-langchain.mcpClientTool`) — exposes external tools to the agent.
- **Configuration:** no options configured in this JSON.
- **Connections:**
  - **Tool →** Correction Agent (as `ai_tool`)
- **Edge cases / failures:**
  - With no configured MCP server/tool definitions, this may be non-functional or environment-dependent.
  - If the agent prompt/tooling causes attempted tool calls, executions can error.

---

### Block 1.3 — Sync + Automated Distribution

**Overview:** Sends corrections to the accounting system API, compiles a monthly-style summary, emails the tax agent, and sends a WhatsApp alert.

**Nodes involved:**
- Sync to Accounting Software
- Format Compliance Summary
- Send Summary to Tax Agent
- Send WhatsApp Alert

#### Node: **Sync to Accounting Software**
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — pushes correction payload.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `{{ $('Workflow Configuration').first().json.accountingApiUrl }}`
  - **Body:** JSON = current item JSON (`{{ $json }}`), which will be the Correction Agent output (parsed structure).
  - **Headers:**
    - `Authorization: Bearer {{ $('Workflow Configuration').first().json.accountingApiKey }}`
    - `Content-Type: application/json`
- **Connections:**
  - **Input ←** Correction Agent
  - **Output →** Format Compliance Summary
- **Edge cases / failures:**
  - Wrong URL / network issues → timeouts, DNS, 4xx/5xx.
  - If the accounting API expects a different schema than the Correction Output Parser produces, sync will fail.
  - If `accountingApiKey` is empty/placeholder → 401/403.

#### Node: **Format Compliance Summary**
- **Type / role:** Set (`n8n-nodes-base.set`) — creates report fields for notifications.
- **Configuration (interpreted):**
  - Adds:
    - `summaryTitle`: `Tax Compliance Report - {{ $now.format("MMMM YYYY") }}`
    - `reportDate`: ISO timestamp `{{ $now.toISO() }}`
    - `totalTransactionsReviewed`: `{{ $('Tax Categorization Agent').all().length }}`
    - `anomaliesDetected`: `{{ $('Anomaly Detection Agent').first().json.anomalies?.length || 0 }}`
    - `correctionsApplied`: `{{ $('Correction Agent').first().json.corrections?.length || 0 }}`
    - `syncStatus`: `{{ $('Sync to Accounting Software').first().json.status || 'completed' }}`
  - `includeOtherFields: true` (keeps prior payload too).
- **Connections:**
  - **Input ←** Sync to Accounting Software
  - **Output →** Send Summary to Tax Agent
- **Edge cases / failures:**
  - `Tax Categorization Agent`.all().length counts **items**, not necessarily “transactions” if batching differs.
  - Many HTTP APIs don’t return `status` in JSON; `syncStatus` will become `'completed'` even if HTTP call returned an error unless you configure the HTTP node to fail on non-2xx and stop execution.

#### Node: **Send Summary to Tax Agent**
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — email notification.
- **Configuration (interpreted):**
  - **To:** `{{ $('Workflow Configuration').first().json.taxAgentEmail }}`
  - **Subject:** `{{ $json.summaryTitle }}`
  - **HTML message:** includes title/date/counts/status and a note about “attached compliance data” (no attachment is actually configured in this node).
- **Credentials:** Gmail OAuth2 credential (“Gmail account”).
- **Connections:**
  - **Input ←** Format Compliance Summary
  - **Output →** Send WhatsApp Alert
- **Edge cases / failures:**
  - Missing/invalid Gmail OAuth token → auth failure.
  - If you truly need attachments, you must add binary generation + attachment configuration; currently it sends summary only.

#### Node: **Send WhatsApp Alert**
- **Type / role:** WhatsApp (`n8n-nodes-base.whatsApp`) — instant messaging alert.
- **Configuration (interpreted):**
  - **Operation:** send
  - **Recipient:** `{{ $('Workflow Configuration').first().json.whatsappPhoneNumber }}`
  - **Message body:** includes summary title, anomalies, corrections, sync status, and notes that full report is emailed.
- **Credentials:** WhatsApp API credential (“WhatsApp account”).
- **Connections:**
  - **Input ←** Send Summary to Tax Agent
  - **Output:** none (end)
- **Edge cases / failures:**
  - Phone number must be in correct international format; WhatsApp providers often require pre-approved templates depending on account type.
  - WhatsApp API credential/provider-specific limitations (rate limits, sandbox restrictions).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly/Monthly Schedule | scheduleTrigger | Scheduled trigger | — | Workflow Configuration | ## Scheduled Data Retrieval<br>**Why:** Automated weekly/monthly fetching ensures continuous monitoring without manual intervention, maintaining up-to-date tax records. |
| Workflow Configuration | set | Central config variables | Weekly/Monthly Schedule | Fetch Revenue Data | ## Scheduled Data Retrieval<br>**Why:** Automated weekly/monthly fetching ensures continuous monitoring without manual intervention, maintaining up-to-date tax records. |
| Fetch Revenue Data | n8n | Retrieve revenue-like data (currently executions) | Workflow Configuration | Tax Categorization Agent | ## Scheduled Data Retrieval<br>**Why:** Automated weekly/monthly fetching ensures continuous monitoring without manual intervention, maintaining up-to-date tax records. |
| Tax Categorization Agent | langchain.agent | AI categorization of transactions | Fetch Revenue Data | Anomaly Detection Agent | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Categorization Output Parser | outputParserStructured | Enforce categorization JSON schema | — (parser linked to agent) | Tax Categorization Agent (parser) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Anthropic Model - Categorization | lmChatAnthropic | LLM backend for categorization | — | Tax Categorization Agent (model) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Anomaly Detection Agent | langchain.agent | AI detection of tax anomalies | Tax Categorization Agent | Correction Agent | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Anomaly Detection Output Parser | outputParserStructured | Enforce anomalies JSON schema | — (parser linked to agent) | Anomaly Detection Agent (parser) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Anthropic Model - Anomaly Detection | lmChatAnthropic | LLM backend for anomaly detection | — | Anomaly Detection Agent (model) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Correction Agent | langchain.agent | AI proposal of corrections | Anomaly Detection Agent | Sync to Accounting Software | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Correction Output Parser | outputParserStructured | Enforce corrections JSON schema | — (parser linked to agent) | Correction Agent (parser) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Anthropic Model - Correction | lmChatAnthropic | LLM backend for correction | — | Correction Agent (model) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| MCP Client Tool | mcpClientTool | Optional external tool for agent actions | — | Correction Agent (tool) | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Sync to Accounting Software | httpRequest | POST corrections to accounting API | Correction Agent | Format Compliance Summary | ## Automated Distribution<br>**Why:** Simultaneous sync to accounting software and multi-channel notifications (Gmail, WhatsApp) ensure stakeholders receive timely compliance updates. |
| Format Compliance Summary | set | Build summary metrics and title | Sync to Accounting Software | Send Summary to Tax Agent | ## Automated Distribution<br>**Why:** Simultaneous sync to accounting software and multi-channel notifications (Gmail, WhatsApp) ensure stakeholders receive timely compliance updates. |
| Send Summary to Tax Agent | gmail | Email compliance summary | Format Compliance Summary | Send WhatsApp Alert | ## Automated Distribution<br>**Why:** Simultaneous sync to accounting software and multi-channel notifications (Gmail, WhatsApp) ensure stakeholders receive timely compliance updates. |
| Send WhatsApp Alert | whatsApp | WhatsApp alert message | Send Summary to Tax Agent | — | ## Automated Distribution<br>**Why:** Simultaneous sync to accounting software and multi-channel notifications (Gmail, WhatsApp) ensure stakeholders receive timely compliance updates. |
| Sticky Note | stickyNote | Commentary | — | — | ## How It Works<br>This workflow automates tax compliance monitoring and revenue analysis for accounting teams and finance managers handling multi-source income data. It solves the critical problem of manually tracking revenue streams, identifying tax anomalies, and ensuring regulatory compliance across multiple data sources. The system fetches revenue data from accounting software via MagicCSV, processes it through three specialized AI models for categorization, anomaly detection, and compliance verification, then automatically syncs validated results to accounting systems and sends tax summary reports via email and WhatsApp. This eliminates hours of manual review, reduces compliance errors, and provides real-time tax insights. |
| Sticky Note1 | stickyNote | Commentary | — | — | ## Setup Steps<br>1. Configure MagicCSV integration with your accounting software API credentials<br>2. Add Anthropic API key for categorization, anomaly detection, and compliance models<br>3. Connect accounting software webhook/API for bidirectional sync<br>4. Set up Gmail authentication for automated report distribution<br>5. Configure WhatsApp Business API credentials for instant alerts |
| Sticky Note2 | stickyNote | Commentary | — | — | ## Prerequisites<br>Anthropic API access, MagicCSV account, accounting software with API capabilities<br>## Use Cases<br>Multi-entity corporations tracking cross-border revenue, e-commerce businesses with diverse income streams<br>## Customization<br>Modify AI prompts for industry-specific tax rules, add custom anomaly thresholds<br>## Benefits<br>Reduces manual tax review time by 80%, minimizes compliance errors through triple AI validation |
| Sticky Note3 | stickyNote | Commentary | — | — | ## Automated Distribution<br>**Why:** Simultaneous sync to accounting software and multi-channel notifications (Gmail, WhatsApp) ensure stakeholders receive timely compliance updates. |
| Sticky Note4 | stickyNote | Commentary | — | — | ## AI-Powered Triple Validation<br>**Why:** Three-stage AI analysis (categorization, anomaly detection, compliance checking) ensures accurate tax classification and identifies potential issues before submission. |
| Sticky Note5 | stickyNote | Commentary | — | — | ## Scheduled Data Retrieval<br>**Why:** Automated weekly/monthly fetching ensures continuous monitoring without manual intervention, maintaining up-to-date tax records. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Revenue Tax Compliance Monitoring and Automated Correction System*.

2. **Add Trigger: “Schedule Trigger”**  
   - Name: **Weekly/Monthly Schedule**  
   - Configure: Weekly → Day = 1, Hour = 9 (adjust timezone/locale as needed).  
   - Connect → **Workflow Configuration**.

3. **Add “Set” node**  
   - Name: **Workflow Configuration**  
   - Enable “Include Other Fields”.  
   - Add string fields:
     - `accountingApiUrl` = your accounting API endpoint
     - `accountingApiKey` = your API bearer token (or replace with a credential strategy)
     - `taxAgentEmail` = destination email
     - `whatsappPhoneNumber` = destination phone (E.164 format)  
   - Connect → **Fetch Revenue Data**.

4. **Add data retrieval node** (as in the JSON)  
   - Node type: **n8n**  
   - Name: **Fetch Revenue Data**  
   - Resource: **execution**, Return All: **true** (per current JSON).  
   - Connect → **Tax Categorization Agent**.  
   - If your intention is “MagicCSV → accounting revenue”: replace this step with the appropriate MagicCSV/accounting connector and ensure it outputs transaction items.

5. **Add Anthropic chat model for categorization**
   - Node type: **Anthropic Chat Model**  
   - Name: **Anthropic Model - Categorization**  
   - Model: `claude-sonnet-4-5-20250929` (or your preferred available model).  
   - Credentials: configure **Anthropic API** credential in n8n and select it.

6. **Add structured output parser for categorization**
   - Node type: **Structured Output Parser**  
   - Name: **Categorization Output Parser**  
   - Schema type: Manual  
   - Paste/define a schema with fields: `transactionId, revenueType, taxCategory, taxRate, amount, taxAmount, specialTreatment`.

7. **Add LangChain Agent for categorization**
   - Node type: **AI Agent**  
   - Name: **Tax Categorization Agent**  
   - Prompt type: Define  
   - Text: `{{ $json }}`  
   - System message: (use the one from this workflow, tailored to your jurisdictions).  
   - Attach:
     - Language Model connection from **Anthropic Model - Categorization**
     - Output Parser connection from **Categorization Output Parser**  
   - Connect → **Anomaly Detection Agent**.

8. **Repeat for anomaly detection**
   - Add **Anthropic Model - Anomaly Detection** (same credential/model).
   - Add **Anomaly Detection Output Parser** with `anomalies: []` array schema.
   - Add **Anomaly Detection Agent** with system instructions for anomaly detection, attach the model and parser.
   - Connect → **Correction Agent**.

9. **Repeat for correction proposal**
   - Add **Anthropic Model - Correction** (same credential/model).
   - Add **Correction Output Parser** with `corrections: []` array schema.
   - Add **MCP Client Tool** (optional; leave default if not using tools, or configure MCP server/tools if you are).
   - Add **Correction Agent** with correction instructions, attach:
     - model (**Anthropic Model - Correction**)
     - parser (**Correction Output Parser**)
     - tool (**MCP Client Tool**) if required.
   - Connect → **Sync to Accounting Software**.

10. **Add “HTTP Request” node to sync corrections**
    - Name: **Sync to Accounting Software**
    - Method: **POST**
    - URL: `{{ $('Workflow Configuration').first().json.accountingApiUrl }}`
    - Body: JSON = `{{ $json }}`
    - Headers:
      - `Authorization: Bearer {{ $('Workflow Configuration').first().json.accountingApiKey }}`
      - `Content-Type: application/json`
    - Connect → **Format Compliance Summary**.

11. **Add “Set” node to format summary**
    - Name: **Format Compliance Summary**
    - Include Other Fields: enabled
    - Add fields using expressions:
      - `summaryTitle` = `Tax Compliance Report - {{ $now.format("MMMM YYYY") }}`
      - `reportDate` = `{{ $now.toISO() }}`
      - `totalTransactionsReviewed` = `{{ $('Tax Categorization Agent').all().length }}`
      - `anomaliesDetected` = `{{ $('Anomaly Detection Agent').first().json.anomalies?.length || 0 }}`
      - `correctionsApplied` = `{{ $('Correction Agent').first().json.corrections?.length || 0 }}`
      - `syncStatus` = `{{ $('Sync to Accounting Software').first().json.status || 'completed' }}`
    - Connect → **Send Summary to Tax Agent**.

12. **Add Gmail node**
    - Name: **Send Summary to Tax Agent**
    - Operation: Send email
    - To: `{{ $('Workflow Configuration').first().json.taxAgentEmail }}`
    - Subject: `{{ $json.summaryTitle }}`
    - Body: HTML summary (as in workflow)
    - Credentials: create/select **Gmail OAuth2** credential.
    - Connect → **Send WhatsApp Alert**.

13. **Add WhatsApp node**
    - Name: **Send WhatsApp Alert**
    - Operation: Send
    - Recipient: `{{ $('Workflow Configuration').first().json.whatsappPhoneNumber }}`
    - Text: use the provided template referencing summary fields
    - Credentials: configure **WhatsApp API** credential for your provider.
    - End of workflow.

14. **Activate workflow** (optional) once credentials and placeholders are replaced and tested.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates tax compliance monitoring… fetches revenue data … via MagicCSV … then syncs … and sends … Gmail and WhatsApp.” | Sticky note “How It Works” (note: MagicCSV is mentioned but not present as a node in the JSON) |
| Setup steps include MagicCSV integration, Anthropic API key, accounting webhook/API, Gmail auth, WhatsApp Business API credentials. | Sticky note “Setup Steps” |
| Prerequisites: Anthropic API access, MagicCSV account, accounting software with API capabilities. Benefits: reduces manual review time by 80% (claimed). | Sticky note “Prerequisites / Benefits” |
| Rationale: scheduled retrieval; three-stage AI validation; multi-channel distribution. | Sticky notes “Scheduled Data Retrieval”, “AI-Powered Triple Validation”, “Automated Distribution” |
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | User-provided disclaimer |

