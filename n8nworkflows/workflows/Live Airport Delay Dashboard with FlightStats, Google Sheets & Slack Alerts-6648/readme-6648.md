Live Airport Delay Dashboard with FlightStats, Google Sheets & Slack Alerts

https://n8nworkflows.xyz/workflows/live-airport-delay-dashboard-with-flightstats--google-sheets---slack-alerts-6648


# Live Airport Delay Dashboard with FlightStats, Google Sheets & Slack Alerts

### 1. Workflow Overview

This workflow automates the monitoring of live airport delay data, storing it for reporting and triggering team alerts for severe delays. It is designed for operational teams needing timely updates on airport delays, integrating FlightStats API, Google Sheets for data logging, and Slack for real-time notifications.

Logical blocks:

- **1.1 Schedule Trigger:** Automatically triggers the workflow hourly to fetch delay data.
- **1.2 Data Retrieval:** Calls FlightStats API to obtain live delay statistics for a specific airport.
- **1.3 Data Storage:** Appends the retrieved delay data into a Google Sheet for dashboarding or analysis.
- **1.4 Delay Evaluation:** Checks if the delay exceeds a threshold (30 minutes) to decide alert necessity.
- **1.5 Alert Dispatch:** Sends Slack alerts for severe delays; otherwise, no action is taken.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow execution every hour without manual intervention to ensure continuous monitoring of airport delays.

- **Nodes Involved:**  
  - Set Schedule

- **Node Details:**  
  - **Set Schedule**  
    - Type: Cron Trigger  
    - Configuration: Default cron node set to trigger hourly (exact timing defaults, but can be customized). No parameters specified in the JSON, meaning the default hourly trigger is used.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "FlightStats API" node  
    - Version: v1  
    - Edge Cases: Cron misconfiguration could cause missed triggers or too frequent executions. Timezone considerations may impact the timing relative to API updates.  

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches live delay data from FlightStats API for a specified airport and current hour.

- **Nodes Involved:**  
  - FlightStats API

- **Node Details:**  
  - **FlightStats API**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.flightstats.com/flex/delay/index` (FlightStats delay endpoint)  
      - Method: GET (default for HTTP Request node)  
      - Parameters: Not specified in JSON but expected to include airport code (e.g., JFK) and current hour as query parameters for live data.  
    - Inputs: Trigger from "Set Schedule"  
    - Outputs: Passes data to "Set Output Data"  
    - Version: v1  
    - Edge Cases:  
      - API authentication or quota limits failures  
      - Network timeouts or errors  
      - Unexpected API response format changes  
      - Missing or invalid airport code parameters  
    - Notes: No explicit authentication setup shown; likely configured via credentials or headers not detailed here.  

#### 1.3 Data Storage

- **Overview:**  
  Appends the fetched airport delay data (airport code and delay minutes) into a Google Sheet for record-keeping and dashboard visualization.

- **Nodes Involved:**  
  - Set Output Data

- **Node Details:**  
  - **Set Output Data**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append rows  
      - Range: Columns A:B (assumed to hold airport code and delay minutes)  
      - Sheet ID: Placeholder `"your_sheet_id"` (needs to be replaced with actual Google Sheet ID)  
    - Credentials: Google API OAuth2 credentials named "Google Sheets- test"  
    - Inputs: Data from "FlightStats API" node  
    - Outputs: Passes data to "Merge API Data"  
    - Version: v1  
    - Edge Cases:  
      - Invalid or missing sheet ID  
      - Permission/credential errors  
      - API rate limits or quota exceeded  
      - Data format mismatch causing append failures  

#### 1.4 Delay Evaluation

- **Overview:**  
  Applies conditional logic to determine if the delay is severe (greater than 30 minutes). Based on this, it routes the workflow to either alert or no action.

- **Nodes Involved:**  
  - Merge API Data

- **Node Details:**  
  - **Merge API Data**  
    - Type: If node (conditional branching)  
    - Configuration:  
      - Condition: Numeric check if `delayMinutes` from FlightStats API response is greater than 30  
    - Inputs: Data from "Set Output Data"  
    - Outputs:  
      - True branch: Sends data to "Send Response via Slack"  
      - False branch: Sends data to "No Action for Minor Delays"  
    - Version: v1  
    - Edge Cases:  
      - Missing or malformed `delayMinutes` field causing expression errors  
      - Type coercion issues if delayMinutes is not numeric  
      - Unhandled nulls or API errors propagating  

#### 1.5 Alert Dispatch

- **Overview:**  
  Notifies the ops-sales team on Slack when severe delays are detected; otherwise, no alert is sent.

- **Nodes Involved:**  
  - Send Response via Slack  
  - No Action for Minor Delays

- **Node Details:**  
  - **Send Response via Slack**  
    - Type: Slack  
    - Configuration:  
      - Channel: `ops-sales`  
      - Message Text: Dynamic text including airport code, delay minutes, and timestamp in ISO format  
    - Credentials: Slack OAuth2 credentials named "Slack account - test"  
    - Inputs: True branch from "Merge API Data"  
    - Outputs: None (terminal node)  
    - Version: v1  
    - Edge Cases:  
      - Slack API authentication failures  
      - Channel not found or permission errors  
      - Slack API rate limiting  
      - Message formatting or expression errors  
  - **No Action for Minor Delays**  
    - Type: NoOp (no operation)  
    - Configuration: None (serves as a placeholder for minor delays)  
    - Inputs: False branch from "Merge API Data"  
    - Outputs: None (terminal node)  
    - Version: v1  
    - Edge Cases: None (safe placeholder)  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                        | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                                                       |
|-------------------------|---------------------|-------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Set Schedule            | Cron Trigger        | Triggers workflow hourly             | None                  | FlightStats API           | Triggers the workflow every hour to fetch live airport delay data automatically.                                                 |
| FlightStats API         | HTTP Request        | Fetches live delay data from API     | Set Schedule           | Set Output Data           | Fetches live delay data from the FlightStats API for the specified airport (e.g., JFK) using the current hour.                  |
| Set Output Data         | Google Sheets       | Stores delay data into Google Sheets | FlightStats API        | Merge API Data            | Stores the fetched delay data (airport code and delay minutes) into a Google Sheet for display.                                  |
| Merge API Data          | If (Conditional)    | Checks if delay exceeds 30 minutes   | Set Output Data         | Send Response via Slack, No Action for Minor Delays | Checks if the delay exceeds 30 minutes to determine if alert is needed.                                                          |
| Send Response via Slack | Slack               | Sends severe delay alert to Slack    | Merge API Data (true)  | None                      | Sends a Slack alert to the 'ops-sales' channel if a severe delay is detected, including airport and delay details.              |
| No Action for Minor Delays | NoOp              | Placeholder for minor delays          | Merge API Data (false) | None                      | Placeholder node for delays under 30 minutes, where no alert is sent.                                                            |
| Sticky Note             | Sticky Note         | Architectural overview note           | None                  | None                      | ## System Architecture - Delay Monitoring Pipeline: Set Schedule, FlightStats API; Data Management Flow: Set Output Data, Merge API Data; Alert and Display: Send Response via Slack, No Action for Minor Delays |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Set Schedule" Node**  
   - Type: Cron Trigger  
   - Parameters: Set to trigger every hour (e.g., minute 0 of every hour)  
   - No credentials required  
   - Position at workflow start  
   - Connect output to "FlightStats API" node  

2. **Create "FlightStats API" Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.flightstats.com/flex/delay/index`  
     - Method: GET  
     - Query Parameters: Include airport code (e.g., `airportCode=JFK`), current date/time as required by FlightStats API (use expressions for dynamic time)  
   - Configure authentication as per FlightStats API (API key or OAuth if required)  
   - Connect input from "Set Schedule"  
   - Connect output to "Set Output Data"  

