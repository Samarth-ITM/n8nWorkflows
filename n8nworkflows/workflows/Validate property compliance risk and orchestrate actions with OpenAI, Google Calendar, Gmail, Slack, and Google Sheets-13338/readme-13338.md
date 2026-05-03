Validate property compliance risk and orchestrate actions with OpenAI, Google Calendar, Gmail, Slack, and Google Sheets

https://n8nworkflows.xyz/workflows/validate-property-compliance-risk-and-orchestrate-actions-with-openai--google-calendar--gmail--slack--and-google-sheets-13338


# Validate property compliance risk and orchestrate actions with OpenAI, Google Calendar, Gmail, Slack, and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow runs every 15 minutes to monitor property conditions by pulling sensor readings and compliance reference data, using OpenAI-based agents to validate compliance/maintenance status, compute a risk score/level, then orchestrate operational actions (calendar scheduling, emails, Slack alerts, audit logging). It also stores validation results and orchestration decisions in n8n Data Tables for traceability.

**Target use cases:**
- Continuous building/asset monitoring (structural, environmental, equipment health)
- Automated compliance checks (occupancy, safety, zoning-like constraints)
- Automated escalation and audit trails for governance/risk operations teams

**Important note about embedded sticky notes:**  
Several sticky notes describe a *medical imaging/radiology* workflow (PACS/DICOM/HIPAA). They do **not** match the actual nodes (property monitoring/compliance). In the table below, those notes are still preserved as requested, but treat them as **out-of-context documentation** unless you intentionally repurpose this workflow for imaging.

### Logical blocks
1.1 **Timed Trigger & Configuration** → sets URLs and thresholds  
1.2 **Data Acquisition** → fetch sensor + compliance DB data  
1.3 **Data Consolidation** → merge datasets  
1.4 **AI Validation (OpenAI Agent)** → produce structured validation findings  
1.5 **Risk Scoring & Persistence** → compute riskScore/riskLevel, store validation result  
1.6 **Routing & Governance Orchestration (OpenAI Agent + Tools)** → decide and execute actions via Calendar/Gmail/Slack/Sheets and sub-agents  
1.7 **Decision Storage, Escalation Check & Report Formatting** → store orchestration decisions, merge, format audit report

---

## 2. Block-by-Block Analysis

### 1.1 Timed Trigger & Configuration

**Overview:**  
Kicks off the workflow on a fixed schedule and defines runtime configuration (API endpoints and numeric thresholds) used later for risk classification.

**Nodes involved:**  
- Schedule Property Monitoring  
- Workflow Configuration

#### Node: Schedule Property Monitoring
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) – entry point, periodic execution.
- **Configuration:** Runs every **15 minutes**.
- **Inputs/outputs:** No input. Output → **Workflow Configuration**
- **Edge cases / failures:** None typical; missed runs can occur if n8n is down.

#### Node: Workflow Configuration
- **Type / role:** `Set` – central config object creation.
- **Configuration choices:** Writes fields while **including other fields** (passes through any inbound JSON, though schedule trigger usually provides none).
- **Key fields set:**
  - `sensorApiUrl` (placeholder)
  - `complianceApiUrl` (placeholder)
  - `criticalRiskThreshold` = 75
  - `highRiskThreshold` = 50
  - `mediumRiskThreshold` = 25
- **Outputs:** → Fetch Property Sensor Data; Fetch Compliance Database
- **Edge cases:**
  - Placeholder URLs not replaced → downstream HTTP errors.
  - Threshold logic assumes ascending thresholds (medium < high < critical).

---

### 1.2 Data Acquisition

**Overview:**  
Fetches two datasets in parallel: live property sensor readings and compliance/reference data.

**Nodes involved:**  
- Fetch Property Sensor Data  
- Fetch Compliance Database

#### Node: Fetch Property Sensor Data
- **Type / role:** `HTTP Request` – pulls sensor payload.
- **Configuration choices:**
  - URL from expression: `{{ $('Workflow Configuration').first().json.sensorApiUrl }}`
  - Sends header `Content-Type: application/json`
- **Connections:** Input ← Workflow Configuration. Output → Combine Property Data (Input 1)
- **Edge cases / failures:**
  - DNS/timeout/5xx from sensor API
  - URL expression fails if config node missing or returns no items
  - Auth not configured (no auth in node); if API needs auth, request will fail

