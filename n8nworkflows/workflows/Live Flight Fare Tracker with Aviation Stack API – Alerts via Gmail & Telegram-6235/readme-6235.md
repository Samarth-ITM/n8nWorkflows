Live Flight Fare Tracker with Aviation Stack API – Alerts via Gmail & Telegram

https://n8nworkflows.xyz/workflows/live-flight-fare-tracker-with-aviation-stack-api---alerts-via-gmail---telegram-6235


# Live Flight Fare Tracker with Aviation Stack API – Alerts via Gmail & Telegram

### 1. Workflow Overview

This workflow automates live tracking of flight fares for a specific route (JFK to LAX) using the Aviation Stack API. It periodically fetches flight data, simulates fare pricing, compares current fares with historical data stored in Google Sheets, and triggers alerts when significant price changes occur. Notifications are sent via email (Gmail) and Telegram, with detailed logging of alert activities.

The workflow is logically divided into the following blocks:

- **1.1 Data Ingestion & Processing**: Scheduled trigger initiates the workflow; flight data is fetched from Aviation Stack API and processed to extract relevant flight and fare details. Historical fare data is read from Google Sheets.

- **1.2 Fare Comparison & Alert Logic**: Current fares are compared against stored fares to detect significant changes (drops or increases). Alerts are generated based on predefined thresholds.

- **1.3 Notification & Logging**: Alert messages are formatted for multiple channels, notifications are sent via Gmail and Telegram, and alert activities are logged for auditing.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Ingestion & Processing

**Overview:**  
This block handles the periodic fetching of live flight data and prepares it for comparison by extracting and formatting required details. It also retrieves historical fare data from Google Sheets to enable change detection.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Flight Data  
- Process Flight Data  
- Previous flight data (Google Sheets node)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Cron Trigger  
  - *Role:* Initiates the workflow every 15 minutes.  
  - *Configuration:* Executes every 15 minutes (`everyX` mode).  
  - *Connections:* Outputs to "Fetch Flight Data".  
  - *Edge Cases:* If the node fails or the schedule is interrupted, data fetching will be delayed. No authentication required.  

- **Fetch Flight Data**  
  - *Type:* HTTP Request  
  - *Role:* Calls Aviation Stack API to fetch live flight data for JFK to LAX route.  
  - *Configuration:*  
    - URL: `https://api.aviationstack.com/v1/flights`  
    - Query Parameters:  
      - `access_key` (API key)  
      - `dep_iata` = "JFK"  
      - `arr_iata` = "LAX"  
      - `limit` = 10 (limit results to 10 flights)  
    - Authentication: HTTP Query Auth with stored credentials.  
  - *Connections:* Outputs to "Process Flight Data".  
  - *Edge Cases:*  
    - API rate limits or invalid API key may cause auth failures.  
    - Network timeouts or malformed responses must be handled downstream.  

- **Process Flight Data**  
  - *Type:* Function Node  
  - *Role:* Parses API response, filters scheduled flights, extracts flight details, and generates mock fare data (randomized).  
  - *Configuration:* Custom JavaScript code extracts:  
    - Flight number, airline name, departure/arrival airports and times  
    - Generates a mock current fare between $200 and $700 (randomized)  
    - Includes route code and timestamp  
  - *Connections:* Outputs to "Previous flight data" node.  
  - *Edge Cases:*  
    - Missing or malformed flight data (no departure info) causes exclusion.  
    - Mock fare is placeholder; real fare integration needs API support.  

- **Previous flight data (Google Sheets)**  
  - *Type:* Google Sheets Read  
  - *Role:* Reads stored historical fare data to compare with current fares.  
  - *Configuration:*  
    - Document ID and Sheet Name preconfigured (linked to a Google Sheet with fare logs).  
    - Authentication via Service Account credentials.  
  - *Connections:* Outputs to "Code" node for fare comparison.  
  - *Edge Cases:*  
    - Credential expiry or permission issues can break data reading.  
    - Empty or inconsistent spreadsheet data may affect logic downstream.  

---

