Weekly Client Re-engagement System with GoHighLevel, Gmail, Sheets, and Slack

https://n8nworkflows.xyz/workflows/weekly-client-re-engagement-system-with-gohighlevel--gmail--sheets--and-slack-9237


# Weekly Client Re-engagement System with GoHighLevel, Gmail, Sheets, and Slack

---
### 1. Workflow Overview

This workflow automates the weekly client re-engagement process by identifying clients in the HighLevel CRM who have not been contacted in over 14 days and triggering a multi-channel engagement and notification sequence. It is ideal for agencies, consultants, and service-oriented businesses aiming to maintain regular client contact and reduce churn.

The workflow logically divides into the following blocks:

- **1.1 Scheduled Trigger:** Initiates workflow execution every Monday at 9:00 AM.
- **1.2 Data Retrieval:** Fetches all contacts from HighLevel CRM.
- **1.3 Filtering:** Identifies clients inactive for 14+ days based on last activity timestamps.
- **1.4 Data Splitting:** Splits the filtered array into individual contacts for processing.
- **1.5 Conditional Processing:** Checks if each contact truly requires attention.
- **1.6 Communication:** Sends personalized re-engagement emails to inactive clients.
- **1.7 Logging:** Appends inactive client details to a Google Sheet for tracking.
- **1.8 Slack Notification:** Aggregates inactive clients and sends a summary notification to account managers on Slack.
- **1.9 Error Handling:** Captures any errors during execution and immediately alerts the team via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Automatically triggers the workflow weekly on Mondays at 9:00 AM.
- **Nodes Involved:**  
  - Trigger: Weekly Scheduler
  - Note: Schedule Trigger (sticky note)
- **Node Details:**

  - **Trigger: Weekly Scheduler**
    - Type: Schedule Trigger
    - Role: Starts workflow based on a cron schedule
    - Configuration: Cron expression `0 9 * * 1` (9:00 AM every Monday)
    - Inputs: Manual trigger only (for testing)
    - Outputs: Triggers downstream nodes when scheduled
    - Edge cases: Cron misconfiguration can cause no trigger; manual test recommended before scheduling
    - Version: 1.1

  - **Note: Schedule Trigger**
    - Type: Sticky Note
    - Role: Documentation for schedule configuration
    - Content: Explains cron syntax and testing instructions
    - No connections

#### 1.2 Data Retrieval

- **Overview:** Fetches all contact records from HighLevel CRM with no initial filters.
- **Nodes Involved:**  
  - Fetch HighLevel Contacts  
  - Note: HighLevel Fetch (sticky note)
- **Node Details:**

  - **Fetch HighLevel Contacts**
    - Type: HighLevel CRM node
    - Role: Retrieves all contacts from CRM, sorted by most recent update
    - Configuration:
      - Operation: Get Many Contacts (getAll)
      - Sort by: `date_updated` descending
      - No filters applied at this stage
      - Credential: HighLevel OAuth2 API (authorized CRM access)
    - Inputs: Trigger node output
    - Outputs: Array of all contacts with full data (names, emails, phones, tags, timestamps)
    - Edge cases: API limits, expired tokens, network errors
    - Version: 2

  - **Note: HighLevel Fetch**
    - Type: Sticky Note
    - Role: Documentation for node setup and output structure

#### 1.3 Filtering

- **Overview:** Filters contacts to only those inactive for 14 or more days based on `dateUpdated` or `dateAdded`.
- **Nodes Involved:**  
  - Filter Inactive Contacts (14+ days)  
  - Note: Filter Logic (sticky note)
- **Node Details:**

  - **Filter Inactive Contacts (14+ days)**
    - Type: Code node (JavaScript)
    - Role: Processes all contacts, computes inactivity duration, flags those needing attention
    - Configuration:
      - Custom JS code calculates a date 14 days prior to now
      - For each contact, uses `dateUpdated` (or fallback `dateAdded`)
      - Compares last activity date to threshold; if older, adds:
        - `daysSinceContact` (integer)
        - `lastContactDate` (timestamp)
        - `needsAttention` (boolean true)
      - Returns filtered array of flagged contacts
    - Inputs: Contacts from HighLevel node
    - Outputs: Array of inactive contacts with extra fields
    - Edge cases: Missing or malformed date fields, time zone differences, empty input arrays
    - Version: 2

  - **Note: Filter Logic**
    - Type: Sticky Note
    - Role: Explains filtering logic and how to customize the inactivity period

#### 1.4 Data Splitting