#### Node: Fetch Compliance Database
- **Type / role:** `HTTP Request` – pulls compliance/reference dataset.
- **Configuration choices:**
  - URL from expression: `{{ $('Workflow Configuration').first().json.complianceApiUrl }}`
  - Header `Content-Type: application/json`
- **Connections:** Input ← Workflow Configuration. Output → Combine Property Data (Input 2)
- **Edge cases / failures:** Same as above.

---

### 1.3 Data Consolidation

**Overview:**  
Combines sensor and compliance data into a single enriched item for the validation agent.

**Nodes involved:**  
- Combine Property Data

#### Node: Combine Property Data
- **Type / role:** `Merge` – combines two inputs.
- **Configuration choices:**
  - Mode: **combine**
  - Join mode: **enrichInput1** (input1 is enriched with fields from input2)
- **Connections:** Inputs ← Fetch Property Sensor Data (main 0), Fetch Compliance Database (main 1). Output → Property Validation Agent
- **Edge cases:**
  - If one branch returns 0 items, merge behavior can produce no output (depends on n8n merge semantics).
  - Field collisions: compliance fields may overwrite sensor fields if names overlap (or vice versa depending on merge rules).

---

### 1.4 AI Validation (OpenAI Agent)

**Overview:**  
Uses a LangChain Agent powered by OpenAI to validate property integrity/compliance/maintenance signals and produce a **structured** JSON result.

**Nodes involved:**  
- Property Validation Agent  
- OpenAI Model - Validation Agent  
- Validation Output Parser

#### Node: Property Validation Agent
- **Type / role:** LangChain `Agent` (`@n8n/n8n-nodes-langchain.agent`) – performs reasoning + structured output.
- **Configuration choices:**
  - Prompt text includes the merged JSON: `JSON.stringify($json, null, 2)`
  - Strong system message with validation criteria and severity classes.
  - `hasOutputParser: true` (expects parser node attached).
- **Connections:**
  - Input ← Combine Property Data
  - Output → Calculate Risk Scores
  - AI connections:
    - Language model ← OpenAI Model - Validation Agent
    - Output parser ← Validation Output Parser
- **Edge cases / failures:**
  - Model returns text that does not conform to schema → parser failure.
  - Large sensor payload may exceed model context.
  - The agent is instructed to reference “compliance standards” but these standards must be present in the merged data, otherwise evidence may be weak or hallucinated unless constrained.

#### Node: OpenAI Model - Validation Agent
- **Type / role:** `lmChatOpenAi` – chat model provider for the agent.
- **Configuration:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential “OpenAi account”
- **Connections:** Provides model to Property Validation Agent.
- **Edge cases:** Invalid API key, quota exceeded, timeouts.

#### Node: Validation Output Parser
- **Type / role:** `outputParserStructured` – enforces JSON schema.
- **Schema highlights (required):**
  - `validationStatus`: PASS/FAIL/WARNING
  - `issues[]`: {category, severity, description, evidence, recommendation}
  - `complianceScore`: 0–100
  - `timestamp`: string
  - `additionalProperties: false` (strict)
- **Connections:** Feeds parser into Property Validation Agent.
- **Edge cases:** Any extra field from model output causes failure due to `additionalProperties: false`.

---

### 1.5 Risk Scoring & Persistence

**Overview:**  
Computes a numeric `riskScore` from issue severities, maps it to `riskLevel` using configured thresholds, then stores the validation output and routes to orchestration.

**Nodes involved:**  
- Calculate Risk Scores  
- Store Validation Results  
- Route by Validation Status

#### Node: Calculate Risk Scores
- **Type / role:** `Code` – deterministic risk scoring.
- **Logic (interpreted):**
  - For each validation item:
    - Sum per-issue points: CRITICAL +25, HIGH +15, MEDIUM +8, LOW +3
    - Cap at 100
    - Assign `riskLevel` by comparing to thresholds in **Workflow Configuration**
    - Add `calculatedAt` ISO timestamp
- **Key expressions / variables:**
  - Reads config via: `$('Workflow Configuration').first().json`
- **Connections:** Input ← Property Validation Agent. Output → Store Validation Results; Route by Validation Status
- **Edge cases:**
  - If `issues` missing/not array → treated as no issues (score 0).
  - If Workflow Configuration node isn’t available in execution context (rare, but possible in partial executions) → throws.
  - Severity values outside expected enum → score not incremented for that issue.