#### 1.2 Fare Comparison & Alert Logic

**Overview:**  
This critical logic block compares current flight fares against historical fares, calculates fare changes, and determines if an alert should be triggered based on predefined thresholds (≥10% drop or ≥15% increase). Alerts are only forwarded if a significant change is detected.

**Nodes Involved:**  
- Code (JavaScript)  
- Check if Alert Needed (If node)

**Node Details:**

- **Code**  
  - *Type:* Function Node (v2)  
  - *Role:* Compares current and previous fares, calculates absolute and percentage changes, determines alert conditions, prepares fare update data for Google Sheets.  
  - *Key Logic:*  
    - Builds lookup from Google Sheets data keyed by flight number and route to get previous fares.  
    - Iterates current flights, compares fares, triggers alerts for significant changes:  
      - Price drop ≥ 10%  
      - Price increase ≥ 15%  
    - Prepares updates for Google Sheets to refresh stored fares after processing.  
  - *Connections:* Outputs alert flights to "Check if Alert Needed".  
  - *Edge Cases:*  
    - Missing flight numbers or routes are skipped with logs.  
    - Zero or invalid current fares ignored.  
    - Duplicate rows in sheets handled by deduplication.  
    - If no alerts, returns empty to halt notification flow.  

- **Check if Alert Needed**  
  - *Type:* If Node  
  - *Role:* Checks if fare_change value is non-zero to decide if alert formatting and notification should proceed.  
  - *Configuration:* Condition: fare_change != 0  
  - *Connections:* True branch to "Format Alert Message"; false branch ends workflow for that item.  
  - *Edge Cases:* Reliable only if previous node outputs correct `fare_change`; malformed data may cause false negatives.  

---

#### 1.3 Notification & Logging

**Overview:**  
This block formats alert messages for multiple channels, sends notifications via Gmail and Telegram, and logs alert activity for audit purposes.

**Nodes Involved:**  
- Format Alert Message (Function)  
- Send a message (Gmail)  
- Telegram  
- Log Alert Activity (Function)

**Node Details:**

- **Format Alert Message**  
  - *Type:* Function Node  
  - *Role:* Generates formatted alert messages in email HTML, SMS text, and Slack-like attachment formats based on alert type (price drop or increase).  
  - *Configuration:*  
    - Uses flight data and alert type to customize message content, emojis, colors, and recommendations.  
  - *Connections:* Outputs to "Send a message" and "Telegram" nodes.  
  - *Edge Cases:*  
    - Date formatting depends on client locale; could vary.  
    - Missing alert_type or fare data may produce incomplete messages.  

- **Send a message (Gmail)**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email with the formatted alert message.  
  - *Configuration:*  
    - Recipient email hardcoded as `abc@gmail.com` (change as needed).  
    - Subject and message body from formatted message.  
    - OAuth2 authentication with configured Gmail credentials.  
  - *Connections:* Outputs to "Log Alert Activity".  
  - *Edge Cases:*  
    - OAuth token expiry or permission issues may cause send failures.  
    - Invalid recipient address will cause delivery errors.  

- **Telegram**  
  - *Type:* Telegram Node  
  - *Role:* Sends SMS-style alert message via Telegram bot.  
  - *Configuration:*  
    - Chat ID hardcoded (`123SSHSJNASB`) and uses formatted SMS message.  
    - Uses Telegram API credentials.  
  - *Connections:* Outputs to "Log Alert Activity".  
  - *Edge Cases:*  
    - Incorrect chat ID or revoked bot token will cause failures.  
    - Network or API rate limit issues possible.  
  - *Error Handling:* Configured to continue on error without stopping workflow.  

