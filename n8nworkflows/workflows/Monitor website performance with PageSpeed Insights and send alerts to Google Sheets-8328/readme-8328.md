Monitor website performance with PageSpeed Insights and send alerts to Google Sheets

https://n8nworkflows.xyz/workflows/monitor-website-performance-with-pagespeed-insights-and-send-alerts-to-google-sheets-8328


# Monitor website performance with PageSpeed Insights and send alerts to Google Sheets

### 1. Workflow Overview

This workflow automates monitoring website performance using Google's PageSpeed Insights API, storing results in Google Sheets, and sending alerts when performance metrics fall below defined thresholds. It supports two primary use cases:

- **Scheduled audits**: Periodically test multiple websites listed in a Google Sheet, track historical performance, and alert on issues.
- **On-demand testing**: Accept immediate single-URL test requests via webhook for instant analysis.

The workflow is organized into logical blocks:

- **1.1 Scheduled Audit Block**: Reads website URLs from a Google Sheet, filters to avoid duplicate daily testing, runs PageSpeed tests, processes results, saves them, updates audit dates, and triggers alerts if thresholds are breached.
- **1.2 On-Demand Testing Block**: Listens for webhook POST requests with URL and parameters, runs a PageSpeed test immediately, processes the result, and returns the audit response.
- **1.3 Alerting Block**: Evaluates performance against thresholds, sends email alerts for under-threshold scores, or skips alerting if performance is acceptable.
- **1.4 Supporting Documentation and Notes**: Sticky notes provide detailed explanations, setup instructions, and usage guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Audit Block

- **Overview:**  
  Automates periodic website performance checks by loading site details from Google Sheets, filtering unprocessed sites for the current day, requesting audits via PageSpeed Insights API, processing the results, saving them back to Google Sheets, updating audit dates, and triggering alerts based on thresholds.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Load Sites from Google Sheet  
  - Filter Unprocessed Sites  
  - Run PageSpeed Test  
  - Process Audit Results  
  - Save Audit Results  
  - Update Audit Date  
  - Check Alert Threshold  
  - Send a message  
  - Skip Alert (Performance OK)  

- **Node Details:**

  1. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Initiates scheduled execution (weekly interval)  
     - Configuration: Runs once every week (interval: weeks)  
     - Inputs: None  
     - Outputs: Triggers "Load Sites from Google Sheet"  
     - Edge cases: Missed triggers if n8n is down; consider retry settings  

  2. **Load Sites from Google Sheet**  
     - Type: Google Sheets node  
     - Role: Reads website info (URL, Site_Name, Category, Alert_Threshold, Last_Processed_Date, Device) from the 'sites' sheet  
     - Configuration: Uses OAuth2 credentials; reads specific sheet and document IDs  
     - Inputs: Trigger from Schedule Trigger  
     - Outputs: Passes site data items to "Filter Unprocessed Sites"  
     - Edge cases: API auth failure, quota limits, empty sheet handling  

  3. **Filter Unprocessed Sites**  
     - Type: Code node  
     - Role: Filters out sites already processed today to avoid duplicate audits  
     - Configuration: Uses JavaScript to compare Last_Processed_Date with current date in YYYY-MM-DD format  
     - Inputs: Site data array from Google Sheets  
     - Outputs: Filtered array of sites not audited today to "Run PageSpeed Test"  
     - Edge cases: Date format mismatches, empty Last_Processed_Date fields handled as unprocessed  

  4. **Run PageSpeed Test**  
     - Type: HTTP Request  
     - Role: Calls Google PageSpeed Insights API using site URL, API key, device type, and category  
     - Configuration: Query params include url, key (replace YOUR-API-KEY), strategy (device), category (from site data)  
     - Inputs: Site info filtered for processing  
     - Outputs: API response JSON to "Process Audit Results"  
     - Edge cases: API rate limits, invalid URLs, network errors, invalid API key  

  5. **Process Audit Results**  
     - Type: Code node  
     - Role: Parses API response to extract performance score, Core Web Vitals (LCP, FID, CLS), recommendations, and alert logic  
     - Configuration: Implements improved alert severity levels (critical, high, medium, low, info) based on performance score and threshold  
     - Inputs: API response and site data  
     - Outputs: Structured audit JSON to "Save Audit Results"  
     - Edge cases: Missing audit data, null scores, parsing errors  

  6. **Save Audit Results**  
     - Type: Google Sheets node  
     - Role: Appends audit results to 'audit_results' sheet in Google Sheets  
     - Configuration: Maps performance metrics and metadata into predefined columns; uses OAuth2 credentials  
     - Inputs: Processed audit data  
     - Outputs: To "Update Audit Date"  
     - Edge cases: API quota, write permissions errors  

  7. **Update Audit Date**  
     - Type: Google Sheets node  
     - Role: Updates the Last_Processed_Date in 'sites' sheet for the audited URL to today's date  
     - Configuration: Uses filters to find row by URL, updates Last_Processed_Date to current date  
     - Inputs: Audit results saved confirmation  
     - Outputs: To "Check Alert Threshold"  
     - Edge cases: Concurrency if multiple runs update same rows, permissions issues  

  8. **Check Alert Threshold**  
     - Type: If node  
     - Role: Checks if the performance score indicates an alert (boolean should_alert)  
     - Configuration: Condition: should_alert == true  
     - Inputs: Updated audit data including alert flags  
     - Outputs: If true: to "Send a message"; else: to "Skip Alert (Performance OK)"  
     - Edge cases: Expression evaluation errors  

  9. **Send a message**  
     - Type: Gmail node  
     - Role: Sends email alert with audit summary if performance is below threshold  
     - Configuration: Uses Gmail OAuth2 credentials; sends to designated email; subject and message dynamically generated from audit results  
     - Inputs: Alert condition met  
     - Outputs: None  
     - Edge cases: Email delivery failures, auth errors  

  10. **Skip Alert (Performance OK)**  
      - Type: NoOp node  
      - Role: Terminates alert branch when no alert is needed  
      - Inputs: Alert condition false  
      - Outputs: None  

