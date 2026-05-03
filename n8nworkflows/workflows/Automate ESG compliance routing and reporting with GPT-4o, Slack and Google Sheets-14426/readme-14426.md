Automate ESG compliance routing and reporting with GPT-4o, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/automate-esg-compliance-routing-and-reporting-with-gpt-4o--slack-and-google-sheets-14426


# Automate ESG compliance routing and reporting with GPT-4o, Slack and Google Sheets

# 1. Workflow Overview

This workflow automates ESG sustainability governance by collecting environmental data from either a scheduled trigger or an external webhook, normalizing the payload, evaluating it through an AI-driven governance layer, routing the result by approval status, and then logging, notifying, synchronizing, and reporting outcomes.

It is designed for organizations that want to:
- automate ESG compliance checks,
- standardize sustainability event intake,
- generate structured governance decisions,
- route items for approval/review/rejection,
- maintain auditable lineage and KPI tracking,
- synchronize approved decisions to downstream enterprise systems.

## 1.1 Input Reception and Normalization

The workflow starts from two entry points:
- a periodic scheduler running every 6 hours,
- a webhook receiving POST events from an external ESG platform.

Both inputs converge into a normalization node that standardizes event structure and fills defaults for missing values.

## 1.2 AI Governance and ESG Validation

The normalized data is sent to a primary AI agent, the **Green Governance Agent**, which uses:
- a governance LLM,
- a structured output parser,
- a nested oversight/validation tool agent,
- Slack, Airtable, and other tool integrations.

The oversight sub-agent can validate metrics, calculate ESG scores, query external APIs, and read benchmark data from Google Sheets.

## 1.3 Approval-Based Routing

The AI output is expected to include an `approvalStatus` field. A Switch node routes the result to one of three branches:
- `approved`
- `needs_review`
- `rejected`

Each branch applies different handling logic.

## 1.4 Approved Record Storage, Reporting, and Synchronization

Approved records are transformed into a storage-friendly format, saved to an n8n Data Table, used to update a Google Sheets dashboard, emailed to stakeholders, synchronized to an enterprise ESG platform, and logged for lineage and KPI tracking.

## 1.5 Error Handling for External Sync

If synchronization to the enterprise ESG platform fails, the workflow continues via the error output, prepares a structured error record, logs the failure, and sends a Slack alert.

## 1.6 Environmental Lineage and KPI Tracking

Approved records also feed a lineage record and KPI record flow, ensuring traceability of original versus validated ESG data and periodic KPI performance tracking.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

**Overview:**  
This block ingests ESG events from two sources and converts them into a standardized internal format. It prevents downstream AI and routing logic from dealing with inconsistent field names or missing metrics.

**Nodes Involved:**  
- Periodic Sustainability Monitoring
- External ESG Platform Events
- Normalize Sustainability Input

### Node: Periodic Sustainability Monitoring
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry point for recurring ESG monitoring.
- **Configuration choices:**  
  Configured to run every 6 hours.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to **Normalize Sustainability Input**.
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  Usually low-risk; the main issue is whether the workflow is active. If inactive, the trigger will not fire.
- **Sub-workflow reference:**  
  None.

### Node: External ESG Platform Events
- **Type and technical role:** `n8n-nodes-base.webhook`  
  HTTP entry point for ESG events submitted externally.
- **Configuration choices:**  
  - Method: `POST`
  - Path: `sustainability-events`
- **Key expressions or variables used:**  
  None in configuration.
- **Input and output connections:**  
  No input; outputs to **Normalize Sustainability Input**.
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Incorrect webhook URL or method
  - Payload shape mismatch
  - Authentication not implemented at webhook layer
  - Large request bodies or malformed JSON
- **Sub-workflow reference:**  
  None.

### Node: Normalize Sustainability Input
- **Type and technical role:** `n8n-nodes-base.set`  
  Standardizes inbound event fields for downstream processing.
- **Configuration choices:**  
  Creates these normalized fields:
  - `eventType`: defaults to `scheduled_monitoring`
  - `carbonFootprint`: from root or `metrics.carbonFootprint`, else `0`
  - `energyConsumption`: from root or `metrics.energyConsumption`, else `0`
  - `resourceUtilization`: from root or `metrics.resourceUtilization`, else `0`
  - `complianceSignals`: from root or nested metrics, else `[]`
  - `timestamp`: input timestamp or current time
  - `source`: input source or `internal`
