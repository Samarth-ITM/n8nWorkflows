Monitor Website Uptime with Google Sheets, Slack, Email & Phone Call Alerts

https://n8nworkflows.xyz/workflows/monitor-website-uptime-with-google-sheets--slack--email---phone-call-alerts-11655


# Monitor Website Uptime with Google Sheets, Slack, Email & Phone Call Alerts

### 1. Workflow Overview

This n8n workflow monitors website uptime for a list of URLs stored in a Google Sheet. It follows a scheduled trigger to periodically check each website’s availability by sending HTTP requests. Depending on the response, it logs the website’s status ("up" or "down") with timestamps into a separate Google Sheet. If any website is detected as down, the workflow sends multi-channel alerts via Slack message, Gmail email, and an automated phone call using Vapi.ai API.

The workflow is logically divided into these main blocks:

- **1.1 Schedule Trigger & Data Retrieval:** Starts periodically and loads the list of websites from a Google Sheet.
- **1.2 URL Processing Loop:** Iterates over each website URL to check status.
- **1.3 Website Status Check:** Performs HTTP requests to determine if each website is up.
- **1.4 Status Logging:** Logs the current status and timestamp to a Google Sheet.
- **1.5 Alerting on Downtime:** If a website is down, sends alerts via Slack, Gmail, and initiates a phone call.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Data Retrieval

**Overview:**  
This block initiates the workflow on a time-based schedule and retrieves the list of website URLs from a configured Google Sheet.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow periodically every minute.  
  - Configuration: Interval set to every 1 minute (field: minutes).  
  - Input: None (start node).  
  - Output: Connects to "Get row(s) in sheet".  
  - Edge cases: Scheduling misconfiguration could cause workflow not to trigger.  

- **Get row(s) in sheet**  
  - Type: Google Sheets (read rows)  
  - Role: Reads all rows from a Google Sheet containing the list of website URLs.  
  - Configuration: Document ID and Sheet name selected (Sheet1). Uses Google Sheets OAuth2 credentials.  
  - Output: Passes retrieved rows (websites) to "Loop Over Items".  
  - Edge cases:  
    - Google Sheets API rate limits or credential failures.  
    - Empty or malformed data in sheet rows.  

---

#### 1.2 URL Processing Loop

**Overview:**  
Loops over each website URL retrieved from the sheet to process them individually.

**Nodes Involved:**  
- Loop Over Items

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each website URL one by one, enabling individual HTTP checks per website.  
  - Configuration: Default batch size (process one item at a time).  
  - Input: Receives array of website URLs from "Get row(s) in sheet".  
  - Output: For each item, passes data to "htttps request".  
  - Edge cases:  
    - Large number of URLs could slow workflow.  
    - If no URLs retrieved, no iterations occur.  

---

#### 1.3 Website Status Check

**Overview:**  
Executes an HTTP GET request to each website URL and evaluates if the site is up based on HTTP status code 200.

**Nodes Involved:**  
- htttps request  
- If website up?  
- Edit Fields  
- Edit Fields1

**Node Details:**

- **htttps request**  
  - Type: HTTP Request  
  - Role: Checks availability of the website by sending an HTTP GET request.  
  - Configuration:  
    - URL set dynamically to the website’s URL from the current item.  
    - Timeout set to 5000 ms (5 seconds).  
    - Configured to get the full HTTP response including status code.  
  - Input: Receives website URL from the loop.  
  - Output: Passes HTTP response to "If website up?".  
  - Edge cases:  
    - Timeout or network errors.  
    - DNS failures or invalid URLs.  
    - Non-200 status codes indicating downtime.  

- **If website up?**  
  - Type: If  
  - Role: Evaluates if HTTP status code equals 200 (OK).  
  - Configuration: Checks: `{{$json.statusCode}} === 200`  
  - Input: HTTP response from "htttps request".  
  - Output:  
    - True path: Website is up → goes to "Edit Fields" (sets status "up").  
    - False path: Website is down → goes to alerting nodes and "Edit Fields1" (sets status "down").  
  - Edge cases:  
    - Missing or malformed statusCode in response.  

- **Edit Fields** (status = "up")  
  - Type: Set  
  - Role: Adds a field "status" with value "up" to the item.  
  - Input: True branch from "If website up?".  
  - Output: Passes data to "Append row in sheet" to log uptime.  

- **Edit Fields1** (status = "down")  
  - Type: Set  
  - Role: Adds a field "status" with value "down" to the item.  
  - Input: False branch from "If website up?".  
  - Output: Passes data to append log and alert nodes.  

---

#### 1.4 Status Logging

**Overview:**  
Logs each website’s status and the current timestamp into a Google Sheet named "uptime log".

