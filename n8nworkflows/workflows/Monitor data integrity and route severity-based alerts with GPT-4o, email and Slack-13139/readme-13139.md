Monitor data integrity and route severity-based alerts with GPT-4o, email and Slack

https://n8nworkflows.xyz/workflows/monitor-data-integrity-and-route-severity-based-alerts-with-gpt-4o--email-and-slack-13139


# Monitor data integrity and route severity-based alerts with GPT-4o, email and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Intelligent AI-Driven Data Integrity Monitoring System*  
**Provided title:** *Monitor data integrity and route severity-based alerts with GPT-4o, email and Slack*

**Purpose:**  
Runs an automated, scheduled data integrity check by fetching two data sources (software metrics + BI dashboard), merging them, using GPT‑4o to detect anomalies and compliance concerns, then orchestrating severity-based notifications (email + Slack for critical, Slack for high, executive report email for medium/low). It also creates a compliance-focused audit trail log for every alert/report path.

**Target use cases:**
- Operational monitoring (SaaS health metrics, logs, availability)
- BI/data quality monitoring (dashboard freshness/consistency)
- Compliance-aware anomaly reporting (GDPR/ISO considerations)

### 1.1 Scheduled Execution & Configuration
- Triggered every 15 minutes.
- Central “Workflow Configuration” node holds URLs, thresholds, and notification targets (email + Slack channel IDs).

### 1.2 Multi-Source Data Collection
- Fetches software metrics from an API endpoint.
- Fetches BI dashboard data from another endpoint.

### 1.3 Data Consolidation
- Merges both responses into a single structured object with a timestamp.

### 1.4 AI-Powered Validation (Anomaly Detection)
- Uses a LangChain Agent backed by GPT‑4o to analyze the merged payload.
- Structured output is enforced by a JSON schema output parser.

### 1.5 AI-Powered Orchestration (Notification Strategy)
- If anomalies exist, a second GPT‑4o agent determines notification channels, escalation, audit requirements, and priority (P0–P3).
- Output is also schema-enforced.

### 1.6 Severity-Based Routing & Notifications
- A Switch routes execution:
  - **Critical:** email + Slack
  - **High:** Slack
  - **Medium/Low:** executive report email

### 1.7 Compliance Audit Logging
- All notification/report branches feed into a “Log Compliance Audit Trail” code node that builds a structured audit log entry (with GDPR/ISO references and approval status fields).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled Execution & Workflow Configuration
**Overview:** Triggers the workflow periodically and defines all runtime configuration values (API URLs, alert recipients, Slack channel IDs, and thresholds).  
**Nodes involved:**  
- Schedule Data Integrity Check  
- Workflow Configuration

#### Node: Schedule Data Integrity Check
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration (interpreted):** Runs every **15 minutes**.
- **Inputs / outputs:** No inputs; outputs to **Workflow Configuration**.
- **Version:** 1.3
- **Failure/edge cases:**
  - n8n instance timezone affects perceived schedule timing.
  - High frequency schedules can overlap if executions are slow (consider concurrency settings).

#### Node: Workflow Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — centralized configuration.
- **Key fields defined:**
  - `metricsApiUrl` (placeholder)
  - `biDashboardApiUrl` (placeholder)
  - `anomalyThreshold` = `0.75` (note: not currently used in routing/agents explicitly)
  - `criticalAlertEmail` (placeholder)
  - `executiveReportEmail` (placeholder)
  - `slackCriticalChannel` (placeholder channel ID)
  - `slackHighPriorityChannel` (placeholder channel ID)
- **Behavior:** `includeOtherFields: true` (passes through existing fields if any exist).
- **Outputs:** Fan-out to:
  - **Fetch Software Metrics**
  - **Fetch BI Dashboard Data**
- **Version:** 3.4
- **Failure/edge cases:**
  - Placeholder values must be replaced or HTTP/notification nodes will fail.
  - Storing secrets here is not recommended; prefer credentials/environment variables.

---

### Block 2.2 — Multi-Source Data Collection
**Overview:** Pulls operational metrics and BI dashboard data via HTTP.  
**Nodes involved:**  
- Fetch Software Metrics  
- Fetch BI Dashboard Data

