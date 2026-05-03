Multiple Websites Monitoring with Notifications including Phone Calls

https://n8nworkflows.xyz/workflows/multiple-websites-monitoring-with-notifications-including-phone-calls-4833


# Multiple Websites Monitoring with Notifications including Phone Calls

### 1. Workflow Overview

This workflow implements a **Multiple Websites Monitoring with Notifications including Phone Calls** system using n8n. Its primary goal is to periodically check the uptime status of a list of websites, log downtime events, and notify stakeholders via email, Slack, Telegram, and automated phone calls when a website goes down or recovers.

The workflow is structured into the following logical blocks:

- **1.1 Scheduling and Input Acquisition:** Periodic trigger to start the workflow and read the list of websites to monitor from a Google Sheet.
- **1.2 Website Status Checking:** Loop through each website URL and perform HTTP requests to verify site availability.
- **1.3 Downtime Record Management:** Query existing downtime logs, detect new downtime or recovery events, update records and calculate total downtime durations.
- **1.4 Notifications Dispatch:** Send notifications through multiple channels (email, Slack, Telegram, phone call) on downtime occurrence and recovery.
- **1.5 Control Flow and Decision Making:** Conditional nodes to handle different scenarios such as site up/down, existing records, and looping logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Input Acquisition

- **Overview:**  
  Initiates the workflow on a scheduled interval and retrieves the list of websites to monitor from a Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Website URLs  
  - Loop Over Items

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically triggers the workflow to start the monitoring process.  
    - Configuration: Default interval (every minute) configured via the interval setting.  
    - Input/Output: No input; outputs to "Website URLs".  
    - Edge Cases: If the n8n instance is down or paused, scheduled runs could be missed.

  - **Website URLs**  
    - Type: Google Sheets  
    - Role: Reads website URLs from a Google Sheet (Sheet1) for monitoring.  
    - Configuration:  
      - Document ID: Google Sheet with website list.  
      - Sheet Name: Sheet1 (contains URLs).  
      - Credentials: Google Sheets OAuth2.  
    - Output: List of website URLs passed to "Loop Over Items".  
    - Edge Cases: API errors, OAuth token expiry, empty sheet.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each website URL to process individually.  
    - Configuration: Default batch size (1 item per batch).  
    - Input: Website URLs list.  
    - Output: Each item sent sequentially to "Check website status".  
    - Edge Cases: Large lists may slow down processing; empty input list.

#### 2.2 Website Status Checking

- **Overview:**  
  Performs HTTP requests to each website URL to check if the site is up or down.

- **Nodes Involved:**  
  - Check website status  
  - If Site Up  
  - Get Active Down Record

- **Node Details:**

  - **Check website status**  
    - Type: HTTP Request  
    - Role: Sends an HTTP GET request to the website URL.  
    - Configuration:  
      - URL: Dynamically set from current item’s "Website URL".  
      - Timeout: 5000 ms (5 seconds).  
      - Response: Full HTTP response.  
    - Input: Single website URL from "Loop Over Items".  
    - Output: HTTP response passed to "If Site Up".  
    - Edge Cases: Timeout, DNS failure, non-200 status codes, network errors.

  - **If Site Up**  
    - Type: If  
    - Role: Determines if site is down based on HTTP status code.  
    - Configuration:  
      - Condition: Checks if statusCode equals 400 (indicating failure).  
    - Input: HTTP response from "Check website status".  
    - Output:  
      - True (statusCode == 400): Site is down, proceeds to "Get Active Down Record".  
      - False: Site is up, proceeds to "Get Active Down Record".  
    - Edge Cases: Other failure codes besides 400 not explicitly handled.

  - **Get Active Down Record**  
    - Type: Google Sheets  
    - Role: Fetches existing downtime records for the current website where "Up Time" is empty (indicating ongoing downtime).  
    - Configuration:  
      - Filters: Match "Website URL" and "Up Time" empty.  
      - Document and Sheet: Same Google Sheet, Sheet2 contains downtime logs.  
      - Credentials: Google Sheets OAuth2.  
    - Input: Website URL and site status from "If Site Up".  
    - Output: Downtime record to "If Down Record Exist".  
    - Edge Cases: API access errors, multiple records returned.

