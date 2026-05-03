Generate a weekly business health report from Google Sheets with Claude

https://n8nworkflows.xyz/workflows/generate-a-weekly-business-health-report-from-google-sheets-with-claude-14931


# Generate a weekly business health report from Google Sheets with Claude

# Technical Documentation: AI Weekly Business Health Report

## 1. Workflow Overview

The **AI Weekly Business Health Report** is an automated analysis pipeline designed to replace manual spreadsheet reporting. The workflow triggers weekly, extracts business performance metrics from Google Sheets, utilizes Claude AI to perform a comparative analysis between the current and previous week, and distributes the findings via Gmail. Finally, it maintains a historical log of all generated reports.

### 1.1 Logical Blocks
- **1.1 Schedule & Configuration:** Handles the timing of the execution and defines global variables (Business name, email, and date ranges).
- **1.2 Data Acquisition & Formatting:** Extracts raw data for two distinct time periods from Google Sheets and transforms it into a structured text format optimized for LLM consumption.
- **1.3 AI Analysis:** Uses a Large Language Model (Claude Sonnet) to analyze trends, flag anomalies, and suggest actionable business steps.
- **1.4 Distribution & Archiving:** Delivers the final report via email and logs the execution metadata back to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule & Configuration
**Overview:** This block initializes the workflow and calculates the specific date windows required to filter Google Sheets data.

- **Nodes Involved:** `Weekly Schedule Trigger`, `Configure Report Settings`.
- **Node Details:**
    - **Weekly Schedule Trigger** (Schedule Trigger): 
        - **Configuration:** Set to trigger every Monday at 07:00 AM.
        - **Output:** Trigger signal to start the workflow.
    - **Configure Report Settings** (Code Node):
        - **Technical Role:** Variable definition and date calculation.
        - **Configuration:** Defines constants for `BUSINESS_NAME`, `REPORT_EMAIL`, and `METRICS_DESCRIPTION`.
        - **Key Expressions:** Uses JavaScript `Date` objects to calculate `this_week_start/end` (last 7 days) and `last_week_start/end` (8-14 days ago).
        - **Potential Failures:** Incorrectly formatted email addresses or hardcoded business names not updated by the user.

### 2.2 Data Acquisition & Formatting
**Overview:** Retrieves quantitative data from a spreadsheet and prepares a comparison table.

- **Nodes Involved:** `Fetch This Week Data`, `Fetch Last Week Data`, `Format Data for Claude`.
- **Node Details:**
    - **Fetch This Week Data** (Google Sheets):
        - **Role:** Reads the primary data sheet.
        - **Configuration:** Targeted at a specific Spreadsheet ID and "Sheet1".
        - **Failure Point:** Authentication errors (OAuth2) or missing Spreadsheet ID.
    - **Fetch Last Week Data** (Google Sheets):
        - **Role:** Reads the same primary data sheet for the previous period.
        - **Configuration:** Mirror of the "This Week" node.
    - **Format Data for Claude** (Code Node):
        - **Technical Role:** Data transformation.
        - **Logic:** Filters the combined input arrays based on the dates provided by the `Configure Report Settings` node. It converts JSON rows into a pipe-separated text table (`Header | Header` \n `Value | Value`).
        - **Output:** A single JSON object containing `formatted_data` (the text table) and configuration metadata.

### 2.3 AI Analysis
**Overview:** Transforms raw numbers into professional business insights using a prompt-engineered LLM chain.

- **Nodes Involved:** `Analyse with Claude`, `Claude Sonnet`.
- **Node Details:**
    - **Analyse with Claude** (AI Chain):
        - **Role:** Orchestrates the prompt and the LLM call.
        - **Configuration:** A strict system prompt requiring a specific structure (Overall Health, WoW Summary, Wins, Concerns, Top 3 Actions, Trend).
        - **Input:** Injects `business_name`, `metrics_description`, and `formatted_data`.
    - **Claude Sonnet** (Anthropic Chat Model):
        - **Role:** The "brain" providing the analysis.
        - **Configuration:** Uses `claude-sonnet-4-5` with a low `temperature` (0.2) to ensure consistency and reduce hallucinations.
        - **Failure Point:** API Key expiration or quota limits on the Anthropic account.

### 2.4 Distribution & Archiving
**Overview:** Finalizes the process by communicating the result to stakeholders and recording the event.