#### Node: Store Validation Results
- **Type / role:** `Data Table` – persistent storage for auditability.
- **Configuration:** Auto-map input columns to table columns; table ID is a placeholder.
- **Connections:** Input ← Calculate Risk Scores. No downstream output connected.
- **Edge cases:**
  - Data table ID placeholder not replaced.
  - Schema mismatch/column creation issues depending on n8n Data Table behavior and permissions.

#### Node: Route by Validation Status
- **Type / role:** `Switch` – routes based on risk level.
- **Configuration details:**
  - Despite the name, it routes on: `{{ $json.riskLevel }}`
  - Outputs: CRITICAL/HIGH/MEDIUM/LOW, fallback “extra”
- **Connections:** Input ← Calculate Risk Scores. Output (all paths) → Governance Orchestration Agent (single connected path in JSON)
- **Edge cases / design gap:**
  - All switch outputs are wired to the same downstream node; the workflow does not currently implement different branches per severity.
  - If `riskLevel` is missing, goes to fallback output “extra” (still connected to orchestration via the switch’s main output connection shown).

---

### 1.6 Routing & Governance Orchestration (OpenAI Agent + Tools)

**Overview:**  
A second OpenAI-driven agent decides what actions to take (repairs, inspections, notifications, escalations) and can invoke multiple tools (Calendar/Gmail/Slack/Sheets) plus three specialist sub-agents (repair/compliance/lease).

**Nodes involved:**  
- Governance Orchestration Agent  
- OpenAI Model - Orchestration Agent  
- Orchestration Output Parser  
- Google Calendar Tool - Schedule Repairs  
- Gmail Tool - Send Notifications  
- Slack Tool - Alert Teams  
- Google Sheets Tool - Log Actions  
- Repair Scheduling Agent Tool (+ its model + parser)  
- Compliance Inspection Agent Tool (+ its model + parser)  
- Lease Management Agent Tool (+ its model + parser)

#### Node: Governance Orchestration Agent
- **Type / role:** LangChain `Agent` – decision maker and tool orchestrator.
- **Configuration choices:**
  - Prompt includes full upstream JSON validation/risk data
  - System message defines decision framework by severity and requires structured/auditable output
  - `hasOutputParser: true`
- **Connections:**
  - Input ← Route by Validation Status
  - Output → Check Critical Threshold
  - AI connections:
    - Model ← OpenAI Model - Orchestration Agent
    - Output parser ← Orchestration Output Parser
    - Tools available:
      - Google Calendar Tool - Schedule Repairs
      - Gmail Tool - Send Notifications
      - Slack Tool - Alert Teams
      - Google Sheets Tool - Log Actions
      - Repair Scheduling Agent Tool
      - Compliance Inspection Agent Tool
      - Lease Management Agent Tool
- **Edge cases / failures:**
  - Tool calls depend on `$fromAI()` parameters; if the model doesn’t provide them, tool nodes fail.
  - The agent *describes* severity-based action timelines but the workflow does not enforce SLA timing outside the agent’s reasoning.
  - If upstream data is incomplete, the agent may generate under-specified actions.

#### Node: OpenAI Model - Orchestration Agent
- **Type / role:** OpenAI chat model for orchestration agent.
- **Configuration:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential “OpenAi account”
- **Edge cases:** quota/timeouts/auth errors.

#### Node: Orchestration Output Parser
- **Type / role:** Structured output schema enforcement.
- **Schema highlights (required):**
  - `actionsExecuted[]`: {actionType, tool, details, timestamp}
  - `agentsInvoked[]`: string
  - `escalationLevel`: none/operational/managerial/executive
  - `reasoning`: string
  - `nextReviewDate`: string
  - Strict `additionalProperties: false`
- **Edge cases:** Any extra keys → parser fail.

#### Tool Node: Google Calendar Tool - Schedule Repairs
- **Type / role:** `googleCalendarTool` – calendar event creation as an agent tool.
- **Configuration:**
  - Calendar ID placeholder
  - Uses `$fromAI()` for `startDateTime`, `endDateTime`, `eventTitle`, `eventDescription`