- **Log Alert Activity**  
  - *Type:* Function Node  
  - *Role:* Logs alert details including timestamp, flight info, fare changes, and notification status to console for monitoring.  
  - *Connections:* End node (no further outputs).  
  - *Edge Cases:*  
    - No external persistent logging; console output only. Consider extending with database or file write for production.  

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)                                | Sticky Note                                                                                                                  |
|-----------------------|---------------------|----------------------------------------|-------------------------|-----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Cron Trigger        | Initiates workflow every 15 minutes    | —                       | Fetch Flight Data                             | ### 📊 Data Ingestion & Processing                                                                                            |
| Fetch Flight Data      | HTTP Request        | Fetch flight data from AviationStack   | Schedule Trigger        | Process Flight Data                           | ### 📊 Data Ingestion & Processing                                                                                            |
| Process Flight Data    | Function            | Extract and format flight data, mock fare | Fetch Flight Data       | Previous flight data                          | ### 📊 Data Ingestion & Processing                                                                                            |
| Previous flight data   | Google Sheets       | Reads historical fare data             | Process Flight Data      | Code                                          | ### 📊 Data Ingestion & Processing                                                                                            |
| Code                  | Function            | Compare fares, generate alerts, prepare updates | Previous flight data     | Check if Alert Needed                         | ### 🔍 Fare Comparison & Alert Logic                                                                                          |
| Check if Alert Needed  | If                  | Check if fare change warrants alert    | Code                    | Format Alert Message                          | ### 🔍 Fare Comparison & Alert Logic                                                                                          |
| Format Alert Message   | Function            | Format alert messages for channels     | Check if Alert Needed    | Send a message, Telegram                      | ### 🔔 Notification & Logging                                                                                                |
| Send a message        | Gmail               | Send email notification                 | Format Alert Message     | Log Alert Activity                            | ### 🔔 Notification & Logging                                                                                                |
| Telegram              | Telegram            | Send Telegram notification              | Format Alert Message     | Log Alert Activity                            | ### 🔔 Notification & Logging                                                                                                |
| Log Alert Activity     | Function            | Log alert details                       | Send a message, Telegram | —                                             | ### 🔔 Notification & Logging                                                                                                |
| Workflow Overview     | Sticky Note         | Describes workflow purpose              | —                       | —                                             | ## ✈️ Flight Fare Tracker & Alert System<br>This workflow is designed to monitor flight prices and send alerts when significant fare changes occur (drops or increases). It automates the process of fetching flight data, comparing current fares against historical records, and notifying users via email or Telegram if an alert condition is met. 📉📈 |
| Data Ingestion & Processing | Sticky Note     | Explains nodes for data fetching and processing | —                       | —                                             | ### 📊 Data Ingestion & Processing<br>1.  **Schedule Trigger**: Starts the workflow at regular intervals.<br>2.  **Fetch Flight Data**: Retrieves flight information from the AviationStack API.<br>3.  **Process Flight Data**: Extracts and formats flight details.<br>4.  **Google Sheets**: Reads historical fare data. |
| Fare Comparison & Alert Logic | Sticky Note    | Explains fare comparison and alert logic | —                       | —                                             | ### 🔍 Fare Comparison & Alert Logic<br>1.  **Code**: Compares fares and determines alerts.<br>2.  **Check if Alert Needed**: Filters flights needing notification. |
| Notification & Logging | Sticky Note         | Explains notification and logging steps | —                       | —                                             | ### 🔔 Notification & Logging<br>1.  **Format Alert Message**: Prepares message formats.<br>2.  **Gmail**: Sends email.<br>3.  **Telegram**: Sends Telegram message.<br>4.  **Log Alert Activity**: Logs alerts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new Workflow in n8n.**

2. **Add a "Schedule Trigger" node:**
   - Type: Cron Trigger  
   - Configure to run every 15 minutes (`everyX` mode, value: 15, unit: minutes).  
   - Position the node at the start.  

3. **Add an "HTTP Request" node named "Fetch Flight Data":**
   - URL: `https://api.aviationstack.com/v1/flights`  
   - Method: GET  
   - Authentication: HTTP Query Auth, using your Aviation Stack API key credential.  
   - Query Parameters:  
     - `access_key`: your API key  
     - `dep_iata`: "JFK"  
     - `arr_iata`: "LAX"  
     - `limit`: 10  
   - Connect "Schedule Trigger" main output to this node.  