- **Overview:** Converts the filtered array into individual execution items for per-contact processing.
- **Nodes Involved:**  
  - Split Clients Array  
  - Note: Split Array (sticky note)
- **Node Details:**

  - **Split Clients Array**
    - Type: Split Out node
    - Role: Takes array of contacts, emits one item per contact downstream
    - Configuration:
      - Field to split on includes all relevant contact fields (names, emails, phone, dates, tags, flags)
    - Inputs: Filtered array from code node
    - Outputs: Individual contact items
    - Edge cases: Empty input array results in no downstream executions
    - Version: 1

  - **Note: Split Array**
    - Type: Sticky Note
    - Role: Explains the reason for splitting and fields included

#### 1.5 Conditional Processing

- **Overview:** Ensures only contacts flagged with `needsAttention` proceed to email and logging.
- **Nodes Involved:**  
  - IF: Needs Attention  
  - Note: Condition Check (sticky note)
- **Node Details:**

  - **IF: Needs Attention**
    - Type: If node
    - Role: Checks boolean `needsAttention` field equals true
    - Configuration:
      - Condition: `needsAttention === true`
    - Inputs: Individual contact from Split node
    - Outputs: 
      - True branch: sends email and logs to Google Sheets
      - False branch: no action (contact skipped)
    - Edge cases: Missing or incorrectly typed `needsAttention` field may cause false negatives
    - Version: 1

  - **Note: Condition Check**
    - Type: Sticky Note
    - Role: Explains the importance of validation before emailing or logging

#### 1.6 Communication

- **Overview:** Sends personalized, branded re-engagement emails to inactive clients using Gmail.
- **Nodes Involved:**  
  - Send Re-engagement Email  
  - Note: Email Sender (sticky note)
- **Node Details:**

  - **Send Re-engagement Email**
    - Type: Gmail node
    - Role: Sends HTML email to each inactive client
    - Configuration:
      - Credential: Gmail OAuth2 (authorized sender account)
      - To: Dynamic email `{{ $json.email }}`
      - Subject: "Just Checking In!"
      - Message: Professional HTML template with personalized greetings using variables like `{{ $json.firstName }}`, company name, and days since last contact
      - Features gradient header, call-to-action button, and responsive design
    - Inputs: True branch from IF node
    - Outputs: Email send response
    - Edge cases: Invalid email addresses, OAuth token expiry, Gmail API rate limits
    - Version: 2.1

  - **Note: Email Sender**
    - Type: Sticky Note
    - Role: Details email content structure and customization options

#### 1.7 Logging

- **Overview:** Appends each inactive contact’s details to a Google Sheet for tracking and historical analysis.
- **Nodes Involved:**  
  - Log Inactive Client in Google Sheets  
  - Note: Google Sheets Logger (sticky note)
- **Node Details:**

  - **Log Inactive Client in Google Sheets**
    - Type: Google Sheets node
    - Role: Appends contact data as a new row or updates existing row keyed by contact `id`
    - Configuration:
      - Credential: Google Sheets OAuth2
      - Document ID: User’s Google Sheet ID
      - Sheet name: Configured sheet (e.g., "Sheet1" or "Client Engagement")
      - Operation: Append with auto-mapping input data to columns
      - Matching column: `id` to prevent duplicates
      - Columns: Includes contactName, firstName, lastName, email, phone, companyName, address, dates, flags, and tags
    - Inputs: True branch from IF node (same as email)
    - Outputs: Confirmation of append/update
    - Edge cases: Invalid spreadsheet ID, permission errors, rate limits, missing columns
    - Version: 4

  - **Note: Google Sheets Logger**
    - Type: Sticky Note
    - Role: Setup instructions and purpose of logging

#### 1.8 Slack Notification

- **Overview:** Aggregates all inactive contacts after processing and sends a summary notification to Slack channel(s) for account managers.
- **Nodes Involved:**  
  - Format Slack Message  
  - Send Slack Notification to Account Manager  
  - Note: Message Formatter  
  - Note: Slack Notifier