- **Credentials:** Google Calendar OAuth2
- **Edge cases:**
  - Invalid calendar ID, missing scopes, invalid ISO date formats
  - The tool requires times; model might omit them

#### Tool Node: Gmail Tool - Send Notifications
- **Type / role:** `gmailTool` – email sending as an agent tool.
- **Configuration:** `$fromAI()` for recipients/subject/message
- **Credentials:** Gmail OAuth2
- **Edge cases:** rate limits, invalid recipient formatting, missing Gmail scopes.

#### Tool Node: Slack Tool - Alert Teams
- **Type / role:** `slackTool` – Slack message as an agent tool.
- **Configuration:**
  - OAuth2 auth
  - Channel selected via `$fromAI('slackChannel')`
  - Text via `$fromAI('alertMessage')`
- **Credentials:** Slack OAuth2
- **Edge cases:** channel not found, missing permission scopes, invalid channel ID vs name.

#### Tool Node: Google Sheets Tool - Log Actions
- **Type / role:** `googleSheetsTool` – append row(s) for audit log as an agent tool.
- **Configuration:** Operation `append`, Spreadsheet ID + Sheet name placeholders.
- **Credentials:** Google Sheets OAuth2
- **Edge cases:** sheet/tab missing, permissions, row schema mismatch.

---

#### Specialist sub-agent tool: Repair Scheduling Agent Tool (+ model + parser)
- **Type / role:** `agentTool` – callable sub-agent that returns structured repair recommendations.
- **Input:** `$fromAI('repairRequest', ..., 'json')`
- **Model:** OpenAI Model - Repair Agent (`gpt-4.1-mini`)
- **Parser schema:** priority, duration, cost, resources, schedule, contractorType (optional)
- **Edge cases:**
  - If orchestration agent doesn’t provide valid JSON in `repairRequest`, parsing/tool execution fails.
  - Strict schema forbids extra fields.

#### Specialist sub-agent tool: Compliance Inspection Agent Tool (+ model + parser)
- **Type / role:** `agentTool` – returns structured compliance inspection plan.
- **Input:** `$fromAI('complianceRequest', ..., 'json')`
- **Model:** OpenAI Model - Compliance Agent (`gpt-4.1-mini`)
- **Parser schema:** inspectionType, regulatoryFramework, riskLevel, penalties (optional), timeline, preparationSteps[]
- **Edge cases:** same as above; ensure enum values match schema exactly.

#### Specialist sub-agent tool: Lease Management Agent Tool (+ model + parser)
- **Type / role:** `agentTool` – returns tenant/lease communication plan.
- **Input:** `$fromAI('leaseRequest', ..., 'json')`
- **Model:** OpenAI Model - Lease Agent (`gpt-4.1-mini`)
- **Parser schema:** notificationType, affectedTenants[], noticeRequirement, leaseImplications (optional), timeline, actions[]
- **Edge cases:** same; ensure affectedTenants is an array.

---

### 1.7 Decision Storage, Escalation Check & Report Formatting

**Overview:**  
Checks whether escalation is “executive”, stores orchestration decisions, merges action streams, and formats an audit report object.

**Nodes involved:**  
- Check Critical Threshold  
- Store Orchestration Decisions  
- Consolidate All Actions  
- Format Audit Report

#### Node: Check Critical Threshold
- **Type / role:** `IF` – branches based on escalation level.
- **Condition:** `{{ $json.escalationLevel }} == "executive"`
- **Connections:**
  - Input ← Governance Orchestration Agent
  - True branch → Store Orchestration Decisions → Consolidate All Actions (input 0)
  - False branch → Consolidate All Actions (input 1)
- **Edge cases:**
  - If `escalationLevel` missing due to parser failure upstream, this node may evaluate false or error depending on runtime.

#### Node: Store Orchestration Decisions
- **Type / role:** `Data Table` – persists orchestration output.
- **Configuration:** Auto-map; table ID placeholder.
- **Connections:** Input ← Check Critical Threshold (true). Output → Consolidate All Actions
- **Edge cases:** placeholder table ID, permission/column mapping.

#### Node: Consolidate All Actions
- **Type / role:** `Merge` – consolidates both branches.
- **Configuration:** default merge parameters (not explicitly set).
- **Connections:** Inputs ← (true path) Store Orchestration Decisions, (false path) Check Critical Threshold. Output → Format Audit Report
- **Edge cases:**
  - If one input has no items, merge behavior may drop/duplicate depending on default mode (verify in UI; defaults differ by merge version/settings).

