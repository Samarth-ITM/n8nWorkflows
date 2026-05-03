Coordinate smart factory operations with OpenAI GPT-4.1-mini and Slack alerts

https://n8nworkflows.xyz/workflows/coordinate-smart-factory-operations-with-openai-gpt-4-1-mini-and-slack-alerts-13709


# Coordinate smart factory operations with OpenAI GPT-4.1-mini and Slack alerts

### 1. Workflow Overview

This workflow automates the coordination of smart factory operations by deploying a multi-agent AI system. It is designed to handle complex manufacturing environments (Chemicals, Electronics, Motor Vehicles, Furniture, and Food Products) where production data and supply chain logistics must be synchronized. 

The process follows a linear progression through four logical stages:
1.  **Data Ingestion:** Regularly fetches and merges production and supply chain data from external APIs.
2.  **Validation:** An initial AI agent validates the data integrity and identifies immediate anomalies.
3.  **Orchestration (Multi-Agent):** A central Coordination Agent utilizes three specialized sub-agents (Scheduling, Procurement, and Quality) to formulate a comprehensive operational plan.
4.  **Actionable Output:** Based on the assessed priority, the workflow either triggers high-urgency Slack alerts or logs routine operational summaries.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Ingestion & Configuration
This block initializes variables and retrieves the raw data required for analysis.
*   **Nodes Involved:** `Schedule Trigger`, `Workflow Configuration`, `Fetch Production Data`, `Fetch Supply Chain Data`, `Merge Operations Data`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires every 15 minutes to ensure near real-time monitoring.
    *   **Workflow Configuration (Set):** Defines placeholders for API endpoints and Slack Channel IDs. Sets `criticalThreshold` (90) and `highThreshold` (70).
    *   **Fetch Nodes (HTTP Request):** Two nodes perform GET requests to the production and supply chain APIs defined in the configuration.
    *   **Merge Operations Data:** Uses a "Merge" mode to combine the two JSON streams into a single object for the AI.
*   **Edge Cases:** API timeouts or 404 errors will halt the workflow.

#### 1.2 Operations Validation
Before making decisions, the system verifies the operational "signals."
*   **Nodes Involved:** `Operations Validation Agent`, `OpenAI Model - Validation Agent`, `Validation Output Parser`.
*   **Node Details:**
    *   **Operations Validation Agent (LangChain Agent):** Acts as the first filter. It analyzes output rates, defect rates, and inventory risks.
    *   **OpenAI Model:** Uses `gpt-4o-mini`.
    *   **Validation Output Parser:** Enforces a strict JSON schema including `validationStatus` (valid/warning/critical) and `priorityLevel`.
*   **Role:** Prevents "garbage in, garbage out" by ensuring subsequent agents work with verified data.

#### 1.3 Cross-Factory Coordination (The Multi-Agent Hub)
This is the core logic engine where the orchestration happens.
*   **Nodes Involved:** `Cross-Factory Coordination Agent`, `Coordination Agent Tool`, `Scheduling Agent Tool`, `Procurement Agent Tool`, `Quality Escalation Agent Tool` (and their respective Models/Parsers).
*   **Node Details:**
    *   **Cross-Factory Coordination Agent:** The "Supervisor" agent. It doesn't solve problems directly but calls the specific tools.
    *   **Scheduling Agent Tool:** Optimizes shifts and resolves conflicts between product lines.
    *   **Procurement Agent Tool:** Monitors stock levels and recommends specific suppliers.
    *   **Quality Escalation Agent Tool:** Determines if quality issues need to be raised to the Plant Director or Executive level.
*   **Failure Types:** Token limit exhaustion (due to large merged data) or tool-call looping if the prompt isn't specific.

#### 1.4 Response Routing & Alerts
Finalizes the workflow based on the AI's priority assessment.
*   **Nodes Involved:** `Route by Priority`, `Critical Alert - Slack`, `High Priority Alert - Slack`, `Log Routine Operations`.
*   **Node Details:**
    *   **Route by Priority (Switch):** Routes data based on the `priorityLevel` string (CRITICAL, HIGH, or others).
    *   **Slack Nodes:** Send formatted blocks to a specific channel. Critical alerts include a "Traceability Report."
    *   **Log Routine Operations (Set):** Cleans up the data for storage/logging when no urgent action is required.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | scheduleTrigger | Workflow Entry | (None) | Workflow Configuration | |
