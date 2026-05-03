Extract Business Leads from Gmail to Google Sheets with Slack Notifications

https://n8nworkflows.xyz/workflows/extract-business-leads-from-gmail-to-google-sheets-with-slack-notifications-7752


# Extract Business Leads from Gmail to Google Sheets with Slack Notifications

### 1. Workflow Overview

This workflow, titled **"Extract Business Leads from Gmail to Google Sheets with Slack Notifications"**, automates the extraction of potential business leads from Gmail inboxes, filters and processes the email data, stores the leads in a Google Sheets document, and sends notifications to a Slack channel. It targets sales, marketing, and business development professionals who want to mine their email inbox for actionable leads without manual effort.

The workflow logically divides into these blocks:

- **1.1 Input Reception:** Two triggers initiate the workflow — a manual trigger for historical email fetching and a Gmail trigger for real-time periodic fetching.
- **1.2 Email Retrieval:** Fetch emails either historically (up to 500) or in real-time.
- **1.3 Email Parsing and Filtering:** Extract sender details, filter out non-relevant/system/generic/personal emails using custom JavaScript code nodes.
- **1.4 Duplicate Removal:** Ensure only unique email leads are passed forward.
- **1.5 Data Storage:** Append or update lead data into a Google Sheets spreadsheet, keyed by email address.
- **1.6 Notification:** Send Slack messages to notify about new leads added.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow execution. It supports two modes: manual trigger for historical data extraction and a Gmail trigger configured for scheduled periodic runs.

**Nodes Involved:**  
- Manual Trigger (Historical Run)  
- Gmail Trigger (Real-time)

**Node Details:**

- **Manual Trigger (Historical Run)**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually to fetch historical emails.  
  - Config: No parameters; activated by user click.  
  - Inputs: None  
  - Outputs: Connected to "Get many messages" node.  
  - Edge Cases: None significant; manual user error possible if triggered too frequently.

- **Gmail Trigger (Real-time)**  
  - Type: Gmail Trigger  
  - Role: Triggers workflow based on schedule to fetch new emails.  
  - Config: Set to poll once daily at 8 AM (hour: 8).  
  - Credentials: Uses Gmail OAuth2 credentials.  
  - Inputs: None  
  - Outputs: Connected to "Code" node for parsing.  
  - Edge Cases: OAuth token expiration, Gmail API rate limits, network timeouts.

---

#### 2.2 Email Retrieval

**Overview:**  
Fetches emails from Gmail either historically via manual trigger or in real-time via scheduled Gmail Trigger.

**Nodes Involved:**  
- Get many messages

**Node Details:**

- **Get many messages**  
  - Type: Gmail node  
  - Role: Retrieves up to 500 emails from the inbox.  
  - Config: Operation "getAll", limit 500 emails, no specific filters.  
  - Credentials: Gmail OAuth2.  
  - Inputs: From "Manual Trigger (Historical Run)" node.  
  - Outputs: Connected to "Code" node for parsing and filtering.  
  - Edge Cases: Gmail API quota exceeded, network failures, large volume causing timeouts.

---

#### 2.3 Email Parsing and Filtering

**Overview:**  
This block parses the emails to extract company name, email, domain, subject, and formatted date. It filters out system/service emails and generic company inboxes. Then it filters out personal email domains.

**Nodes Involved:**  
- Code (parse and filter system/generic emails)  
- Code2 (filter personal emails)

**Node Details:**

- **Code (parse and filter system/generic emails)**  
  - Type: Code (JavaScript)  
  - Role: Parses "From" field to extract company name and email; filters out emails with system/generic/local parts.  
  - Key Logic:  
    - Regex parses display name and email.  
    - Filters emails like no-reply@, info@, support@, etc.  
    - Formats email received date in US locale with weekday and time.  
  - Inputs: From "Get many messages" or "Gmail Trigger (Real-time)".  
  - Outputs: To "Code2" node.  
  - Edge Cases: Malformed email addresses, missing "From" field, unexpected formats.