- **Key expressions or variables used:**  
  - `{{ $json.eventType || 'scheduled_monitoring' }}`
  - `{{ $json.metrics?.carbonFootprint || 0 }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**  
  Inputs from **Periodic Sustainability Monitoring** and **External ESG Platform Events**; output to **Green Governance Agent**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Numeric fields may arrive as strings; implicit casting may or may not behave as intended
  - Missing nested `metrics` object is handled safely via optional chaining
  - Defaulting to `0` may hide upstream data quality issues
- **Sub-workflow reference:**  
  None.

---

## 2.2 AI Governance Orchestration

**Overview:**  
This is the core decisioning layer. The Green Governance Agent receives the normalized ESG event, calls the oversight validator tool if needed, and returns structured compliance output suitable for deterministic routing.

**Nodes Involved:**  
- Green Governance Agent
- Governance Model
- ESG Compliance Output Schema
- Sustainability Oversight Agent Tool
- Governance Alerts Tool
- Compliance Documentation Tool

### Node: Green Governance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Primary AI orchestration node that evaluates ESG events and produces a governance decision.
- **Configuration choices:**  
  - Prompt type: defined prompt
  - Input text is assembled from normalized fields into a concise metric summary
  - Uses a detailed system message defining governance, approvals, KPI tracking, documentation, and compliance responsibilities
  - Structured output parsing is enabled
- **Key expressions or variables used:**  
  Builds text from:
  - `$json.eventType`
  - `$json.carbonFootprint`
  - `$json.energyConsumption`
  - `$json.resourceUtilization`
  - `JSON.stringify($json.complianceSignals)`
  - `$json.source`
  - `$json.timestamp`
- **Input and output connections:**  
  - Main input from **Normalize Sustainability Input**
  - AI language model from **Governance Model**
  - AI output parser from **ESG Compliance Output Schema**
  - AI tools from:
    - **Sustainability Oversight Agent Tool**
    - **Governance Alerts Tool**
    - **Compliance Documentation Tool**
  - Main output to **Route by Approval Status**
- **Version-specific requirements:**  
  Type version `3.1`; requires LangChain-compatible AI nodes in the same workflow.
- **Edge cases or potential failure types:**  
  - LLM may fail to respect schema if prompting is weak
  - Tool call failures can degrade result quality
  - OpenAI auth/rate-limit errors from linked model
  - Missing expected output fields can break downstream routing
- **Sub-workflow reference:**  
  None.

### Node: Governance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM backend for the Green Governance Agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model output to **Green Governance Agent**.
- **Version-specific requirements:**  
  Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Model availability issues
  - Token/rate limits
  - Regional or account access restrictions
- **Sub-workflow reference:**  
  None.

### Node: ESG Compliance Output Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured JSON-like response schema from the AI agent.
- **Configuration choices:**  
  Example schema includes:
  - `approvalStatus`
  - `complianceScore`
  - validated metrics
  - KPI fields
  - governance actions
  - audit trail
  - risk level
  - recommendations
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI output parser connected to **Green Governance Agent**.
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - AI may omit fields or use incompatible data types
  - Parser failures can stop the workflow if schema cannot be satisfied
- **Sub-workflow reference:**  
  None.

### Node: Sustainability Oversight Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Nested AI tool used by the main governance agent to validate ESG metrics.
- **Configuration choices:**  
  - Accepts AI-provided `metrics_data`
  - Has its own system message focused on metric validation, ESG requirement analysis, KPI calculation, and gap detection
  - Tool description clearly instructs the parent agent when to use it
- **Key expressions or variables used:**  
  `{{ $fromAI('metrics_data', 'Environmental metrics and compliance data to validate') }}`
- **Input and output connections:**  
  - AI language model from **Oversight Model**
  - AI tools from:
    - **Sustainability Metrics Calculator**
    - **ESG Scoring Engine**
    - **Multi-Cloud Sustainability API**
    - **ESG Reporting Sheets Tool**
  - Output as a tool to **Green Governance Agent**
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Tool-input shape may not match what child tools expect
  - Nested tool orchestration adds latency and more failure points
  - LLM may overuse or underuse available tools
- **Sub-workflow reference:**  
  None; this is a nested agent tool, not a separate workflow.

### Node: Governance Alerts Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`  
  AI-callable Slack tool for sending governance notifications.
- **Configuration choices:**  
  - Slack OAuth2 authentication
  - Channel chosen dynamically by AI via `slack_channel`
  - Message content provided dynamically by AI via `alert_message`
- **Key expressions or variables used:**  
  - `{{ $fromAI('alert_message', ...) }}`
  - `{{ $fromAI('slack_channel', ...) }}`
- **Input and output connections:**  
  Tool output to **Green Governance Agent**.
- **Version-specific requirements:**  
  Type version `2.4`; requires Slack OAuth2 credentials and channel access.
- **Edge cases or potential failure types:**  
  - AI may return an invalid channel ID
  - Slack permission issues
  - Rate limiting
  - Bot not invited to target channel
- **Sub-workflow reference:**  
  None.

### Node: Compliance Documentation Tool
- **Type and technical role:** `n8n-nodes-base.airtableTool`  
  AI-callable Airtable writer for compliance records.
- **Configuration choices:**  
  - Operation: `create`
  - Base selected manually
  - Table chosen dynamically by AI
- **Key expressions or variables used:**  
  `{{ $fromAI('table_name', 'The Airtable table to store compliance documentation', 'string') }}`
- **Input and output connections:**  
  Tool output to **Green Governance Agent**.
- **Version-specific requirements:**  
  Type version `2.2`; Airtable credentials are implied but not present in the JSON export.
- **Edge cases or potential failure types:**  
  - Missing Airtable credentials
  - Invalid or inaccessible base/table
  - AI returning a table ID/name incompatible with expected mode
- **Sub-workflow reference:**  
  None.

---

## 2.3 Sustainability Oversight and Scoring Sub-Agent

**Overview:**  
This block acts as the analytical validation layer. It supports the main governance agent by running calculations, external lookups, and benchmark checks.

**Nodes Involved:**  
- Oversight Model
- Sustainability Metrics Calculator
- ESG Scoring Engine
- Multi-Cloud Sustainability API
- ESG Reporting Sheets Tool

### Node: Oversight Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM backend for the oversight tool agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model output to **Sustainability Oversight Agent Tool**.
- **Version-specific requirements:**  
  Type version `1.3`; requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  Same as the governance model: auth, rate limits, model availability.
- **Sub-workflow reference:**  
  None.

### Node: Sustainability Metrics Calculator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  AI-callable calculation utility.
- **Configuration choices:**  
  No special parameters set.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Tool output to **Sustainability Oversight Agent Tool**.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  Depends on valid mathematical inputs from the agent.
- **Sub-workflow reference:**  
  None.

### Node: ESG Scoring Engine
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Custom JavaScript AI tool for ESG scoring.
- **Configuration choices:**  
  Implements custom scoring logic:
  - parses input JSON,
  - computes carbon, energy, and resource scores,
  - computes weighted ESG score,
  - derives energy rating,
  - derives risk level,
  - sets compliance status,
  - returns validated metrics and KPI details.
- **Key expressions or variables used:**  
  Uses the implicit `query` input expected by the tool code runtime.