3. **Create "Set Output Data" Node**  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: Append  
     - Range: A:B (two columns for airport code and delay minutes)  
     - Sheet ID: Insert your Google Sheet ID  
   - Connect input from "FlightStats API"  
   - Connect output to "Merge API Data"  
   - Assign Google API credentials with edit permissions  

4. **Create "Merge API Data" Node**  
   - Type: If (Conditional)  
   - Parameters:  
     - Condition: Check if `{{$node["FlightStats API"].json["delayMinutes"]}} > 30`  
   - Connect input from "Set Output Data"  
   - True branch connects to "Send Response via Slack"  
   - False branch connects to "No Action for Minor Delays"  

5. **Create "Send Response via Slack" Node**  
   - Type: Slack  
   - Parameters:  
     - Channel: `ops-sales`  
     - Text: `"🚨 Severe Delay Alert: {{$node['FlightStats API'].json['airportCode']}} has a delay of {{$node['FlightStats API'].json['delayMinutes']}} minutes at {{$now.toISOString()}}"`  
   - Assign Slack credentials with permission to post in `ops-sales` channel  
   - Connect input from true branch of "Merge API Data"  
   - No output (terminal)  

6. **Create "No Action for Minor Delays" Node**  
   - Type: NoOp  
   - Parameters: None  
   - Connect input from false branch of "Merge API Data"  
   - No output (terminal)  

7. **Optional: Add Sticky Note**  
   - Content summarizing the system architecture as described in section 1  
   - Place visually for clarity  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| System Architecture includes three main pipelines: Delay Monitoring, Data Management, and Alert/Display.              | Sticky Note node content summarized in section 3                                               |
| Slack alert message uses ISO 8601 timestamp for clarity in alert timing.                                              | Message template in "Send Response via Slack" node                                             |
| Google Sheet ID must be replaced with actual sheet ID with appropriate permissions for append operations.            | Google Sheets node configuration                                                               |
| FlightStats API endpoint and parameters need correct API key and parameter setup as per FlightStats documentation.   | FlightStats API official docs: https://developer.flightstats.com/api-docs                      |
| OAuth2 credentials must be properly configured for Slack and Google APIs before running workflow.                     | Credential setup required in n8n settings                                                      |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.