Audit contract governance risk with OpenAI, Slack and email routing

https://n8nworkflows.xyz/workflows/audit-contract-governance-risk-with-openai--slack-and-email-routing-13683


# Audit contract governance risk with OpenAI, Slack and email routing

This technical document provides a comprehensive breakdown of the **Intelligent Contract Governance Auditor** workflow. This system automates the auditing of smart contracts by utilizing a multi-agent AI architecture to validate security, assess business risks, verify compliance, and route alerts based on calculated risk levels.

---

### 1. Workflow Overview

The workflow is designed to replace manual contract reviews with a scheduled, AI-driven governance pipeline. It initiates by generating or receiving contract audit data, processes it through specialized AI agents, and ends with conditional notifications and a persistent audit trail.

#### Logical Blocks:
*   **1.1 Data Initialization:** Sets environment variables (thresholds) and generates simulated contract data for auditing.
*   **1.2 Initial Validation:** A dedicated AI agent performs technical validation of the contract signals (security flags, gas efficiency).
*   **1.3 Governance Orchestration:** A primary agent coordinates specialized sub-tools (Risk and Compliance agents) to form a final governance decision.
*   **1.4 Risk-Based Routing:** Logic to branch the workflow based on the severity of the identified risks (Low, Medium, High, Critical).
*   **1.5 Notification & Escalation:** Delivery of results via Slack (for operational tracking) or Email (for critical escalations).
*   **1.6 Audit Logging:** Final aggregation of the decision process into a structured log.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Initialization
*   **Overview:** Establishes the operational parameters and the input data for the audit.
*   **Nodes Involved:** `Schedule Trigger`, `Workflow Configuration`, `Simulate Contract Audit Data`.
*   **Node Details:**
    *   **Workflow Configuration (Set):** Defines numeric thresholds for risk (30, 60, 85) and placeholder strings for Slack channels and escalation emails.
    *   **Simulate Contract Audit Data (Code):** A JavaScript-based node that generates a random contract object containing `contractId`, `auditScore`, `securityFlags` (e.g., reentrancy), and `complianceChecks` (GDPR, SOX, HIPAA).
*   **Edge Cases:** Missing placeholder values in the configuration will cause downstream failures in the notification nodes.

#### 2.2 Initial Validation
*   **Overview:** Analyzes the technical signals of the contract to determine if it meets basic deployment standards.
*   **Nodes Involved:** `Contract Validation Agent`, `OpenAI Chat Model - Validation`, `Structured Output Parser - Validation`.
*   **Node Details:**
    *   **Contract Validation Agent:** Uses a system message to act as a "Contract Validation Specialist."
    *   **Structured Output Parser:** Enforces a JSON schema requiring `validationStatus` (APPROVED/CONDITIONAL/REJECTED), `securityScore`, and `criticalIssues`.
*   **Edge Cases:** If the AI model fails to return valid JSON, the output parser will trigger an error.

#### 2.3 Governance Orchestration
*   **Overview:** A multi-agent coordination block where the Orchestrator uses tools to perform deep-dive assessments.
*   **Nodes Involved:** `Governance Orchestrator Agent`, `Risk Assessment Agent Tool`, `Compliance Checker Agent Tool`, `Calculator Tool`, and associated OpenAI models/parsers.
*   **Node Details:**
    *   **Governance Orchestrator:** Acts as the "manager." It calls the Risk and Compliance tools as needed.
    *   **Risk Assessment Agent Tool:** Evaluates financial exposure and operational impact (Output: `riskScore`, `financialExposure`).
    *   **Compliance Checker Agent Tool:** Maps findings to frameworks like GDPR and HIPAA (Output: `complianceStatus`, `regulatoryRisk`).
    *   **Calculator Tool:** Used by the agent for precise scoring calculations.
*   **Edge Cases:** High token usage due to multiple agent loops; potential timeouts if the LLM response is slow.

#### 2.4 Risk-Based Routing
*   **Overview:** Filters the orchestration results into specific action paths.
*   **Nodes Involved:** `Route by Risk Level`.
*   **Node Details:**
    *   **Route by Risk Level (Switch):** Uses string comparison on `{{ $json.output.riskLevel }}`. 
        *   **LOW:** Direct to approval.
        *   **MEDIUM:** Direct to Slack notification.
        *   **HIGH/CRITICAL:** Direct to Email escalation.

#### 2.5 Notification & Escalation
*   **Overview:** Communicates the AI's findings to human stakeholders.
*   **Nodes Involved:** `Prepare Low/Medium/High Notification`, `Send Slack Notification`, `Send Email Escalation`.
*   **Node Details:**
    *   **Prepare High Risk Notification (Set):** Constructs a complex HTML body for the email, including a red alert banner, a table of risk metrics, and an audit trail.
    *   **Send Email Escalation (Email):** Uses SMTP/Gmail to send the critical alert to the `escalationEmail` defined in Block 1.
    *   **Send Slack Notification (Slack):** Posts a summary message to the configured channel.