#### Node: Format Audit Report
- **Type / role:** `Set` – creates final report fields.
- **Key fields:**
  - `reportTitle`: constant
  - `reportDate`: `{{ $now.toISO() }}`
  - `validationSummary`: `{{ $json.validationStatus }}`
  - `riskLevel`: `{{ $json.riskLevel }}`
  - `riskScore`: `{{ $json.riskScore }}`
  - `actionsExecuted`: stringified JSON `{{ JSON.stringify($json.actionsExecuted, null, 2) }}`
  - `escalationLevel`, `nextReviewDate`
  - `auditTrail`: constant text
- **Connections:** Input ← Consolidate All Actions. No downstream nodes.
- **Edge cases:**
  - If the merged input item does not include `validationStatus/riskLevel/riskScore` (e.g., only orchestration decision item survives), fields may be empty/undefined.
  - Stringifying `actionsExecuted` fails only if it contains circular structures (unlikely).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Property Monitoring | Schedule Trigger | Periodic workflow entry (every 15 min) | — | Workflow Configuration |  |
| Workflow Configuration | Set | Defines API URLs + risk thresholds | Schedule Property Monitoring | Fetch Property Sensor Data; Fetch Compliance Database |  |
| Fetch Property Sensor Data | HTTP Request | Retrieve live sensor data | Workflow Configuration | Combine Property Data |  |
| Fetch Compliance Database | HTTP Request | Retrieve compliance/reference data | Workflow Configuration | Combine Property Data |  |
| Combine Property Data | Merge | Enrich sensor data with compliance data | Fetch Property Sensor Data; Fetch Compliance Database | Property Validation Agent |  |
| Property Validation Agent | LangChain Agent | Validate property integrity/compliance/maintenance | Combine Property Data | Calculate Risk Scores | ## AI Quality Validation<br>**What**: Validation Agent assesses image quality, completeness, and technical adequacy for diagnostic interpretation<br>**Why**: Quality checks prevent misdiagnosis from poor images and trigger automatic re-acquisition |
| OpenAI Model - Validation Agent | OpenAI Chat Model | LLM for validation agent | — (AI language model connection) | Property Validation Agent (ai_languageModel) | ## AI Quality Validation<br>**What**: Validation Agent assesses image quality, completeness, and technical adequacy for diagnostic interpretation<br>**Why**: Quality checks prevent misdiagnosis from poor images and trigger automatic re-acquisition |
| Validation Output Parser | Structured Output Parser | Enforces validation JSON schema | — (AI parser connection) | Property Validation Agent (ai_outputParser) | ## AI Quality Validation<br>**What**: Validation Agent assesses image quality, completeness, and technical adequacy for diagnostic interpretation<br>**Why**: Quality checks prevent misdiagnosis from poor images and trigger automatic re-acquisition |
| Calculate Risk Scores | Code | Compute riskScore/riskLevel from issues + thresholds | Property Validation Agent | Store Validation Results; Route by Validation Status | ## Risk-Based Prioritization<br>**What**: Calculates clinical risk scores, routes by validation status and urgency through multi-path workflows<br>**Why**: Priority queuing ensures critical cases (stroke, trauma, cancer) receive immediate radiologist attention |
| Store Validation Results | Data Table | Persist validation/risk results | Calculate Risk Scores | — | ## Risk-Based Prioritization<br>**What**: Calculates clinical risk scores, routes by validation status and urgency through multi-path workflows<br>**Why**: Priority queuing ensures critical cases (stroke, trauma, cancer) receive immediate radiologist attention |
| Route by Validation Status | Switch | Route by `riskLevel` (CRITICAL/HIGH/MEDIUM/LOW) | Calculate Risk Scores | Governance Orchestration Agent | ## Risk-Based Prioritization<br>**What**: Calculates clinical risk scores, routes by validation status and urgency through multi-path workflows<br>**Why**: Priority queuing ensures critical cases (stroke, trauma, cancer) receive immediate radiologist attention |
| Governance Orchestration Agent | LangChain Agent | Decide + execute actions using tools and sub-agents | Route by Validation Status | Check Critical Threshold | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| OpenAI Model - Orchestration Agent | OpenAI Chat Model | LLM for orchestration agent | — (AI language model connection) | Governance Orchestration Agent (ai_languageModel) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Orchestration Output Parser | Structured Output Parser | Enforces orchestration decision schema | — (AI parser connection) | Governance Orchestration Agent (ai_outputParser) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Google Calendar Tool - Schedule Repairs | Google Calendar Tool | Create repair/inspection events | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Gmail Tool - Send Notifications | Gmail Tool | Send stakeholder notifications | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Slack Tool - Alert Teams | Slack Tool | Send operational alerts | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Google Sheets Tool - Log Actions | Google Sheets Tool | Append audit log rows | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Repair Scheduling Agent Tool | LangChain Agent Tool | Sub-agent: propose repair schedule/cost/resources | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| OpenAI Model - Repair Agent | OpenAI Chat Model | LLM for repair sub-agent | — (AI language model connection) | Repair Scheduling Agent Tool (ai_languageModel) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Repair Agent Output Parser | Structured Output Parser | Enforces repair plan schema | — (AI parser connection) | Repair Scheduling Agent Tool (ai_outputParser) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Compliance Inspection Agent Tool | LangChain Agent Tool | Sub-agent: propose inspection/regulatory plan | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| OpenAI Model - Compliance Agent | OpenAI Chat Model | LLM for compliance sub-agent | — (AI language model connection) | Compliance Inspection Agent Tool (ai_languageModel) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Compliance Agent Output Parser | Structured Output Parser | Enforces inspection plan schema | — (AI parser connection) | Compliance Inspection Agent Tool (ai_outputParser) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Lease Management Agent Tool | LangChain Agent Tool | Sub-agent: tenant/lease notification plan | — (tool called by agent) | Governance Orchestration Agent (ai_tool) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| OpenAI Model - Lease Agent | OpenAI Chat Model | LLM for lease sub-agent | — (AI language model connection) | Lease Management Agent Tool (ai_languageModel) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Lease Agent Output Parser | Structured Output Parser | Enforces lease plan schema | — (AI parser connection) | Lease Management Agent Tool (ai_outputParser) | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |
| Check Critical Threshold | IF | Branch if escalationLevel == executive | Governance Orchestration Agent | Store Orchestration Decisions; Consolidate All Actions |  |
| Store Orchestration Decisions | Data Table | Persist orchestration decisions | Check Critical Threshold (true) | Consolidate All Actions |  |
| Consolidate All Actions | Merge | Merge true/false branches | Store Orchestration Decisions; Check Critical Threshold (false) | Format Audit Report |  |
| Format Audit Report | Set | Produce final audit report object | Consolidate All Actions | — |  |
| Sticky Note | Sticky Note | Comment | — | — | ## Prerequisites<br>PACS/VNA system API access, HIPAA-compliant AI service accounts<br>## Use Cases<br>Emergency radiology triage (stroke, trauma), lung nodule detection and tracking<br>## Customization<br>Modify AI models for modality-specific analysis (CT, MRI, X-ray, ultrasound)<br>## Benefits<br>Reduces diagnosis turnaround time by 60%, improves critical finding detection rates |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Setup Steps<br>1. Connect **imaging trigger** for automatic study notifications<br>2. Configure **PACS/VNA system APIs** with credentials for DICOM image retrieval and metadata access<br>3. Add **AI model API keys** to Validation Agent and specialized diagnostic agents<br>4. Define **risk stratification criteria** in routing logic based on clinical protocols and imaging findings<br>5. Link **Google Calendar API** for radiologist scheduling and case assignment workflows<br>6. Configure **Slack** integration for care team communication and critical finding alerts<br>7. Connect **email system** for patient/referring physician notifications and report distribution |
| Sticky Note2 | Sticky Note | Comment | — | — | ## How It Works<br>This workflow automates medical imaging analysis and diagnostic reporting for radiology departments, imaging centers, and hospital networks managing high patient volumes. Designed for radiologists, medical imaging technicians, and diagnostic coordinators, it solves the challenge of rapidly analyzing imaging studies, prioritizing critical findings, routing cases appropriately, and generating structured reports while maintaining diagnostic accuracy and regulatory compliance. The system triggers on new imaging studies, fetches imagery and metadata, prepares data through AI agents (Validation ensures image quality and completeness), calculates risk scores, routes by validation status and risk level through multiple pathways, deploys specialized AI agents for comprehensive analysis (Orchestration coordinates findings, Google Calendar manages scheduling, Slack Tool enables team communication, Email Actions handles notifications, Water Monitoring tracks contrast protocols, Compliance Validation ensures regulatory adherence, Leave Management coordinates radiologist availability), and generates final diagnostic reports with complete audit trails. Organizations reduce diagnosis turnaround time by 60%, improve critical finding detection rates, ensure consistent reporting standards, and enable radiologists to focus on complex cases requiring expert judgment. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## AI Quality Validation<br>**What**: Validation Agent assesses image quality, completeness, and technical adequacy for diagnostic interpretation<br>**Why**: Quality checks prevent misdiagnosis from poor images and trigger automatic re-acquisition |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Automated Image Acquisition<br>**What**: Trigger captures new imaging studies, fetches DICOM data and patient metadata from PACS systems<br>**Why**: Immediate processing upon study completion eliminates delays |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Risk-Based Prioritization<br>**What**: Calculates clinical risk scores, routes by validation status and urgency through multi-path workflows<br>**Why**: Priority queuing ensures critical cases (stroke, trauma, cancer) receive immediate radiologist attention |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Multi-Agent Diagnostic Coordination<br>**What**: Orchestration Agent coordinates specialized agents for scheduling, communication, compliance, and report generation<br>**Why**: Parallel workflows manage complex diagnostic processes while maintaining regulatory compliance and care coordination |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** (inactive by default).