- **Node Details:**

  - **Format Slack Message**
    - Type: Code node (JavaScript)
    - Role: Aggregates contact items into a formatted Slack message
    - Configuration:
      - Maps all input contacts to objects with name, email, company, phone, days since last contact
      - Handles missing last names gracefully
      - Builds a bulleted list string for Slack message
      - Outputs JSON with:
        - `count` (number of clients)
        - `clientList` (formatted string)
        - `clients` (array of objects)
    - Inputs: Google Sheets node output (accumulates all logged contacts)
    - Outputs: Single JSON summary for Slack
    - Edge cases: Empty arrays yield zero count and empty list
    - Version: 2

  - **Send Slack Notification to Account Manager**
    - Type: Slack node
    - Role: Posts formatted message to Slack channel
    - Configuration:
      - Credential: Slack OAuth2 app
      - Channel: Configured target (e.g., `#client-engagement`)
      - Message: Uses markdown, includes alert emoji, client count, bulleted client list, call to action
    - Inputs: Output from Format Slack Message node
    - Outputs: Slack post confirmation
    - Edge cases: Invalid channel ID, token expiration, permission errors
    - Version: 2.1

  - **Note: Message Formatter**
    - Type: Sticky Note
    - Role: Describes logic of aggregation and output formatting

  - **Note: Slack Notifier**
    - Type: Sticky Note
    - Role: Slack setup and best practices for notification channels

#### 1.9 Error Handling

- **Overview:** Catches any error occurring anywhere in the workflow and immediately alerts the responsible team via Slack.
- **Nodes Involved:**  
  - Error Trigger  
  - Send Error Alert to Slack  
  - Note: Error Trigger  
  - Note: Error Alerting
- **Node Details:**

  - **Error Trigger**
    - Type: Error Trigger node
    - Role: Listens for any error thrown by any node in the workflow
    - Configuration: Default, no parameters
    - Inputs: N/A (automatic)
    - Outputs: Error data including node name, error message, timestamp, stack trace
    - Edge cases: Must be first in error handling chain to capture all errors
    - Version: 1

  - **Send Error Alert to Slack**
    - Type: Slack node
    - Role: Sends immediate error notification to Slack
    - Configuration:
      - Credential: Slack OAuth2
      - Channel: Configured error alert channel (e.g., `#errors`)
      - Message template includes:
        - Node name that failed
        - Error message text
        - Timestamp in `yyyy-MM-dd HH:mm:ss` format
        - Call to immediate investigation
    - Inputs: Error Trigger output
    - Outputs: Slack message confirmation
    - Edge cases: Slack API issues, token expiry
    - Version: 2.1

  - **Note: Error Trigger**
    - Type: Sticky Note
    - Role: Explains importance of error capture and common failure causes

  - **Note: Error Alerting**
    - Type: Sticky Note
    - Role: Describes alert message format and Slack channel setup

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                       | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                                                        |
|-----------------------------------|-----------------------|------------------------------------|-----------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview                 | Sticky Note           | Documentation overview              | —                           | —                                     | Contains full workflow purpose and use case                                                                                       |
| Note: Schedule Trigger            | Sticky Note           | Explains schedule trigger setup    | —                           | —                                     | Details cron expression, examples, and testing                                                                                    |
| Note: HighLevel Fetch             | Sticky Note           | Explains HighLevel contact fetch   | —                           | —                                     | Setup instructions and output details                                                                                            |
| Note: Filter Logic                | Sticky Note           | Explains filter criteria           | —                           | —                                     | Shows JS filter logic and fields used                                                                                            |
| Note: Split Array                 | Sticky Note           | Explains splitting array to items  | —                           | —                                     | Reason for splitting and fields included                                                                                         |
| Note: Condition Check             | Sticky Note           | Explains IF node condition         | —                           | —                                     | Importance of validation before action                                                                                           |
| Note: Email Sender                | Sticky Note           | Explains Gmail email setup         | —                           | —                                     | Details email template and personalization                                                                                        |
| Note: Google Sheets Logger        | Sticky Note           | Explains Google Sheets logging     | —                           | —                                     | Setup and purpose of logging                                                                                                     |
| Note: Message Formatter           | Sticky Note           | Explains Slack message formatting  | —                           | —                                     | JS code explanation for message assembly                                                                                        |
| Note: Slack Notifier              | Sticky Note           | Explains Slack notification setup  | —                           | —                                     | Slack app permissions and channel suggestions                                                                                   |
| Note: Error Trigger               | Sticky Note           | Explains error trigger importance  | —                           | —                                     | Failure modes and why error capture is critical                                                                                  |
| Note: Error Alerting              | Sticky Note           | Explains Slack error alerts        | —                           | —                                     | Message format and recommended channels                                                                                          |
| Trigger: Weekly Scheduler         | Schedule Trigger      | Starts workflow on schedule        | —                           | Fetch HighLevel Contacts               | Refer to schedule trigger note                                                                                                  |
| Fetch HighLevel Contacts          | HighLevel CRM node    | Retrieves all contacts             | Trigger: Weekly Scheduler    | Filter Inactive Contacts (14+ days)   | Refer to HighLevel fetch note                                                                                                   |
| Filter Inactive Contacts (14+ days) | Code node            | Filters inactive clients           | Fetch HighLevel Contacts     | Split Clients Array                   | Refer to filter logic note                                                                                                       |
| Split Clients Array               | Split Out node        | Splits array to individual items   | Filter Inactive Contacts     | IF: Needs Attention                   | Refer to split array note                                                                                                       |
| IF: Needs Attention              | If node               | Checks if contact needs attention  | Split Clients Array          | True: Send Re-engagement Email, Log Inactive Client in Google Sheets | Refer to condition check note                                                                                                   |
| Send Re-engagement Email         | Gmail node            | Sends personalized email           | IF: Needs Attention (true)   | Log Inactive Client in Google Sheets  | Refer to email sender note                                                                                                      |
| Log Inactive Client in Google Sheets | Google Sheets node    | Logs client data                   | IF: Needs Attention (true)   | Format Slack Message                  | Refer to Google Sheets logger note                                                                                              |
| Format Slack Message             | Code node             | Aggregates clients for Slack       | Log Inactive Client in Google Sheets | Send Slack Notification to Account Manager | Refer to message formatter note                                                                                                |
| Send Slack Notification to Account Manager | Slack node           | Sends Slack summary message        | Format Slack Message         | —                                     | Refer to Slack notifier note                                                                                                    |
| Error Trigger                   | Error Trigger node    | Captures workflow errors           | —                           | Send Error Alert to Slack             | Refer to error trigger note                                                                                                     |
| Send Error Alert to Slack       | Slack node            | Sends error notification           | Error Trigger               | —                                     | Refer to error alerting note                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add and Configure Scheduled Trigger:**
   - Node Type: Schedule Trigger
   - Set Cron Expression: `0 9 * * 1` (runs every Monday at 9:00 AM)
   - Save node as "Trigger: Weekly Scheduler"