- **Code2 (filter personal emails)**  
  - Type: Code (JavaScript)  
  - Role: Filters out emails from personal domains such as gmail.com, yahoo.com, outlook.com, etc.  
  - Logic: Checks domain against a predefined personal domain list and removes matches.  
  - Inputs: From "Code".  
  - Outputs: To "Code1" node for duplicate removal.  
  - Edge Cases: New/unrecognized personal domains not in the list; emails with uncommon domains.

---

#### 2.4 Duplicate Removal

**Overview:**  
Filters out duplicate emails, keeping only the first occurrence of each unique email address.

**Nodes Involved:**  
- Code1

**Node Details:**

- **Code1 (remove duplicates)**  
  - Type: Code (JavaScript)  
  - Role: Iterates through items, using a Set to keep track of seen email addresses and only passes unique emails onward.  
  - Inputs: From "Code2".  
  - Outputs: To "Append or update row in sheet" node.  
  - Edge Cases: Case sensitivity handled by lowercasing emails in prior nodes; possible null or missing emails filtered out earlier.

---

#### 2.5 Data Storage

**Overview:**  
Appends new or updates existing rows in a Google Sheets spreadsheet with lead data, keyed on email address to prevent duplicates.

**Nodes Involved:**  
- Append or update row in sheet

**Node Details:**

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Inserts or updates lead data into a specified sheet within a Google Sheets document.  
  - Config:  
    - Operation: appendOrUpdate  
    - Spreadsheet ID and Sheet Name are user-supplied placeholders (to be replaced).  
    - Columns: company_name, email, domain, subject, date_received.  
  - Credentials: Google Sheets OAuth2.  
  - Inputs: From "Code1".  
  - Outputs: To "Send a message" node.  
  - Edge Cases: Invalid spreadsheet ID, permission issues, API quotas, schema mismatches.

---

#### 2.6 Notification

**Overview:**  
Sends a Slack notification to a specified channel about the new lead added.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - Type: Slack node  
  - Role: Sends a message to a Slack channel with the company name and email of the lead.  
  - Config:  
    - Text template: "New Lead added {{company_name}} | {{email}}"  
    - Channel ID: placeholder to be replaced with actual Slack channel ID  
  - Credentials: Slack API credentials.  
  - Inputs: From "Append or update row in sheet".  
  - Outputs: None (end node).  
  - Edge Cases: Slack API rate limits, invalid channel ID, expired credentials.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                       | Input Node(s)                 | Output Node(s)               | Sticky Note                                         |