2) **Add Trigger**
   1. Add **Schedule Trigger** named **“Schedule Property Monitoring”**.  
   2. Set interval: **Every 15 minutes**.

3) **Add configuration node**
   1. Add **Set** named **“Workflow Configuration”**.  
   2. Add fields:
      - `sensorApiUrl` (string): your sensor API endpoint  
      - `complianceApiUrl` (string): your compliance/reference API endpoint  
      - `criticalRiskThreshold` (number): 75  
      - `highRiskThreshold` (number): 50  
      - `mediumRiskThreshold` (number): 25  
   3. Enable **Include Other Fields**.
   4. Connect: **Schedule Property Monitoring → Workflow Configuration**.

4) **Add parallel HTTP fetches**
   1. Add **HTTP Request** named **“Fetch Property Sensor Data”**:
      - URL: `={{ $('Workflow Configuration').first().json.sensorApiUrl }}`
      - Header: `Content-Type: application/json`
      - Configure authentication if your API requires it (e.g., Header Auth/Bearer token).
   2. Add **HTTP Request** named **“Fetch Compliance Database”**:
      - URL: `={{ $('Workflow Configuration').first().json.complianceApiUrl }}`
      - Header: `Content-Type: application/json`
   3. Connect: **Workflow Configuration → Fetch Property Sensor Data** and **Workflow Configuration → Fetch Compliance Database**.