#### 2.6 Audit Logging
*   **Overview:** Finalizes the process by creating a record of the governance event.
*   **Nodes Involved:** `Merge Notifications`, `Log Audit Trail`.
*   **Node Details:**
    *   **Log Audit Trail (Code):** Aggregates data from any path (Low, Medium, or High) and generates a structured log entry with a unique `auditId` and a timestamp.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Workflow Start | (None) | Workflow Configuration | |
| Workflow Configuration | Set | Global Variables | Schedule Trigger | Simulate Data | |
| Simulate Contract Audit Data | Code | Input Simulation | Workflow Configuration | Contract Validation Agent | |
| Contract Validation Agent | AI Agent | Technical Review | Simulate Data | Governance Orchestrator | Validates contract data using OpenAI with structured output parsing. |
| OpenAI Chat Model - Validation | OpenAI Model | LLM Provider | (Internal) | Contract Validation Agent | Ensures only well-formed, complete contracts proceed. |
| Structured Output Parser - Validation | Output Parser | Data Structuring | (Internal) | Contract Validation Agent | Ensures only well-formed, complete contracts proceed. |
| Governance Orchestrator Agent | AI Agent | Logic Controller | Validation Agent | Route by Risk Level | Delegates to Risk Assessment and Compliance Checker sub-agents. |
| Risk Assessment Agent Tool | Agent Tool | Business Risk Eval | Governance Agent | Governance Agent | Centralizes governance logic for modular risk processing. |
| Compliance Checker Agent Tool | Agent Tool | Regulatory Eval | Governance Agent | Governance Agent | Centralizes governance logic for modular risk processing. |
| Calculator Tool | AI Tool | Math Processing | Governance Agent | Governance Agent | Centralizes governance logic for modular risk processing. |
| Route by Risk Level | Switch | Path Routing | Governance Agent | Prepare Low/Med/High | Directs flow to low, medium, or high-risk notification paths. |
| Prepare Low Risk Notification | Set | Data Formatting | Route by Risk Level | Merge Notifications | |
| Prepare Medium Risk Notification | Set | Data Formatting | Route by Risk Level | Send Slack Notification | |
| Prepare High Risk Notification | Set | Data Formatting | Route by Risk Level | Send Email Escalation | |
| Send Slack Notification | Slack | Communication | Prepare Medium | Merge Notifications | Sends Slack alerts or escalation emails based on risk level. |
| Send Email Escalation | Email | Communication | Prepare High | Merge Notifications | Sends Slack alerts or escalation emails based on risk level. |
| Merge Notifications | Merge | Data Aggregation | Notification paths | Log Audit Trail | |
| Log Audit Trail | Code | Audit Recording | Merge Notifications | (None) | Ensures urgent findings are delivered immediately. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Create a new workflow. Drag a **Schedule Trigger** and set it to 15-minute intervals.
2.  **Configuration:** Add a **Set** node ("Workflow Configuration"). Define four variables: `riskThresholdLow` (Number), `riskThresholdMedium` (Number), `riskThresholdHigh` (Number), and `slackChannelId` (String).
3.  **Data Source:** Add a **Code** node ("Simulate Contract Audit Data"). Use `Math.random()` to generate a JSON object representing a contract (ID, status, audit score, security flags).
4.  **The Validation Layer:**
    *   Create an **AI Agent** node ("Contract Validation Agent").
    *   Connect an **OpenAI Chat Model** (set to `gpt-4o-mini`).
    *   Connect a **Structured Output Parser**. Define a JSON schema including `validationStatus`, `securityScore`, and `reasoning`.
5.  **The Governance Layer:**
    *   Create a second **AI Agent** ("Governance Orchestrator").
    *   Connect an **OpenAI Chat Model** and a **Structured Output Parser** (schema: `riskLevel`, `riskScore`, `approvalStatus`, `escalationReason`).
    *   Add two **Agent Tool** nodes: "Risk Assessment Agent Tool" and "Compliance Checker Agent Tool".
    *   Connect separate **OpenAI Chat Models** and **Structured Output Parsers** to each of these tools to define their specific outputs.
    *   Connect a **Calculator Tool** to the Orchestrator.
6.  **Logic Branching:**
    *   Add a **Switch** node ("Route by Risk Level"). 
    *   Rule 1: If `riskLevel` equals `LOW`.
    *   Rule 2: If `riskLevel` equals `MEDIUM`.
    *   Rule 3: If `riskLevel` equals `HIGH` OR `CRITICAL`.
7.  **Output Channels:**
    *   **Slack:** Add a **Slack** node. Configure OAuth2 credentials. Set the message to reference the output of the "Prepare Medium Risk" node.
    *   **Email:** Add a **Send Email** node. Set the body to "HTML" and use expressions to pull in the `emailBody` from the "Prepare High Risk" node.
8.  **Finalization:** Add a **Merge** node to collect all branches. Finish with a **Code** node to format the final audit log.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Needs Slack bot token, SMTP/Gmail credentials, and OpenAI API Key. |
| **Use Cases** | Procurement team auditing, automated legal compliance breach detection. |
| **Customization** | Replace the "Simulate Data" node with an "HTTP Request" node to fetch real contracts from a database or blockchain explorer. |
| **Project Benefits** | Eliminates manual review bottlenecks and provides a 24/7 audit trail. |