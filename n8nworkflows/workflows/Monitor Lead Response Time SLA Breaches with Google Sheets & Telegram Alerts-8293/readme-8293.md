Monitor Lead Response Time SLA Breaches with Google Sheets & Telegram Alerts

https://n8nworkflows.xyz/workflows/monitor-lead-response-time-sla-breaches-with-google-sheets---telegram-alerts-8293


# Monitor Lead Response Time SLA Breaches with Google Sheets & Telegram Alerts

### 1. Workflow Overview

**Purpose:**  
This workflow monitors lead response times stored in a Google Sheet and alerts a Telegram chat when any lead breaches the SLA (Service Level Agreement) for response time. Specifically, it checks every 5 minutes for leads marked as “unreplied” for more than 15 minutes and sends a detailed notification to a Telegram channel/group to prompt immediate action.

**Target Use Cases:**  
- Sales or customer support teams needing to ensure timely follow-up on new leads.  
- SLA compliance monitoring with automated alerts for delayed responses.  
- Centralized alerting using Google Sheets as a lead source and Telegram for immediate notification.

**Logical Blocks:**  
- **1.1 Periodic Trigger:** Periodically triggers the workflow every 5 minutes to initiate the SLA check.  
- **1.2 Data Retrieval:** Fetches lead data from a specific Google Sheet.  
- **1.3 SLA Breach Filtering:** Filters leads that are “unreplied” and whose last update timestamp exceeds the 15-minute SLA threshold.  
- **1.4 Data Enrichment:** Adds a direct link to the Google Sheet row for easy access in alerts.  
- **1.5 Alert Dispatch:** Sends a formatted Telegram message with the lead details and SLA breach alert.

---

### 2. Block-by-Block Analysis

#### 2.1 Periodic Trigger

- **Overview:**  
  Initiates the workflow execution every 5 minutes to ensure timely SLA breach checks.

- **Nodes Involved:**  
  - Every 5 Minutes

- **Node Details:**  
  - **Every 5 Minutes**  
    - Type: Cron Trigger  
    - Configuration: Runs every 5 minutes (default configuration, no additional parameters)  
    - Input: None (trigger node)  
    - Output: Triggers next node “Get row(s) in sheet”  
    - Edge Cases: Workflow depends on this trigger for periodic checks; if disabled or misconfigured, SLA checks will not run. Cron misfires due to system downtime would delay alerts.

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves all lead data from the designated Google Sheet, serving as the data source for SLA checks.

- **Nodes Involved:**  
  - Get row(s) in sheet

- **Node Details:**  
  - **Get row(s) in sheet**  
    - Type: Google Sheets (Read)  
    - Configuration:  
      - Document ID set to a specific Google Sheet (ID: `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`)  
      - Sheet Name referencing a sheet by GID `540316831` (named “sample_leads_50”)  
      - Reads all rows (no filters or ranges specified)  
    - Credentials: Google Sheets OAuth2 account “Google Sheets account 3”  
    - Input: Trigger from “Every 5 Minutes”  
    - Output: Full lead dataset passed to “SLA Breach Check” node  
    - Edge Cases:  
      - Authorization errors if OAuth2 token expires or invalid  
      - Rate limits or API quota exceeded on Google Sheets API  
      - Empty or malformed sheets could cause downstream errors  
    - Sticky Note: This node fetches all lead data including status, contact details, and timestamps.

#### 2.3 SLA Breach Filtering

- **Overview:**  
  Filters leads that have status “unreplied” and whose last modification time is older than 15 minutes ago, indicating SLA breach.

- **Nodes Involved:**  
  - SLA Breach Check

- **Node Details:**  
  - **SLA Breach Check**  
    - Type: If (Conditional Filter)  
    - Configuration:  
      - Two conditions combined:  
        1. Status equals "unreplied" (string comparison)  
        2. Date/time condition: Lead timestamp is before (now - 15 minutes)  
      - Date/time expressions:  
        - Lower bound: `new Date(Date.now() - 15 * 60 * 1000)` (15 minutes ago)  
        - Upper bound: current time `Date.now()`  
    - Input: Data rows from Google Sheets  
    - Output: Leads that meet both conditions forwarded to “Code1” for enrichment  
    - Edge Cases:  
      - Expression evaluation failure if data fields are missing or malformed  
      - Timezone mismatches between stored timestamps and system time may cause false positives/negatives  
      - Leads without a “Status” or timestamp field will not pass the filter  
    - Sticky Note: Filters leads unreplied for more than 15 minutes; only those are escalated.

#### 2.4 Data Enrichment

- **Overview:**  
  Enriches the filtered lead data by adding a direct, clickable URL linking to the corresponding Google Sheet row, facilitating quick access during alerts.

- **Nodes Involved:**  
  - Code1

- **Node Details:**  
  - **Code1**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Iterates over all lead rows passed from the SLA Breach Check node.  
      - Copies all original JSON fields as-is.  
      - Adds a new field `link` containing a URL to the specific Google Sheet tab.  
      - The link is static and points to the lead data sheet’s main tab for quick review.  
    - Input: Filtered lead items from SLA Breach Check  
    - Output: Enriched lead data with added `link` property, passed to Telegram alert node  
    - Edge Cases:  
      - Hardcoded link means if the sheet structure changes or rows move, direct row linking is not dynamic  
      - If input data is empty, output will be empty (normal case)  
    - Sticky Note: Adds a direct Google Sheet link for quick lead detail access.

#### 2.5 Alert Dispatch