| Workflow Configuration | set | Variable Definition | Schedule Trigger | Fetch Data nodes | |
| Fetch Production Data | httpRequest | Data Retrieval | Workflow Configuration | Merge Operations Data | |
| Fetch Supply Chain Data | httpRequest | Data Retrieval | Workflow Configuration | Merge Operations Data | |
| Merge Operations Data | merge | Data Consolidation | Fetch Data nodes | Operations Validation Agent | Combines production and supply chain data into unified context. |
| Operations Validation Agent | agent | Data Integrity | Merge / Validation Model | Coordination Agent | Validates merged data integrity using OpenAI before coordination. |
| Cross-Factory Coordination Agent | agent | Multi-Agent Orchestrator | Validation Agent / Tools | Route by Priority | Orchestrates Scheduling, Procurement, and Quality Escalation sub-agents. |
| Scheduling Agent Tool | agentTool | Specialized Sub-Agent | Coordination Agent | Coordination Agent | |
| Procurement Agent Tool | agentTool | Specialized Sub-Agent | Coordination Agent | Coordination Agent | |
| Quality Escalation Agent Tool | agentTool | Specialized Sub-Agent | Coordination Agent | Coordination Agent | |
| Route by Priority | switch | Logic Branching | Coordination Agent | Slack / Log nodes | Separates critical, high, and routine operational outcomes. |
| Critical Alert - Slack | slack | Urgent Notification | Route by Priority | (None) | |
| High Priority Alert - Slack | slack | Important Notification | Route by Priority | (None) | |
| Log Routine Operations | set | Data Logging | Route by Priority | (None) | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create a new workflow and add a **Schedule Trigger** set to 15-minute intervals.
    *   Add a **Set Node** ("Workflow Configuration") to define your Production API and Supply Chain API URLs as string variables.

2.  **Data Pipeline:**
    *   Create two **HTTP Request Nodes** to fetch data from the URLs defined in Step 1.
    *   Connect both to a **Merge Node**. Use the "Combine" setting to create a single JSON object.

3.  **Primary Agent (Validation):**
    *   Add a **LangChain Agent** node ("Operations Validation Agent").
    *   Attach an **OpenAI Chat Model** node (Configured with `gpt-4o-mini`).
    *   Attach a **Structured Output Parser** node. Define a schema that requires `validationStatus` and `priorityLevel`.
    *   Input the system prompt: "You are an Operations Validation Agent... Inspect production signals for chemicals, electronics, etc."

4.  **Orchestrator Agent (Coordination):**
    *   Add a second **LangChain Agent** node ("Cross-Factory Coordination Agent").
    *   Connect the output of the Validation Agent to its input.
    *   Attach three **Agent Tool** nodes to this agent:
        *   **Scheduling Tool:** Prompt to optimize shifts.
        *   **Procurement Tool:** Prompt to monitor inventory/suppliers.
        *   **Quality Tool:** Prompt to assess defect severity and escalation levels.
    *   Each tool requires its own Model and Structured Output Parser to ensure the tool returns clean data to the main agent.

5.  **Output Logic:**
    *   Add a **Switch Node** ("Route by Priority"). Create three output paths:
        *   Path 1: String `priorityLevel` equals `CRITICAL`.
        *   Path 2: String `priorityLevel` equals `HIGH`.
        *   Path 3: Fallback (Routine).
    *   Connect **Slack Nodes** to the Critical and High paths. Configure with OAuth2 credentials and your Slack Channel ID.
    *   Connect a **Set Node** to the fallback path to format the log entries.

6.  **Credentials:**
    *   Configure **OpenAI API** credentials for all model nodes.
    *   Configure **Slack OAuth2** credentials for alert nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Slack workspace with bot token; Production and supply chain data sources (API or database). |
| **Multi-Agent Benefit** | Eliminates manual coordination delays by automating cross-factory logic across scheduling, procurement, and quality. |
| **Customization** | Users can add additional sub-agents for maintenance or logistics optimization. |
| **Traceability** | The workflow creates a "Traceability Report" in critical alerts for audit purposes. |
| **Setup Steps** | Ensure OpenAI API credentials are added to all individual Model nodes within the tools. |