#### 2.3 Downtime Record Management

- **Overview:**  
  Manages downtime records by updating existing entries or adding new ones, and calculates total downtime duration.

- **Nodes Involved:**  
  - If Down Record Exist  
  - Update Uptime and Total Downtime  
  - Add Downtime Record  
  - Find Existing Record  
  - Check Downtime Exists  
  - Get Existing Down Records

- **Node Details:**

  - **If Down Record Exist**  
    - Type: If  
    - Role: Checks if there's an active downtime record.  
    - Condition: True if the record returned is not empty.  
    - Input: Output from "Get Active Down Record".  
    - Output:  
      - True: Proceed to update uptime and total downtime.  
      - False: Loop back to process next website.  
    - Edge Cases: Empty data handling.

  - **Update Uptime and Total Downtime**  
    - Type: Google Sheets (Update)  
    - Role: Updates existing downtime record with current up time and recalculates total downtime.  
    - Configuration:  
      - Fields updated: "Up Time" with current time, "Total downtime" calculated as difference between Down Time and current Up Time.  
      - Uses a JavaScript function in expressions to parse and calculate time difference, considering overnight scenarios.  
      - Matches rows by "row_number".  
    - Input: Active downtime record and current time.  
    - Output: Notifications for site recovery triggered next.  
    - Edge Cases: Time parsing errors, overnight downtime periods.

  - **Add Downtime Record**  
    - Type: Google Sheets (Append)  
    - Role: Adds a new downtime record for the website when a new downtime is detected.  
    - Configuration:  
      - Fields: "Down Time" with current time, "Website URL" from current item.  
      - Matches by "Website URL".  
    - Input: Triggered when downtime is new (after checking no existing active downtime record).  
    - Output: Notifications for downtime triggered next.  
    - Edge Cases: Duplicate records, API failure.

  - **Find Existing Record**  
    - Type: Code (JavaScript)  
    - Role: Checks if there is any existing incomplete downtime record (Down Time exists but Up Time or Total downtime missing).  
    - Logic: Iterates all inputs and sets status "yes" if incomplete record found, otherwise "no".  
    - Input: Items from "Get Existing Down Records".  
    - Output: Status flag to "Check Downtime Exists".  
    - Edge Cases: Empty input, malformed data.

  - **Check Downtime Exists**  
    - Type: If  
    - Role: Evaluates if an incomplete downtime record exists.  
    - Condition: Checks if "status" equals "yes".  
    - Input: Output from "Find Existing Record".  
    - Output:  
      - True: Loop back to "Loop Over Items" (skip adding new record).  
      - False: Proceed to "Add Downtime Record".  
    - Edge Cases: Logic mismatch.

  - **Get Existing Down Records**  
    - Type: Google Sheets  
    - Role: Retrieves existing downtime records for the current website URL (all records, not only active).  
    - Configuration: Filter by "Website URL".  
    - Input: From "If Site Up" node.  
    - Output: Sent to "Find Existing Record".  
    - Edge Cases: API errors.

#### 2.4 Notifications Dispatch

- **Overview:**  
  Sends notifications via email, Slack, Telegram, and phone calls depending on website status changes (down or up).

- **Nodes Involved:**  
  - Email Notification for Down  
  - Slack Notification for Down  
  - Telegram Notification for Down  
  - Notify over Phone Call  
  - Email Notification for Up  
  - Slack Notification for Up  
  - Telegram Notification for Up