**Nodes Involved:**  
- Append row in sheet  
- Append row in sheet1

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets (append row)  
  - Role: Appends a row logging "up" status to the uptime log sheet.  
  - Configuration:  
    - Document ID and sheet ID set to the uptime log Google Sheet.  
    - Columns: date (current timestamp), status ("up"), website URL (from original sheet).  
    - Uses Google Sheets OAuth2 credentials.  
  - Input: From "Edit Fields" node.  
  - Output: Loops back to "Loop Over Items" (for batch continuation).  
  - Edge cases:  
    - Credential or API errors.  
    - Data type mismatches or missing fields.  

- **Append row in sheet1**  
  - Type: Google Sheets (append row)  
  - Role: Appends a row logging "down" status to the uptime log sheet.  
  - Configuration: Similar to above but logs "down" status.  
  - Input: From "Edit Fields1" node.  
  - Output: Loops back to "Loop Over Items".  
  - Edge cases: Same as above.  

---

#### 1.5 Alerting on Downtime

**Overview:**  
When a website is detected as down, sends alerts via Slack message, Gmail email, and initiates an automated phone call.

**Nodes Involved:**  
- Send a message (Slack)  
- Send a message1 (Gmail)  
- HTTP Request1 (Vapi.ai call)

**Node Details:**

- **Send a message (Slack)**  
  - Type: Slack  
  - Role: Sends a Slack message alert to a specified user notifying that the website is down.  
  - Configuration:  
    - Message text: Static text indicating the website is down.  
    - User selected by Slack User ID (configured with Slack credentials).  
  - Input: False branch of "If website up?" (website down path).  
  - Output: None (end node).  
  - Edge cases:  
    - Slack API authentication failure.  
    - Invalid user ID or revoked permissions.  

- **Send a message1 (Gmail)**  
  - Type: Gmail  
  - Role: Sends an email alert about the website downtime.  
  - Configuration:  
    - Recipient email configured statically.  
    - Subject and message body indicate downtime.  
    - Gmail OAuth2 credentials used.  
  - Input: False branch of "If website up?".  
  - Output: None (end node).  
  - Edge cases:  
    - Gmail OAuth token expiration or invalid credentials.  
    - Email quota limits.  

- **HTTP Request1 (Vapi.ai call)**  
  - Type: HTTP Request  
  - Role: Initiates an automated phone call via Vapi.ai API to notify about downtime.  
  - Configuration:  
    - POST method to https://api.vapi.ai/call  
    - JSON body contains assistantId, phoneNumberId, and customer phone number (hardcoded +919537805470).  
    - Authorization header requires Bearer token (placeholder "YOUR_TOKEN_HERE").  
  - Input: False branch of "If website up?".  
  - Output: None (end node).  
  - Edge cases:  
    - Missing or invalid API token.  
    - Network or service downtime at Vapi.ai.  
    - Hardcoded phone number limits flexibility.  

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                      | Input Node(s)             | Output Node(s)                         | Sticky Note                                                                                              |
|--------------------|--------------------|------------------------------------|---------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger   | Starts workflow on schedule        | -                         | Get row(s) in sheet                   |                                                                                                        |
| Get row(s) in sheet | Google Sheets      | Retrieves website URLs list         | Schedule Trigger           | Loop Over Items                       | ## 1. get data from google sheet                                                                        |
| Loop Over Items     | SplitInBatches     | Iterates over each website URL      | Get row(s) in sheet        | htttps request                       | ## 2. loop over urls to check status                                                                    |
| htttps request      | HTTP Request       | Checks website availability via HTTP GET | Loop Over Items            | If website up?                       |                                                                                                        |
| If website up?      | If                 | Checks if HTTP status code is 200   | htttps request             | Edit Fields (true), Send a message, HTTP Request1, Send a message1, Edit Fields1 (false) |                                                                                                        |
| Edit Fields        | Set                | Sets status="up"                    | If website up? (true)      | Append row in sheet                   | ## 3. log the status to ggogle sheet if up                                                              |
| Append row in sheet | Google Sheets      | Logs "up" status to uptime log     | Edit Fields                | Loop Over Items                      |                                                                                                        |
| Send a message     | Slack               | Sends Slack alert for downtime      | If website up? (false)     | -                                   | ## 4. inform user via slack, mail and call, and log the status to sheets                                |
| HTTP Request1      | HTTP Request       | Initiates automated phone call      | If website up? (false)     | -                                   |                                                                                                        |
| Send a message1    | Gmail              | Sends email alert about downtime    | If website up? (false)     | -                                   |                                                                                                        |
| Edit Fields1       | Set                | Sets status="down"                  | If website up? (false)     | Append row in sheet1                  |                                                                                                        |
| Append row in sheet1| Google Sheets      | Logs "down" status to uptime log   | Edit Fields1               | Loop Over Items                      |                                                                                                        |
| Sticky Note        | Sticky Note        | Explains overall workflow purpose   | -                         | -                                   | ## Website Uptime Monitor with Multi-Channel Alerts ... (full note on workflow description and setup)  |
| Sticky Note1       | Sticky Note        | Marks block 1: data retrieval        | -                         | -                                   | ## 1. get data from google sheet                                                                        |
| Sticky Note2       | Sticky Note        | Marks block 2: looping URLs          | -                         | -                                   | ## 2. loop over urls to check status                                                                    |
| Sticky Note3       | Sticky Note        | Marks block 3: logging status if up | -                         | -                                   | ## 3. log the status to ggogle sheet if up                                                              |
| Sticky Note4       | Sticky Note        | Marks block 4: alert and logging down | -                         | -                                   | ## 4. inform user via slack, mail and call, and log the status to sheets                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 1 minute.  