#### Node: Fetch Software Metrics
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — fetch JSON metrics.
- **Configuration:**
  - **URL:** `={{ $('Workflow Configuration').first().json.metricsApiUrl }}`
  - Sends header `Content-Type: application/json`
- **Inputs:** From **Workflow Configuration**
- **Outputs:** To **Merge Data Sources**
- **Version:** 4.3
- **Failure/edge cases:**
  - 401/403 if API requires auth and none is configured.
  - Non-JSON responses can break downstream assumptions.
  - Timeouts/5xx; consider retry options and explicit response format settings.

#### Node: Fetch BI Dashboard Data
- **Type / role:** HTTP Request — fetch BI/dashboard JSON.
- **Configuration:**
  - **URL:** `={{ $('Workflow Configuration').first().json.biDashboardApiUrl }}`
  - Sends header `Content-Type: application/json`
- **Inputs:** From **Workflow Configuration**
- **Outputs:** To **Merge Data Sources**
- **Version:** 4.3
- **Failure/edge cases:** Same as above; additionally, BI endpoints often paginate—this node currently assumes a single payload.

---

### Block 2.3 — Data Consolidation
**Overview:** Merges both HTTP responses into one object `{ metrics, biData, timestamp }` for AI analysis.  
**Nodes involved:**  
- Merge Data Sources

#### Node: Merge Data Sources
- **Type / role:** Code (`n8n-nodes-base.code`) — custom merging logic.
- **Logic (interpreted):**
  - Collects all incoming items via `$input.all()`.
  - Tries to identify which payload is metrics vs BI by checking fields:
    - Metrics heuristic: `item.json.metrics` or `item.json.software`
    - BI heuristic: `item.json.dashboard` or `item.json.bi`
  - If unknown, assigns first unassigned to metrics then BI.
  - Emits **one item**:
    ```js
    {
      metrics: <metricsData||{}>,
      biData: <biData||{}>,
      timestamp: ISOString
    }
    ```
- **Inputs:** Both HTTP request nodes
- **Outputs:** To **Data Validation Agent**
- **Version:** 2
- **Failure/edge cases:**
  - If both responses have similar shapes (no distinguishing keys), misclassification is possible.
  - If either HTTP node returns multiple items, the merge heuristics may behave unexpectedly.
  - If either branch fails and sends no data, merged output contains `{}` for that source (agent must tolerate missing data).

---

### Block 2.4 — AI-Powered Anomaly Detection (Validation)
**Overview:** Uses GPT‑4o to detect anomalies, integrity issues, and compliance concerns, returning schema-validated structured JSON.  
**Nodes involved:**  
- OpenAI Model - Data Validation  
- Validation Output Parser  
- Data Validation Agent

