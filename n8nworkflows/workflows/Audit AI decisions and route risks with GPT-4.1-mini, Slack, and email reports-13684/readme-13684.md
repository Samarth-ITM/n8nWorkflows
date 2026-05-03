Audit AI decisions and route risks with GPT-4.1-mini, Slack, and email reports

https://n8nworkflows.xyz/workflows/audit-ai-decisions-and-route-risks-with-gpt-4-1-mini--slack--and-email-reports-13684


# Audit AI decisions and route risks with GPT-4.1-mini, Slack, and email reports

This document provides a technical breakdown of the **AI Decision Governance Auditor** workflow. This system automates the auditing of AI-driven decisions by evaluating risk and compliance through a multi-agent orchestration layer, providing real-time alerts for high-risk outcomes and detailed explainability reports.

---

### 1. Workflow Overview

The workflow is designed to ensure accountability in automated decision-making (e.g., financial approvals). It follows a linear progression from data ingestion to multi-layered AI analysis, ending with conditional routing based on risk severity.

**Logical Blocks:**
*   **1.1 Input & Environment Setup:** Triggers the workflow and defines global configuration variables (thresholds, contact points) and simulated decision data.
*   **1.2 Decision Tracing:** Extracts and structures raw input into a standardized metadata format for auditability.
*   **1.3 Governance Orchestration:** A central agent manages two specialized sub-agents (Risk and Compliance) to synthesize a final governance verdict.
*   **1.4 Risk Routing & Alerting:** Evaluates the risk score against thresholds to trigger immediate Slack notifications for critical cases.
*   **1.5 Reporting & Archiving:** Generates a comprehensive HTML email report and persists the audit trail into data tables for regulatory review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Environment Setup
Initializes the audit process with either scheduled triggers or simulated data.
*   **Nodes Involved:** `Schedule Trigger`, `Workflow Configuration`, `Simulate Decision Request`.
*   **Node Details:**
    *   **Workflow Configuration (Set):** Defines `riskThresholdHigh` (75) and `riskThresholdCritical` (90). Sets placeholders for Slack and Email.
    *   **Simulate Decision Request (Set):** Generates a mock "Financial Approval" request ($150,000 for Engineering) with a unique ID and justification.
*   **Edge Cases:** Missing placeholder values will cause failures in downstream notification nodes.

#### 2.2 Decision Tracing
Standardizes input data into a machine-readable schema.
*   **Nodes Involved:** `Decision Trace Agent`, `OpenAI Model - Decision Trace`, `Decision Metadata Parser`.
*   **Node Details:**
    *   **Decision Trace Agent:** Uses a system message to identify stakeholders and rationale.
    *   **Metadata Parser:** Enforces a JSON schema containing `decisionId`, `stakeholders`, and `rationale`.
*   **Technical Role:** Acts as the "Data Cleaning" layer before analysis begins.

#### 2.3 Governance Orchestration (The "Brain")
Synthesizes findings from specialized domains.
*   **Nodes Involved:** `Governance Agent`, `OpenAI Model - Governance`, `Governance Decision Parser`.
*   **Node Details:**
    *   **Governance Agent:** Orchestrates the sub-tools. It is configured to *not* return a final answer until both Risk and Compliance tools have been consulted.
    *   **Governance Decision Parser:** A complex manual schema requiring a `governanceDecision`, `overallRiskLevel`, and a boolean `escalationRequired`.

#### 2.4 Risk & Compliance Tools (Sub-Agents)
Functional sub-units called by the Governance Agent.
*   **Nodes Involved:** `Risk Assessment Agent Tool`, `Compliance Checker Agent Tool` (and their respective models/parsers).
*   **Node Details:**
    *   **Risk Assessment:** Calculates a 0–100 score based on financial and operational impact.
    *   **Compliance Checker:** Validates against regulations (GDPR, SOX) and internal delegation limits.
*   **Failure Types:** LLM timeouts during tool calling; schema mismatch if the model produces hallucinated fields.

#### 2.5 Risk Routing & Alerting
Conditional logic for escalation.
*   **Nodes Involved:** `Route by Risk Level`, `Store High Risk Decisions`, `Notify High Risk Alert`.
*   **Node Details:**
    *   **Route by Risk Level (Switch):** Compares the AI-generated `riskScore` against the variables set in Block 1.1.
    *   **Notify High Risk Alert (Slack):** Sends a formatted block message to the configured channel if risk is $\ge 75$.