- **Node Details:**

  - **Email Notification for Down**  
    - Type: Gmail  
    - Role: Sends an email alert when a website is down.  
    - Configuration:  
      - Recipient: example@gmail.com (placeholder).  
      - Subject: "Your site is down!!".  
      - Message: Includes website URL and prompt to take action.  
      - Credentials: Gmail OAuth2 account.  
    - Input: Triggered after adding a downtime record.  
    - Output: None.  
    - Edge Cases: Email sending failures, quota limits.

  - **Slack Notification for Down**  
    - Type: Slack  
    - Role: Sends Slack message alerting about downtime.  
    - Configuration:  
      - Message text includes website URL and alert emoji.  
      - Recipient user ID configured.  
      - OAuth2 authentication.  
    - Input: Triggered after downtime record creation.  
    - Output: None.  
    - Edge Cases: API limits, invalid user ID.

  - **Telegram Notification for Down**  
    - Type: Telegram  
    - Role: Sends Telegram channel/group message for downtime alert.  
    - Configuration:  
      - Chat ID: Specific Telegram group/channel.  
      - Message text with alert and website URL.  
      - Telegram API credentials.  
    - Input: Triggered upon downtime detection.  
    - Output: None.  
    - Edge Cases: Invalid chat ID, API errors.

  - **Notify over Phone Call**  
    - Type: HTTP Request  
    - Role: Initiates a phone call via vapi.ai API to notify about website downtime.  
    - Configuration:  
      - URL: https://api.vapi.ai/call  
      - Method: POST  
      - JSON Body: Includes assistant ID, variables (name, web_domain), customer phone number, and phoneNumberId.  
      - Authorization Header with bearer token.  
    - Input: Triggered after downtime record creation.  
    - Output: Loops back to "Loop Over Items" for next website.  
    - Edge Cases: API key invalid, call failures, network issues.

  - **Email Notification for Up**  
    - Type: Gmail  
    - Role: Sends email notification when a website recovers.  
    - Configuration:  
      - Recipient: example@gmail.com.  
      - Subject: "Site is now working fine."  
      - Message: Announces website is live again.  
      - Credentials: Gmail OAuth2.  
    - Input: Triggered after updating uptime record.  
    - Output: None.  
    - Edge Cases: Same as down email.

  - **Slack Notification for Up**  
    - Type: Slack  
    - Role: Sends Slack message for website recovery.  
    - Configuration:  
      - Message text announcing site is live.  
      - User ID and OAuth2 authentication.  
    - Input: Triggered after uptime update.  
    - Output: None.  
    - Edge Cases: Same as down Slack.

  - **Telegram Notification for Up**  
    - Type: Telegram  
    - Role: Sends Telegram message announcing website is back online.  
    - Configuration:  
      - Chat ID and message text with site URL.  
      - Telegram API credentials.  
    - Input: Triggered after uptime update.  
    - Output: Loops back to "Loop Over Items".  
    - Edge Cases: Same as down Telegram.

#### 2.5 Control Flow and Documentation

- **Overview:**  
  Contains nodes for controlling branching and a sticky note with documentation.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides documentation inside the workflow for users.  
    - Content:  
      - Link to sample Google Sheet used for websites and downtime records.  
      - Instructions for setting up the Vapi.ai assistant with a sample message template for phone call alerts.  
    - Position: At the top-left area for easy visibility.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                               | Input Node(s)                  | Output Node(s)                                                                                      | Sticky Note                                                                                                                                                                        |
