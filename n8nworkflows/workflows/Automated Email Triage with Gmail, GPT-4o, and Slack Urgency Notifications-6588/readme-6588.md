Automated Email Triage with Gmail, GPT-4o, and Slack Urgency Notifications

https://n8nworkflows.xyz/workflows/automated-email-triage-with-gmail--gpt-4o--and-slack-urgency-notifications-6588


# Automated Email Triage with Gmail, GPT-4o, and Slack Urgency Notifications

---
### 1. Workflow Overview

This workflow automates email triage for a Gmail inbox by leveraging OpenAI’s GPT-4o model to analyze and classify emails, then uses Slack to notify about urgent messages and send daily digests. It integrates Gmail, Google Sheets, OpenAI, and Slack to streamline the founder’s inbox management, automatically labeling emails based on intent and urgency, logging data for tracking, and summarizing lower-priority messages in periodic Slack digests.

**Target Use Cases:**  
- Busy professionals needing automated email prioritization  
- Teams requiring Slack notifications for critical emails  
- Centralized tracking of email metadata in Google Sheets  
- Automated classification and labeling of Gmail messages using AI  

**Logical Blocks:**  
- **1.1 Trigger and Email Retrieval:** Scheduled trigger initiates workflow; fetch Gmail labels and recent emails  
- **1.2 AI Processing and Classification:** Use GPT-4o to classify each email into summary, urgency, category, and intent in JSON format  
- **1.3 Conditional Slack Alerts and Gmail Labeling:** Send Slack alerts for high urgency emails; apply Gmail labels based on AI classification  
- **1.4 Email Metadata Logging:** Append processed email data to two Google Sheets tabs for record-keeping and digest preparation  
- **1.5 Daily Digest Creation and Slack Notification:** Prepare and send a Slack daily digest for medium and low urgency emails received recently  

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Retrieval

- **Overview:**  
  This block initiates the workflow every 4 hours, retrieves all Gmail labels, then fetches all emails received in the last 4 hours.

- **Nodes Involved:**  
  - Schedule Trigger1  
  - Get many labels  
  - Get many messages  

- **Node Details:**  
  - **Schedule Trigger1**  
    - Type: Schedule Trigger (interval trigger)  
    - Configured to run every 4 hours automatically  
    - No inputs; triggers downstream nodes  
    - Failure modes: n8n scheduler downtime or misconfiguration  

  - **Get many labels**  
    - Type: Gmail node (operation: getAll labels)  
    - Retrieves all Gmail labels for the connected account to enable accurate labeling later  
    - Input: Trigger from Schedule Trigger  
    - Output: List of labels for reference  
    - Requires valid Gmail OAuth2 credentials  
    - Potential failure: Auth errors, rate limits  

  - **Get many messages**  
    - Type: Gmail node (operation: getAll messages)  
    - Filters emails received after current time minus 4 hours (dynamic expression)  
    - Returns all matching emails for processing  
    - Input: Output of "Get many labels"  
    - Output: List of recent email messages  
    - Credentials: Gmail OAuth2  
    - Edge cases: No emails in timeframe, Gmail API quota exceeded  

---

#### 1.2 AI Processing and Classification

- **Overview:**  
  Emails fetched are analyzed individually by GPT-4o to extract structured JSON data describing summary, urgency, category, and intent.

- **Nodes Involved:**  
  - Message a model1  
  - Calculate Intent (Code node)  

- **Node Details:**  
  - **Message a model1**  
    - Type: OpenAI LangChain node (model: gpt-4o-mini)  
    - Sends email subject and snippet to GPT-4o with prompt requesting JSON classification  
    - Returns raw JSON with fields: summary, urgency (High, Medium, Low), category (Investor, Customer, Support, Spam, Other), intent (e.g., To respond, FYI, Notification, etc.)  
    - Input: Each email item from "Get many messages"  
    - Output: GPT response per email  
    - Credentials: OpenAI API key  
    - Failure modes: API rate limits, malformed input, GPT output parsing errors  

  - **Calculate Intent**  
    - Type: Code node (JavaScript)  
    - Runs once per GPT response item, parses raw JSON output, handles formatting fixes (removes markdown, fixes quotes)  
    - Extracts classification fields, fetches corresponding Gmail label ID for the intent (labels named like "AI Agent/<Intent>")  
    - Returns structured object with summary, urgency, category, intent, message_id, label_id  
    - Input: GPT output from "Message a model1" and references to Gmail labels and messages  
    - Output: Parsed and enriched email classification data  
    - Edge cases: JSON parse failure returns error message and original raw GPT output  