---

#### 2.2 On-Demand Testing Block

- **Overview:**  
  Provides an HTTP webhook endpoint for immediate performance testing of a single URL, processing the results and returning a JSON response.

- **Nodes Involved:**  
  - Webhook  
  - Format Webhook Input  
  - Run On-Demand Test  
  - Process On-Demand Results  
  - Return Audit Response  

- **Node Details:**

  1. **Webhook**  
     - Type: Webhook (POST)  
     - Role: Receives JSON payload with url, site_name, alert_threshold, device  
     - Configuration: Path and method configured; responseMode set to respond via node  
     - Inputs: External HTTP POST request  
     - Outputs: To "Format Webhook Input"  
     - Edge cases: Invalid payload, missing fields, HTTP errors  

  2. **Format Webhook Input**  
     - Type: Set node  
     - Role: Normalizes and assigns input data fields with defaults (e.g., site_name defaults to "On-Demand Test", alert_threshold defaults to 75, device defaults to "mobile")  
     - Inputs: Webhook data  
     - Outputs: To "Run On-Demand Test"  
     - Edge cases: Missing or malformed fields  

  3. **Run On-Demand Test**  
     - Type: HTTP Request  
     - Role: Calls PageSpeed Insights API similarly to scheduled audit but with webhook input parameters  
     - Configuration: URL, API key, strategy (device), category fixed to performance  
     - Inputs: Formatted webhook input  
     - Outputs: API response to "Process On-Demand Results"  
     - Edge cases: API limits, invalid URL, network errors  

  4. **Process On-Demand Results**  
     - Type: Code node  
     - Role: Parses API response extracting key metrics and recommendations, formats output JSON  
     - Configuration: Similar extraction logic as scheduled audit but tailored for on-demand context  
     - Inputs: API response and formatted webhook data  
     - Outputs: To "Return Audit Response"  
     - Edge cases: Missing data, null values, parsing errors  

  5. **Return Audit Response**  
     - Type: Respond to Webhook  
     - Role: Sends back JSON audit result to webhook caller  
     - Inputs: Processed audit data  
     - Outputs: HTTP response  
     - Edge cases: Network issues, response timeouts  

---

#### 2.3 Alerting Block

- **Overview:**  
  Separates alert decision logic and handles notification delivery or graceful skipping based on performance results.

- **Nodes Involved:**  
  - Check Alert Threshold  
  - Send a message  
  - Skip Alert (Performance OK)  

- **Node Details:**

  - Already detailed in Scheduled Audit Block, nodes included here for clarity.

---

#### 2.4 Supporting Documentation and Notes

- **Overview:**  
  Provides user guidance, setup instructions, and workflow explanations via sticky notes.

- **Nodes Involved:**  
  - Main Template Explanation (large sticky note)  
  - Scheduled Audit Note  
  - Scheduled Audit Note1  
  - Scheduled Audit Note2  
  - On-Demand Testing Note  