5) **Merge datasets**
   1. Add **Merge** named **“Combine Property Data”**:
      - Mode: **Combine**
      - Join mode: **Enrich Input 1**
   2. Connect:
      - **Fetch Property Sensor Data → Combine Property Data (Input 1)**
      - **Fetch Compliance Database → Combine Property Data (Input 2)**

6) **Add Validation Agent (LangChain)**
   1. Add **AI Agent** (`@n8n/n8n-nodes-langchain.agent`) named **“Property Validation Agent”**.
   2. Set **Prompt** (Define mode) to embed the merged JSON and request validation.
   3. Set the **System Message** to your property validation policy (you can reuse the one in the workflow).
   4. Add **OpenAI Chat Model** node named **“OpenAI Model - Validation Agent”**:
      - Model: **gpt-4.1-mini**
      - Credentials: configure **OpenAI API** in n8n credentials.
   5. Add **Structured Output Parser** named **“Validation Output Parser”**:
      - Schema: the validation schema (PASS/FAIL/WARNING, issues[], complianceScore, timestamp)
      - Ensure `additionalProperties` is false if you want strict parsing.
   6. Wire AI connections:
      - Model node → Validation Agent (AI language model connection)
      - Output Parser → Validation Agent (AI output parser connection)
   7. Connect: **Combine Property Data → Property Validation Agent**

