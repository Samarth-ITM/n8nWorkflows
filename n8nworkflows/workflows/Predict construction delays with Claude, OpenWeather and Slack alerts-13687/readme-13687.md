Predict construction delays with Claude, OpenWeather and Slack alerts

https://n8nworkflows.xyz/workflows/predict-construction-delays-with-claude--openweather-and-slack-alerts-13687


# Predict construction delays with Claude, OpenWeather and Slack alerts

# AI Construction Delay Predictor Reference Document

## 1. Workflow Overview
The **AI Construction Delay Predictor** is an advanced automation designed to monitor active construction projects and proactively identify potential schedule slippage. By integrating real-time weather forecasts, supplier procurement data, and labor availability, the workflow uses Claude AI (Anthropic) to generate risk scores and mitigation strategies.

The workflow is organized into four functional blocks:
1.  **Input & Project Loading:** Ingests project data via webhook or scheduled batches.
2.  **Risk Data Enrichment:** Fetches external context (Weather, Supply Chain, Resources).
3.  **AI Analysis:** Utilizes LLM agents to calculate delay probability and impact.
4.  **Action & Reporting:** Distributes alerts via Slack, creates Jira tickets, updates dashboards, and sends daily executive email briefings.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization
This block manages how the workflow starts and ensures all incoming data follows a consistent schema, whether it's a single project update or a daily audit of all active sites.

*   **Nodes Involved:** `Receive Project Alert Request`, `Morning Schedule Trigger`, `Load Active Projects`, `Normalize Project Records`.
*   **Node Details:**
    *   **Receive Project Alert Request (Webhook):** Listens for POST requests. Useful for real-time triggers from external PM tools.
    *   **Morning Schedule Trigger:** Fires every Monday–Saturday at 6:00 AM.
    *   **Load Active Projects (Airtable):** Searches for records where `Status = 'Active'`. Requires Airtable API credentials.
    *   **Normalize Project Records (Code):** A JavaScript node that maps fields from different sources (Webhook vs. Airtable) into a unified `project` object. It handles default values for coordinates and calculates a `jobRunId`.
    *   **Edge Cases:** Throws an error if site coordinates (Lat/Lon) are missing, as these are required for weather data.

### 2.2 Risk Data Enrichment
Collects the specific variables that influence construction timelines.

*   **Nodes Involved:** `Fetch 7-Day Site Weather`, `Fetch Supplier Delivery Status`, `Fetch Crew & Resource Availability`, `Merge Risk Data Streams`, `Compile Project Risk Profile`.
*   **Node Details:**
    *   **Fetch 7-Day Site Weather (HTTP Request):** Calls OpenWeatherMap OneCall API using project coordinates.
    *   **Fetch Supplier Delivery Status (HTTP Request):** Connects to the Procore API to retrieve purchase order statuses.
    *   **Fetch Crew & Resource Availability (Airtable):** Checks resource tables to compare assigned vs. available headcount.
    *   **Compile Project Risk Profile (Code):** The logic engine of this block. It parses raw API responses into "Risk Levels" (LOW/MEDIUM/HIGH) for weather, suppliers, and labor.
    *   **Failure Handling:** All fetch nodes have "Continue on Fail" enabled; the Code node contains "Synthetic demo fallback" logic to allow the workflow to proceed even if an API returns an error.

### 2.3 AI Processing & Scoring
The core intelligence phase where raw data is turned into a narrative and numerical risk assessment.

*   **Nodes Involved:** `Predict Delays with Claude AI`, `Claude AI Model`, `Parse AI Delay Prediction`.
*   **Node Details:**
    *   **Predict Delays with Claude AI (AI Agent):** A LangChain-based agent. The prompt instructs Claude to act as a "Senior Construction Risk Analyst (PMP)". It provides a structured prompt containing all enriched data from Block 2.
    *   **Claude AI Model (Anthropic):** Uses `claude-sonnet-4-20250514`. Temperature is set to `0.1` for high consistency and low "hallucination" in risk scoring.
    *   **Parse AI Delay Prediction (Code):** Extracts the JSON response from the AI. It maps the AI's severity (CRITICAL, HIGH, etc.) to internal priority labels (P1, P2) and determines if a Jira ticket or Slack escalation is required.

### 2.4 Severity Routing & Multi-Channel Reporting
The final stage where the workflow takes action based on the AI's findings.

*   **Nodes Involved:** `Route by Delay Severity`, `Send Delay Alert to Slack`, `Open Risk Ticket in Jira`, `Update Project Risk Dashboard`, `Build Daily Risk Briefing`, `Send Daily Email Briefing`, `Return Prediction to Caller`, `Log Low Risk — No Escalation`.
*   **Node Details:**
    *   **Route by Delay Severity (Switch):** Splits the path based on the `severity` string.
    *   **Send Delay Alert to Slack:** Uses Slack Blocks to send a rich, formatted alert with "Top Drivers" and "Immediate Actions" to specific channels (e.g., `#construction-critical`).
    *   **Open Risk Ticket in Jira:** Created only for "CRITICAL" and "HIGH" risks. Uses the Atlassian Document Format for the description.
    *   **Build Daily Risk Briefing (Code):** Aggregates results from all processed projects into a single HTML email report with color-coded status badges.
    *   **Send Daily Email Briefing (SMTP):** Sends the aggregated HTML report to stakeholders.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Project Alert Request | Webhook | Trigger | (None) | Load Active Projects | 1. Trigger & Project Loading |