- **Node Details:**

  1. **Main Template Explanation**  
     - Type: Sticky Note  
     - Content: Comprehensive project description, use cases, setup steps, Google Sheets structure, API usage, customization tips  
     - Position: Left side, large and prominent  
     - Use: Onboarding and reference  

  2. **Scheduled Audit Note**  
     - Type: Sticky Note  
     - Content: Explains scheduled audit workflow and interval configuration  
     - Position: Near Schedule Trigger  

  3. **Scheduled Audit Note1**  
     - Type: Sticky Note  
     - Content: Suggests options for alerting methods (email, Slack)  
     - Position: Near alert nodes  

  4. **Scheduled Audit Note2**  
     - Type: Sticky Note  
     - Content: Explains the filtering logic for unprocessed sites, date format, and usage rationale  
     - Position: Near Filter Unprocessed Sites  

  5. **On-Demand Testing Note**  
     - Type: Sticky Note  
     - Content: Notes that the webhook endpoint is for immediate single URL testing  
     - Position: Near webhook nodes  

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                  |
|--------------------------|---------------------|----------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Main Template Explanation | Sticky Note         | Documentation and setup guidance       | None                             | None                            | # Monitor website performance with PageSpeed Insights and save to Google Sheets with alerts... (full content)|
| Scheduled Audit Note      | Sticky Note         | Explains scheduled audit workflow      | None                             | None                            | ## Scheduled Audit Workflow - Automatically tests all websites from Google Sheet on a defined schedule.      |
| Scheduled Audit Note1     | Sticky Note         | Alert method options                    | None                             | None                            | ## Choose what works for you - Send an email or a Slack notification.                                        |
| Scheduled Audit Note2     | Sticky Note         | Explains filter unprocessed sites logic| None                             | None                            | ### Filter Unprocessed Sites - Prevents duplicate processing on same day, explains date format.               |
| On-Demand Testing Note    | Sticky Note         | Explains webhook purpose                | None                             | None                            | ## On-Demand Testing - Webhook endpoint for immediate single-URL performance testing                         |
| Schedule Trigger          | Schedule Trigger    | Initiates scheduled audit flow          | None                             | Load Sites from Google Sheet    |                                                                                                              |
| Load Sites from Google Sheet | Google Sheets    | Reads site list for auditing            | Schedule Trigger                 | Filter Unprocessed Sites         |                                                                                                              |
| Filter Unprocessed Sites  | Code                | Filters sites not audited today         | Load Sites from Google Sheet     | Run PageSpeed Test               |                                                                                                              |
| Run PageSpeed Test        | HTTP Request        | Calls PageSpeed Insights API            | Filter Unprocessed Sites          | Process Audit Results            |                                                                                                              |
| Process Audit Results     | Code                | Parses API response, computes metrics   | Run PageSpeed Test                | Save Audit Results              |                                                                                                              |
| Save Audit Results        | Google Sheets       | Saves audit data to Google Sheets       | Process Audit Results             | Update Audit Date               |                                                                                                              |
| Update Audit Date         | Google Sheets       | Updates last processed date              | Save Audit Results                | Check Alert Threshold           |                                                                                                              |
| Check Alert Threshold     | If                  | Decides if alert is needed               | Update Audit Date                 | Send a message / Skip Alert     |                                                                                                              |
| Send a message            | Gmail               | Sends email alert                        | Check Alert Threshold            | None                           |                                                                                                              |
| Skip Alert (Performance OK) | NoOp              | Ends flow without alert                  | Check Alert Threshold            | None                           |                                                                                                              |
| Webhook                  | Webhook             | Receives on-demand test requests         | None                             | Format Webhook Input            |                                                                                                              |
| Format Webhook Input      | Set                 | Normalizes webhook input data             | Webhook                         | Run On-Demand Test              |                                                                                                              |
| Run On-Demand Test        | HTTP Request        | Calls PageSpeed API on-demand             | Format Webhook Input             | Process On-Demand Results       |                                                                                                              |
| Process On-Demand Results | Code                | Parses on-demand API response             | Run On-Demand Test               | Return Audit Response           |                                                                                                              |
| Return Audit Response     | Respond to Webhook  | Sends audit result to webhook client      | Process On-Demand Results        | None                           |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to desired frequency (e.g., weekly) to run scheduled audits.  
   - Connect output to next node.

2. **Create a Google Sheets node ("Load Sites from Google Sheet")**  
   - Operation: Read rows from a sheet named "sites" in your Google Sheets document.  
   - Configure OAuth2 credentials for Google Sheets access.  
   - Map columns: URL, Site_Name, Category, Alert_Threshold, Last_Processed_Date, Device.  
   - Connect Schedule Trigger output to this node.

3. **Create a Code node ("Filter Unprocessed Sites")**  
   - JavaScript code: Filter sites where Last_Processed_Date is missing or not equal to today's date (format YYYY-MM-DD).  
   - Connect output of Google Sheets node to this node.

4. **Create an HTTP Request node ("Run PageSpeed Test")**  
   - Method: GET  
   - URL: https://www.googleapis.com/pagespeedonline/v5/runPagespeed  
   - Query parameters:  
     - url = {{$json.URL}}  
     - key = YOUR-API-KEY (replace with valid Google API key)  
     - strategy = {{$json.Device}}  
     - category = {{$json.Category}}  
   - Connect output from Filter Unprocessed Sites to this node.