#### Node: OpenAI Model - Data Validation
- **Type / role:** OpenAI Chat Model connector (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- **Model:** `gpt-4o`
- **Temperature:** `0.2` (more deterministic)
- **Used by:** **Data Validation Agent** as its language model input (`ai_languageModel` connection).
- **Credentials:** `OpenAi account`
- **Version:** 1.3
- **Failure/edge cases:**
  - Invalid/expired OpenAI API key, quota limits, model availability.
  - Response truncation if prompts are too large (merged payload can grow).

#### Node: Validation Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
- **Schema:** Manual JSON schema requiring:
  - `hasAnomaly` (boolean)
  - `anomalyScore` (number)
  - `anomalyType` (string)
  - `affectedSystems` (string[])
  - `dataIntegrityIssues` (string[])
  - `complianceViolations` (string[])
  - `gdprConcerns` (string[])
  - `isoConcerns` (string[])
  - `recommendations` (string[])
  - `severity` (string: expected low/medium/high/critical by system prompt, but schema doesn’t restrict values)
- **Connection:** Feeds into **Data Validation Agent** as its output parser (`ai_outputParser`).
- **Version:** 1.3
- **Failure/edge cases:**
  - If the LLM returns non-JSON or missing fields, parsing fails and the workflow errors.
  - Consider adding defaulting or a “try/catch” style fallback (not present).

#### Node: Data Validation Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — produces anomaly assessment.
- **Prompt input:**  
  `Data to validate: {{ JSON.stringify($json) }}`
- **System message (interpreted):**
  - Read-only posture; no autonomous production changes.
  - Detect data completeness/consistency issues, anomalies, and compliance concerns.
  - Compute anomaly score 0–1; classify anomaly type; list affected systems; give recommendations; set severity.
- **Inputs:** From **Merge Data Sources** (+ attached model + output parser)
- **Outputs:** To **Check for Anomalies**
- **Version:** 3.1
- **Failure/edge cases:**
  - Large JSON payloads can exceed context limits; consider summarizing before sending.
  - If upstream returns non-serializable data, `JSON.stringify` could fail (rare in n8n, but possible with circular structures).
  - Model may comply but still produce borderline schema mismatches; parser will fail hard.

---

### Block 2.5 — Anomaly Gate
**Overview:** Only proceeds to orchestration if an anomaly is detected.  
**Nodes involved:**  
- Check for Anomalies

#### Node: Check for Anomalies
- **Type / role:** IF (`n8n-nodes-base.if`) — boolean gate.
- **Condition:**  
  `={{ $('Data Validation Agent').item.json.hasAnomaly }}` **equals** `true`
- **Outputs:**  
  - **True path:** to **Orchestration Agent**
  - **False path:** not connected (workflow effectively ends for “no anomaly” runs)
- **Version:** 2.3
- **Failure/edge cases:**
  - If the validation agent fails or doesn’t output `hasAnomaly`, expression evaluation can error.
  - Because the “false” branch is unused, “no anomaly” cases produce no reporting/audit record.

---

### Block 2.6 — AI-Powered Orchestration (Response Planning)
**Overview:** Uses GPT‑4o to decide alert strategy (channels, escalation, executive reporting, audit requirements) based on anomaly findings, outputting schema-validated JSON.  
**Nodes involved:**  
- OpenAI Model - Orchestration  
- Orchestration Output Parser  
- Orchestration Agent

#### Node: OpenAI Model - Orchestration
- **Type / role:** OpenAI Chat Model connector
- **Model:** `gpt-4o`
- **Temperature:** `0.3`
- **Connection:** Provides model to **Orchestration Agent** (`ai_languageModel`).
- **Credentials:** `OpenAi account`
- **Version:** 1.3
- **Failure/edge cases:** Same as other model node.

#### Node: Orchestration Output Parser
- **Type / role:** Structured output parser
- **Schema fields:**
  - `alertType` (string)
  - `notificationChannels` (string[])
  - `escalationRequired` (boolean)
  - `executiveReportRequired` (boolean)
  - `actionItems` (string[])
  - `complianceDocumentation` (string)
  - `auditTrailRequired` (boolean)
  - `severity` (string)
  - `priority` (string, expected P0–P3)
- **Connection:** into **Orchestration Agent** (`ai_outputParser`)
- **Version:** 1.3
- **Failure/edge cases:** Parser failures if LLM misses fields; schema does not constrain enumerations.

#### Node: Orchestration Agent
- **Type / role:** LangChain Agent — translates anomaly assessment into routing decisions and action items.
- **Prompt input:**  
  `Anomaly data: {{ JSON.stringify($json) }}`
- **System message highlights:**
  - Severity → channels mapping:
    - critical → email + Slack
    - high → Slack
    - medium/low → executive report only
  - Escalation required for critical/high
  - Must not take production actions; maintain audit trails
- **Inputs:** From **Check for Anomalies** (true branch) plus attached model/parser
- **Outputs:** To **Route by Severity**
- **Version:** 3.1
- **Failure/edge cases:**
  - The agent output must include `severity` used by the downstream Switch.
  - If the agent returns “Critical” instead of “critical”, Switch rules won’t match (rules use strict equals).

---

### Block 2.7 — Severity-Based Routing & Notifications
**Overview:** Routes to the correct notification path and sends tailored alerts/reports.  
**Nodes involved:**  
- Route by Severity  
- Send Critical Alert Email  
- Send Critical Slack Alert  
- Send High Priority Slack Alert  
- Prepare Executive Report  
- Send Executive Report

#### Node: Route by Severity
- **Type / role:** Switch (`n8n-nodes-base.switch`) — branch by severity.
- **Rules (strict):**
  - Output “Critical” if `={{ $json.severity }}` equals `"critical"`
  - Output “High” if equals `"high"`
  - Fallback renamed to “Medium/Low”
- **Inputs:** From **Orchestration Agent**
- **Outputs:**
  - Critical → **Send Critical Alert Email** and **Send Critical Slack Alert**
  - High → **Send High Priority Slack Alert**
  - Medium/Low → **Prepare Executive Report**
- **Version:** 3.4
- **Failure/edge cases:**
  - Case sensitivity: conditions are strict, but caseSensitive false; still, value must match exactly by operation semantics—best to normalize.
  - If orchestration omits `severity`, everything goes to fallback.

#### Node: Send Critical Alert Email
- **Type / role:** Email Send (`n8n-nodes-base.emailSend`) — critical notification.
- **To:** `={{ $('Workflow Configuration').first().json.criticalAlertEmail }}`
- **Subject:** `CRITICAL: Data Integrity Anomaly Detected - {{ $json.anomalyType }}`
- **Body:** HTML template (includes “🚨” in header text)
- **From:** `user@example.com` (placeholder)
- **Output:** To **Log Compliance Audit Trail**
- **Version:** 2.1
- **Failure/edge cases / important mismatch:**
  - Template references fields likely **not produced** by upstream agents:
    - `anomalyDescription`, `detectedAt`, `recommendedActions` (not in validation/orchestration schemas)
  - If these fields are missing, the email renders blank placeholders (not a hard failure).
  - SMTP/Email credentials are not shown in JSON; node requires a configured email transport in n8n.

#### Node: Send Critical Slack Alert
- **Type / role:** Slack (`n8n-nodes-base.slack`) — critical Slack message.
- **Auth:** OAuth2 (`slackOAuth2Api`)
- **Channel:** `={{ $('Workflow Configuration').first().json.slackCriticalChannel }}`
- **Message uses arrays:** `.join(', ')`, `.join('\n')` on:
  - `affectedSystems`, `complianceViolations`, `recommendations`
- **Output:** To **Log Compliance Audit Trail**
- **Version:** 2.4
- **Failure/edge cases:**
  - If any of those fields are not arrays, `.join()` throws (runtime expression error).
  - Ensure validation agent always outputs arrays per schema (parser enforces it).

#### Node: Send High Priority Slack Alert
- **Type / role:** Slack — high-priority Slack message.
- **Channel:** `={{ $('Workflow Configuration').first().json.slackHighPriorityChannel }}`
- **Message joins arrays:** `affectedSystems`, `dataIntegrityIssues`, `recommendations`
- **Output:** To **Log Compliance Audit Trail**
- **Version:** 2.4
- **Failure/edge cases:** Same `.join()` risks if upstream schema enforcement is bypassed (parser failures would stop earlier).

#### Node: Prepare Executive Report
- **Type / role:** Set — builds concise fields for an executive report email.
- **Fields created:**
  - `reportTitle` = `Data Integrity Monitoring Report - {{ $now.toFormat('yyyy-MM-dd HH:mm') }}`
  - `anomalySummary` = `{{ $json.anomalyType }}`
  - `severityLevel` = `{{ $json.severity }}`
  - `affectedSystemsList` = `{{ $json.affectedSystems.join(', ') }}`
  - `complianceStatus` = `Violations Detected` if `complianceViolations.length > 0` else `Compliant`
  - `actionItemsSummary` = `recommendations.join('; ')`
- **Output:** To **Send Executive Report**
- **Version:** 3.4
- **Failure/edge cases:**
  - Assumes arrays exist; if orchestration output differs from validation output, fields may be missing (but arrays are expected from validation agent; note: the Medium/Low branch uses orchestration output, not validation output).

#### Node: Send Executive Report
- **Type / role:** Email Send — executive HTML report.
- **To:** `={{ $('Workflow Configuration').first().json.executiveReportEmail }}`
- **Subject:** `Executive Report: Data Integrity Monitoring - {{ $now.toFormat('yyyy-MM-dd') }}`
- **Body:** Rich HTML with many expressions referencing multiple nodes (Fetch/Merge/Agents).
- **Output:** To **Log Compliance Audit Trail**
- **Version:** 2.1
- **Failure/edge cases / important mismatch:**
  - The template references many fields that are **not produced** by the agents/schemas, e.g.:
    - `overallStatus`, `criticalCount`, `gdprCompliant`, `isoCompliant`, `integrityIssuesDescription`, etc.
  - These will render as fallback strings (`|| '...'`) in many places, so it won’t necessarily fail, but the report may be largely placeholders unless you extend agent outputs.
  - Same email transport/credential requirements as other email node.

---

### Block 2.8 — Compliance Audit Trail Logging
**Overview:** Generates a structured audit log entry that captures anomaly metadata, actions taken, compliance references (GDPR + ISO 27001), and approval status expectations.  
**Nodes involved:**  
- Log Compliance Audit Trail

#### Node: Log Compliance Audit Trail
- **Type / role:** Code — constructs an audit log object.
- **Execution mode:** `runOnceForEachItem`
- **Key generated structure:**
  - `logId` unique ID
  - `timestamp`
  - `workflowExecutionId` = `$execution.id`
  - `anomalyDetails` (type, severity, affectedSystems, etc.)
  - `actionsTaken`, `notificationChannels`
  - `complianceDocumentation`:
    - GDPR legal basis: “Legitimate interest - Article 6(1)(f) GDPR”
    - ISO reference: “ISO 27001:2013 A.12.4.1 Event Logging”
  - `userApprovalStatus.requiresApproval` true if severity critical/high
  - `metadata.retentionDate` set to +90 days
- **Inputs:** From:
  - Send Critical Alert Email
  - Send Critical Slack Alert
  - Send High Priority Slack Alert
  - Send Executive Report
- **Outputs:** Final (no downstream connections)
- **Version:** 2
- **Failure/edge cases:**
  - This node does **not** persist logs anywhere (only returns JSON and `console.log`). If you need audit durability, add a database/storage node (Postgres, S3, SIEM, etc.).
  - Many fields (`alertsSent`, `emailSent`, `slackSent`, etc.) are expected but never set upstream, so the audit log will default to empty/false unless you enrich the data before logging.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Data Integrity Check | Schedule Trigger | Periodic workflow trigger (15 min) | — | Workflow Configuration | ## Multi-Source Data Collection<br>**What:** Fetches and merges data from software metrics APIs and BI dashboard sources on scheduled intervals<br>**Why:** Ensures comprehensive monitoring coverage by combining operational metrics with business intelligence for complete visibility |
| Workflow Configuration | Set | Centralized config (URLs, emails, channels, threshold) | Schedule Data Integrity Check | Fetch Software Metrics; Fetch BI Dashboard Data | ## Multi-Source Data Collection<br>**What:** Fetches and merges data from software metrics APIs and BI dashboard sources on scheduled intervals<br>**Why:** Ensures comprehensive monitoring coverage by combining operational metrics with business intelligence for complete visibility |
| Fetch Software Metrics | HTTP Request | Pull software/ops metrics JSON | Workflow Configuration | Merge Data Sources | ## Multi-Source Data Collection<br>**What:** Fetches and merges data from software metrics APIs and BI dashboard sources on scheduled intervals<br>**Why:** Ensures comprehensive monitoring coverage by combining operational metrics with business intelligence for complete visibility |
| Fetch BI Dashboard Data | HTTP Request | Pull BI/dashboard JSON | Workflow Configuration | Merge Data Sources | ## Multi-Source Data Collection<br>**What:** Fetches and merges data from software metrics APIs and BI dashboard sources on scheduled intervals<br>**Why:** Ensures comprehensive monitoring coverage by combining operational metrics with business intelligence for complete visibility |
| Merge Data Sources | Code | Consolidate both sources into one payload | Fetch Software Metrics; Fetch BI Dashboard Data | Data Validation Agent | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |
| OpenAI Model - Data Validation | OpenAI Chat Model (LangChain) | LLM backend for validation agent | — (model provider) | Data Validation Agent (ai_languageModel) | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |
| Validation Output Parser | Structured Output Parser (LangChain) | Enforce schema for validation output | — | Data Validation Agent (ai_outputParser) | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |
| Data Validation Agent | LangChain Agent | Detect anomalies/compliance issues; output structured JSON | Merge Data Sources (+ model/parser) | Check for Anomalies | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |
| Check for Anomalies | IF | Gate: proceed only if hasAnomaly=true | Data Validation Agent | Orchestration Agent (true path) | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |
| OpenAI Model - Orchestration | OpenAI Chat Model (LangChain) | LLM backend for orchestration agent | — | Orchestration Agent (ai_languageModel) | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Orchestration Output Parser | Structured Output Parser (LangChain) | Enforce schema for orchestration output | — | Orchestration Agent (ai_outputParser) | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Orchestration Agent | LangChain Agent | Decide channels/escalation/priority based on anomaly | Check for Anomalies (+ model/parser) | Route by Severity | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Route by Severity | Switch | Branch: critical/high/medium-low | Orchestration Agent | Send Critical Alert Email; Send Critical Slack Alert; Send High Priority Slack Alert; Prepare Executive Report | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Send Critical Alert Email | Email Send | Email critical incident | Route by Severity (Critical) | Log Compliance Audit Trail | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Send Critical Slack Alert | Slack | Slack message for critical incidents | Route by Severity (Critical) | Log Compliance Audit Trail | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Send High Priority Slack Alert | Slack | Slack message for high severity | Route by Severity (High) | Log Compliance Audit Trail | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Prepare Executive Report | Set | Build executive summary fields | Route by Severity (Medium/Low) | Send Executive Report | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Send Executive Report | Email Send | Send executive HTML report | Prepare Executive Report | Log Compliance Audit Trail | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Log Compliance Audit Trail | Code | Create compliance audit log entry | Send Critical Alert Email; Send Critical Slack Alert; Send High Priority Slack Alert; Send Executive Report | — |  |
| Sticky Note | Sticky Note | Documentation block | — | — | ## Prerequisites<br>OpenAI or Nvidia API credentials for AI-powered analysis, API access to software metrics platforms<br>## Use Cases<br>SaaS platforms monitoring service health metrics, e-commerce businesses tracking inventory data quality<br>## Customization<br>Adjust scheduling frequency for monitoring intervals, modify severity thresholds for alert classification<br>## Benefits<br>Reduces mean time to detection by 75%, eliminates manual data quality checks |
| Sticky Note1 | Sticky Note | Documentation block | — | — | ## Setup Steps<br>1. Configure Schedule Data Integrity Check trigger with monitoring frequency<br>2. Connect Workflow Configuration node with data source parameters<br>3. Set up Fetch Software Metrics and Fetch BI Dashboard Data nodes with respective API credentials<br>4. Configure Merge Data Sources node for data consolidation logic<br>5. Connect Data Validation Agent with OpenAI/Nvidia API credentials for anomaly detection<br>6. Set up Orchestration Agent with AI API credentials for severity assessment<br>7. Configure Check for Anomalies node with routing conditions<br>8. Connect Route by Severity node with classification logic |
| Sticky Note2 | Sticky Note | Documentation block | — | — | ## How It Works<br>This workflow automates continuous data integrity monitoring and intelligent alert management across multiple data sources. Designed for data engineers, IT operations teams, and business intelligence analysts, it solves the critical challenge of detecting data anomalies and orchestrating appropriate responses based on severity levels. The system operates on scheduled intervals, fetching data from software metrics APIs and BI dashboards, then merging these sources for comprehensive analysis. It employs AI-powered validation and orchestration agents to detect anomalies, assess severity, and determine optimal response strategies. The workflow intelligently routes alerts based on severity classification, triggering critical notifications via email and Slack for high-priority issues while sending standard reports for routine findings. By maintaining detailed compliance audit logs and preparing executive summaries, it ensures stakeholders receive timely, actionable intelligence while creating audit trails for data quality monitoring initiatives. |
| Sticky Note3 | Sticky Note | Documentation block | — | — | ## Multi-Source Data Collection<br>**What:** Fetches and merges data from software metrics APIs and BI dashboard sources on scheduled intervals<br>**Why:** Ensures comprehensive monitoring coverage by combining operational metrics with business intelligence for complete visibility |
| Sticky Note4 | Sticky Note | Documentation block | — | — | ## Severity-Based Alert Routing<br>**What:** Routes alerts through severity-specific channels with tailored messaging for critical, high-priority, and standard notifications<br>**Why:** Ensures urgent issues receive immediate attention while preventing alert fatigue from routine findings |
| Sticky Note5 | Sticky Note | Documentation block | — | — | ## AI-Powered Anomaly Detection<br>**What:** Processes merged data through dual AI agents for validation and intelligent orchestration of anomaly findings<br>**Why:** Leverages AI to detect subtle patterns and determine appropriate severity classifications beyond simple threshold-based rules |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *Intelligent AI-Driven Data Integrity Monitoring System*
   - Set workflow to inactive initially.

2. **Add Trigger: “Schedule Data Integrity Check”**
   - Node type: **Schedule Trigger**
   - Schedule rule: every **15 minutes**

3. **Add config node: “Workflow Configuration”**
   - Node type: **Set**
   - Add fields:
     - `metricsApiUrl` (string) – your metrics endpoint
     - `biDashboardApiUrl` (string) – your BI endpoint
     - `anomalyThreshold` (number) – `0.75` (optional unless you integrate it)
     - `criticalAlertEmail` (string)
     - `executiveReportEmail` (string)
     - `slackCriticalChannel` (string; Slack channel ID)
     - `slackHighPriorityChannel` (string; Slack channel ID)
   - Enable **Include Other Fields**.

4. **Connect:** Schedule Trigger → Workflow Configuration

5. **Add HTTP node: “Fetch Software Metrics”**
   - Node type: **HTTP Request**
   - URL expression: `{{ $('Workflow Configuration').first().json.metricsApiUrl }}`
   - Header: `Content-Type: application/json`
   - Configure authentication if your API requires it (Header auth, OAuth2, etc.)

6. **Add HTTP node: “Fetch BI Dashboard Data”**
   - Node type: **HTTP Request**
   - URL expression: `{{ $('Workflow Configuration').first().json.biDashboardApiUrl }}`
   - Header: `Content-Type: application/json`
   - Configure authentication if required.

7. **Connect:** Workflow Configuration → Fetch Software Metrics  
   **Connect:** Workflow Configuration → Fetch BI Dashboard Data

8. **Add code node: “Merge Data Sources”**
   - Node type: **Code**
   - Paste logic to:
     - read `$input.all()`
     - identify metrics vs BI payload
     - output a single merged item `{ metrics, biData, timestamp }`
   - Keep as “Run Once for All Items” (default for merge-style logic).

9. **Connect:** Fetch Software Metrics → Merge Data Sources  
   **Connect:** Fetch BI Dashboard Data → Merge Data Sources

10. **Add OpenAI model node: “OpenAI Model - Data Validation”**
    - Node type: **OpenAI Chat Model (LangChain)**
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Credentials: configure **OpenAI API** credential in n8n.

11. **Add output parser: “Validation Output Parser”**
    - Node type: **Structured Output Parser**
    - Schema: create manual schema with fields listed in section 2.4.

12. **Add agent: “Data Validation Agent”**
    - Node type: **AI Agent (LangChain)**
    - Prompt: `Data to validate: {{ JSON.stringify($json) }}`
    - System message: include the read-only anomaly detection instructions (as in the workflow).
    - Attach connections:
      - Model: connect **OpenAI Model - Data Validation** → **Data Validation Agent** (AI Language Model)
      - Parser: connect **Validation Output Parser** → **Data Validation Agent** (AI Output Parser)

13. **Connect:** Merge Data Sources → Data Validation Agent

14. **Add gate: “Check for Anomalies”**
    - Node type: **IF**
    - Condition: boolean equals
      - Left: `{{ $('Data Validation Agent').item.json.hasAnomaly }}`
      - Right: `true`

15. **Connect:** Data Validation Agent → Check for Anomalies (main)

16. **Add OpenAI model node: “OpenAI Model - Orchestration”**
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Use same OpenAI credentials (or separate if desired).

17. **Add output parser: “Orchestration Output Parser”**
    - Node type: **Structured Output Parser**
    - Manual schema with orchestration fields (section 2.6).

18. **Add agent: “Orchestration Agent”**
    - Prompt: `Anomaly data: {{ JSON.stringify($json) }}`
    - System message: include severity→channels mapping + audit trail constraints.
    - Connect:
      - **OpenAI Model - Orchestration** → **Orchestration Agent** (AI Language Model)
      - **Orchestration Output Parser** → **Orchestration Agent** (AI Output Parser)

19. **Connect:** Check for Anomalies (true output) → Orchestration Agent

20. **Add switch: “Route by Severity”**
    - Node type: **Switch**
    - Value to evaluate: `{{ $json.severity }}`
    - Rules:
      - “Critical”: equals `critical`
      - “High”: equals `high`
    - Fallback output name: “Medium/Low”

21. **Connect:** Orchestration Agent → Route by Severity

22. **Add notification nodes**
    - **Send Critical Alert Email** (Email Send)
      - To: `{{ $('Workflow Configuration').first().json.criticalAlertEmail }}`
      - From: set a valid from address for your SMTP
      - Subject/body: use your HTML template (ensure referenced fields exist)
    - **Send Critical Slack Alert** (Slack)
      - Auth: Slack OAuth2 credential
      - Channel ID: `{{ $('Workflow Configuration').first().json.slackCriticalChannel }}`
      - Message text using `affectedSystems`, `complianceViolations`, `recommendations`
    - **Send High Priority Slack Alert** (Slack)
      - Channel ID: `{{ $('Workflow Configuration').first().json.slackHighPriorityChannel }}`
      - Message text using `affectedSystems`, `dataIntegrityIssues`, `recommendations`
    - **Prepare Executive Report** (Set)
      - Build report fields as per section 2.7
    - **Send Executive Report** (Email Send)
      - To: `{{ $('Workflow Configuration').first().json.executiveReportEmail }}`

23. **Wire severity routes**
    - Route by Severity (Critical) → Send Critical Alert Email
    - Route by Severity (Critical) → Send Critical Slack Alert
    - Route by Severity (High) → Send High Priority Slack Alert
    - Route by Severity (Medium/Low) → Prepare Executive Report → Send Executive Report

24. **Add “Log Compliance Audit Trail”**
    - Node type: **Code**
    - Mode: **Run Once for Each Item**
    - Paste audit log construction logic (section 2.8).
    - (Optional but recommended) Add a persistence node afterwards (DB/S3/SIEM).

25. **Connect all notification/report nodes to audit logging**
    - Send Critical Alert Email → Log Compliance Audit Trail
    - Send Critical Slack Alert → Log Compliance Audit Trail
    - Send High Priority Slack Alert → Log Compliance Audit Trail
    - Send Executive Report → Log Compliance Audit Trail

26. **Credentials checklist**
    - OpenAI credential for both model nodes
    - Slack OAuth2 credential for Slack nodes
    - Email/SMTP configuration for both Email Send nodes (depends on n8n setup)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** OpenAI or Nvidia API credentials for AI-powered analysis, API access to software metrics platforms | Sticky note “Prerequisites” |
| **Use Cases:** SaaS platforms monitoring service health metrics, e-commerce businesses tracking inventory data quality | Sticky note “Use Cases” |
| **Customization:** Adjust scheduling frequency for monitoring intervals, modify severity thresholds for alert classification | Sticky note “Customization” |
| **Benefits:** Reduces mean time to detection by 75%, eliminates manual data quality checks | Sticky note “Benefits” |
| **How it works:** Workflow description covering scheduled fetch → merge → AI validation/orchestration → severity routing → alerts/reports → audit trails | Sticky note “How It Works” |