3. **Add HighLevel CRM Node:**
   - Node Type: HighLevel
   - Operation: Get Many Contacts (`getAll`)
   - Sort by: `date_updated` descending
   - Credentials: Set up or select existing HighLevel OAuth2 credential authorized to access your CRM
   - Connect input from "Trigger: Weekly Scheduler"
   - Name node "Fetch HighLevel Contacts"

4. **Add Code Node to Filter Inactive Contacts:**
   - Node Type: Code
   - Language: JavaScript
   - Paste code (see filtering logic):
     ```javascript
     const fourteenDaysAgo = new Date();
     fourteenDaysAgo.setDate(fourteenDaysAgo.getDate() - 14);

     const inactiveContacts = [];

     for (const item of $input.all()) {
       const contact = item.json;
       const lastActivity = contact.dateUpdated || contact.dateAdded;
       const lastActivityDate = new Date(lastActivity);

       if (lastActivityDate < fourteenDaysAgo) {
         inactiveContacts.push({
           json: {
             ...contact,
             daysSinceContact: Math.floor((new Date() - lastActivityDate) / (1000 * 60 * 60 * 24)),
             lastContactDate: lastActivity,
             needsAttention: true
           }
         });
       }
     }

     return inactiveContacts;
     ```
   - Connect input from "Fetch HighLevel Contacts"
   - Name node "Filter Inactive Contacts (14+ days)"

5. **Add Split Out Node:**
   - Node Type: Split Out
   - Field to split: Include all relevant fields, e.g., `contactName, firstName, lastName, companyName, email, phone, address1, dateUpdated, dateAdded, id, tags, lastContactDate, needsAttention`
   - Connect input from "Filter Inactive Contacts (14+ days)"
   - Name node "Split Clients Array"

6. **Add IF Node to Check `needsAttention`:**
   - Node Type: If
   - Condition: Boolean equals, `{{$json.needsAttention}}` equals `true`
   - Connect input from "Split Clients Array"
   - Name node "IF: Needs Attention"

7. **Add Gmail Node to Send Email:**
   - Node Type: Gmail
   - Credential: Connect or create Gmail OAuth2 credential with send email permission
   - To: Use expression `{{$json.email}}`
   - Subject: "Just Checking In!"
   - Message: Use professional HTML template with personalization:
     - Include variables like `{{$json.firstName}}`, `{{$json.lastName}}`, `{{$json.companyName}}`, `{{$json.daysSinceContact}}`
     - Include gradient header, greeting, call-to-action button, and footer as per original design
   - Connect input from IF node "True" branch
   - Name node "Send Re-engagement Email"