|-------------------------------|--------------------|----------------------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger   | Periodic workflow start                       | None                           | Website URLs                                                                                      |                                                                                                                                                                                    |
| Website URLs                  | Google Sheets      | Read website list                             | Schedule Trigger               | Loop Over Items                                                                                   |                                                                                                                                                                                    |
| Loop Over Items               | SplitInBatches     | Iterate over each website                      | Website URLs                  | Check website status, loops back from notifications                                              |                                                                                                                                                                                    |
| Check website status          | HTTP Request       | Check website availability                     | Loop Over Items               | If Site Up                                                                                       |                                                                                                                                                                                    |
| If Site Up                   | If                 | Decision: site status up/down                   | Check website status          | Get Active Down Record (both branches)                                                           |                                                                                                                                                                                    |
| Get Active Down Record        | Google Sheets      | Fetch active downtime record                    | If Site Up                   | If Down Record Exist                                                                             |                                                                                                                                                                                    |
| If Down Record Exist          | If                 | Check if active downtime record exists         | Get Active Down Record        | Update Uptime and Total Downtime (true), Loop Over Items (false)                                  |                                                                                                                                                                                    |
| Update Uptime and Total Downtime | Google Sheets  | Update downtime record with Up Time and total downtime | If Down Record Exist          | Email Notification for Up, Slack Notification for Up, Telegram Notification for Up               |                                                                                                                                                                                    |
| Email Notification for Up     | Gmail              | Notify recovery via email                       | Update Uptime and Total Downtime | Loop Over Items                                                                                   |                                                                                                                                                                                    |
| Slack Notification for Up     | Slack              | Notify recovery via Slack                       | Update Uptime and Total Downtime | Loop Over Items                                                                                   |                                                                                                                                                                                    |
| Telegram Notification for Up  | Telegram           | Notify recovery via Telegram                    | Update Uptime and Total Downtime | Loop Over Items                                                                                   |                                                                                                                                                                                    |
| Get Existing Down Records     | Google Sheets      | Get all downtime records for the website       | If Site Up                   | Find Existing Record                                                                             |                                                                                                                                                                                    |
| Find Existing Record          | Code (JavaScript)  | Check for incomplete downtime records           | Get Existing Down Records     | Check Downtime Exists                                                                            |                                                                                                                                                                                    |
| Check Downtime Exists         | If                 | Check if incomplete downtime record exists     | Find Existing Record          | Loop Over Items (true), Add Downtime Record (false)                                             |                                                                                                                                                                                    |
| Add Downtime Record           | Google Sheets      | Append new downtime record                       | Check Downtime Exists         | Email Notification for Down, Slack Notification for Down, Telegram Notification for Down, Notify over Phone Call |                                                                                                                                                                                    |
| Email Notification for Down   | Gmail              | Notify downtime via email                        | Add Downtime Record           | None                                                                                             |                                                                                                                                                                                    |
| Slack Notification for Down   | Slack              | Notify downtime via Slack                        | Add Downtime Record           | None                                                                                             |                                                                                                                                                                                    |
| Telegram Notification for Down| Telegram           | Notify downtime via Telegram                     | Add Downtime Record           | None                                                                                             |                                                                                                                                                                                    |
| Notify over Phone Call        | HTTP Request       | Initiate phone call alert via vapi.ai           | Add Downtime Record           | Loop Over Items                                                                                   |                                                                                                                                                                                    |
| Sticky Note                  | Sticky Note        | Documentation and instructions                   | None                         | None                                                                                             | Sample Sheet: https://docs.google.com/spreadsheets/d/1_VVpkIvpYQigw5q0KmPXUAC2aV2rk1nRQLQZ7YK2KwY/edit?usp=sharing. Instructions to create Vapi assistant with message template included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to run at a desired interval (e.g., every minute) to initiate the workflow.

2. **Add a Google Sheets node ("Website URLs"):**  
   - Connect from Schedule Trigger.  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Set Document ID to your Google Sheet containing website URLs.  
   - Set Sheet Name to the appropriate sheet (e.g., "Sheet1").  
   - This node reads the list of websites to monitor.

3. **Add a SplitInBatches node ("Loop Over Items"):**  
   - Connect from "Website URLs".  
   - Default batch size = 1 to process one website at a time.

4. **Add an HTTP Request node ("Check website status"):**  
   - Connect from second output of "Loop Over Items".  
   - Set URL dynamically with expression: `={{ $json["Website URL"] }}`.  
   - Set timeout to 5000 ms.  
   - Configure to return full response.

5. **Add an If node ("If Site Up"):**  
   - Connect from "Check website status".  
   - Condition: Check if `{{$json["statusCode"]}}` equals 400 (indicates down).  
   - True branch means site down; false means site up.

6. **Add a Google Sheets node ("Get Active Down Record"):**  
   - Connect True and False outputs from "If Site Up" to this node (parallel).  
   - Configure with same Google Sheets OAuth2 credentials.  
   - Document ID: downtime log sheet.  
   - Sheet Name: downtime logs sheet (e.g., "Sheet2").  
   - Filter rows where "Website URL" matches current website and "Up Time" is empty (active downtime).