#### 2.6 Reporting & Archiving
Final output and persistence.
*   **Nodes Involved:** `Send Governance Report`, `Store Decision Audit Trail`, `Store Explainability Report`.
*   **Node Details:**
    *   **Send Governance Report (Email):** A dynamic HTML template that changes color based on risk (Red for CRITICAL, Yellow for HIGH). Includes the full `auditTrail`.
    *   **DataTables:** Three separate tables (`DecisionAuditTrail`, `HighRiskDecisions`, `ExplainabilityReports`) are used to store the records.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Workflow Entry | (None) | Workflow Configuration | Set schedule trigger interval to match governance audit frequency. |
| Workflow Configuration | n8n-nodes-base.set | Global Variables | Schedule Trigger | Simulate Decision Request | |
| Simulate Decision Request | n8n-nodes-base.set | Data Input | Workflow Configuration | Decision Trace Agent | Replace simulated decision request with live AI system webhook. |
| Decision Trace Agent | @n8n/n8n-nodes-langchain.agent | Metadata Extraction | Simulate Decision Request | Governance Agent | Extracts decision metadata using OpenAI with structured parsing. |
| Governance Agent | @n8n/n8n-nodes-langchain.agent | Audit Orchestrator | Decision Trace Agent | Route by Risk Level, Store Decision Audit Trail | Orchestrates Risk Assessment and Compliance Checker sub-agents. |
| Risk Assessment Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Risk Analysis | Governance Agent | (Tool Output) | |
| Compliance Checker Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Policy Validation | Governance Agent | (Tool Output) | |
| Route by Risk Level | n8n-nodes-base.switch | Logic Routing | Governance Agent | Store High Risk Decisions, Low/Med Path | Separates high-risk decisions from standard outcomes. |
| Notify High Risk Alert | n8n-nodes-base.slack | Critical Alerting | Store High Risk Decisions | Merge Notification Paths | Sends Slack alert and stores high-risk records separately. |
| Send Governance Report | n8n-nodes-base.emailSend | Reporting | Store High Risk Decisions | Merge Notification Paths | Emails governance report and stores explainability data. |
| Store Decision Audit Trail | n8n-nodes-base.dataTable | Data Archiving | Governance Agent | (None) | |
| Store High Risk Decisions | n8n-nodes-base.dataTable | Segmented Storage | Route by Risk Level | Notify High Risk Alert, Send Governance Report | Provides real-time escalation and isolated audit evidence. |
| Store Explainability Report | n8n-nodes-base.dataTable | Regulatory Compliance | Merge Notification Paths | (None) | Satisfies regulatory requirements for decision transparency. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create a `Schedule Trigger` (set to your preferred audit interval).
    *   Add a `Set` node ("Workflow Configuration") with four assignments: `slackChannelId` (string), `governanceEmail` (string), `riskThresholdHigh` (number: 75), and `riskThresholdCritical` (number: 90).
    *   Add another `Set` node to simulate or receive your input data (JSON object with ID, Type, Amount, etc.).

2.  **The Metadata Layer:**
    *   Create a `Decision Trace Agent`. Attach an `OpenAI Chat Model` (Model: `gpt-4.1-mini`).
    *   Attach a `Structured Output Parser` and define the JSON schema for decision metadata.

3.  **The Governance Layer (Advanced Agent):**
    *   Create the `Governance Agent`.
    *   Create two `Agent Tool` nodes: "Risk Assessment Agent Tool" and "Compliance Checker Agent Tool."
    *   **Risk Tool Config:** Attach its own OpenAI Model and a `Structured Output Parser` with fields for `riskScore` and `mitigationMeasures`.
    *   **Compliance Tool Config:** Attach its own OpenAI Model and a `Structured Output Parser` with fields for `complianceStatus` and `policyViolations`.
    *   Connect both tools to the `Governance Agent`.

4.  **Logic & Routing:**
    *   Connect the `Governance Agent` to a `Switch` node.
    *   **Rule 1:** `riskScore` >= `{{$node["Workflow Configuration"].json["riskThresholdHigh"]}}`.
    *   **Fallback:** Send all other results to the "Low/Medium" path.

5.  **Notifications & Persistence:**
    *   Set up a `Slack` node for the High Risk path using the `slackChannelId` variable.
    *   Set up an `Email Send` node for the report. Use an HTML template to display the `explainabilityReport` and `auditTrail` fields from the Governance Agent.
    *   Create three n8n `DataTables` or use an external database (PostgreSQL/Google Sheets) to map the output of the Governance Agent for permanent storage.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Needs Slack Bot Token, SMTP/Gmail credentials, and OpenAI API Key. |
| **Use Case: Finance** | Regulatory compliance auditing for AI-driven loan or insurance decisions. |
| **Use Case: HR** | Automated fairness and bias detection in hiring systems. |
| **Customization** | Swap simulated input with a "Webhook Trigger" for live API integration. |