4. **Add a "Function" node named "Process Flight Data":**
   - Purpose: Extract flight details and mock fare.  
   - Code: Use JavaScript to parse `items[0].json.data` array, filter scheduled flights, and for each flight return:  
     - flight_number (iata code)  
     - airline name  
     - departure and arrival airport names  
     - scheduled departure and arrival time  
     - current_fare (random between 200 and 700)  
     - route string (e.g., JFK-LAX)  
     - timestamp (current ISO string)  
   - Connect "Fetch Flight Data" output to this node.  

5. **Add a "Google Sheets" node named "Previous flight data":**
   - Operation: Read rows from your Google Sheet that stores previous fare information.  
   - Authentication: Service Account credentials configured.  
   - Choose the spreadsheet ID and sheet name (e.g., "fare details change logs", Sheet1).  
   - Connect "Process Flight Data" output to this node.  

6. **Add a "Function" node named "Code":**
   - Purpose: Compare current fares with stored fares, calculate changes, determine alerts, prepare updates.  
   - Code:  
     - Build a lookup from Google Sheets data keyed by flight number + route.  
     - Iterate over current flight data, compare current_fare and previous_fare.  
     - Calculate absolute and percentage change.  
     - Trigger alert if % change ≤ -10 (price drop) or ≥ 15 (price increase).  
     - Prepare data to update Google Sheets fares.  
     - Return alert flights only if alerts exist; else return empty array.  
   - Connect "Previous flight data" output to this node.  

7. **Add an "If" node named "Check if Alert Needed":**
   - Condition: `fare_change` field is not equal to 0.  
   - Connect "Code" output to this node.  
   - True branch proceeds to alert formatting.  

8. **Add a "Function" node named "Format Alert Message":**
   - Purpose: Format alert messages for email, SMS, and Slack-like attachments.  
   - Code:  
     - Use flight data and alert type to compose:  
       - Email subject and HTML body (includes flight details, fare changes, recommendations).  
       - SMS message text.  
       - Slack attachment object with colors and fields.  
   - Connect "Check if Alert Needed" true output to this node.  

9. **Add a "Gmail" node named "Send a message":**
   - Purpose: Send email notification.  
   - Credentials: Gmail OAuth2 account configured.  
   - Recipient: Set the target email (e.g., `abc@gmail.com`).  
   - Subject and message body pulled from formatted alert message.  
   - Connect "Format Alert Message" output to this node.  

10. **Add a "Telegram" node:**
    - Purpose: Send Telegram message with alert SMS text.  
    - Credentials: Telegram Bot API configured.  
    - Chat ID: Set to your Telegram chat ID.  
    - Message: Use SMS formatted message from alert data.  
    - Set error handling to continue on error to avoid workflow interruption.  
    - Connect "Format Alert Message" output to this node.  

11. **Add a "Function" node named "Log Alert Activity":**
    - Purpose: Log alert details (timestamp, flight number, fare changes, notification sent).  
    - Code: Log to console or extend to persistent storage.  
    - Connect outputs of both "Send a message" and "Telegram" nodes to this node.  

12. **Set execution order and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                                        |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses mock fare data due to API limitations; integrate with a real fare API for production use.       | Customization for actual fare data retrieval.                                                                                          |
| Telegram node error handling is set to continue on failure to avoid blocking workflow execution.                  | Reliable alert delivery without workflow interruption.                                                                                 |
| Google Sheets acts as the historical fare database; ensure sheet structure matches expected fields.               | Sheet must include flight_number, route, current_fare, row_number, and timestamps.                                                     |
| The Send a message node uses Gmail OAuth2 credentials; ensure tokens are refreshed and valid for uninterrupted mail sending. | Gmail API quota and permission management.                                                                                            |
| For more info on Aviation Stack API, see: [https://aviationstack.com/documentation](https://aviationstack.com/documentation) | Useful for customizing flight data queries and expanding data fields.                                                                 |

---

This document provides a complete, detailed reference for understanding, modifying, or recreating the Live Flight Fare Tracker workflow with alerts via Gmail and Telegram.