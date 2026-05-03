Forecast property CAPEX and ROI weekly using Google Sheets and GPT-4o

https://n8nworkflows.xyz/workflows/forecast-property-capex-and-roi-weekly-using-google-sheets-and-gpt-4o-15027


# Forecast property CAPEX and ROI weekly using Google Sheets and GPT-4o

# Workflow Analysis: Multi-agent Property CAPEX Forecasting with ROI Simulation

## 1. Workflow Overview
This workflow is a sophisticated automated system designed for commercial real estate portfolio management. Its primary purpose is to forecast Capital Expenditure (CAPEX) and simulate Return on Investment (ROI) on a weekly basis. It transforms raw data from Google Sheets into actionable financial insights using a multi-agent AI architecture.

The logic is divided into the following functional blocks:
- **1.1 Data Acquisition:** Scheduled retrieval of maintenance history, property details, and tenant feedback.
- **1.2 Data Unification:** Merging multiple data streams into a single context for the AI.
- **1.3 Multi-Agent AI Orchestration:** A central "Main Prediction Agent" that coordinates three specialized agents (Prioritization, ROI Simulation, and Procurement) to analyze the data.
- **1.4 Output Structuring & Distribution:** Parsing the AI's natural language response into structured data and exporting it to a database (Google Sheets) and an external budgeting API.

---

## 2. Block-by-Block Analysis

### 2.1 Data Acquisition & Unification
**Overview:** This block triggers the workflow weekly and gathers all necessary context from Google Sheets to ensure the AI has a holistic view of the property portfolio.

- **Nodes Involved:** `Weekly Maintenance Analysis`, `Get Maintenance Data`, `Get Property Data`, `Get Tenant Feedback`, `Combine All Data`.
- **Node Details:**
    - **Weekly Maintenance Analysis (Schedule Trigger):** Triggers every Monday at 06:00.
    - **Get Maintenance Data, Get Property Data, Get Tenant Feedback (Google Sheets):** 
        - Reads specific tabs ("Maintenance History", "Property Details", "Tenant Feedback").
        - Configuration: Reads range `A:Z`.
        - Potential Failures: Authentication expiry or missing sheet tabs.
    - **Combine All Data (Merge):** 
        - Mode: Combine by position.
        - Role: Aggregates the three separate Google Sheet inputs into a single JSON object.

### 2.2 Multi-Agent AI Orchestration
**Overview:** The core intelligence layer. It uses a "Manager-Worker" pattern where a main agent delegates complex tasks to specialized sub-agents.

- **Nodes Involved:** `Main Prediction Agent`, `Main Agent Model`, `Prediction Output Parser`, `CAPEX Prioritizer Agent`, `ROI Simulator Agent`, `Quote Requester Agent`, and their respective models.
- **Node Details:**
    - **Main Prediction Agent (AI Agent):**
        - Role: Orchestrator.
        - System Message: Analyzes historical data to predict repair/upgrade needs.
        - Connections: Linked to the Model, Output Parser, and three Agent Tools.
    - **Main Agent Model (Chat OpenAI):** Configured with `gpt-4o` and temperature `0.2` for high consistency.
    - **Prediction Output Parser (Structured Output Parser):** 
        - Enforces a strict JSON schema: `predictions` array containing `propertyId`, `issueType`, `urgencyLevel`, `estimatedTimeframe`, `estimatedCost`, and `reasoning`.
    - **CAPEX Prioritizer Agent (Agent Tool):** 
        - Specialization: Ranks spending based on safety, compliance, and property value impact.
    - **ROI Simulator Agent (Agent Tool):** 
        - Specialization: Financial modeling (NPV, IRR, Payback period).
        - Tools: Connected to the `Calculator Tool` and `Financial Modeling Tool`.
    - **Quote Requester Agent (Agent Tool):** 
        - Specialization: Procurement; generates professional technical specifications for contractors.

### 2.3 Financial Tooling (Custom Logic)
**Overview:** Provides the AI with deterministic mathematical capabilities to avoid "AI hallucinations" during financial calculations.

- **Nodes Involved:** `Calculator Tool`, `Financial Modeling Tool`.
- **Node Details:**
    - **Calculator Tool:** Standard mathematical operations.
    - **Financial Modeling Tool (Code/Python):** 
        - Custom Python script to calculate **Net Present Value (NPV)**, **Internal Rate of Return (IRR)**, and **Payback Period**.
        - Input: Expects JSON with `cash_flows`, `discount_rate`, and `initial_investment`.

### 2.4 Output Distribution
**Overview:** Converts the AI's complex analysis into a clean format and pushes it to downstream systems.

- **Nodes Involved:** `Split Predictions`, `Format Results`, `Save Predictions`, `Update Budgeting System`.
- **Node Details:**
    - **Split Predictions (Split Out):** Breaks the `predictions` array into individual items for row-by-row processing.
    - **Format Results (Set):** Maps the AI outputs to final fields (e.g., `propertyId`, `priorityScore`, `roiAnalysis`, `analysisDate`).
    - **Save Predictions (Google Sheets):** Appends the formatted results to the "Predictions & Analysis" sheet.
    - **Update Budgeting System (HTTP Request):** Sends the JSON data via POST to an external API endpoint.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Maintenance Analysis | Schedule Trigger | Workflow Entry Point | - | Data Get Nodes | - |