---

#### 1.3 Conditional Slack Alerts and Gmail Labeling

- **Overview:**  
  Based on urgency, sends immediate Slack alerts for high urgency emails and applies appropriate Gmail labels to the messages.

- **Nodes Involved:**  
  - If1  
  - Send a message2 (Slack)  
  - Add label to message2 (Gmail)  
  - Add label to message3 (Gmail)  

- **Node Details:**  
  - **If1**  
    - Type: If node  
    - Condition: Checks if email urgency equals "High"  
    - Input: Output from "Calculate Intent"  
    - Outputs: True (high urgency), False (others)  
    - Edge cases: Case sensitivity in urgency string, missing urgency field  

  - **Send a message2**  
    - Type: Slack node (send message)  
    - Sends an alert message to a configured Slack channel with email details (From, Summary, Urgency, Category, Intent)  
    - Input: True path from If1 (high urgency emails)  
    - Requires Slack OAuth2 credentials  
    - Failure: Slack API errors, channel permissions  

  - **Add label to message2**  
    - Type: Gmail node (addLabels operation)  
    - Applies Gmail label corresponding to intent for high urgency emails  
    - Input: After Slack alert sent  
    - Uses label_id from "Calculate Intent" and message ID from "Get many messages"  
    - Credentials: Gmail OAuth2  

  - **Add label to message3**  
    - Type: Gmail node (addLabels operation)  
    - Applies Gmail label for non-high urgency emails (false path from If1)  
    - Input: False path from If1  
    - Credentials: Gmail OAuth2  

---

#### 1.4 Email Metadata Logging

- **Overview:**  
  Logs all processed email data into two Google Sheets tabs: one for full record and another for digest preparation.

- **Nodes Involved:**  
  - Append row in sheet2  
  - Wait  
  - Append row in sheet1  

- **Node Details:**  
  - **Append row in sheet2**  
    - Type: Google Sheets node (append operation)  
    - Appends email metadata (From, Intent, Summary, Urgency, Category, TimeStamp) to “Sheet2” tab  
    - Input: After Gmail labeling of messages (from "Add label to message2")  
    - Credentials: Google Sheets OAuth2  
    - Sheet: “N8N - Emails” document, tab ID for Sheet2  
    - Use: Stores latest batch for digest  

  - **Wait**  
    - Type: Wait node (delay)  
    - Pauses workflow for 30 seconds before next step to ensure Sheet2 data is updated  
    - Input: After appending to Sheet2  

  - **Append row in sheet1**  
    - Type: Google Sheets node (append operation)  
    - Appends same email metadata to “Sheet1” tab (full archive)  
    - Input: After “Add label to message3” (non-high urgency path)  
    - Credentials: Google Sheets OAuth2  
    - Sheet: “N8N - Emails” document, tab ID for Sheet1  

---

#### 1.5 Daily Digest Creation and Slack Notification

- **Overview:**  
  Retrieves recent emails from Sheet2, filters them by timestamp and urgency (medium or low), compiles a digest message, and sends it to Slack.

- **Nodes Involved:**  
  - Get row(s) from sheet2  
  - Filter timestamp (Code)  
  - Daily Digest Preparation (Code)  
  - Send a message3 (Slack)  