|---------------------------|---------------------|------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------|
| Manual Trigger (Historical Run) | Manual Trigger      | Manual start for historical email fetch | None                         | Get many messages           | ## Run email fetching for the existing emails       |
| Get many messages          | Gmail               | Fetch up to 500 emails historically | Manual Trigger (Historical Run) | Code                        | ## Run email fetching periodically default is set to daily |
| Gmail Trigger (Real-time)  | Gmail Trigger       | Scheduled daily trigger at 8 AM     | None                         | Code                        | ## Run email fetching periodically default is set to daily |
| Code                      | Code                | Parse email fields and filter system/generic emails | Get many messages / Gmail Trigger (Real-time) | Code2                       | ## Filter out the service emails (i.e. info@company.com) |
| Code2                     | Code                | Filter out personal email domains   | Code                         | Code1                       | ## Filter out the Personal emails (i.e. name@gmail.com) |
| Code1                     | Code                | Remove duplicate email leads        | Code2                        | Append or update row in sheet | ## Remove Duplicates                                  |
| Append or update row in sheet | Google Sheets       | Append or update lead rows in sheet | Code1                        | Send a message              | ## Store leads Into Google Sheet                      |
| Send a message            | Slack               | Notify Slack channel of new lead    | Append or update row in sheet | None                       | ## Notify in Slack channel                            |
| Sticky Note               | Sticky Note         | Visual comment                      | None                         | None                       | ## Notify in Slack channel                            |
| Sticky Note1              | Sticky Note         | Visual comment                      | None                         | None                       | ## Store leads Into Google Sheet                      |
| Sticky Note2              | Sticky Note         | Visual comment                      | None                         | None                       | ## Filter out the service emails (i.e. info@company.com) |
| Sticky Note3              | Sticky Note         | Visual comment                      | None                         | None                       | ## Run email fetching periodically default is set to daily |
| Sticky Note4              | Sticky Note         | Visual comment                      | None                         | None                       | ## Run email fetching for the existing emails         |
| Sticky Note5              | Sticky Note         | Visual comment                      | None                         | None                       | ## Remove Duplicates                                  |
| Sticky Note6              | Sticky Note         | Visual comment                      | None                         | None                       | ## Filter out the Personal emails (i.e. name@gmail.com) |
| Sticky Note7              | Sticky Note         | Detailed workflow description       | None                         | None                       | ## Reverse Outreach\n\nThis workflow allows users to extract potential leads sitting on their inboxes. The idea of a reverse outreach is based on the notion that the next big client/customer/partner might be sitting in your inbox waiting to be mined. This automation has two workflows, one that extracts from the historical emails, and the other is a scheduled event, default set to run everyday morning.\nThe workflow intelligently filters out emails from personal domains, system addresses (no-reply, updates), and generic company inboxes (info@, support@). The remaining emails are parsed to extract key information—company name, email address, domain, and subject—which is then stored in a Google Sheets spreadsheet. The Google Sheets node is configured to append or update based on the email address, ensuring that you never store duplicate entries. Finally, you will get a slack message with key information about the lead.\n\n🚀 **How it works**\nManual Trigger: A manual click initiate will fetch all historical emails. the limit set to 500, which you can increase up to 5000\n\nPeriodic Trigger: The workflow is triggered by time, default is set to daily fetch.\n\nCode nodes: Three Code nodes filter the emails based on custom rules - personal domains, system addresses, generic inboxes.\n\nGoogle Sheets: The processed data is sent to a Google Sheets spreadsheet. The append or update operation automatically handles whether to create a new row or update an existing one based on the email address, preventing duplicates.\n\n🔑 **Required Credentials**\nGoogle (Gmail): To access your Gmail account and retrieve email messages.\n\nGoogle Sheets: To connect to your spreadsheet.\n\nSlack Bot: To Send message in a designated slack channel\n\n🛠️ **Setup Instructions**\nConfigure Gmail Trigger:\n\nConnect your Google cloud account credential in Gmail and Google sheet nodes.\n\nChoose the schedule to run the Email fetch node that periodically watch for new emails.\n\n\nConfigure Code Node:\n\nThis node is pre-configured with the filtering logic. You can customize the lists of personal, blocked, or generic email parts to fit your needs.\n\nConfigure Google Sheets Node:\n\nConnect your Google Sheets credentials.\nCreate a Spreadsheet with the following columns\ncompany_name, email, domain, subject, date_received\n\nEnter the Spreadsheet ID of your target spreadsheet in the Google sheet node along with the Sheet Name (e.g., Leads).\n\nAnd that should do it! Now run the manual trigger workflow  and see the lead information showing up in your selected Slack Channel and also in the populated google sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "Manual Trigger (Historical Run)"  
   - No parameters required.

2. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Name: "Gmail Trigger (Real-time)"  
   - Set poll time to trigger daily at 8:00 AM (hour: 8).  
   - Connect Gmail OAuth2 credentials.  
   - Leave filters empty to fetch all new emails.

3. **Create Gmail Node to Fetch Historical Emails**  
   - Type: Gmail  
   - Name: "Get many messages"  
   - Operation: "getAll"  
   - Limit: 500 emails (adjustable up to 5000)  
   - Connect Gmail OAuth2 credentials.

4. **Connect Nodes:**  
   - Connect "Manual Trigger (Historical Run)" output to "Get many messages" input.  
   - Connect "Gmail Trigger (Real-time)" output directly to next processing node (Code).