| Get Maintenance Data | Google Sheets | Data Retrieval | Weekly Analysis | Combine All Data | Data Combination: Merges three data streams |
| Get Property Data | Google Sheets | Data Retrieval | Weekly Analysis | Combine All Data | Data Combination: Merges three data streams |
| Get Tenant Feedback | Google Sheets | Data Retrieval | Weekly Analysis | Combine All Data | Data Combination: Merges three data streams |
| Combine All Data | Merge | Data Aggregation | All 3 GS Nodes | Main Prediction Agent | Data Combination: Merges three data streams |
| Main Prediction Agent | AI Agent | Logic Orchestrator | Combine All Data | Split Predictions | Main Prediction Agent: Orchestrates specialist agents |
| Main Agent Model | Chat OpenAI | LLM Brain | - | Main Prediction Agent | - |
| Prediction Output Parser | Output Parser | Schema Enforcement | - | Main Prediction Agent | Output Parsing & Formatting: Ensures clean data |
| CAPEX Prioritizer Agent | Agent Tool | Priority Scoring | Main Prediction Agent | Main Prediction Agent | Main Prediction Agent: Orchestrates specialist agents |
| CAPEX Agent Model | Chat OpenAI | LLM Brain | - | CAPEX Prioritizer | - |
| ROI Simulator Agent | Agent Tool | Financial Simulation | Main Prediction Agent | Main Prediction Agent | Main Prediction Agent: Orchestrates specialist agents |
| ROI Agent Model | Chat OpenAI | LLM Brain | - | ROI Simulator Agent | - |
| Quote Requester Agent | Agent Tool | Procurement Spec | Main Prediction Agent | Main Prediction Agent | Main Prediction Agent: Orchestrates specialist agents |
| Quote Agent Model | Chat OpenAI | LLM Brain | - | Quote Requester Agent | - |
| Calculator Tool | Calculator | Math Operations | - | ROI Simulator Agent | - |
| Financial Modeling Tool | Code (Python) | Financial Metrics | - | ROI Simulator Agent | - |
| Split Predictions | Split Out | Data Flattening | Main Prediction Agent | Format Results | Output Parsing & Formatting: Ensures clean data |
| Format Results | Set | Data Mapping | Split Predictions | Save/Update Nodes | Output Parsing & Formatting: Ensures clean data |
| Save Predictions | Google Sheets | Data Archiving | Format Results | - | Output Parsing & Formatting: Ensures clean data |
| Update Budgeting System | HTTP Request | API Integration | Format Results | - | Output Parsing & Formatting: Ensures clean data |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger & Data Ingestion
1. Create a **Schedule Trigger** set to weekly (Monday, 06:00).
2. Create three **Google Sheets** nodes. Configure each to "Get Many" rows from a specific sheet:
   - Node 1: "Maintenance History"
   - Node 2: "Property Details"
   - Node 3: "Tenant Feedback"
3. Add a **Merge** node. Set the mode to "Combine" and "Combine by Position". Connect all three Google Sheets nodes to it.

### Step 2: AI Agent Architecture
1. Create an **AI Agent** node (Main Prediction Agent).
2. Connect a **Chat OpenAI** node to the agent. Select model `gpt-4o` and set temperature to `0.2`.
3. Connect a **Structured Output Parser** node to the agent. Define the JSON schema as an object containing a `predictions` array with the properties: `propertyId`, `issueType`, `urgencyLevel`, `estimatedTimeframe`, `estimatedCost`, and `reasoning`.
4. Create three **Agent Tool** nodes:
   - **CAPEX Prioritizer:** System prompt focusing on urgency, cost-benefit, and strategic alignment.
   - **ROI Simulator:** System prompt focusing on NPV, IRR, and payback periods.
   - **Quote Requester:** System prompt focusing on technical specs and procurement.
5. Assign a dedicated **Chat OpenAI** model node to each of these three tools.

### Step 3: Financial Tooling
1. Create a **Calculator Tool** and connect it to the **ROI Simulator Agent**.
2. Create a **Code Tool** (Python). Paste the provided Python script for NPV/IRR/Payback calculation. Connect this to the **ROI Simulator Agent**.

### Step 4: Processing & Export
1. Add a **Split Out** node after the Main Agent. Set the field to split as `predictions`.
2. Add a **Set** node to map the split data into clean keys (e.g., `propertyId`, `roiAnalysis`, etc.) and add a timestamp using `{{ $now.toISO() }}`.
3. Add a **Google Sheets** node set to "Append" to save results into a "Predictions & Analysis" sheet.
4. Add an **HTTP Request** node (POST) to send the final JSON payload to your budgeting system's API endpoint.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites:** Requires Google Sheets API, OpenAI API, and a POST-compatible budgeting API. | Setup Requirements |
| **Use Case:** Ideal for multi-property portfolios requiring automated maintenance budgeting. | Target Audience |
| **Customization:** The workflow can be expanded by adding IoT sensor data or ERP exports as additional input nodes. | Scalability |
| **Benefit:** Replaces manual spreadsheet analysis with an autonomous, auditable AI pipeline. | Value Proposition |