| Morning Schedule Trigger | Schedule | Trigger | (None) | Load Active Projects | 1. Trigger & Project Loading |
| Load Active Projects | Airtable | Data Ingestion | Webhook / Schedule | Normalize Project Records | 1. Trigger & Project Loading |
| Normalize Project Records | Code | Data Formatting | Load Active Projects | Fetch Nodes | 1. Trigger & Project Loading |
| Fetch 7-Day Site Weather | HTTP Request | Weather Enrichment | Normalize Records | Merge Risk Data | 2. Risk Data Enrichment |
| Fetch Supplier Delivery Status | HTTP Request | Supply Enrichment | Normalize Records | Merge Risk Data | 2. Risk Data Enrichment |
| Fetch Crew & Resource | Airtable | Labor Enrichment | Normalize Records | Merge Risk Data | 2. Risk Data Enrichment |
| Merge Risk Data Streams | Merge | Sync Data | Fetch Nodes | Compile Risk Profile | 2. Risk Data Enrichment |
| Compile Project Risk Profile | Code | Logic/Parsing | Merge Risk Data | Predict Delays (AI) | 2. Risk Data Enrichment |
| Predict Delays with Claude AI | AI Agent | Risk Analysis | Compile Risk Profile | Parse AI Prediction | 3. Claude AI Delay Prediction |
| Claude AI Model | Anthropic | LLM Engine | Predict Delays (AI) | Predict Delays (AI) | 3. Claude AI Delay Prediction |
| Parse AI Delay Prediction | Code | JSON Parser | Predict Delays (AI) | Route by Severity | 3. Claude AI Delay Prediction |
| Route by Delay Severity | Switch | Routing | Parse AI Prediction | Slack / Log Node | 4. Severity Routing & Alerts |
| Send Delay Alert to Slack | HTTP Request | High-Risk Alert | Route by Severity | Open Risk Ticket | 4. Severity Routing & Alerts |
| Open Risk Ticket in Jira | HTTP Request | Task Creation | Slack Alert | Update Dashboard | 4. Severity Routing & Alerts |
| Update Project Risk Dashboard | Google Sheets | Reporting | Jira / Log Node | Build Daily Briefing | 4. Severity Routing & Alerts |
| Build Daily Risk Briefing | Code | Report Generation | Update Dashboard | Send Daily Email | 4. Severity Routing & Alerts |
| Send Daily Email Briefing | Email Send | Stakeholder Comms | Build Briefing | Return to Caller | 4. Severity Routing & Alerts |
| Return Prediction to Caller | Webhook Resp. | Response | Send Email | (None) | 4. Severity Routing & Alerts |
| Log Low Risk — No Escalation | Set | Logging | Route by Severity | Update Dashboard | 4. Severity Routing & Alerts |

---

## 4. Reproducing the Workflow from Scratch

1.  **Set up Triggers:** Create a Webhook node (POST) and a Schedule node (6 AM Cron). Link both to an Airtable node.
2.  **Airtable Integration:** Configure the Airtable node to "Search" your "Projects" table. Ensure your table has columns for `Status`, `Phase`, `Project Name`, `Site Lat`, and `Site Lon`.
3.  **Data Normalization:** Add a Code node using JavaScript to create a standard `project` object. This ensures that whether data comes from Airtable or Webhook, it looks the same for downstream nodes.
4.  **Parallel Enrichment:**
    *   Add an **HTTP Request** node for OpenWeatherMap. Use the expression `{{ $json.project.siteLat }}` in the query.
    *   Add an **HTTP Request** node for Procore (or your procurement tool).
    *   Add an **Airtable** node to search your "Resources" table using the Project ID.
5.  **Merge & Profile:** Use a **Merge** node (By Position) followed by a **Code** node. In this Code node, write the logic to calculate "Workable Days" (Weather) and "Capacity %" (Staffing).
6.  **AI Integration:**
    *   Add an **AI Agent** node.
    *   Connect an **Anthropic Chat Model** node to it (Claude 3.5 Sonnet recommended).
    *   In the Agent's "Prompt", copy the construction risk analyst instructions provided in the analysis section above.
7.  **Response Parsing:** Add a Code node to `JSON.parse` the AI's string response into a usable n8n object.
8.  **Routing:** Use a **Switch** node based on the severity returned by the AI (CRITICAL, HIGH, MEDIUM, LOW).
9.  **Action Nodes:**
    *   **Slack:** Configure an HTTP node or Slack node with "Blocks" to visualize the risk.
    *   **Jira:** Configure an HTTP node to the Jira API to create "Issue Type: Risk".
    *   **Google Sheets:** Use the "Append" operation to keep a log of every prediction.
10. **Aggregation:** Add a final **Code** node to aggregate all items in the run and build an HTML table for the daily email.
11. **Finalize:** Add the **Email Send** node and a **Respond to Webhook** node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Custom AI Growth Strategy & Support | [Contact OneClick IT Solution](https://www.oneclickitsolution.com/contact-us/) |
| Recommended AI Model | Claude 3.5 Sonnet for best JSON adherence |
| Weather API Requirement | Requires OpenWeatherMap OneCall 3.0 subscription |
| Procore API Requirement | Requires Developer App Client ID and Company ID |

**Disclaimer:** This workflow is a logical framework. Ensure all API keys and Base IDs are updated in the parameters before activation.