7. **Add an If node ("If Down Record Exist"):**  
   - Connect from "Get Active Down Record".  
   - Condition: Check if output data is empty or not (true if record exists).

8. **Add a Google Sheets node ("Update Uptime and Total Downtime"):**  
   - Connect True output of "If Down Record Exist".  
   - Set operation to 'update'.  
   - Map fields:  
     - "Up Time" = current time in HH:mm:ss, e.g., `={{ new Date().toLocaleTimeString('en-GB', { hour12: false }) }}`  
     - "Total downtime" = calculated duration between Down Time and Up Time using JavaScript expression (handle overnight cases).  
   - Match rows by "row_number".

9. **Add Gmail, Slack, and Telegram nodes for "Up" notifications:**  
   - Connect all from "Update Uptime and Total Downtime".  
   - Configure each with appropriate credentials and recipient details.  
   - Use dynamic messages incorporating website URL.

10. **Add a Google Sheets node ("Get Existing Down Records"):**  
    - Connect False output of "If Site Up".  
    - Fetch all downtime records for current website (filter by "Website URL").

11. **Add a JavaScript Code node ("Find Existing Record"):**  
    - Connect from "Get Existing Down Records".  
    - Script checks if any record has "Down Time" but missing "Up Time" or "Total downtime".  
    - Outputs a status flag.

12. **Add an If node ("Check Downtime Exists"):**  
    - Connect from "Find Existing Record".  
    - Condition: status equals "yes" (incomplete record exists).

13. **Connect True output of "Check Downtime Exists" back to "Loop Over Items"** (skip adding new record).

14. **Add a Google Sheets node ("Add Downtime Record"):**  
    - Connect False output of "Check Downtime Exists".  
    - Operation: Append.  
    - Set "Down Time" to current time and "Website URL" to current item.

15. **Add Gmail, Slack, Telegram, and HTTP Request nodes for "Down" notifications:**  
    - Connect all from "Add Downtime Record".  
    - Configure messages for downtime alerts.  
    - For HTTP Request node ("Notify over Phone Call"):  
      - POST to https://api.vapi.ai/call  
      - Include JSON body with assistantId, variables (name, web_domain), customer number, and phoneNumberId.  
      - Set Authorization header with Bearer token.  
    - Connect output back to "Loop Over Items".

16. **Add a Sticky Note node to document the workflow:**  
    - Include links to sample Google Sheets.  
    - Provide instructions for setting up Vapi.ai assistant with example message.

17. **Verify and test workflow:**  
    - Ensure all OAuth2 credentials (Google Sheets, Gmail, Slack, Telegram) are correctly configured.  
    - Replace placeholders like email addresses, Slack user IDs, Telegram chat IDs, phone numbers, API keys with actual values.  
    - Run manual tests and adjust as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Sample Google Sheet used for websites and downtime logs: https://docs.google.com/spreadsheets/d/1_VVpkIvpYQigw5q0KmPXUAC2aV2rk1nRQLQZ7YK2KwY/edit?usp=sharing        | Source spreadsheet referenced in workflow                                                          |
| Vapi.ai assistant creation instructions: Set first message as "Hello {{name}}, I'm Website Monitoring Assistant. This is a system alert. The {{web_domain}} is down." | Phone call assistant setup for automated downtime alerts (https://vapi.ai)                          |
| Ensure all credentials (Google Sheets OAuth2, Gmail OAuth2, Slack OAuth2, Telegram API) are properly authorized and have necessary permissions configured.       | General credential setup                                                                              |
| The workflow handles overnight downtime calculation by adjusting date when Up Time is earlier than Down Time, avoiding negative durations.                      | Important for accurate total downtime reporting                                                   |
| HTTP request timeout set to 5 seconds to avoid long waits, but consider network reliability and possible false negatives if timeout too short.                 | Timeout configuration note                                                                          |
| The workflow assumes HTTP status code 400 indicates site down; consider expanding condition for other error codes (e.g., 500, 404) for broader detection.         | Potential enhancement                                                                                |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.