- **Node Details:**  
  - **Get row(s) from sheet2**  
    - Type: Google Sheets node (read operation)  
    - Fetches all rows from Sheet2 tab (latest batch)  
    - Input: After Wait node  
    - Credentials: Google Sheets OAuth2  

  - **Filter timestamp**  
    - Type: Code node (JavaScript)  
    - Filters rows within the past 0.5 hours (30 minutes) and urgency of low or medium only  
    - Returns filtered list for digest preparation  
    - Input: Rows from Sheet2  

  - **Daily Digest Preparation**  
    - Type: Code node (JavaScript)  
    - Separates filtered items into medium and low urgency groups  
    - Formats Slack message text with numbered summaries and metadata  
    - If no messages, returns “_No important messages today._”  
    - Input: Filtered emails from previous node  

  - **Send a message3**  
    - Type: Slack node (send message)  
    - Sends the daily digest message to the configured Slack channel with timestamp and formatted summaries  
    - Input: Output of Daily Digest Preparation  
    - Credentials: Slack OAuth2  

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                        | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                  |
|-----------------------|---------------------------|-------------------------------------|-------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1      | Schedule Trigger          | Initiates workflow every 4 hours    | —                       | Get many labels             | ✅ Steps to Use: Connect Gmail, Slack, Google Sheets, OpenAI credentials; create Gmail labels; set up sheets. |
| Get many labels        | Gmail                     | Fetches all Gmail labels             | Schedule Trigger1         | Get many messages           | Same as above                                                                                                |
| Get many messages      | Gmail                     | Fetch emails received in last 4h    | Get many labels           | Message a model1            | Same as above                                                                                                |
| Message a model1       | OpenAI LangChain          | Classify emails with GPT-4o          | Get many messages         | Calculate Intent            | Same as above                                                                                                |
| Calculate Intent       | Code                      | Parses GPT JSON, fetches label IDs   | Message a model1          | If1                        | Same as above                                                                                                |
| If1                   | If                        | Checks if urgency is High            | Calculate Intent          | Send a message2 (true); Add label to message3 (false) | Same as above                                                                                                |
| Send a message2        | Slack                     | Sends Slack alert for high urgency  | If1 (true)                | Add label to message2       | Same as above                                                                                                |
| Add label to message2  | Gmail                     | Adds Gmail label for high urgency   | Send a message2           | Append row in sheet2        | Same as above                                                                                                |
| Add label to message3  | Gmail                     | Adds Gmail label for non-high urgency| If1 (false)               | Append row in sheet1        | Same as above                                                                                                |
| Append row in sheet2   | Google Sheets             | Logs email to Sheet2 (digest data)  | Add label to message2     | Wait                       | Same as above                                                                                                |
| Wait                  | Wait                      | 30s delay before digest processing   | Append row in sheet2      | Get row(s) from sheet2      | Same as above                                                                                                |
| Get row(s) from sheet2 | Google Sheets             | Reads recent emails for digest       | Wait                      | Filter timestamp            | Same as above                                                                                                |
| Filter timestamp       | Code                      | Filters emails by timestamp & urgency| Get row(s) from sheet2    | Daily Digest Preparation    | Same as above                                                                                                |
| Daily Digest Preparation| Code                     | Formats digest message text          | Filter timestamp          | Send a message3             | Same as above                                                                                                |
| Send a message3        | Slack                     | Sends daily digest to Slack channel  | Daily Digest Preparation  | —                          | Same as above                                                                                                |
| Append row in sheet1   | Google Sheets             | Logs email to Sheet1 (full archive) | Add label to message3     | —                          | Same as above                                                                                                |
| Sticky Note            | Sticky Note               | Provides setup instructions          | —                         | —                          | ✅ Steps to Use: Connect accounts, create labels, set up sheets, configure Slack, run test, activate workflow  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run every 4 hours (hoursInterval: 4)  
   - No inputs  

3. **Add a Gmail node to get labels:**  
   - Type: Gmail  
   - Operation: getAll (resource: label)  
   - Connect input from Schedule Trigger  
   - Use Gmail OAuth2 credentials  

4. **Add a Gmail node to get recent emails:**  
   - Type: Gmail  
   - Operation: getAll (resource: message)  
   - Filter: receivedAfter = expression `={{ new Date(Date.now() - 4 * 60 * 60 * 1000).toISOString() }}`  
   - Return all matching messages  
   - Connect input from "Get many labels" output  
   - Use Gmail OAuth2 credentials  

5. **Add an OpenAI LangChain node for classification:**  
   - Model: gpt-4o-mini  
   - Input messages: prompt GPT with email Subject and snippet, requesting JSON classification with fields summary, urgency, category, intent  
   - Connect input from "Get many messages" output  
   - Use OpenAI API credentials  

6. **Add a Code node to parse GPT output ("Calculate Intent"):**  
   - Run once per item  
   - JavaScript code to clean GPT output, parse JSON, extract fields  
   - Lookup Gmail label ID matching `AI Agent/<Intent>`  
   - Connect input from OpenAI node  
   - Output structured data (summary, urgency, category, intent, message_id, label_id)  

7. **Add an If node ("If1") to check urgency:**  
   - Condition: urgency equals "High" (case sensitive)  
   - Connect input from "Calculate Intent" output  
   - True branch for high urgency  
   - False branch for others  

