Route and escalate student advising requests with OpenAI, Gmail and Slack

https://n8nworkflows.xyz/workflows/route-and-escalate-student-advising-requests-with-openai--gmail-and-slack-13680


# Route and escalate student advising requests with OpenAI, Gmail and Slack

# AI Student Academic Advisor: Reference Documentation

This document provides a technical analysis of the **AI Student Academic Advisor** workflow, designed to automate student status validation, advising recommendations, and multi-channel escalation using a multi-agent AI architecture.

---

### 1. Workflow Overview
The workflow functions as an intelligent triage and support system for academic institutions. It receives student event data (like GPA changes or enrollment updates) and uses a hierarchical AI structure to decide whether to notify the student, alert an advisor, or escalate to faculty.

**Logical Blocks:**
*   **1.1 Input & Config:** Receives the student event and sets global environment variables.
*   **1.2 Status Validation:** A specialized agent classifies the student's current lifecycle stage (e.g., At-Risk, Graduating).
*   **1.3 Academic Orchestration:** A "Manager" agent coordinates three specialized tools (Advising, Notification, and Escalation) to create a unified action plan.
*   **1.4 Communication Dispatch:** Sends targeted messages via Email (Gmail/SMTP) or Slack based on the orchestration plan.
*   **1.5 Logging:** Records the final outcome for administrative auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
This block initializes the process and defines the operational parameters used across all nodes.
*   **Nodes Involved:** `Student Event Webhook`, `Workflow Configuration`.
*   **Node Details:**
    *   **Student Event Webhook (Webhook):** Listens for `POST` requests at the `/student-event` endpoint. It triggers the entire flow.
    *   **Workflow Configuration (Set):** Defines placeholders for Slack Channel IDs, Faculty Emails, and threshold values (e.g., GPA < 2.0).
    *   **Edge Cases:** Missing payload fields in the webhook will cause expression failures in subsequent AI nodes.

#### 2.2 Student Status Agent
The "Gatekeeper" that interprets raw data into a structured academic status.
*   **Nodes Involved:** `Student Status Agent`, `OpenAI Model - Status Agent`, `Status Validation Output Parser`.
*   **Node Details:**
    *   **Student Status Agent (AI Agent):** Uses a System Message to classify students into stages like `AT_RISK` or `GRADUATION_ELIGIBLE`.
    *   **Status Validation Output Parser (Structured Output Parser):** Enforces a specific JSON schema including `lifecycleStage`, `gpa`, and `riskFactors`.
    *   **Role:** Ensures downstream logic operates on validated classifications rather than raw, noisy data.

#### 2.3 Academic Orchestration (Multi-Agent Layer)
The core intelligence block where specialized sub-agents are called as tools.
*   **Nodes Involved:** `Academic Orchestration Agent`, `Advising Agent Tool`, `Notification Agent Tool`, `Escalation Agent Tool` (and their respective Models/Parsers).
*   **Node Details:**
    *   **Academic Orchestration Agent (AI Agent):** Acts as the supervisor. It is instructed to call **all three** tools for every event.
    *   **Advising Agent Tool:** Recommends specific academic actions (e.g., tutoring, course selection).
    *   **Notification Agent Tool:** Determines the priority and drafts messages for Student, Advisor, and Faculty.
    *   **Escalation Agent Tool:** Specifically looks for triggers like policy violations or severe GPA drops.
    *   **Failure Types:** If the Orchestration Agent fails to call a tool, the final action plan will be incomplete.

#### 2.4 Communication & Escalation Dispatch
Routes the AI-generated content to the correct stakeholders.
*   **Nodes Involved:** `Route by Action Type`, `Check if Escalation Required`, `Send Student Notification Email`, `Send Advisor Slack Alert`, `Send Faculty Escalation Email`.
*   **Node Details:**
    *   **Route by Action Type (Switch):** Splits the flow into Student, Advisor, or Faculty paths.
    *   **Check if Escalation Required (If):** A safety check that confirms the `escalationRequired` boolean from the AI is `true` before emailing faculty.
    *   **Send Student Notification Email:** Uses a professional HTML template to deliver "Recommended Actions" and "Support Resources."
    *   **Send Advisor Slack Alert:** Sends a formatted Markdown message to a specific Slack channel.