2. **Create Google Sheets Node to Get Rows**  
   - Type: Google Sheets (Read rows)  
   - Connect credentials for Google Sheets OAuth2.  
   - Set Document ID to your sheet containing website URLs.  
   - Set Sheet Name to the correct tab (e.g., "Sheet1").  
   - Connect output of Schedule Trigger to this node.  

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Connect input from "Get row(s) in sheet".  
   - Default batch size (process one URL at a time).  

4. **Create HTTP Request Node to Check Website**  
   - Type: HTTP Request  
   - Set URL dynamically from current item’s website URL field (e.g., `{{$json.url}}`).  
   - Method: GET  
   - Set timeout to 5000 ms.  
   - Enable full response output including status code.  
   - Connect input from "Loop Over Items".  

5. **Create If Node to Check HTTP Status**  
   - Type: If  
   - Condition: `{{$json.statusCode}}` equals `200`.  
   - Connect input from HTTP Request node.  

6. **Create Set Node for Status 'up'**  
   - Type: Set  
   - Add field "status" with value "up".  
   - Connect input from If node’s true output.  

7. **Create Google Sheets Node to Append Row (for 'up')**  
   - Type: Google Sheets (Append row)  
   - Connect credentials for Google Sheets OAuth2.  
   - Document: your uptime log sheet ID.  
   - Sheet: correct sheet/tab for logging.  
   - Map columns:  
     - date: current timestamp (`{{$now}}`)  
     - status: from Set node ("up")  
     - website url: from original URL field  
   - Connect input from Set node (status "up").  
   - Connect output back to "Loop Over Items" for batch continuation.  

8. **Create Set Node for Status 'down'**  
   - Type: Set  
   - Add field "status" with value "down".  
   - Connect input from If node’s false output.  

9. **Create Google Sheets Node to Append Row (for 'down')**  
   - Type: Google Sheets (Append row)  
   - Configure same as the "up" append node but for "down" status.  
   - Connect input from Set node (status "down").  
   - Connect output back to "Loop Over Items".  

10. **Create Slack Node to Send Message**  
    - Type: Slack  
    - Connect Slack API credentials.  
    - Set message text to alert about downtime (e.g., "pixcelsthemes.com is down").  
    - Set recipient user or channel.  
    - Connect input from If node’s false output (down).  

11. **Create Gmail Node to Send Email**  
    - Type: Gmail  
    - Connect Gmail OAuth2 credentials.  
    - Set recipient email, subject, and message content for downtime alert.  
    - Connect input from If node’s false output (down).  

12. **Create HTTP Request Node for Phone Call**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://api.vapi.ai/call  
    - Headers: Authorization Bearer token (replace with your actual token).  
    - Body (JSON): includes assistantId, phoneNumberId, and customer phone number.  
    - Connect input from If node’s false output (down).  

13. **Connect all nodes according to the logic described above.**  

14. **Test the workflow with live data and verify Google Sheets logs and alerts.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow monitors website uptime and sends multi-channel alerts (Slack, Gmail, Phone call) on downtime detection.                                | Workflow purpose and alerting mechanism.                                                                                                              |
| Ensure you replace placeholder API tokens and phone numbers with valid credentials and targets for your environment.                            | Important for HTTP Request1 (Vapi.ai API) and Slack/Gmail nodes configuration.                                                                         |
| Google Sheets must have two documents: one for website URLs and one for uptime logs, both accessible with proper OAuth2 credentials.             | Setup requirement for correct data flow.                                                                                                             |
| Slack user ID used is specific; adjust to your Slack workspace and intended recipient.                                                           | Slack node configuration note.                                                                                                                       |
| Gmail OAuth2 credentials must be properly configured with permission to send emails from the connected account.                                   | Gmail node configuration note.                                                                                                                       |
| Vapi.ai API usage requires valid subscription and API key for phone call automation.                                                             | External API integration note.                                                                                                                       |
| Workflow includes sticky notes explaining blocks and setup instructions for user clarity.                                                       | Sticky notes provide useful guidance within the workflow editor interface.                                                                            |

---

_Disclaimer: The provided text is extracted solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public._