8. **On True branch:**  
   - Add Slack node ("Send a message2"):  
     - Send message to Slack channel with email details (From, Summary, Urgency, Category, Intent)  
     - Use Slack OAuth2 credentials  
   - Connect Slack node output to Gmail node ("Add label to message2"):  
     - Operation: addLabels  
     - Label IDs from "Calculate Intent" (label_id)  
     - Message ID from "Get many messages"  
     - Use Gmail OAuth2 credentials  
   - Connect Gmail node output to Google Sheets node ("Append row in sheet2"):  
     - Append row with email metadata (From, Intent, Summary, Urgency, Category, TimeStamp)  
     - Sheet2 tab of specified Google Sheet  
     - Use Google Sheets OAuth2 credentials  

9. **On False branch:**  
   - Add Gmail node ("Add label to message3"):  
     - Add label similarly for non-high urgency emails  
     - Use Gmail OAuth2 credentials  
   - Connect output to Google Sheets node ("Append row in sheet1"):  
     - Append row with email metadata to Sheet1 tab (full archive)  
     - Use Google Sheets OAuth2 credentials  

10. **Add Wait node:**  
    - Delay 30 seconds  
    - Connect input from "Append row in sheet2"  

11. **Add Google Sheets node ("Get row(s) from sheet2"):**  
    - Read all rows from Sheet2 tab  
    - Use Google Sheets OAuth2 credentials  
    - Connect input from Wait node  

12. **Add Code node ("Filter timestamp"):**  
    - Filter rows within last 0.5 hours and urgency low or medium  
    - Connect input from previous node  

13. **Add Code node ("Daily Digest Preparation"):**  
    - Separate filtered emails by urgency (medium, low) and format Slack digest message text  
    - Connect input from filter node  

14. **Add Slack node ("Send a message3"):**  
    - Send the compiled daily digest message to Slack channel  
    - Use Slack OAuth2 credentials  
    - Connect input from daily digest node  

15. **Add Sticky Note for instructions:**  
    - Content detailing setup steps: connecting credentials, creating Gmail labels (e.g., AI Agent/To Respond), setting up Google Sheets tabs with columns, configuring Slack channel, running tests, and activating workflow  

16. **Ensure all nodes have appropriate credentials configured: Gmail OAuth2, OpenAI API, Google Sheets OAuth2, Slack OAuth2.**

17. **Verify connections as per above flow.**

18. **Test workflow manually before activating scheduled runs.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Create Gmail labels named exactly like "AI Agent/To Respond", "AI Agent/Awaiting Reply", "AI Agent/Notification", "AI Agent/Marketing", etc. These labels are critical for the workflow to auto-tag emails based on GPT intent classification.       | Gmail labels setup                                                                                              |
| Google Sheets document should have two sheets/tabs: Sheet1 (for full archive) and Sheet2 (for latest batch digest). Columns required: From, Summary, Intent, Category, TimeStamp, Urgency — all as plain text columns.                             | Google Sheets setup                                                                                             |
| Slack channel ID must be set in Slack nodes to receive urgent alerts and daily digests. Use Slack OAuth2 credentials with appropriate permissions to post messages.                                                                             | Slack configuration                                                                                             |
| OpenAI API model used is "gpt-4o-mini". Prompt requests strict JSON output without markdown to ease parsing downstream.                                                                                                                       | OpenAI API usage                                                                                                |
| Workflow runs every 4 hours by default; adjust Schedule Trigger interval as needed for more or less frequent triage.                                                                                                                           | Schedule adjustment                                                                                            |
| Potential failure points include API rate limits (Gmail, OpenAI, Slack), malformed GPT JSON output, missing Gmail labels, and permission issues in Slack or Google Sheets. Implement error handling or monitoring as needed.                      | Error handling considerations                                                                                   |
| Workflow designed for busy startup founders or professionals needing automated inbox prioritization with AI assistance and Slack integration for alerts and summaries.                                                                        | Use case context                                                                                                |
| Slack digest message format groups medium urgency and low urgency emails separately and excludes marketing emails from low urgency digest.                                                                                                    | Digest message formatting                                                                                       |
| For detailed setup, see sticky note inside the workflow interface which provides step-by-step instructions and checklist to connect accounts and create necessary labels and sheets.                                                           | Sticky note inside workflow                                                                                      |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, strictly compliant with content policies, and does not contain any illegal, offensive, or protected material. All data handled is legal and public.