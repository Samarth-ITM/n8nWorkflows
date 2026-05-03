Track SEO Keyword Position in Google SERP (Google Sheets + SerpAPI Integration)

https://n8nworkflows.xyz/workflows/track-seo-keyword-position-in-google-serp--google-sheets---serpapi-integration--3792


# Track SEO Keyword Position in Google SERP (Google Sheets + SerpAPI Integration)

### 1. Workflow Overview

This n8n workflow automates the monitoring of SEO keyword rankings on Google by integrating SerpAPI with Google Sheets and notification services (WhatsApp and Email). It periodically checks the position of specified keywords in Google’s organic search results, logs the current rankings in a Google Sheet, compares them with previous rankings, and sends notifications if there is any change in position.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Initialization**: Periodically marks all keywords in the Google Sheet as “notchecked” to prepare for a new ranking check cycle.
- **1.2 Keyword Processing Trigger**: Detects rows marked “notchecked” and triggers the ranking check process for each keyword.
- **1.3 SerpAPI Request & Data Mapping**: Sends requests to SerpAPI to fetch Google SERP data for each keyword and maps the organic results.
- **1.4 Position Extraction**: Parses the SERP data to find the position of the target website URL.
- **1.5 Google Sheets Update**: Updates the Google Sheet with the new position and marks the row as “checked”.
- **1.6 Ranking Comparison & Notification Decision**: Compares previous and current positions to determine if a notification should be sent.
- **1.7 Notification Dispatch**: Sends WhatsApp and/or Email notifications based on ranking improvements or drops.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Initialization

- **Overview:**  
  This block triggers the workflow on a schedule (e.g., daily or weekly) and updates all rows in the Google Sheet to mark them as “notchecked” in the `checkstatus` column, signaling that these keywords need to be processed.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Sheets2 (Update all rows to `notchecked`)  
  - WA Start Checks Notification (WhatsApp notification on start)  
  - GMAIL Start Checks Notification (Email notification on start)

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a user-defined schedule.  
    - Configuration: Default schedule parameters (e.g., daily/weekly).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to Google Sheets2 node.  
    - Edge Cases: Misconfigured schedule may cause no triggers; ensure time zone correctness.

  - **Google Sheets2**  
    - Type: Google Sheets  
    - Role: Updates all rows in the sheet, setting `checkstatus` to “notchecked”.  
    - Configuration: Uses Google Sheets credentials; targets the correct spreadsheet and worksheet; performs batch update.  
    - Inputs: From Schedule Trigger  
    - Outputs: Connects to WA and GMAIL Start Checks Notification nodes.  
    - Edge Cases: API quota limits, authentication errors, or incorrect sheet range may cause failures.

  - **WA Start Checks Notification**  
    - Type: WhatsApp  
    - Role: Sends a WhatsApp message notifying that the keyword check cycle has started.  
    - Configuration: Uses WhatsApp API credentials (e.g., Twilio or Ultramsg).  
    - Inputs: From Google Sheets2  
    - Outputs: None  
    - Edge Cases: API authentication failure, message sending limits.

  - **GMAIL Start Checks Notification**  
    - Type: Gmail  
    - Role: Sends an email notification that the keyword check cycle has started.  
    - Configuration: Uses Gmail OAuth2 credentials.  
    - Inputs: From Google Sheets2  
    - Outputs: None  
    - Edge Cases: OAuth token expiration, Gmail API limits.

---

#### 1.2 Keyword Processing Trigger

- **Overview:**  
  This block detects rows in the Google Sheet where `checkstatus` is “notchecked” and triggers the keyword ranking check process for each such row.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Get Row (notChecked Column)  
  - Set Keyword (s)  
  - When clicking ‘Test workflow’ (manual trigger for testing)

- **Node Details:**  

  - **Google Sheets Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Watches the Google Sheet for changes or triggers manually to detect rows needing processing.  
    - Configuration: Monitors the sheet and worksheet where keywords are stored.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to Get Row (notChecked Column).  
    - Edge Cases: Trigger delay or missed triggers if API limits are hit.

  - **Get Row (notChecked Column)**  
    - Type: Google Sheets  
    - Role: Retrieves rows where `checkstatus` equals “notchecked”.  
    - Configuration: Uses filter or query to select only relevant rows.  
    - Inputs: From Google Sheets Trigger or manual trigger  
    - Outputs: Connects to Set Keyword (s).  
    - Edge Cases: Empty results if no rows are “notchecked”.

  - **Set Keyword (s)**  
    - Type: Code (JavaScript)  
    - Role: Prepares the keyword(s) for the SerpAPI request, possibly formatting or extracting necessary data.  
    - Configuration: Custom JavaScript code to set keyword parameters.  
    - Inputs: From Get Row (notChecked Column)  
    - Outputs: Connects to Google Serp Request.  
    - Edge Cases: Code errors if input data is malformed or missing.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the keyword processing for testing purposes.  
    - Configuration: None  
    - Inputs: None  
    - Outputs: Connects to Get Row (notChecked Column).  
    - Edge Cases: None.