#### 2.5 Logging
Finalizes the execution by creating a record.
*   **Nodes Involved:** `Merge Notification Paths`, `Log Workflow Completion`.
*   **Node Details:**
    *   **Log Workflow Completion (Code):** Consolidates data from the Status Agent and Orchestration Agent into a clean JSON log entry and prints it to the console.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Student Event Webhook** | Webhook | Trigger | (External) | Workflow Configuration | |
| **Workflow Configuration** | Set | Config | Student Event Webhook | Student Status Agent | |
| **Student Status Agent** | AI Agent | Analysis | Workflow Configuration | Route by Student Status | Classifies student status using OpenAI. |
| **Route by Student Status**| Switch | Routing | Student Status Agent | Academic Orchestration Agent | Directs flow based on status classification. |
| **Academic Orchestration Agent** | AI Agent | Controller | Route by Student Status | Route by Action Type | Delegates to sub-agents. |
| **Advising Agent Tool** | Agent Tool | Specialized Logic | (Orchestration Agent) | (Orchestration Agent) | Provides academic advising recommendations. |
| **Notification Agent Tool**| Agent Tool | Specialized Logic | (Orchestration Agent) | (Orchestration Agent) | Determines notification strategy. |
| **Escalation Agent Tool** | Agent Tool | Specialized Logic | (Orchestration Agent) | (Orchestration Agent) | Assesses faculty escalation requirements. |
| **Route by Action Type** | Switch | Router | Academic Orchestration Agent | Email/Slack/If Nodes | Determines output channel and escalation need. |
| **Check if Escalation Req.**| If | Safety Gate | Route by Action Type | Send Faculty Email | Ensures urgent cases reach faculty. |
| **Send Student Email** | Email Send | Delivery | Route by Action Type | Merge Paths | Delivers outcomes through email. |
| **Send Advisor Slack** | Slack | Delivery | Route by Action Type | Merge Paths | Sends alert to advisor via Slack. |
| **Send Faculty Email** | Email Send | Delivery | Check if Escalation Req. | Merge Paths | Delivers escalation alerts. |
| **Log Workflow Comp.** | Code | Analytics | Merge Paths | (End) | Logs completion status. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Obtain an **OpenAI API Key**.
    *   Set up a **Slack App** with `chat:write` scopes and a Bot Token.
    *   Prepare a **Gmail/SMTP** account for sending emails.
2.  **Input Setup:** Create a **Webhook Node** (POST) and a **Set Node** containing your target email addresses and Slack Channel ID.
3.  **Status Agent Setup:**
    *   Add an **AI Agent Node**. Connect an **OpenAI Chat Model** (GPT-4o mini).
    *   Add a **Structured Output Parser**. Define a schema for `lifecycleStage` (enum), `gpa` (number), and `riskFactors` (array).
4.  **Routing:** Add a **Switch Node** to filter out status types that don't require action (e.g., skip if "NORMAL_PROGRESSION" is not needed).
5.  **Orchestration Logic:**
    *   Create a second **AI Agent Node** (Orchestration).
    *   Create three **Agent Tool Nodes** (Advising, Notification, Escalation).
    *   For each Tool, attach a Model and a **Structured Output Parser** to define the specific fields needed for that domain.
6.  **Dispatching:**
    *   Add a **Switch Node** to route based on the `actionType` returned by the Orchestration Agent.
    *   Add an **If Node** specifically for the Escalation path to check the `escalationRequired` boolean.
7.  **Communication Nodes:**
    *   Configure **Email Send Nodes** using HTML. Use expressions like `{{ $json.output.notificationStrategy.studentMessage }}` to map AI content.
    *   Configure the **Slack Node** to send a message to the channel ID defined in the configuration.
8.  **Completion:** Add a **Merge Node** (Combine by position) and a **Code Node** to aggregate results into a final log object.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | OpenAI API key, Gmail OAuth2, Slack bot token. |
| **Scalability** | You can add new sub-agent tools for "Financial Aid" or "Career Services" easily. |
| **Benefit** | Reduces manual triage and ensures 24/7 coverage for "At-Risk" student detection. |
| **Setup Step** | Remember to configure the Webhook URL in your external student management system. |