- **Input and output connections:**  
  Tool output to **Sustainability Oversight Agent Tool**.
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid JSON input returns an error payload as a string
  - Score thresholds are simplistic and may not match real regulations
  - Negative or unrealistic values are not explicitly sanitized
  - Returned stringified JSON must still be interpreted correctly by the AI chain
- **Sub-workflow reference:**  
  None.

### Node: Multi-Cloud Sustainability API
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`  
  AI-callable external API retriever.
- **Configuration choices:**  
  URL is selected dynamically by the AI.
- **Key expressions or variables used:**  
  `{{ $fromAI('api_endpoint', 'The API endpoint to query for sustainability data', 'string') }}`
- **Input and output connections:**  
  Tool output to **Sustainability Oversight Agent Tool**.
- **Version-specific requirements:**  
  Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Invalid AI-generated endpoints
  - Missing authentication if API requires it
  - Timeout, SSL, DNS, or response format errors
- **Sub-workflow reference:**  
  None.

### Node: ESG Reporting Sheets Tool
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI-callable Google Sheets reader for ESG baselines and thresholds.
- **Configuration choices:**  
  - Document ID: not filled
  - Sheet name: not filled
  - Manual tool description explaining use for thresholds and benchmarks
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Tool output to **Sustainability Oversight Agent Tool**.
- **Version-specific requirements:**  
  Type version `4.7`; requires Google Sheets OAuth2.
- **Edge cases or potential failure types:**  
  - Missing document/sheet configuration makes the tool unusable
  - Permissions or sharing issues
  - Schema mismatch in the referenced sheet
- **Sub-workflow reference:**  
  None.

---

## 2.4 Approval Routing

**Overview:**  
This block deterministically routes the AI result based on the `approvalStatus` field. It separates approved, review-required, and rejected items for distinct operational treatment.

**Nodes Involved:**  
- Route by Approval Status

### Node: Route by Approval Status
- **Type and technical role:** `n8n-nodes-base.switch`  
  Conditional router based on approval status.
- **Configuration choices:**  
  Rules:
  1. `approvalStatus == approved`
  2. `approvalStatus == needs_review`
  3. `approvalStatus == rejected`
- **Key expressions or variables used:**  
  `{{ $json.approvalStatus }}`
- **Input and output connections:**  
  Input from **Green Governance Agent**. Outputs to:
  - **Prepare Approved Record**
  - **Send Review Request**
  - **Prepare Rejected Record**
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If the AI returns any other status value, no branch will match
  - Case sensitivity matters
  - Missing `approvalStatus` leads to unrouted items
- **Sub-workflow reference:**  
  None.

---

## 2.5 Review and Rejection Handling

**Overview:**  
This block handles outcomes that are not immediately approved. Review-required items are sent to Slack, while rejected items are transformed into a compact log record and stored.

**Nodes Involved:**  
- Send Review Request
- Prepare Rejected Record
- Log Rejected Items

### Node: Send Review Request
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a human review request for borderline or escalated items.
- **Configuration choices:**  
  Sends a formatted Slack message containing:
  - compliance score
  - validated carbon footprint
  - energy rating
  - risk level
  - ESG requirements
  - recommendations
  - audit trail
- **Key expressions or variables used:**  
  Uses values such as:
  - `$json.complianceScore`
  - `$json.carbonFootprintValidated`
  - `$json.recommendations.join('\n')`
- **Input and output connections:**  
  Input from **Route by Approval Status** on the `needs_review` branch.
- **Version-specific requirements:**  
  Type version `2.4`; requires Slack OAuth2.
- **Edge cases or potential failure types:**  
  - `recommendations` may be absent or not an array, causing expression failure
  - Placeholder channel ID must be replaced
  - Slack auth or channel permission errors
- **Sub-workflow reference:**  
  None.

### Node: Prepare Rejected Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Shapes rejected decisions into a logging record.
- **Configuration choices:**  
  Stores:
  - status
  - score
  - validated carbon footprint
  - risk
  - rejection reasons from recommendations
  - audit trail
  - timestamp
- **Key expressions or variables used:**  
  `{{ JSON.stringify($json.recommendations) }}`
- **Input and output connections:**  
  Input from **Route by Approval Status** on the `rejected` branch; output to **Log Rejected Items**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If `recommendations` is missing, JSON stringify still works but may yield undefined-like behavior
- **Sub-workflow reference:**  
  None.

### Node: Log Rejected Items
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists rejected items in an n8n Data Table.
- **Configuration choices:**  
  - Auto-map input data
  - Data Table ID placeholder must be configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Prepare Rejected Record**.
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Invalid Data Table ID
  - Column inference may create schema inconsistency over time
- **Sub-workflow reference:**  
  None.

---

## 2.6 Approved Record Processing

**Overview:**  
This block prepares approved decisions for storage and fans them out to multiple downstream consumers: internal persistence, dashboarding, email reporting, enterprise sync, and lineage tracking.

**Nodes Involved:**  
- Prepare Approved Record
- Store Approved Sustainability Records
- Update ESG Dashboard
- Send Compliance Report
- Sync to Enterprise ESG Platform

### Node: Prepare Approved Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Converts AI output into a flat approved-record schema for downstream systems.
- **Configuration choices:**  
  Produces fields such as:
  - `approvalStatus`
  - `complianceScore`
  - `carbonFootprint`
  - `energyRating`
  - `resourceScore`
  - `greenCompliance`
  - serialized ESG requirements, KPIs, and governance actions
  - `auditTrail`
  - `riskLevel`
  - current timestamp
- **Key expressions or variables used:**  
  - `{{ $json.carbonFootprintValidated }}`
  - `{{ JSON.stringify($json.esgRequirementsMet) }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**  
  Input from **Route by Approval Status** on the `approved` branch; output to **Store Approved Sustainability Records**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Missing fields from AI output
  - Boolean field coercion may fail if AI returns strings instead of booleans
- **Sub-workflow reference:**  
  None.

### Node: Store Approved Sustainability Records
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Stores approved items internally.
- **Configuration choices:**  
  - Auto-map input data
  - Uses placeholder Data Table ID
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Prepare Approved Record**. Outputs to:
  - **Update ESG Dashboard**
  - **Send Compliance Report**
  - **Sync to Enterprise ESG Platform**
  - **Prepare Lineage Record**
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Same Data Table concerns as above
  - A failure here prevents all downstream approved-side actions
- **Sub-workflow reference:**  
  None.

### Node: Update ESG Dashboard
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends approved record data to a Google Sheets dashboard/log.
- **Configuration choices:**  
  - Operation: `append`
  - Document ID: not configured
  - Sheet name: not configured
- **Key expressions or variables used:**  
  None shown explicitly.
- **Input and output connections:**  
  Input from **Store Approved Sustainability Records**.
- **Version-specific requirements:**  
  Type version `4.7`; requires Google Sheets OAuth2.
- **Edge cases or potential failure types:**  
  - Missing spreadsheet configuration
  - Header mismatch
  - Permission errors
  - Append format issues
- **Sub-workflow reference:**  
  None.

### Node: Send Compliance Report
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email report to stakeholders for approved records.
- **Configuration choices:**  
  - Recipient is placeholder email
  - Subject includes approval status and current date
  - Message body contains governance and metric summary
- **Key expressions or variables used:**  
  - `{{ $json.approvalStatus.toUpperCase() }}`
  - `{{ JSON.stringify($json.esgRequirements, null, 2) }}`
  - `{{ $json.greenCompliance ? 'Yes' : 'No' }}`
- **Input and output connections:**  
  Input from **Store Approved Sustainability Records**.
- **Version-specific requirements:**  
  Type version `2.2`; requires Gmail OAuth2.
- **Edge cases or potential failure types:**  
  - Placeholder recipient must be replaced
  - Gmail quota/auth issues
  - Serialized fields are strings already, so pretty-printing may not behave as intended
- **Sub-workflow reference:**  
  None.

### Node: Sync to Enterprise ESG Platform
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Pushes approved records to an external ESG platform API.
- **Configuration choices:**  
  - Method: `POST`
  - Body enabled
  - Sends all approved record fields
  - `onError` set to `continueErrorOutput`
- **Key expressions or variables used:**  
  Uses prepared approved record fields such as:
  - `$json.approvalStatus`
  - `$json.complianceScore`
  - `$json.kpis`
  - `$json.timestamp`
- **Input and output connections:**  
  Input from **Store Approved Sustainability Records**.  
  Success output is unused. Error output goes to **Prepare Error Record**.
- **Version-specific requirements:**  
  Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Placeholder endpoint must be replaced
  - Missing auth headers if target API requires them
  - Non-2xx responses go to error handling
  - Success path currently has no further node
- **Sub-workflow reference:**  
  None.

---

## 2.7 Sync Error Handling

**Overview:**  
This block captures enterprise sync failures without stopping the overall workflow. It converts the error into a structured record, logs it, and alerts operators in Slack.

**Nodes Involved:**  
- Prepare Error Record
- Log API Sync Errors
- Alert on Sync Failure

### Node: Prepare Error Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts relevant sync failure details and combines them with the approved record context.
- **Configuration choices:**  
  Stores:
  - error message
  - error type
  - approval status
  - compliance score
  - carbon footprint
  - timestamp
  - sync target label
- **Key expressions or variables used:**  
  - `{{ $json.error.message }}`
  - `{{ $('Prepare Approved Record').item.json.complianceScore }}`
- **Input and output connections:**  
  Input from error output of **Sync to Enterprise ESG Platform**; outputs to:
  - **Log API Sync Errors**
  - **Alert on Sync Failure**
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Error object shape may vary by request failure type
  - Cross-node item lookup can fail if execution context changes unexpectedly
- **Sub-workflow reference:**  
  None.

### Node: Log API Sync Errors
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists API synchronization failures.
- **Configuration choices:**  
  - Auto-map input data
  - Uses placeholder Data Table ID
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Prepare Error Record**.
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Invalid table ID or schema issues.
- **Sub-workflow reference:**  
  None.

### Node: Alert on Sync Failure
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an operational alert when enterprise sync fails.
- **Configuration choices:**  
  Sends a formatted message containing:
  - error message
  - timestamp
  - approved record details
- **Key expressions or variables used:**  
  - `{{ $json.error.message }}`
  - `{{ $('Prepare Approved Record').item.json.carbonFootprint }}`
- **Input and output connections:**  
  Input from **Prepare Error Record**.
- **Version-specific requirements:**  
  Type version `2.4`; requires Slack OAuth2.
- **Edge cases or potential failure types:**  
  - Placeholder channel ID must be configured
  - If error shape differs, the Slack expression may fail
- **Sub-workflow reference:**  
  None.

---

## 2.8 Environmental Lineage and KPI Tracking

**Overview:**  
This block creates traceability records for approved ESG decisions and tracks sustainability KPI performance over time. It compares original input values with validated/approved values and stores a KPI-oriented snapshot.

**Nodes Involved:**  
- Prepare Lineage Record
- Track Environmental Impact Lineage
- Prepare KPI Record
- Track Green KPI Performance

### Node: Prepare Lineage Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds a lineage/audit record joining original normalized input and approved output.
- **Configuration choices:**  
  Stores:
  - generated record ID
  - original event type/source/timestamp
  - validation timestamp
  - approval and score
  - original vs validated carbon/energy/resource fields
  - governance actions
  - audit trail
  - risk level
- **Key expressions or variables used:**  
  Cross-node references:
  - `{{ $('Normalize Sustainability Input').item.json.eventType }}`
  - `{{ $('Normalize Sustainability Input').item.json.carbonFootprint }}`
  - `{{ JSON.stringify($json.governanceActions) }}`
- **Input and output connections:**  
  Input from **Store Approved Sustainability Records**; output to **Track Environmental Impact Lineage**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Relies on cross-node item references
  - If multiple items are processed in parallel, item linking must remain consistent
- **Sub-workflow reference:**  
  None.

### Node: Track Environmental Impact Lineage
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Stores audit lineage records.
- **Configuration choices:**  
  - Auto-map input data
  - Uses placeholder Data Table ID
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Prepare Lineage Record**; output to **Prepare KPI Record**.
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Data table misconfiguration.
- **Sub-workflow reference:**  
  None.

### Node: Prepare KPI Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Converts approved/lineage data into a KPI tracking row.
- **Configuration choices:**  
  Stores:
  - current timestamp
  - KPI type
  - compliance score
  - carbon footprint
  - energy rating
  - resource score
  - green compliance flag
  - risk level
  - KPI details
  - period in `yyyy-MM`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.kpis) }}`
  - `{{ $now.toFormat('yyyy-MM') }}`
- **Input and output connections:**  
  Input from **Track Environmental Impact Lineage**; output to **Track Green KPI Performance**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  If `kpis` was already serialized or missing, KPI details may be malformed.
- **Sub-workflow reference:**  
  None.

### Node: Track Green KPI Performance
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists KPI history records.
- **Configuration choices:**  
  - Auto-map input data
  - Uses placeholder Data Table ID
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Prepare KPI Record**.
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  Data table configuration/schema drift.
- **Sub-workflow reference:**  
  None.

---

## 2.9 Sticky Notes and Embedded Documentation

**Overview:**  
These nodes document the workflow’s purpose, prerequisites, and logical sections. They do not execute but provide important implementation context.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node group: Sticky notes
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotations for maintainers.
- **Configuration choices:**  
  Cover:
  - overall workflow explanation
  - prerequisites
  - setup steps
  - normalization rationale
  - AI orchestration rationale
  - routing rationale
  - sync/lineage/KPI rationale
  - dashboard rationale
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Periodic Sustainability Monitoring | scheduleTrigger | Scheduled ESG monitoring entry point |  | Normalize Sustainability Input | ## Normalise Sustainability Input<br>**Why** — Standardises heterogeneous ESG data before AI processing to prevent scoring errors. |
| External ESG Platform Events | webhook | External ESG event intake endpoint |  | Normalize Sustainability Input | ## Normalise Sustainability Input<br>**Why** — Standardises heterogeneous ESG data before AI processing to prevent scoring errors. |
| Normalize Sustainability Input | set | Standardizes incoming ESG payload fields | Periodic Sustainability Monitoring; External ESG Platform Events | Green Governance Agent | ## Normalise Sustainability Input<br>**Why** — Standardises heterogeneous ESG data before AI processing to prevent scoring errors. |
| Green Governance Agent | langchain agent | Main AI governance orchestrator | Normalize Sustainability Input | Route by Approval Status | ## Green Governance Agent<br>**Why** — Orchestrates ESG scoring, compliance checks, and documentation generation in one coordinated pass. |
| Governance Model | lmChatOpenAi | LLM backend for governance agent |  | Green Governance Agent | ## Green Governance Agent<br>**Why** — Orchestrates ESG scoring, compliance checks, and documentation generation in one coordinated pass. |
| ESG Compliance Output Schema | outputParserStructured | Structured parser for agent response |  | Green Governance Agent | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Sustainability Oversight Agent Tool | agentTool | Nested validation and oversight tool | Oversight Model; Sustainability Metrics Calculator; ESG Scoring Engine; Multi-Cloud Sustainability API; ESG Reporting Sheets Tool | Green Governance Agent | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Oversight Model | lmChatOpenAi | LLM backend for oversight tool agent |  | Sustainability Oversight Agent Tool | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Sustainability Metrics Calculator | toolCalculator | AI-callable math utility |  | Sustainability Oversight Agent Tool | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| ESG Scoring Engine | toolCode | Custom ESG scoring logic |  | Sustainability Oversight Agent Tool | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Multi-Cloud Sustainability API | httpRequestTool | AI-callable external sustainability API lookup |  | Sustainability Oversight Agent Tool | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| ESG Reporting Sheets Tool | googleSheetsTool | AI-callable ESG benchmarks reader |  | Sustainability Oversight Agent Tool | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Governance Alerts Tool | slackTool | AI-callable Slack alert sender |  | Green Governance Agent | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Compliance Documentation Tool | airtableTool | AI-callable compliance record creator |  | Green Governance Agent | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Route by Approval Status | switch | Routes output by approval decision | Green Governance Agent | Prepare Approved Record; Send Review Request; Prepare Rejected Record | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Prepare Approved Record | set | Shapes approved record payload | Route by Approval Status | Store Approved Sustainability Records | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Store Approved Sustainability Records | dataTable | Stores approved records internally | Prepare Approved Record | Update ESG Dashboard; Send Compliance Report; Sync to Enterprise ESG Platform; Prepare Lineage Record | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Send Review Request | slack | Sends manual review notification | Route by Approval Status |  | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Prepare Rejected Record | set | Shapes rejected item log payload | Route by Approval Status | Log Rejected Items | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Log Rejected Items | dataTable | Stores rejected records | Prepare Rejected Record |  | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Update ESG Dashboard | googleSheets | Appends approved data to dashboard | Store Approved Sustainability Records |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting.<br>## Update ESG Dashboard<br>**Why** — Keeps the Google Sheets dashboard current for real-time stakeholder visibility. |
| Send Compliance Report | gmail | Emails compliance report to stakeholders | Store Approved Sustainability Records |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Sync to Enterprise ESG Platform | httpRequest | Pushes approved record to enterprise ESG API | Store Approved Sustainability Records | Prepare Error Record (error output) | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Prepare Lineage Record | set | Builds environmental lineage audit row | Store Approved Sustainability Records | Track Environmental Impact Lineage | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Track Environmental Impact Lineage | dataTable | Stores lineage records | Prepare Lineage Record | Prepare KPI Record | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Prepare Error Record | set | Builds structured sync failure record | Sync to Enterprise ESG Platform (error output) | Log API Sync Errors; Alert on Sync Failure | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Log API Sync Errors | dataTable | Stores API sync failures | Prepare Error Record |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Alert on Sync Failure | slack | Sends Slack alert for sync failures | Prepare Error Record |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Prepare KPI Record | set | Shapes KPI tracking row | Track Environmental Impact Lineage | Track Green KPI Performance | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting.<br>## Update ESG Dashboard<br>**Why** — Keeps the Google Sheets dashboard current for real-time stakeholder visibility. |
| Track Green KPI Performance | dataTable | Stores KPI performance history | Prepare KPI Record |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting.<br>## Update ESG Dashboard<br>**Why** — Keeps the Google Sheets dashboard current for real-time stakeholder visibility. |
| Sticky Note | stickyNote | Canvas documentation |  |  | ## Sustainability Oversight Sub-Agent & ESG Scoring Engine<br>**Why** — Applies governance rules and quantitative scoring to surface non-compliant records early. |
| Sticky Note1 | stickyNote | Canvas documentation |  |  | ## Synchronise, Track Environmental Impact Lineage & Green KPI Performance<br>**Why** — Maintains auditable data lineage and performance records required for regulatory reporting. |
| Sticky Note2 | stickyNote | Canvas documentation |  |  | ## Route by Approval Status<br>**Why** — Rules-based routing separates rejected, under-review, and approved records for appropriate handling. |
| Sticky Note3 | stickyNote | Canvas documentation |  |  | ## Green Governance Agent<br>**Why** — Orchestrates ESG scoring, compliance checks, and documentation generation in one coordinated pass. |
| Sticky Note4 | stickyNote | Canvas documentation |  |  | ## Normalise Sustainability Input<br>**Why** — Standardises heterogeneous ESG data before AI processing to prevent scoring errors. |
| Sticky Note5 | stickyNote | Canvas documentation |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Google Sheets with ESG log tabs pre-created<br>- Enterprise ESG platform API endpoint access<br>## Use Cases<br>- Corporations automating quarterly ESG compliance report generation<br>## Customisation<br>- Swap ESG Scoring Engine thresholds to match regional regulatory frameworks (EU Taxonomy, GRI, SASB)<br>## Benefits<br>- Eliminates manual ESG data aggregation, cutting reporting cycle time significantly |
| Sticky Note6 | stickyNote | Canvas documentation |  |  | ## Setup Steps<br>1. Import workflow and configure the periodic trigger interval and ESG platform webhook URL.<br>2. Add AI model credentials to the Green Governance Agent and Sustainability Oversight Sub-Agent.<br>3. Connect Slack credentials to Governance Alerts Tool and Sync Failure Alert nodes.<br>4. Link Google Sheets credentials; set sheet IDs for Rejected Items, etc.<br>5. Configure the Multi-Cloud Sustainability API and Enterprise ESG Platform sync endpoint URLs.<br>6. Set ESG scoring thresholds in the ESG Scoring Engine node. |
| Sticky Note7 | stickyNote | Canvas documentation |  |  | ## How It Works<br>This workflow automates end-to-end ESG (Environmental, Social, and Governance) sustainability reporting for enterprise sustainability teams, compliance officers, and green governance leads. It solves the challenge of manually aggregating multi-source ESG data, applying scoring logic, and routing records through approval chains, a process that is slow, error-prone, and difficult to audit. Sustainability data enters via two sources: a periodic scheduler and an external ESG platform webhook. Inputs are normalised and passed to a Green Governance Agent equipped with a Sustainability Oversight Sub-Agent, ESG Scoring Engine, Multi-Cloud Sustainability API, Governance Alerts Tool, Compliance Documentation Tool, and ESG Reporting Sheets Tool. The agent produces a structured compliance output, which is then routed by approval status, rejected records are logged, review requests are sent via Slack, and approved records are stored and synced to the enterprise ESG platform. Sync errors trigger Slack alerts and error logging. Approved data simultaneously updates environmental impact lineage, KPI performance tracking, and the ESG dashboard in Google Sheets. |
| Sticky Note8 | stickyNote | Canvas documentation |  |  | ## Update ESG Dashboard<br>**Why** — Keeps the Google Sheets dashboard current for real-time stakeholder visibility. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name such as:  
   `AI-powered ESG sustainability governance with compliance routing and tracking`.

2. **Add a Schedule Trigger node** named **Periodic Sustainability Monitoring**.
   - Type: `Schedule Trigger`
   - Set interval to every `6 hours`.

3. **Add a Webhook node** named **External ESG Platform Events**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `sustainability-events`
   - Decide whether to add authentication in production, since this example does not.

4. **Add a Set node** named **Normalize Sustainability Input**.
   - Create the following fields:
     - `eventType` as string: `{{ $json.eventType || 'scheduled_monitoring' }}`
     - `carbonFootprint` as number: `{{ $json.carbonFootprint || $json.metrics?.carbonFootprint || 0 }}`
     - `energyConsumption` as number: `{{ $json.energyConsumption || $json.metrics?.energyConsumption || 0 }}`
     - `resourceUtilization` as number: `{{ $json.resourceUtilization || $json.metrics?.resourceUtilization || 0 }}`
     - `complianceSignals` as array: `{{ $json.complianceSignals || $json.metrics?.complianceSignals || [] }}`
     - `timestamp` as string: `{{ $json.timestamp || $now.toISO() }}`
     - `source` as string: `{{ $json.source || 'internal' }}`
   - Connect both triggers to this node.

5. **Add an OpenAI Chat Model node** named **Governance Model**.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach valid OpenAI credentials.

6. **Add a Structured Output Parser node** named **ESG Compliance Output Schema**.
   - Use a structured schema matching fields such as:
     - `approvalStatus`
     - `complianceScore`
     - `carbonFootprintValidated`
     - `energyEfficiencyRating`
     - `resourceUtilizationScore`
     - `greenPolicyCompliance`
     - `esgRequirementsMet`
     - `sustainabilityKPIs`
     - `governanceActions`
     - `auditTrail`
     - `riskLevel`
     - `recommendations`

7. **Add an AI Agent node** named **Green Governance Agent**.
   - Set prompt mode to define text manually.
   - Input text:
     `{{ $json.eventType + ': Carbon=' + $json.carbonFootprint + 'kg, Energy=' + $json.energyConsumption + 'kWh, Resources=' + $json.resourceUtilization + '%, Compliance Signals=' + JSON.stringify($json.complianceSignals) + ', Source=' + $json.source + ', Timestamp=' + $json.timestamp }}`
   - Add a system message describing the Green Governance Agent’s role in sustainability approvals, oversight coordination, KPI tracking, governance automation, audit documentation, access controls, and compliance.
   - Enable structured output parsing.
   - Connect:
     - **Normalize Sustainability Input** → main input
     - **Governance Model** → AI language model
     - **ESG Compliance Output Schema** → AI output parser

8. **Add another OpenAI Chat Model node** named **Oversight Model**.
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Reuse or separately configure OpenAI credentials.

9. **Add an AI Agent Tool node** named **Sustainability Oversight Agent Tool**.
   - Input text:
     `{{ $fromAI('metrics_data', 'Environmental metrics and compliance data to validate') }}`
   - Tool description: explain that it validates carbon footprint, energy use, resource utilization, and green-policy compliance.
   - Add a detailed system message focused on ESG validation rigor.
   - Connect **Oversight Model** as its AI language model.

10. **Add a Calculator Tool node** named **Sustainability Metrics Calculator**.
    - Keep default configuration.
    - Connect it as an AI tool to **Sustainability Oversight Agent Tool**.

11. **Add a Code Tool node** named **ESG Scoring Engine**.
    - Paste the custom scoring JavaScript from the design:
      - parse JSON/object input,
      - calculate carbon score,
      - calculate energy score,
      - use resource utilization as resource score,
      - compute weighted ESG score,
      - derive energy rating,
      - derive risk level,
      - derive compliance status,
      - return validated metrics and KPI object.
    - Connect it as an AI tool to **Sustainability Oversight Agent Tool**.

12. **Add an HTTP Request Tool node** named **Multi-Cloud Sustainability API**.
    - URL expression:
      `{{ $fromAI('api_endpoint', 'The API endpoint to query for sustainability data', 'string') }}`
    - Keep default request options unless your API requires auth headers.
    - Connect it as an AI tool to **Sustainability Oversight Agent Tool**.

13. **Add a Google Sheets Tool node** named **ESG Reporting Sheets Tool**.
    - Configure Google Sheets OAuth2 credentials.
    - Select the spreadsheet and sheet containing ESG thresholds or benchmark baselines.
    - Add a manual description stating it reads ESG thresholds, industry benchmarks, and baseline data.
    - Connect it as an AI tool to **Sustainability Oversight Agent Tool**.

14. **Add a Slack Tool node** named **Governance Alerts Tool**.
    - Configure Slack OAuth2 credentials.
    - Set message text:
      `{{ $fromAI('alert_message', 'The alert message content', 'string') }}`
    - Set channel ID dynamically:
      `{{ $fromAI('slack_channel', 'The Slack channel to send the alert to', 'string') }}`
    - Add a tool description explaining it sends sustainability and governance alerts.
    - Connect it as an AI tool to **Green Governance Agent**.

15. **Add an Airtable Tool node** named **Compliance Documentation Tool**.
    - Configure Airtable credentials.
    - Set operation to `Create`.
    - Select the Airtable base.
    - Configure the table dynamically if you want to preserve the original design:
      `{{ $fromAI('table_name', 'The Airtable table to store compliance documentation', 'string') }}`
    - Connect it as an AI tool to **Green Governance Agent**.

16. **Connect Sustainability Oversight Agent Tool** as an AI tool to **Green Governance Agent**.

17. **Add a Switch node** named **Route by Approval Status**.
    - Compare `{{ $json.approvalStatus }}`
    - Rule 1: equals `approved`
    - Rule 2: equals `needs_review`
    - Rule 3: equals `rejected`
    - Connect **Green Governance Agent** to it.

18. **Add a Slack node** named **Send Review Request**.
    - Configure Slack OAuth2 credentials.
    - Choose the review channel.
    - Compose the message with:
      - compliance score
      - validated carbon footprint
      - energy rating
      - risk level
      - ESG requirements
      - recommendations
      - audit trail
    - Connect it to the `needs_review` output of the Switch.

19. **Add a Set node** named **Prepare Rejected Record**.
    - Map:
      - `approvalStatus`
      - `complianceScore`
      - `carbonFootprint` from `carbonFootprintValidated`
      - `riskLevel`
      - `rejectionReasons` from `JSON.stringify($json.recommendations)`
      - `auditTrail`
      - `timestamp` as `{{ $now.toISO() }}`
    - Connect it to the `rejected` output.

20. **Add a Data Table node** named **Log Rejected Items**.
    - Configure or create an n8n Data Table for rejected items.
    - Use auto-map input data.
    - Connect **Prepare Rejected Record** to it.

21. **Add a Set node** named **Prepare Approved Record**.
    - Map:
      - `approvalStatus`
      - `complianceScore`
      - `carbonFootprint` from `carbonFootprintValidated`
      - `energyRating`
      - `resourceScore` from `resourceUtilizationScore`
      - `greenCompliance` from `greenPolicyCompliance`
      - `esgRequirements` as serialized JSON
      - `kpis` as serialized JSON
      - `governanceActions` as serialized JSON
      - `auditTrail`
      - `riskLevel`
      - `timestamp` as current ISO date
    - Connect it to the `approved` output.

22. **Add a Data Table node** named **Store Approved Sustainability Records**.
    - Create/select an n8n Data Table for approved records.
    - Enable auto-map input.
    - Connect **Prepare Approved Record** to it.

23. **Add a Google Sheets node** named **Update ESG Dashboard**.
    - Operation: `Append`
    - Configure Google Sheets OAuth2 credentials.
    - Select the dashboard spreadsheet and target tab.
    - Connect from **Store Approved Sustainability Records**.

24. **Add a Gmail node** named **Send Compliance Report**.
    - Configure Gmail OAuth2 credentials.
    - Set recipient email.
    - Set subject such as:
      `ESG Compliance Report - {{ $json.approvalStatus.toUpperCase() }} - {{ $now.toFormat('yyyy-MM-dd') }}`
    - Build the body with environmental metrics, ESG requirements, KPIs, governance actions, and audit trail.
    - Connect from **Store Approved Sustainability Records**.

25. **Add an HTTP Request node** named **Sync to Enterprise ESG Platform**.
    - Method: `POST`
    - URL: your enterprise ESG platform endpoint
    - Enable request body
    - Send fields from the approved record
    - Set node error handling to continue using the error output
    - Connect from **Store Approved Sustainability Records**.

26. **Add a Set node** named **Prepare Error Record**.
    - Map:
      - `errorMessage` from `{{ $json.error.message }}`
      - `errorType` from `{{ $json.error.name }}`
      - approved context from **Prepare Approved Record**
      - timestamp
      - `syncAttempt` = `enterprise_esg_platform`
    - Connect this to the error output of **Sync to Enterprise ESG Platform**.

27. **Add a Data Table node** named **Log API Sync Errors**.
    - Create/select a Data Table for API sync errors.
    - Auto-map input.
    - Connect from **Prepare Error Record**.

28. **Add a Slack node** named **Alert on Sync Failure**.
    - Configure Slack OAuth2 credentials.
    - Choose an alerts channel.
    - Compose a message including:
      - error message
      - timestamp
      - approval status
      - compliance score
      - carbon footprint
    - Connect from **Prepare Error Record**.

29. **Add a Set node** named **Prepare Lineage Record**.
    - Build a lineage record using both:
      - approved output fields,
      - original normalized input fields referenced via `$('Normalize Sustainability Input')`.
    - Include:
      - generated record ID
      - event type, source, input timestamp
      - validation timestamp
      - original vs validated values
      - governance actions
      - audit trail
      - risk level
    - Connect from **Store Approved Sustainability Records**.

30. **Add a Data Table node** named **Track Environmental Impact Lineage**.
    - Create/select a lineage Data Table.
    - Auto-map input.
    - Connect from **Prepare Lineage Record**.

31. **Add a Set node** named **Prepare KPI Record**.
    - Map:
      - current timestamp
      - `kpiType = sustainability_performance`
      - compliance score
      - carbon footprint
      - energy rating
      - resource score
      - green compliance
      - risk level
      - KPI details
      - period as `{{ $now.toFormat('yyyy-MM') }}`
    - Connect from **Track Environmental Impact Lineage**.

32. **Add a Data Table node** named **Track Green KPI Performance**.
    - Create/select a Data Table for KPI history.
    - Auto-map input.
    - Connect from **Prepare KPI Record**.

33. **Optionally add sticky notes** to document:
    - overall workflow purpose,
    - prerequisites,
    - setup steps,
    - normalization,
    - governance AI,
    - routing,
    - sync/lineage/KPI tracking,
    - dashboard update purpose.

34. **Replace all placeholder values** before activation:
    - approved records Data Table ID
    - rejected items Data Table ID
    - API sync errors Data Table ID
    - lineage Data Table ID
    - KPI Data Table ID
    - Slack review channel ID
    - Slack alerts channel ID
    - stakeholder email
    - enterprise ESG platform API URL
    - Google Sheets document IDs and tab names
    - Airtable base/table details

35. **Test each entry path separately**:
    - run the scheduler manually,
    - send a sample POST request to the webhook.

36. **Validate the AI output schema carefully**.
    - Ensure `approvalStatus` is always one of:
      - `approved`
      - `needs_review`
      - `rejected`
    - Ensure `recommendations` is always an array.
    - Ensure booleans and numbers are not returned as strings unless downstream nodes are adjusted.

37. **Activate the workflow** once credentials, sheets, channels, tables, and endpoint URLs are configured.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: OpenAI API key or compatible LLM, Slack workspace with bot credentials, Google Sheets with ESG log tabs pre-created, enterprise ESG platform API endpoint access | Project setup |
| Use case: corporations automating quarterly ESG compliance report generation | Business context |
| Customization: adjust ESG Scoring Engine thresholds to match regional frameworks such as EU Taxonomy, GRI, or SASB | Scoring and compliance adaptation |
| Benefit: reduces manual ESG data aggregation and shortens reporting cycle time | Operational value |
| Setup guidance: configure trigger interval, webhook URL, AI credentials, Slack credentials, Google Sheets IDs, API endpoints, and scoring thresholds | Implementation notes |
| Workflow purpose: automate end-to-end ESG sustainability reporting, scoring, routing, documentation, lineage, KPI tracking, and dashboard updates | Overall design context |