---

#### 1.3 SerpAPI Request & Data Mapping

- **Overview:**  
  This block sends HTTP requests to SerpAPI to fetch Google SERP data for each keyword and maps the organic results array for further processing.

- **Nodes Involved:**  
  - Google Serp Request  
  - Map Organic Results Array

- **Node Details:**  

  - **Google Serp Request**  
    - Type: HTTP Request  
    - Role: Sends a GET request to SerpAPI with the keyword and API key to retrieve search results.  
    - Configuration:  
      - URL: SerpAPI endpoint (e.g., https://serpapi.com/search)  
      - Query parameters include keyword, API key, search engine (Google), and other SerpAPI options.  
      - Credential: Generic credential type with SerpAPI key.  
    - Inputs: From Set Keyword (s)  
    - Outputs: Connects to Map Organic Results Array.  
    - Edge Cases: API key invalid, rate limits, network timeouts, malformed responses.

  - **Map Organic Results Array**  
    - Type: Set  
    - Role: Extracts and maps the organic search results array from the SerpAPI response for easier processing downstream.  
    - Configuration: Uses expressions to select the `organic_results` array from the HTTP response JSON.  
    - Inputs: From Google Serp Request  
    - Outputs: Connects to Web URL Position Finder.  
    - Edge Cases: Missing or empty organic results, unexpected response structure.

---

#### 1.4 Position Extraction

- **Overview:**  
  This block parses the organic search results to find the position of the target website URL within the SERP.

- **Nodes Involved:**  
  - Web URL Position Finder

- **Node Details:**  

  - **Web URL Position Finder**  
    - Type: Code (JavaScript)  
    - Role: Loops through the organic results array, compares each result’s URL to the target website URL, and determines the current ranking position.  
    - Configuration: Custom JavaScript that returns the position index or a default value if not found.  
    - Inputs: From Map Organic Results Array  
    - Outputs: Connects to Update Check Status Column (to Checked).  
    - Edge Cases: Target URL not found in results, malformed URLs, empty organic results.

---

#### 1.5 Google Sheets Update

- **Overview:**  
  This block updates the Google Sheet with the new keyword position and marks the row’s `checkstatus` as “checked” to indicate processing completion.

- **Nodes Involved:**  
  - Update Check Status Column (to Checked)  
  - Google Sheets (final update)

- **Node Details:**  

  - **Update Check Status Column (to Checked)**  
    - Type: Google Sheets  
    - Role: Updates the current row with the new position and sets `checkstatus` to “checked”.  
    - Configuration: Uses Google Sheets credentials, targets the correct spreadsheet and worksheet, updates specific columns (`currentposition`, `checkstatus`).  
    - Inputs: From Web URL Position Finder  
    - Outputs: Connects to Notifications Switch.  
    - Edge Cases: API errors, concurrent updates, incorrect row references.

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Additional update or batch update node (connected downstream from Google Sheets2 in the schedule block).  
    - Configuration: Similar to above, may be used for other sheet updates.  
    - Inputs: From Google Sheets2  
    - Outputs: Connects to WA Start Checks Notification and GMAIL Start Checks Notification.  
    - Edge Cases: Same as above.

---

#### 1.6 Ranking Comparison & Notification Decision

- **Overview:**  
  This block compares the previous keyword position with the current position and decides whether to send notifications based on improvement, drop, or no change.

- **Nodes Involved:**  
  - Notifications Switch

- **Node Details:**  

  - **Notifications Switch**  
    - Type: Switch  
    - Role: Evaluates the difference between `previousposition` and `currentposition` to route the workflow accordingly:  
      - Improved rank → positive alert branch  
      - Dropped rank → negative alert branch  
      - No change → no alert (ends workflow)  
    - Configuration: Uses expressions comparing numeric values of positions.  
    - Inputs: From Update Check Status Column (to Checked)  
    - Outputs:  
      - No change: no output  
      - Drop: connects to Send Email Dropped and Send WA Message Dropped  
      - Improvement: connects to Send Email Improved and Send WA Message Improved  
    - Edge Cases: Missing or invalid position data, equal positions, zero or null values.

---

#### 1.7 Notification Dispatch

- **Overview:**  
  This block sends notifications via WhatsApp and Email based on the ranking movement detected.

- **Nodes Involved:**  
  - Send Email Improved  
  - Send WA Message Improved  
  - Send Email Dropped  
  - Send WA Message Dropped

- **Node Details:**  

  - **Send Email Improved**  
    - Type: Gmail  
    - Role: Sends an email notification indicating the keyword ranking has improved.  
    - Configuration: Uses Gmail OAuth2 credentials; customizable email subject and body with dynamic data.  
    - Inputs: From Notifications Switch (improvement branch)  
    - Outputs: None  
    - Edge Cases: OAuth token expiration, email quota limits.

  - **Send WA Message Improved**  
    - Type: WhatsApp  
    - Role: Sends a WhatsApp message notifying of ranking improvement.  
    - Configuration: Uses WhatsApp API credentials; customizable message content.  
    - Inputs: From Notifications Switch (improvement branch)  
    - Outputs: None  
    - Edge Cases: API limits, authentication errors.

  - **Send Email Dropped**  
    - Type: Gmail  
    - Role: Sends an email notification indicating the keyword ranking has dropped.  
    - Configuration: Similar to Send Email Improved but with different message content.  
    - Inputs: From Notifications Switch (drop branch)  
    - Outputs: None  
    - Edge Cases: Same as above.

  - **Send WA Message Dropped**  
    - Type: WhatsApp  
    - Role: Sends a WhatsApp message notifying of ranking drop.  
    - Configuration: Similar to Send WA Message Improved but with different message content.  
    - Inputs: From Notifications Switch (drop branch)  
    - Outputs: None  
    - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                | Input Node(s)                          | Output Node(s)                                 | Sticky Note                              |
|-----------------------------------|---------------------|-----------------------------------------------|--------------------------------------|-----------------------------------------------|-----------------------------------------|
| Schedule Trigger                  | Schedule Trigger    | Triggers workflow on schedule                   | None                                 | Google Sheets2                                |                                         |
| Google Sheets2                   | Google Sheets       | Marks all rows `checkstatus` as "notchecked"   | Schedule Trigger                     | WA Start Checks Notification, GMAIL Start Checks Notification, Google Sheets |                                         |
| WA Start Checks Notification     | WhatsApp            | Sends WhatsApp notification on check start     | Google Sheets2                      | None                                          |                                         |
| GMAIL Start Checks Notification  | Gmail               | Sends email notification on check start        | Google Sheets2                      | None                                          |                                         |
| Google Sheets                   | Google Sheets       | Additional sheet update                          | Google Sheets2                      | WA Start Checks Notification, GMAIL Start Checks Notification |                                         |
| Google Sheets Trigger           | Google Sheets Trigger| Detects rows with `checkstatus = notchecked`   | None                               | Get Row (notChecked Column)                    |                                         |
| Get Row (notChecked Column)     | Google Sheets       | Retrieves rows marked "notchecked"              | Google Sheets Trigger, Manual Trigger | Set Keyword (s)                               |                                         |
| Set Keyword (s)                 | Code (JavaScript)   | Prepares keyword parameter for SerpAPI request | Get Row (notChecked Column)          | Google Serp Request                            |                                         |
| Google Serp Request             | HTTP Request        | Calls SerpAPI to fetch Google SERP data         | Set Keyword (s)                     | Map Organic Results Array                      |                                         |
| Map Organic Results Array       | Set                 | Extracts organic results array from SerpAPI response | Google Serp Request               | Web URL Position Finder                        |                                         |
| Web URL Position Finder         | Code (JavaScript)   | Finds position of target URL in organic results | Map Organic Results Array           | Update Check Status Column (to Checked)       |                                         |
| Update Check Status Column (to Checked) | Google Sheets | Updates position and marks row as "checked"    | Web URL Position Finder             | Notifications Switch                           |                                         |
| Notifications Switch            | Switch              | Compares previous and current positions         | Update Check Status Column (to Checked) | Send Email Dropped, Send WA Message Dropped, Send Email Improved, Send WA Message Improved |                                         |
| Send Email Improved             | Gmail               | Sends email notification for ranking improvement | Notifications Switch (improved)    | None                                          |                                         |
| Send WA Message Improved        | WhatsApp            | Sends WhatsApp notification for ranking improvement | Notifications Switch (improved)    | None                                          |                                         |
| Send Email Dropped              | Gmail               | Sends email notification for ranking drop       | Notifications Switch (dropped)     | None                                          |                                         |
| Send WA Message Dropped         | WhatsApp            | Sends WhatsApp notification for ranking drop    | Notifications Switch (dropped)     | None                                          |                                         |
| When clicking ‘Test workflow’   | Manual Trigger      | Manual trigger for testing                        | None                               | Get Row (notChecked Column)                    |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure the desired frequency (e.g., daily at 9:00 AM).  
   - No credentials needed.

2. **Add a Google Sheets node (Google Sheets2)**  
   - Type: Google Sheets  
   - Operation: Update multiple rows or batch update.  
   - Configure credentials for Google Sheets API.  
   - Target your spreadsheet and worksheet containing keywords.  
   - Update the `checkstatus` column for all rows to “notchecked”.  
   - Connect Schedule Trigger output to this node.

3. **Add WhatsApp and Gmail nodes for start notifications**  
   - WhatsApp node: Configure with WhatsApp API credentials (e.g., Twilio).  
   - Gmail node: Configure with Gmail OAuth2 credentials.  
   - Customize message content to notify start of checks.  
   - Connect Google Sheets2 output to both notification nodes.

4. **Add a Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Configure to watch the keywords sheet for changes or manual trigger.  
   - No credentials needed beyond Google Sheets API.

5. **Add a Google Sheets node (Get Row (notChecked Column))**  
   - Operation: Read rows with filter where `checkstatus` = “notchecked”.  
   - Connect Google Sheets Trigger output to this node.

6. **Add a Manual Trigger node (When clicking ‘Test workflow’)**  
   - For manual testing, connect output to the Get Row node.

7. **Add a Code node (Set Keyword (s))**  
   - Type: Code (JavaScript)  
   - Write code to extract and prepare the keyword(s) from the input data for the API request.  
   - Connect Get Row output to this node.

8. **Add an HTTP Request node (Google Serp Request)**  
   - Configure to send GET requests to SerpAPI endpoint: `https://serpapi.com/search`.  
   - Set query parameters: `q` (keyword), `api_key` (SerpAPI key), `engine=google`, and others as needed.  
   - Use Generic credential type with your SerpAPI key.  
   - Connect Set Keyword node output to this node.

9. **Add a Set node (Map Organic Results Array)**  
   - Extract the `organic_results` array from the HTTP response JSON using expressions.  
   - Connect Google Serp Request output to this node.

10. **Add a Code node (Web URL Position Finder)**  
    - Write JavaScript to loop through `organic_results` and find the position of the target website URL.  
    - Return the position or a default value if not found.  
    - Connect Map Organic Results Array output to this node.

11. **Add a Google Sheets node (Update Check Status Column (to Checked))**  
    - Operation: Update the current row with new `currentposition` and set `checkstatus` to “checked”.  
    - Connect Web URL Position Finder output to this node.

12. **Add a Switch node (Notifications Switch)**  
    - Configure to compare `previousposition` and `currentposition` from the updated row.  
    - Define three outputs: no change (no output), position dropped, position improved.  
    - Connect Update Check Status Column output to this node.

13. **Add Gmail and WhatsApp nodes for notifications**  
    - For position improved:  
      - Gmail node (Send Email Improved) with customized message.  
      - WhatsApp node (Send WA Message Improved) with customized message.  
    - For position dropped:  
      - Gmail node (Send Email Dropped) with customized message.  
      - WhatsApp node (Send WA Message Dropped) with customized message.  
    - Connect respective outputs of the Switch node to these notification nodes.

14. **Configure all credentials**  
    - Google Sheets: OAuth2 credentials with access to the target spreadsheet.  
    - SerpAPI: Generic credential with your API key.  
    - Gmail: OAuth2 credentials for sending emails.  
    - WhatsApp: API credentials (e.g., Twilio or Ultramsg).

15. **Test the workflow**  
    - Use the manual trigger to test processing a single keyword row.  
    - Verify updates in Google Sheets and receipt of notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Google Sheets template for keyword tracking with columns: keyword, Website_url, position, timestamp, checkstatus | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1mWXr7qonRLOtNYU8cPYr-PXTNLh6vFasu5L_iAjvdQ0/edit?gid=0#gid=0) |
| SerpAPI API key management and documentation                                                    | [https://serpapi.com/manage-api-key](https://serpapi.com/manage-api-key), [SerpAPI Search API](https://serpapi.com/search-api) |
| Use Generic credential type in n8n for SerpAPI API key                                         | n8n credential setup instructions in node Docs tab                                               |
| WhatsApp API options include Twilio or Ultramsg                                                | Setup instructions available in WhatsApp node Docs tab                                          |
| Gmail OAuth2 credentials required for sending emails                                           | Setup instructions available in Gmail node Docs tab                                            |
| Workflow author contact for customization or support                                          | Twitter: [https://x.com/juppfy](https://x.com/juppfy), Email: joseph@uppfy.com                   |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the “Automated Website Keyword SEO Tracker” workflow in n8n, including all nodes, their roles, configurations, and integration points.