5. **Create First Code Node ("Code") for Parsing and Filtering System/Generic Emails**  
   - Type: Code (JavaScript)  
   - Name: "Code"  
   - Paste the provided JavaScript code which:  
     - Extracts company name and email from "From" field.  
     - Filters out system/service emails by local part (e.g., no-reply, support).  
     - Formats the received date as a readable string.  
   - Input: Connect from "Get many messages" (historical) and "Gmail Trigger (Real-time)" (real-time).

6. **Create Second Code Node ("Code2") for Filtering Personal Emails**  
   - Type: Code (JavaScript)  
   - Name: "Code2"  
   - Paste JavaScript to filter out emails from personal domains like gmail.com, yahoo.com, etc.

7. **Create Third Code Node ("Code1") for Removing Duplicates**  
   - Type: Code (JavaScript)  
   - Name: "Code1"  
   - Paste JavaScript to remove duplicate emails by keeping first occurrence.

8. **Create Google Sheets Node ("Append or update row in sheet")**  
   - Type: Google Sheets  
   - Name: "Append or update row in sheet"  
   - Operation: Append or Update  
   - Configure Spreadsheet ID and Sheet Name (replace placeholders).  
   - Map columns: company_name, email, domain, subject, date_received.  
   - Connect Google Sheets OAuth2 credentials.

9. **Create Slack Node ("Send a message")**  
   - Type: Slack  
   - Name: "Send a message"  
   - Configure message text: `New Lead added {{$json["company_name"]}} | {{$json["email"]}}`  
   - Set channel ID (replace placeholder with actual Slack channel ID).  
   - Connect Slack API credentials.

10. **Connect Nodes in Order:**  
    - "Code" → "Code2" → "Code1" → "Append or update row in sheet" → "Send a message"

11. **Add Sticky Notes (Optional):**  
    - Add textual sticky notes above each logical block for clarity if desired.

12. **Credentials Setup:**  
    - Gmail OAuth2 (Google Cloud project with Gmail API enabled)  
    - Google Sheets OAuth2 (Google Cloud project with Sheets API enabled)  
    - Slack API (Slack app with permissions to post messages to channel)

13. **Run and Test:**  
    - Run the manual trigger to fetch historical emails.  
    - Verify new leads appear in Google Sheets and Slack notifications are sent.  
    - Confirm scheduled Gmail trigger runs daily as configured.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is designed to extract valuable business leads from your inbox by filtering out non-business and generic emails. It helps sales and marketing teams automate lead generation without manual effort.                                                                                                                                                                                                                                                                                                       | Workflow purpose                                                                                          |
| The appendOrUpdate operation in Google Sheets prevents duplicate leads by using the email as a unique key. Ensure your spreadsheet columns exactly match the expected fields: `company_name`, `email`, `domain`, `subject`, `date_received`.                                                                                                                                                                                                                                                                            | Google Sheets schema requirement                                                                          |
| Slack notifications use a placeholder channel ID `REPLACE_WITH_SLACK_CHANNEL_ID`. Replace it with your actual channel ID to receive notifications.                                                                                                                                                                                                                                                                                                                                                                  | Slack configuration                                                                                        |
| Gmail API limits and OAuth token refresh handling are critical for uninterrupted operation. Monitor API quotas especially for high volumes of emails.                                                                                                                                                                                                                                                                                                                                                                | Gmail API best practices                                                                                   |
| The JavaScript code nodes can be customized with additional filters or domain lists to suit specific business needs.                                                                                                                                                                                                                                                                                                                                                                                                  | Customization guidance                                                                                      |
| For more detailed understanding of Gmail triggers and Google Sheets nodes in n8n, refer to official documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.gmail-trigger/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.google-sheets/                                                                                                                                                                                                                                    | Official n8n documentation                                                                                 |
| Workflow contains two entry points: manual trigger (historical fetch) and scheduled Gmail trigger (real-time fetch). Both converge to the same processing path for consistent filtering and storage.                                                                                                                                                                                                                                                                                                                    | Workflow design notes                                                                                       |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, respecting all content policies. It contains no illegal or protected data. All processed data are public and legal.