Detect and enforce abuse cases with OpenAI, Slack, Gmail and Sheets

https://n8nworkflows.xyz/workflows/detect-and-enforce-abuse-cases-with-openai--slack--gmail-and-sheets-13700


# Detect and enforce abuse cases with OpenAI, Slack, Gmail and Sheets

This document provides a technical analysis and reproduction guide for the **Intelligent Abuse Detection and Enforcement Coordination System** workflow in n8n.

---

### 1. Workflow Overview
The workflow is an automated Trust and Safety system designed to handle platform abuse signals (spam, harassment, fraud) in real-time. It uses a multi-agent AI architecture to triage incoming reports, conduct deep investigations, verify policy compliance, and execute enforcement actions (logging, alerting, or escalating).

**Logical Blocks:**
*   **1.1 Input Reception & Config:** Captures incoming webhooks and sets operational thresholds (risk levels, notification channels).
*   **1.2 Initial Triage:** A "Behavior Signal Agent" analyzes the raw input to determine initial severity and risk.
*   **1.3 Advanced Governance:** For high-risk cases, a "Governance Agent" orchestrates specialized sub-tools for investigation, multi-factor risk scoring, and policy verification.
*   **1.4 Enforcement Logic:** Formats the final decision and routes it to the appropriate communication channel based on the severity of the enforcement action.
*   **1.5 Automated Response:** Triggers Slack alerts, emails, or logs to database tables (Google Sheets/n8n Data Tables).

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
*   **Overview:** Receives the abuse signal and initializes global variables.
*   **Nodes Involved:** `Abuse Signal Webhook`, `Workflow Configuration`.
*   **Node Details:**
    *   **Abuse Signal Webhook:** HTTP POST endpoint (`/abuse-signal`). Responds via the "Response Node" to confirm receipt.
    *   **Workflow Configuration (Set):** Defines static parameters used throughout the workflow: `riskThresholdHigh` (80), `riskThresholdMedium` (50), `autoActionThreshold` (30), `slackChannel`, and `escalationEmail`.

#### 2.2 Behavior Triage (Agentic)
*   **Overview:** Uses LLMs to classify the type of abuse and assess initial risk.
*   **Nodes Involved:** `Behavior Signal Agent`, `OpenAI Model - Behavior Agent`, `Behavior Signal Output Parser`.
*   **Node Details:**
    *   **Behavior Signal Agent:** A LangChain agent using a system prompt to extract User IDs, action types, and preliminary severity (CRITICAL, HIGH, MEDIUM, LOW).
    *   **Output Parser:** Forces the LLM to return a valid JSON structure including `riskScore`, `indicators`, and `reasoning`.

#### 2.3 Severity Routing
*   **Overview:** Directs the flow based on the AI's triage.
*   **Nodes Involved:** `Route by Severity`, `Check Auto-Action Threshold`.
*   **Node Details:**
    *   **Route by Severity (Switch):** "Critical" and "High" paths lead to the Governance Agent. "Medium" and "Low" paths are merged into "Low Risk" and sent for potential auto-action.

#### 2.4 Governance & Tool Orchestration
*   **Overview:** The core decision-making hub for complex cases.
*   **Nodes Involved:** `Governance Agent`, `Investigation Agent Tool`, `Risk Scoring Agent Tool`, `Policy Compliance Checker Tool`, and associated Models/Parsers.
*   **Node Details:**
    *   **Governance Agent:** Orchestrates the sub-agents to decide on: warn, suspend, ban, or monitor.
    *   **Investigation Tool:** Analyzes patterns, user history, and malicious intent.
    *   **Risk Scoring Tool:** Calculates a refined score (0-100) based on multi-factor analysis (frequency vs. impact).
    *   **Policy Compliance Checker (Tool Code):** A JavaScript-based tool that checks the case against hard-coded rules (e.g., if `violationCount` > 5, recommend `ban`).

#### 2.5 Action Execution
*   **Overview:** Final formatting and delivery of enforcement.
*   **Nodes Involved:** `Prepare Enforcement Data`, `Route Enforcement Action`, `Alert Security Team`, `Send Escalation Email`, `Log to Abuse Records`.
*   **Node Details:**
    *   **Prepare Enforcement Data (Code):** Merges data from the triage agent and the governance agent into a unified object.
    *   **Route Enforcement Action (Switch):** Branches based on whether the action is `monitor` (Log only), `warn/suspend` (Slack alert), or `ban/escalate` (Email).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Abuse Signal Webhook | Webhook | Entry Point | None | Workflow Configuration | |