- **Nodes Involved:** `Parse Report Output`, `Send Weekly Report`, `Log Report Run`.
- **Node Details:**
    - **Parse Report Output** (Set/Edit Fields):
        - **Role:** Data cleaning.
        - **Configuration:** Maps the AI's text output and the original configuration variables (email, date) into a flat object for easy use in downstream nodes.
    - **Send Weekly Report** (Gmail):
        - **Role:** Communication.
        - **Configuration:** Sends a plain-text email to `report_email`.
        - **Input:** Combines a greeting with the `report_text` generated by Claude.
    - **Log Report Run** (Google Sheets):
        - **Role:** Audit trail.
        - **Configuration:** Appends a new row to the "Report Log" tab.
        - **Fields:** Date, Status ("Sent"), a short summary (first 3 lines of the report), Business Name, and the week covered.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Schedule Trigger | Schedule Trigger | Temporal Trigger | - | Configure Report Settings | Schedule and configure |
| Configure Report Settings | Code | Variable Initialization | Weekly Schedule Trigger | Fetch This Week/Last Week | Schedule and configure |
| Fetch This Week Data | Google Sheets | Data Retrieval | Configure Report Settings | Format Data for Claude | Fetch and format data |
| Fetch Last Week Data | Google Sheets | Data Retrieval | Configure Report Settings | Format Data for Claude | Fetch and format data |
| Format Data for Claude | Code | Data Formatting | Fetch This/Last Week | Analyse with Claude | Fetch and format data |
| Analyse with Claude | AI Chain | Business Analysis | Format Data for Claude | Parse Report Output | Analyse with Claude AI |
| Claude Sonnet | LLM | Intelligence Engine | (Connected to Chain) | Analyse with Claude | Analyse with Claude AI |
| Parse Report Output | Set | Data Mapping | Analyse with Claude | Send Report / Log Run | Send report and log run |
| Send Weekly Report | Gmail | Communication | Parse Report Output | - | Send report and log run |
| Log Report Run | Google Sheets | Audit Logging | Parse Report Output | - | Send report and log run |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger and Setup
1. Create a **Schedule Trigger** node. Set the rule to `Weeks`, selecting `Monday` at `07:00`.
2. Create a **Code** node named `Configure Report Settings`. Paste the JavaScript provided in the workflow to define your `BUSINESS_NAME`, `REPORT_EMAIL`, and `METRICS_DESCRIPTION`.

### Step 2: Data Retrieval
3. Create two **Google Sheets** nodes named `Fetch This Week Data` and `Fetch Last Week Data`.
    - Set operation to `Get Many`.
    - Connect your Google Account.
    - Select your Spreadsheet ID and the sheet name (e.g., `Sheet1`).
    - Connect both to the `Configure Report Settings` node.

### Step 3: Data Formatting
4. Create a **Code** node named `Format Data for Claude`.
    - Connect both Google Sheets nodes to this node.
    - Use JavaScript to filter rows by the dates provided in the configuration node and join the data into a text-based table.

### Step 4: AI Integration
5. Create an **AI Chain** node named `Analyse with Claude`.
    - Set the prompt to the "Business Analyst" persona.
    - Ensure the prompt uses variables for `{{ $json.business_name }}`, `{{ $json.metrics_description }}`, and `{{ $json.formatted_data }}`.
6. Attach an **Anthropic Chat Model** node to the chain.
    - Select model `claude-sonnet-4-5`.
    - Set `temperature` to `0.2`.
    - Configure Anthropic API credentials.

### Step 5: Output and Logging
7. Create a **Set** node named `Parse Report Output` to consolidate the AI's response and the original configuration variables.
8. Create a **Gmail** node named `Send Weekly Report`.
    - Use the `report_email` variable for the recipient.
    - Construct the message body using a template: `Hi, ... [report_text]`.
9. Create a **Google Sheets** node named `Log Report Run`.
    - Set operation to `Append`.
    - Target the `Report Log` tab.
    - Map columns: `Date`, `Status`, `Summary`, `Business`, and `Week Covered`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Google Sheet Setup** | Requires a main data tab (Sheet1) and a logging tab (Report Log). |
| **API Requirements** | Requires active credentials for Google (OAuth2) and Anthropic (API Key). |
| **Cost Optimization** | Switch `Claude Sonnet` to `Claude Haiku` in the model node to reduce API costs for smaller datasets. |
| **Customization** | The system prompt in the `Analyse with Claude` node can be modified to change the "tone" or "KPI priorities" of the report. |