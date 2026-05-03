Generate GLPI Support Performance Reports with SLA Tracking & Email Delivery

https://n8nworkflows.xyz/workflows/generate-glpi-support-performance-reports-with-sla-tracking---email-delivery-9210


# Generate GLPI Support Performance Reports with SLA Tracking & Email Delivery

---

## 1. Workflow Overview

This workflow automates the generation of monthly technical support performance reports from GLPI tickets, focusing on SLA (Service Level Agreement) tracking. It extracts ticket data for the previous month, calculates working hours spent on each ticket within defined business hours, assesses SLA compliance, compiles summarized metrics overall and per technician, generates a styled HTML report, and emails it to designated recipients. The workflow includes the following logical blocks:

- **1.1 Schedule and Date Range Calculation**: Triggered monthly on the 6th, calculates the report month and date range for data extraction.
- **1.2 GLPI Session Management and Data Retrieval**: Authenticates with the GLPI API, retrieves tickets matching the date range and entity.
- **1.3 Data Preparation and Metrics Calculation**: Formats ticket fields, calculates business-hour-based metrics, and evaluates SLA compliance.
- **1.4 Report Generation and Email Delivery**: Creates a detailed HTML report and sends it via Gmail, then logs out of the GLPI session.
- **1.5 Workflow Variables and Configuration**: Centralized node for all configurable parameters such as server URL, tokens, working hours, and SLA limits.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule and Date Range Calculation

**Overview:**  
This block triggers the workflow monthly (on the 6th day) and computes the date range covering the entire previous month to query tickets accurately.

**Nodes Involved:**  
- Schedule Trigger  
- Date range

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow monthly on the 6th day.  
  - Configuration: Interval set to months, triggering at day 6.  
  - Connections: Outputs trigger to Date range node.  
  - Edge Cases: Trigger misfires if server timezone or daylight savings mismatch.  

- **Date range**  
  - Type: Code (JavaScript)  
  - Role: Calculates previous month’s start and end dates, plus a formatted month name for report labeling.  
  - Key Logic:  
    - Extracts current date from trigger input.  
    - Determines previous month.  
    - Sets startDate to last day of two months ago (to cover prior month fully).  
    - Sets endDate to first day of next month after previous month.  
    - Outputs JSON with month label, startDate, endDate.  
  - Connections: Outputs to Variables node.  
  - Edge Cases: Handles date rollovers correctly; formatting strict to ISO date.  

---

### 2.2 GLPI Session Management and Data Retrieval

**Overview:**  
This block manages API session with GLPI, authenticates using credentials, fetches tickets filtered by date range and entity, and splits the retrieved data for processing.

**Nodes Involved:**  
- Variables  
- Get session token  
- Get tickets  
- Split Out

**Node Details:**