| Workflow Configuration | Set | Global Vars | Webhook | Behavior Signal Agent | |
| Behavior Signal Agent | Agent | AI Triage | Configuration | Route by Severity | Classifies signals by severity using OpenAI. |
| Route by Severity | Switch | Logic Routing | Behavior Agent | Governance Agent, Check Threshold | Prevents low-priority signals from consuming resources. |
| Governance Agent | Agent | Orchestrator | Severity Switch | Prepare Enf. Data | Centralises enforcement logic for consistent decisions. |
| Investigation Agent Tool | Agent Tool | Sub-Analysis | Governance Agent | Governance Agent | Conducts deep investigation of abuse patterns. |
| Risk Scoring Agent Tool | Agent Tool | Sub-Analysis | Governance Agent | Governance Agent | Calculates refined risk scores. |
| Policy Compliance Checker | Tool Code | Rule Check | Governance Agent | Governance Agent | Verifies policy compliance against rules. |
| Prepare Enforcement Data | Code | Data Merging | Governance Agent | Route Enf. Action | Formats data and routes by action type. |
| Route Enforcement Action | Switch | Communication Path | Prepare Data | Log, Slack, Email | Ensures case receives the correct response. |
| Alert Security Team | Slack | Notification | Route Action | Log Enf. Actions | Slack workspace with bot token required. |
| Send Escalation Email | Email | Notification | Route Action | Log Enf. Actions | Gmail or SMTP credentials required. |
| Log to Abuse Records | Data Table | Record Keeping | Route Action | Log Enf. Actions | Google Sheets/Data Tables for logging. |
| Check Auto-Action Threshold | If | Threshold Logic | Severity Switch | Format Auto-Action | Prevents over-enforcement. |
| Log Enforcement Actions | Data Table | Final Sink | All Actions | None | Shared logging for all outcomes. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Create two n8n Data Tables: `AbuseRecords` and `EnforcementActions`. Ensure you have an OpenAI API key.
2.  **Initial Webhook:** Add a Webhook node, set method to **POST**, and path to `abuse-signal`.
3.  **Global Constants:** Add a **Set** node named `Workflow Configuration`. Create number variables for `riskThresholdHigh` (80) and `autoActionThreshold` (30). Add string variables for `slackChannel` ID and `escalationEmail`.
4.  **Triage Agent:**
    *   Add a **LangChain Agent** node.
    *   Connect an **OpenAI Chat Model** node (use GPT-4o or GPT-4o-mini).
    *   Connect a **Structured Output Parser**. Define the schema with fields: `userId`, `riskScore`, `severityLevel`, and `reasoning`.
    *   Set the System Message to: "Validate abuse data and classify severity."
5.  **Severity Switch:** Add a **Switch** node. Create three outputs:
    *   `severityLevel` equals "CRITICAL"
    *   `severityLevel` equals "HIGH"
    *   `severityLevel` equals "MEDIUM" OR "LOW"
6.  **Governance Orchestration:**
    *   Add another **Agent** node (`Governance Agent`).
    *   Connect three **Tool** nodes:
        *   `Investigation Tool`: Use another Agent Tool with a prompt for "Deep pattern analysis."
        *   `Risk Scoring Tool`: Use an Agent Tool to "Calculate refined multi-factor risk."
        *   `Policy Checker`: Use a **Tool Code** node. Paste JS logic to compare `violationCount` against thresholds (e.g., > 3 = suspend).
7.  **Data Preparation:** Add a **Code** node to consolidate `{{ $json }}` from the Behavior Agent and Governance Agent into one object including a `timestamp` and `executionId`.
8.  **Routing & Notifications:**
    *   Add a **Switch** node to route by `enforcementAction`.
    *   **Slack:** Configure a Slack node using OAuth2 to post the investigation summary.
    *   **Email:** Configure a Gmail or SMTP node using HTML to send the escalation details.
    *   **Logging:** Use **Data Table** or **Google Sheets** nodes to append the final enforcement decision.
9.  **Low Risk Path:** Connect the "Low Risk" output of the first switch to an **If** node checking if `riskScore` is below the `autoActionThreshold`. If true, route to a **Set** node to format an "auto_monitor" action, then log it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Slack bot token, Gmail/SMTP auth, and Google Sheets access. |
| **Customization** | OpenAI nodes can be replaced with Claude (Anthropic) or local models (Ollama/NVIDIA NIM). |
| **Benefits** | Eliminates manual triage; ensures high-severity violations are actioned within seconds. |
| **Model Version** | Configured for `gpt-4.1-mini` (interpret as `gpt-4o-mini` for current production environments). |