- **Overview:**  
  Sends a detailed Telegram message notifying about the SLA breach lead, including all relevant lead information and the link to the sheet.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Send a text message**  
    - Type: Telegram node  
    - Configuration:  
      - Chat ID set statically to `963318735` (Telegram chat/group ID)  
      - Message text composed with n8n expressions, interpolating lead fields:  
        - Company Name, Contact Person, Job Title, Industry, Location, Email, Phone, Lead Source  
        - Status displayed with emoji indicating unreplied (❌) or replied (✅)  
        - Booking Status displayed similarly with colored emoji  
        - Lead ID with row number  
        - Direct clickable link to Google Sheet row  
      - Parse mode set to HTML for formatting support  
    - Credentials: Telegram API credentials “Telegram account”  
    - Input: Enriched lead data from Code1  
    - Output: No further nodes (end of branch)  
    - Edge Cases:  
      - Telegram API errors due to invalid token, revoked access, or network issues  
      - Chat ID must be valid and the bot must be a member of the chat to send messages  
      - Large or malformed messages may cause send failures  
    - Sticky Note: Sends immediate SLA breach alert via Telegram with lead details and clickable sheet link.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                 | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                                 |
|---------------------|----------------------|--------------------------------|----------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| Every 5 Minutes      | Cron Trigger         | Periodic workflow trigger       | —                    | Get row(s) in sheet     |                                                                                                             |
| Get row(s) in sheet | Google Sheets        | Retrieves lead data             | Every 5 Minutes      | SLA Breach Check        | Fetches all lead data from Google Sheets including status, contacts, and timestamps.                        |
| SLA Breach Check     | If                   | Filters leads breaching SLA     | Get row(s) in sheet  | Code1                   | Filters leads unreplied for more than 15 minutes; only those pass forward for alerting.                     |
| Code1               | Code (JavaScript)    | Adds Google Sheet link to data  | SLA Breach Check     | Send a text message     | Adds a direct Google Sheet link for quick lead detail access.                                              |
| Send a text message  | Telegram             | Sends Telegram SLA breach alert | Code1                | —                       | Sends a Telegram notification with lead details and sheet link to prompt immediate action.                 |
| Sticky Note          | Sticky Note          | Documentation                   | —                    | —                       | This node sends a Telegram notification whenever an SLA breach is detected with lead details and link.     |
| Sticky Note1         | Sticky Note          | Documentation                   | —                    | —                       | This node enriches the output by adding a direct link to the Google Sheet row for quick access.            |
| Sticky Note2         | Sticky Note          | Documentation                   | —                    | —                       | This node applies the condition to filter leads unreplied for more than 15 minutes for escalation.          |
| Sticky Note3         | Sticky Note          | Documentation                   | —                    | —                       | This node fetches all lead data from Google Sheets to pass forward for SLA checking.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node:**  
   - Name: `Every 5 Minutes`  
   - Type: Cron Trigger  
   - Set to run every 5 minutes (default settings).  

2. **Add a Google Sheets node:**  
   - Name: `Get row(s) in sheet`  
   - Type: Google Sheets (Read Rows)  
   - Connect input from `Every 5 Minutes`.  
   - Set Credentials: Use OAuth2 credentials for your Google Sheets account.  
   - Configure Document ID with your Google Sheet ID (e.g., `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`).  
   - Set Sheet Name by GID or name (e.g., `540316831` or “sample_leads_50”).  
   - Leave range empty or select the full sheet.  

3. **Add an If node for filtering SLA breaches:**  
   - Name: `SLA Breach Check`  
   - Connect input from `Get row(s) in sheet`.  
   - Configure two conditions:  
     - String condition: `$json.Status === 'unreplied'`  
     - DateTime condition: `$json.LastModified` or equivalent timestamp field is before `new Date(Date.now() - 15 * 60 * 1000)` (15 minutes ago). Use expressions to define these dynamically.  
   - Configure output to send matched leads forward only.  

4. **Add a Code node to enrich data:**  
   - Name: `Code1`  
   - Connect input from `SLA Breach Check`.  
   - Paste JavaScript code to:  
     - Iterate input items.  
     - Copy all properties.  
     - Add a `link` property with the static URL to your Google Sheet tab (adjust ID and GID accordingly).  
   - Return enriched data.  

5. **Add a Telegram node to send messages:**  
   - Name: `Send a text message`  
   - Connect input from `Code1`.  
   - Set Telegram credentials with a valid bot token.  
   - Set `chatId` to your target Telegram chat/group ID.  
   - Compose the message text with dynamic fields (using expressions) including Company Name, Contact Person, Status with emojis, Booking Status, Lead ID, and the direct Google Sheet link.  
   - Set parse mode to HTML for formatting support.  

6. **Test the workflow:**  
   - Run manually or wait for the cron to trigger.  
   - Ensure messages appear in Telegram only for leads unreplied for more than 15 minutes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The message formatting uses emojis and HTML parse mode to improve readability and urgency in Telegram notifications.                   | Telegram message node configuration                                                                          |
| Google Sheets API quota and OAuth2 token expiration may affect the workflow; monitor and refresh credentials as needed.                 | Google Sheets node                                                                                          |
| Ensure the Telegram bot is added to the target chat/group with permission to send messages for alerts to work.                          | Telegram Bot API requirements                                                                                |
| SLA threshold (15 minutes) and sheet URLs are hardcoded; adjust these parameters in the If node and Code node respectively as needed.   | Workflow customization                                                                                        |
| The original Google Sheet must have columns including Status, Company Name, Contact Person, Job Title, Industry, Location, Email, Phone. | Source data expectations                                                                                     |
| Workflow inactive by default; activate after setup and credential validation.                                                           | Workflow metadata                                                                                           |

---

**Disclaimer:** The content provided is extracted exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal or protected data. All manipulated data is legal and publicly available.