5. **Create a Code node ("Process Audit Results")**  
   - Use JavaScript to:  
     - Extract from response: performance score, LCP, FID, CLS, recommendations.  
     - Evaluate alert severity based on thresholds.  
     - Format output JSON with all needed audit info.  
   - Inputs: API response from Run PageSpeed Test and site data.  
   - Connect output from HTTP Request node to this node.

6. **Create a Google Sheets node ("Save Audit Results")**  
   - Operation: Append data to "audit_results" sheet.  
   - Map columns for Date, URL, Site_Name, Device, Performance_Score, LCP, FID, CLS, Recommendations, Full_Report_URL.  
   - Connect output from Process Audit Results to this node.  
   - Use same Google Sheets OAuth2 credentials.

7. **Create a Google Sheets node ("Update Audit Date")**  
   - Operation: Update row in "sites" sheet.  
   - Filters: Match by URL ({{$json.URL}}).  
   - Update "Last_Processed_Date" column with today's date (new Date().toISOString().split('T')[0]).  
   - Connect output from Save Audit Results to this node.

8. **Create an If node ("Check Alert Threshold")**  
   - Condition: Boolean expression checking if {{$json.should_alert}} is true.  
   - Connect output from Update Audit Date to this node.

9. **Create a Gmail node ("Send a message")**  
   - Configure Gmail OAuth2 credentials.  
   - To: email address of alert recipient.  
   - Subject: "Audit results from {{$json.date}}"  
   - Message: {{$json.alert_message}}  
   - Connect "true" output of If node to this node.

10. **Create a NoOp node ("Skip Alert (Performance OK)")**  
    - Connect "false" output of If node here to end flow cleanly.

---

**On-Demand Testing Setup**

11. **Create a Webhook node**  
    - HTTP Method: POST  
    - Path: e.g., "pagespeed-on-demand"  
    - Response Mode: Use Respond to Webhook node  
    - Connect output to next node.

12. **Create a Set node ("Format Webhook Input")**  
    - Assign variables:  
      - URL = {{$json.body.url}}  
      - Site_Name = {{$json.body.site_name || "On-Demand Test"}}  
      - Alert_Threshold = {{$json.body.alert_threshold || 75}}  
      - device = {{$json.body.device || "mobile"}}  
    - Connect output from Webhook node.

13. **Create an HTTP Request node ("Run On-Demand Test")**  
    - Similar to scheduled test, call PageSpeed Insights API with formatted input.  
    - Connect output from Set node.

14. **Create a Code node ("Process On-Demand Results")**  
    - Parse API response and format output JSON like in scheduled audit.  
    - Connect output from HTTP Request node.

15. **Create a Respond to Webhook node ("Return Audit Response")**  
    - Sends JSON response back to webhook caller.  
    - Connect output from Code node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template requires a valid Google PageSpeed Insights API key obtainable for free from Google Cloud Console.                                                                                                                  | Setup instruction                                                                                   |
| Google Sheets should include two sheets: "sites" (input URLs and settings) and "audit_results" (to store audit history).                                                                                                         | Google Sheets structure                                                                             |
| Alerting is implemented via Gmail but can be extended to Slack or other channels by replacing the "Send a message" node.                                                                                                        | Alert customization                                                                                |
| The filtering logic prevents duplicate tests on the same day by comparing dates in YYYY-MM-DD format, matching Google Sheets date format.                                                                                       | Filter Unprocessed Sites explanation                                                               |
| API rate limits and quota management should be considered when scheduling audits or running on-demand tests to avoid failures.                                                                                                  | Operational consideration                                                                           |
| The workflow includes detailed sticky notes explaining setup, use cases, and customization tips to aid users and maintainers.                                                                                                  | Main Template Explanation and supporting notes                                                    |
| Replace all placeholder API keys and email addresses with your actual credentials before running the workflow.                                                                                                                  | Credential setup                                                                                   |
| Use the Google Sheets OAuth2 credentials with necessary scopes granted for reading and writing to the relevant sheets.                                                                                                          | Credential requirements                                                                             |
| On-demand testing supports immediate URL audits via HTTP POST with JSON payload: { "url": "...", "site_name": "...", "alert_threshold": 75, "device": "mobile" }                                                                   | Webhook input format                                                                                |
| Recommendations extracted include unused CSS, render-blocking resources, minification, and image optimization suggestions to help improve site performance.                                                                      | Audit result details                                                                               |

---

This document fully describes the workflow structure, logic, node configurations, and setup instructions to enable both human operators and automation agents to understand, reproduce, and maintain the website performance monitoring automation using n8n.