7) **Compute risk score/level**
   1. Add **Code** node named **“Calculate Risk Scores”**.
   2. Paste logic to:
      - Sum points by severity
      - Cap at 100
      - Compare against thresholds from `Workflow Configuration`
      - Append `riskScore`, `riskLevel`, `calculatedAt`
   3. Connect: **Property Validation Agent → Calculate Risk Scores**

8) **Persist validation results**
   1. Add **Data Table** node named **“Store Validation Results”**.
   2. Set Data Table ID to your created table (create one in n8n UI first).
   3. Keep auto-mapping unless you need strict columns.
   4. Connect: **Calculate Risk Scores → Store Validation Results**

9) **Route by risk level**
   1. Add **Switch** node named **“Route by Validation Status”** (optional rename to “Route by Risk Level” for clarity).
   2. Add four rules on `={{ $json.riskLevel }}` equals `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`. Enable fallback output.
   3. Connect: **Calculate Risk Scores → Route by Validation Status**

10) **Add Governance Orchestration Agent**
   1. Add **AI Agent** named **“Governance Orchestration Agent”** with a system message describing the decision framework and required audit output.
   2. Add **OpenAI Chat Model** named **“OpenAI Model - Orchestration Agent”** (gpt-4.1-mini, OpenAI credential).
   3. Add **Structured Output Parser** named **“Orchestration Output Parser”** with required fields:
      - actionsExecuted[], agentsInvoked[], escalationLevel, reasoning, nextReviewDate
   4. Connect:
      - **Route by Validation Status → Governance Orchestration Agent**
      - Model → Agent (AI model connection)
      - Parser → Agent (AI parser connection)

11) **Add tool integrations (callable by the orchestration agent)**
   1. Add **Google Calendar Tool** node:
      - Calendar credential: Google Calendar OAuth2
      - Calendar ID: choose your calendar
      - Fields use `$fromAI()` for start/end/title/description
   2. Add **Gmail Tool** node:
      - Gmail OAuth2 credential
      - To/Subject/Message use `$fromAI()`
   3. Add **Slack Tool** node:
      - Slack OAuth2 credential
      - Channel and message use `$fromAI()`
   4. Add **Google Sheets Tool** node:
      - Sheets OAuth2 credential
      - Operation: append
      - Spreadsheet ID + Sheet name set to your audit log sheet
   5. Connect each tool node to **Governance Orchestration Agent** via **AI Tool** connections (not main workflow lines).

12) **Add specialist sub-agents (as tools)**
   - For each of the three:
     1. Add an **Agent Tool** node (`@n8n/n8n-nodes-langchain.agentTool`)
     2. Attach an **OpenAI Chat Model** (gpt-4.1-mini)
     3. Attach a **Structured Output Parser** with the relevant schema
     4. Ensure the Agent Tool node uses `$fromAI(..., 'json')` for its input
     5. Connect the Agent Tool to **Governance Orchestration Agent** via **AI Tool** connection

13) **Escalation check + store decisions**
   1. Add **IF** node named **“Check Critical Threshold”**:
      - Condition: `={{ $json.escalationLevel }}` equals `executive`
   2. Add **Data Table** node named **“Store Orchestration Decisions”** pointing to your decisions table.
   3. Connect:
      - **Governance Orchestration Agent → Check Critical Threshold**
      - **Check Critical Threshold (true) → Store Orchestration Decisions**

14) **Consolidate and format final report**
   1. Add **Merge** node named **“Consolidate All Actions”**.
      - Connect:
        - **Store Orchestration Decisions → Consolidate All Actions (Input 1)**
        - **Check Critical Threshold (false) → Consolidate All Actions (Input 2)**
   2. Add **Set** node named **“Format Audit Report”** to create:
      - reportTitle, reportDate, validationSummary, riskLevel, riskScore, actionsExecuted (stringified), escalationLevel, nextReviewDate, auditTrail
   3. Connect: **Consolidate All Actions → Format Audit Report**

15) **Credentials checklist**
   - OpenAI API credential for all OpenAI model nodes
   - Google Calendar OAuth2 (scopes to create events)
   - Gmail OAuth2 (send email scope)
   - Slack OAuth2 (chat:write, channel access)
   - Google Sheets OAuth2 (spreadsheets scope)
   - If sensor/compliance APIs require auth, configure in HTTP Request nodes

---