8. **Add Google Sheets Node to Log Client:**
   - Node Type: Google Sheets
   - Operation: Append
   - Credentials: Google Sheets OAuth2 with access to tracking sheet
   - Document ID: Your Google Sheet ID
   - Sheet Name: e.g., "Client Engagement" or "Sheet1"
   - Enable auto-map input data
   - Matching column: `id`
   - Columns: Include all fields such as contactName, firstName, lastName, email, phone, companyName, address1, dateUpdated, dateAdded, lastContactDate, needsAttention, tags
   - Connect input from IF node "True" branch (can be connected after Gmail node)
   - Name node "Log Inactive Client in Google Sheets"

9. **Add Code Node to Format Slack Message:**
   - Node Type: Code
   - JavaScript code to aggregate all contacts and build message:
     ```javascript
     const clients = $input.all().map(item => {
       const data = item.json;
       const fullName = data.lastName ? `${data.firstName} ${data.lastName}` : data.firstName || data.contactName;
       const lastContact = new Date(data.lastContactDate);
       const today = new Date();
       const daysSinceContact = Math.floor((today - lastContact) / (1000 * 60 * 60 * 24));
       return {
         name: fullName,
         email: data.email || 'No email',
         company: data.companyName || 'No company',
         phone: data.phone || 'No phone',
         daysSinceContact: daysSinceContact
       };
     });

     const clientList = clients.map(c => `• ${c.name} (${c.email}) - ${c.daysSinceContact} days since last contact`).join('\n');

     return {
       json: {
         count: clients.length,
         clientList: clientList,
         clients: clients
       }
     };
     ```
   - Connect input from "Log Inactive Client in Google Sheets"
   - Name node "Format Slack Message"

10. **Add Slack Node to Send Notification:**
    - Node Type: Slack
    - Credential: Slack OAuth2 app with `chat:write` and `channels:read` scopes
    - Channel: Target channel ID (e.g., `#client-engagement`)
    - Message:
      ```
      🚨 *Weekly Client Engagement Pulse*

      *{{ $json.count }}* clients need attention (no contact in 14+ days):

      {{ $json.clientList }}

      Please review and follow up accordingly.
      ```
    - Enable Markdown (mrkdwn)
    - Connect input from "Format Slack Message"
    - Name node "Send Slack Notification to Account Manager"

11. **Add Error Trigger Node:**
    - Node Type: Error Trigger
    - No parameters
    - Name node "Error Trigger"

12. **Add Slack Node for Error Alerts:**
    - Node Type: Slack
    - Credential: Slack OAuth2 (can be the same app)
    - Channel: Error alert channel ID (e.g., `#errors`)
    - Message:
      ```
      ❌ *Workflow Error: Weekly Client Engagement Pulse*

      *Node:* {{ $json.node.name }}
      *Error:* {{ $json.error.message }}
      *Time:* {{ $now.format('yyyy-MM-dd HH:mm:ss') }}

      Please investigate immediately.
      ```
    - Connect input from "Error Trigger"
    - Name node "Send Error Alert to Slack"

13. **Connect all nodes according to dependencies:**
    - Trigger → Fetch HighLevel Contacts → Filter Inactive Contacts → Split Clients Array → IF node
    - IF True → Send Re-engagement Email → Log Inactive Client in Google Sheets → Format Slack Message → Send Slack Notification
    - Error Trigger → Send Error Alert to Slack

14. **Test workflow manually before activating schedule.**

15. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                            |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| This workflow runs every Monday at 9:00 AM by default. Adjust the cron expression in the Schedule Trigger node as needed to fit your schedule.                                | Schedule Trigger node documentation       |
| Gmail OAuth2 credentials require enabling Gmail API and proper OAuth client setup in Google Cloud Console.                                                                     | Gmail API setup                           |
| HighLevel OAuth2 credentials require authorization with your HighLevel CRM account.                                                                                             | HighLevel API docs                        |
| Google Sheets must have columns matching the mapped fields exactly for proper logging and duplicate prevention.                                                               | Google Sheets setup                       |
| Slack app must have `chat:write` and `channels:read` permissions and be installed in your workspace with access to target channels.                                           | Slack API documentation                   |
| Error handling is essential to prevent silent failures and ensure your team is notified immediately of any issues during execution.                                           | Best practices for error handling         |
| Customize email templates and Slack messages to suit your branding and voice.                                                                                                  | HTML email design and Slack message formatting |
| For scalability, consider API rate limits on HighLevel, Gmail, Google Sheets, and Slack, and adjust workflow frequency or batch sizes accordingly.                            | API rate limits documentation             |
| Always test workflow with a few contacts before going live to ensure email deliverability and correct data handling.                                                           | Testing recommendations                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.