- **Variables**  
  - Type: Set  
  - Role: Defines key configuration parameters such as GLPI server URL, app token, entity name, working hours, and SLA limits.  
  - Key Parameters:  
    - Server URL (e.g., https://server_glpi.com/)  
    - App Token for API authentication  
    - Entity name filter  
    - Working hours (start, lunch break start/end, end)  
    - SLA limit in hours (default 24)  
  - Connections: Outputs to Get session token node.  
  - Edge Cases: Token or URL misconfiguration will fail subsequent API requests.

- **Get session token**  
  - Type: HTTP Request  
  - Role: Initiates GLPI API session, retrieving a session token for authenticated requests.  
  - Configuration:  
    - URL: `${Server URL}apirest.php/initSession`  
    - Headers: Content-Type JSON, App-Token from variables  
    - Authentication: Basic Auth (username/password stored in n8n credential "Api GLPI")  
  - Connections: Outputs session token to Get tickets node.  
  - Edge Cases: Authentication failures, API downtime, token expiration.

- **Get tickets**  
  - Type: HTTP Request  
  - Role: Fetches tickets from GLPI matching the criteria: opening date within date range, entity matches, limited to 999 records.  
  - Configuration:  
    - URL: `${Server URL}apirest.php/search/Ticket`  
    - Query Parameters:  
      - Filter by opening date greater than startDate and less than endDate  
      - Entity contains variable entity name  
      - Order descending, range 0-999  
    - Headers: Content-Type JSON, Session-Token and App-Token from previous nodes  
    - Authentication: Basic Auth (same credential)  
  - Connections: Outputs tickets array to Split Out node.  
  - Edge Cases: Large datasets exceeding range limit, session token expiry, network timeouts.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the tickets array into individual ticket items for downstream processing.  
  - Configuration: Field to split: "data" (from GLPI tickets response)  
  - Connections: Outputs each ticket to Edit Fields node.  
  - Edge Cases: Empty data array, malformed response.

---

### 2.3 Data Preparation and Metrics Calculation

**Overview:**  
This block formats relevant ticket fields, computes business hours spent on each ticket considering work schedules and weekends, evaluates SLA compliance, and summarizes performance metrics both globally and by technician.

**Nodes Involved:**  
- Edit Fields  
- Metrics

**Node Details:**

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts and renames GLPI ticket fields for clarity and further processing.  
  - Key Fields Mapped:  
    - Ticket_id (from JSON field 2)  
    - Title (field 1)  
    - technical_id (field 5)  
    - Entity (field 80)  
    - Opening date (field 15)  
    - Solution date (field 17)  
    - Closing date (field 16)  
    - Status (field 12)  
  - Connections: Outputs to Metrics node.  
  - Edge Cases: Missing or null fields cause inaccurate calculations downstream.

- **Metrics**  
  - Type: Code (JavaScript)  
  - Role: Calculates business hours between ticket opening and resolution/closure, checks SLA compliance, aggregates data into general and per-technician reports.  
  - Key Logic:  
    - Defines working hours from Variables node (WORK_START, LUNCH_START, LUNCH_END, WORK_END)  
    - Defines SLA limit (default 24 hours)  
    - Functions:  
      - `businessHoursDiff` calculates elapsed working hours excluding weekends and lunch breaks  
      - `decimalToHM` formats hours as "Xh Ym"  
    - Processes each ticket:  
      - Parses dates, calculates effective hours  
      - Determines SLA compliance  
      - Aggregates counts by status: open (1), in progress (2), solved (5), closed (6)  
      - Aggregates totals and SLA metrics globally and per technician  
    - Outputs two reports: General summary and Technician-wise detail.  
  - Connections: Outputs report data to Generate report node.  
  - Edge Cases: Tickets without solution or closing dates, invalid date formats, tickets spanning weekends/holidays not accounted beyond weekends.

---

### 2.4 Report Generation and Email Delivery

**Overview:**  
This block converts the metrics data into a visually appealing HTML report, sends it via Gmail, and safely ends the GLPI API session.

**Nodes Involved:**  
- Generate report  
- Send a message  
- End session  
- No Operation, do nothing

**Node Details:**

- **Generate report**  
  - Type: HTML  
  - Role: Builds a styled HTML report presenting general and technician-specific ticket metrics with conditional formatting and alerts.  
  - Key Features:  
    - Responsive design for readability on different devices  
    - Dynamic insertion of data from Metrics node  
    - Color-coded SLA compliance indicators  
    - Alerts for high volume and critical SLA breaches (<50%)  
    - Timestamp of report generation in America/Bogota timezone  
  - Connections: Outputs HTML content to Send a message node.  
  - Edge Cases: Large number of technicians might affect rendering; missing data handled with fallback text.

- **Send a message**  
  - Type: Gmail (OAuth2)  
  - Role: Emails the generated HTML report to a predefined recipient.  
  - Configuration:  
    - Recipient: info@example.com (configurable)  
    - Subject: Includes the report month from Date range node  
    - Message body: HTML content from Generate report  
    - Credentials: Gmail OAuth2 account configured in n8n  
  - Connections: Proceeds to End session node.  
  - Edge Cases: Email authentication failures, network issues, invalid recipient addresses.

- **End session**  
  - Type: HTTP Request  
  - Role: Terminates the GLPI API session to ensure security and resource cleanup.  
  - Configuration:  
    - URL: `${Server URL}apirest.php/killSession`  
    - Headers include Session-Token and App-Token  
  - Connections: Ends with No Operation node.  
  - Edge Cases: Session already expired or invalid token.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Final node for clean workflow termination.  
  - Connections: None.  

---

### 2.5 Workflow Variables and Configuration

**Overview:**  
Centralized node containing all configurable parameters for the workflow, including GLPI connection details, entity filtering, SLA limits, and working hours.

**Nodes Involved:**  
- Variables

**Node Details:**  
(Described in 2.2 Variables node)

---

## 3. Summary Table

| Node Name           | Node Type               | Functional Role                                              | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                                               |
|---------------------|-------------------------|--------------------------------------------------------------|----------------------|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger         | Monthly trigger on day 6                                     | -                    | Date range           | The configuration is set to start on the 6th day of each month to ensure accurate SLA measurement...                                        |
| Date range          | Code                    | Calculates previous month date range and label               | Schedule Trigger      | Variables            | The month prior to the execution month is identified, and the range covering all days of that month is determined.                         |
| Variables           | Set                     | Defines GLPI server URL, tokens, entity, working hours, SLA  | Date range            | Get session token    | In this node, the connection parameters to the GLPI server are updated: GLPI server URL, API token, entity name, working hours, SLA limit. |
| Get session token   | HTTP Request             | Authenticates with GLPI API, retrieves session token         | Variables             | Get tickets          |                                                                                                                                           |
| Get tickets         | HTTP Request             | Retrieves tickets from GLPI filtered by date and entity      | Get session token     | Split Out            | The node retrieves GLPI cases based on the defined entity, applying the date range configured in Data range...                             |
| Split Out           | Split Out                | Splits ticket array into individual ticket items             | Get tickets           | Edit Fields          |                                                                                                                                           |
| Edit Fields         | Set                      | Extracts and renames key ticket fields                        | Split Out             | Metrics              |                                                                                                                                           |
| Metrics             | Code                    | Calculates working hours, SLA compliance, aggregates reports | Edit Fields           | Generate report      | Calculates working hours between ticket opening and resolution, checks SLA, and generates general and technician reports.                  |
| Generate report     | HTML                    | Builds styled HTML report with conditional formatting         | Metrics               | Send a message       |                                                                                                                                           |
| Send a message      | Gmail                   | Sends HTML report email to designated recipient              | Generate report       | End session          | Add your email account                                                                                                                     |
| End session         | HTTP Request             | Closes GLPI API session                                      | Send a message        | No Operation         |                                                                                                                                           |
| No Operation, do nothing | NoOp               | Final node for workflow termination                           | End session           | -                    |                                                                                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to monthly, trigger on day 6.  

2. **Add Code node "Date range":**  
   - Input: from Schedule Trigger  
   - JavaScript (see code in 2.1) to compute previous month start and end dates and formatted month label.  
   - Output fields: `month`, `startDate`, `endDate`.  

3. **Add Set node "Variables":**  
   - Input: from Date range  
   - Assign variables:  
     - Server URL (string, e.g., "https://server_glpi.com/")  
     - App Token (string, your GLPI API app token)  
     - Entity name (string, e.g., "name_entity")  
     - WORK_START (string, e.g., "8")  
     - LUNCH_START (string, e.g., "12")  
     - LUNCH_END (string, e.g., "13")  
     - WORK_END (string, e.g., "18")  
     - SLA_LIMIT_HOURS (string, e.g., "24")  

4. **Create HTTP Request node "Get session token":**  
   - Input: from Variables  
   - URL: `${Server URL}apirest.php/initSession` (use expression referencing Variables node)  
   - Method: GET  
   - Headers:  
     - Content-Type: application/json  
     - App-Token: from Variables  
   - Authentication: Generic Credential Type → Basic Auth  
   - Credential: Create or select "Api GLPI" credential with your GLPI username/password.  

5. **Create HTTP Request node "Get tickets":**  
   - Input: from Get session token  
   - URL: `${Server URL}apirest.php/search/Ticket`  
   - Method: GET  
   - Query parameters:  
     - criteria[0][field] = 15 (opening date)  
     - criteria[0][searchtype] = morethan  
     - criteria[0][value] = `{{ $('Date range').item.json.startDate }}`  
     - criteria[1][link] = AND  
     - criteria[1][field] = 15  
     - criteria[1][searchtype] = lessthan  
     - criteria[1][value] = `{{ $('Date range').item.json.endDate }}`  
     - criteria[2][link] = AND  
     - criteria[2][field] = 80 (entity)  
     - criteria[2][searchtype] = contains  
     - criteria[2][value] = `{{ $('Variables').item.json["Entity name"] }}`  
     - order = DESC  
     - range = 0-999  
   - Headers:  
     - Content-Type: application/json  
     - Session-Token: from Get session token (JSON path: `session_token`)  
     - App-Token: from Variables  
   - Authentication: Generic Credential Type → Basic Auth (same "Api GLPI" credential)  

6. **Add Split Out node "Split Out":**  
   - Input: from Get tickets  
   - Field to split out: "data" (the array of tickets)  

7. **Add Set node "Edit Fields":**  
   - Input: from Split Out  
   - Map fields:  
     - Ticket_id = `{{$json["2"]}}`  
     - Title = `{{$json["1"]}}`  
     - technical_id = `{{$json["5"]}}`  
     - Entity = `{{$json["80"]}}`  
     - Opening date = `{{$json["15"]}}`  
     - Solution date = `{{$json["17"]}}`  
     - Closing date = `{{$json["16"]}}`  
     - Status = `{{$json["12"]}}`  

8. **Add Code node "Metrics":**  
   - Input: from Edit Fields  
   - Paste JavaScript code (see 2.3) to calculate working hours, SLA compliance, and aggregate reports.  
   - Use Variables node to fetch working hours and SLA limit parameters.  

9. **Add HTML node "Generate report":**  
   - Input: from Metrics  
   - Paste the full HTML template provided (see 2.4) with embedded expressions referencing Metrics output.  
   - Purpose: Generate a visually formatted report with metrics and alerts.  

10. **Add Gmail node "Send a message":**  
    - Input: from Generate report  
    - Configure recipient email, e.g., info@example.com  
    - Subject: `Report - {{ $('Date range').first().json.month }}`  
    - Message: use HTML content from Generate report node  
    - Credentials: Configure Gmail OAuth2 credentials for sending email.  

11. **Add HTTP Request node "End session":**  
    - Input: from Send a message  
    - URL: `${Server URL}apirest.php/killSession`  
    - Method: GET  
    - Headers:  
      - Session-Token: from Get session token  
      - App-Token: from Variables  
    - No authentication needed beyond headers.  

12. **Add No Operation node "No Operation, do nothing":**  
    - Input: from End session  
    - Purpose: Final node to terminate workflow cleanly.  

---

## 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Set GLPI API Credentials in n8n under Credential Name "Api GLPI" using Basic Auth with GLPI username and password.                       | Credential setup for HTTP Request nodes "Get session token" and "Get tickets".                   |
| Configure GLPI Tickets panel to include fields: ID, Title, Status, Opening Date, Closing Date, Resolution Date, Priority, Requester, Technician. | Ensures data fields used in the workflow are available in GLPI API responses.                    |
| Schedule trigger is set to the 6th day of each month to avoid SLA distortion from tickets opened late in previous month.                  | Sticky note explaining schedule rationale for SLA accuracy.                                     |
| Email node requires Gmail OAuth2 credentials configured in n8n with appropriate permissions to send emails.                               | Gmail node "Send a message" configuration note.                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---