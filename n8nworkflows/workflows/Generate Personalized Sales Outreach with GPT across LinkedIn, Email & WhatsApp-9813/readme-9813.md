Generate Personalized Sales Outreach with GPT across LinkedIn, Email & WhatsApp

https://n8nworkflows.xyz/workflows/generate-personalized-sales-outreach-with-gpt-across-linkedin--email---whatsapp-9813


# Generate Personalized Sales Outreach with GPT across LinkedIn, Email & WhatsApp

### 1. Workflow Overview

This workflow automates the generation and dispatch of personalized sales outreach messages using AI across LinkedIn, Email, and WhatsApp. It targets sales and marketing professionals seeking to hyper-personalize outreach at scale by leveraging lead data from Google Sheets and AI-generated messaging, with manual approval and multi-channel sending integration.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Preparation:** Fetching lead data from Google Sheets and batching it for processing.
- **1.2 AI Personalization Engine:** Using OpenAI GPT via LangChain agent to generate personalized LinkedIn connection requests, email content, and WhatsApp messages based on lead data.
- **1.3 Output Cleaning and Data Persistence:** Parsing AI JSON output and updating the Google Sheet with personalized messages.
- **1.4 Manual Approval Gate:** Sending an approval request email to the user and conditionally proceeding based on approval.
- **1.5 Multi-Channel Sending:** Sending LinkedIn requests via Phantombuster API, emails via Gmail, WhatsApp messages, and Telegram notifications for monitoring.
- **1.6 Supporting Setup and Guidance:** Sticky notes and checklist nodes provide configuration instructions, best practices, and customization ideas.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Preparation

- **Overview:**  
  This block retrieves lead data from Google Sheets, splits it into batches to avoid API rate limits, and prepares each row for AI processing by packaging the full row JSON.

- **Nodes Involved:**  
  - Get row(s) in sheet4 (Google Sheets)  
  - Loop Over Items4 (SplitInBatches)  
  - Edit Fields2 (Set)

- **Node Details:**  
  - *Get row(s) in sheet4*  
    - Type: Google Sheets  
    - Role: Reads leads data from a specified Google Sheet and tab (sheetName).  
    - Configuration: Requires Google Sheets OAuth2 credentials. The sheet ID and tab (gid) must be set.  
    - Inputs: None (triggered externally or via upstream nodes)  
    - Outputs: Array of lead data rows.  
    - Edge Cases: Credential failures, incorrect Sheet ID or permissions, empty sheets.  
  - *Loop Over Items4*  
    - Type: SplitInBatches  
    - Role: Splits the lead data into smaller batches to process leads one at a time, mitigating API rate limits.  
    - Configuration: Default batch size (not explicitly set, defaults apply).  
    - Inputs: Lead data array.  
    - Outputs: Single lead items sequentially.  
    - Edge Cases: Large datasets may cause delays; batch size should be tuned.  
  - *Edit Fields2*  
    - Type: Set  
    - Role: Adds a new field "Fullrow" containing the entire JSON object of the lead row for AI input.  
    - Configuration: Creates a string assignment with the full lead JSON (`{{$json}}`).  
    - Inputs: Single lead JSON objects.  
    - Outputs: Lead JSON enriched with "Fullrow".  
    - Edge Cases: Expression evaluation issues if input data is malformed.

---

#### 2.2 AI Personalization Engine

- **Overview:**  
  Generates three personalized messages (LinkedIn request, email, WhatsApp) per lead using OpenAI GPT-4.1-mini model through a LangChain AI agent. The prompt uses all lead data for hyper-personalization.

- **Nodes Involved:**  
  - AI Agent (Data Analysis & Personalisation)2 (LangChain Agent)  
  - OpenAI Chat Model3 (OpenAI LM Node)

- **Node Details:**  
  - *AI Agent (Data Analysis & Personalisation)2*  
    - Type: LangChain Agent  
    - Role: Processes the full lead data JSON and generates personalized messages in JSON format.  
    - Configuration: Uses a detailed prompt instructing generation of three messages with specific style, length, and channels; placeholders for user name, title, email, LinkedIn URL to be customized by the user.  
    - Inputs: The enriched lead JSON with "Fullrow".  
    - Outputs: Raw JSON string with keys: connectionMessage, emailMessage, whatsappMessage.  
    - Version-Specific: Requires LangChain integration with n8n, OpenAI API key configured in credentials.  
    - Edge Cases: API rate limits, malformed input JSON, prompt errors, incomplete AI responses.  
  - *OpenAI Chat Model3*  
    - Type: OpenAI Chat Model (GPT)  
    - Role: Provides the underlying GPT-4.1-mini model for AI Agent node.  
    - Configuration: Model set to "gpt-4.1-mini"; user must provide OpenAI API key credential.  
    - Inputs: Text prompt from AI Agent node.  
    - Outputs: AI-generated text response.  
    - Edge Cases: API key invalid, network errors, model unavailability.

---

#### 2.3 Output Cleaning and Data Persistence

- **Overview:**  
  Parses the AI JSON output string into structured JSON fields and writes the personalized messages back into the respective Google Sheet row.

- **Nodes Involved:**  
  - Cleans up2 (Code)  
  - Update row in sheet2 (Google Sheets)

- **Node Details:**  
  - *Cleans up2*  
    - Type: Code (JavaScript)  
    - Role: Parses the AI-generated string output into JSON fields: connectionMessage, emailMessage, whatsappMessage.  
    - Configuration: Inline JS code that checks for "output" field and parses JSON safely.  
    - Inputs: AI Agent output containing JSON string.  
    - Outputs: Structured JSON with message fields.  
    - Edge Cases: Invalid JSON response, missing keys, parsing errors.  
  - *Update row in sheet2*  
    - Type: Google Sheets  
    - Role: Updates the same row in Google Sheet by matching "row_number", writing AI messages into "Connection", "AI Email", and "AI Whatsapp Message" columns.  
    - Configuration: Uses Google Sheets OAuth2 credentials; requires sheet ID, tab, and matching on "row_number".  
    - Inputs: Parsed AI messages and row identifier.  
    - Outputs: Confirmation of sheet update.  
    - Edge Cases: Sheet permission errors, concurrent edits, incorrect row_number.

---

#### 2.4 Manual Approval Gate

- **Overview:**  
  Sends an approval email to the user for reviewing AI-generated messages before sending. Checks user approval and conditionally proceeds with sending outreach.

- **Nodes Involved:**  
  - Send a message5 (Gmail)  
  - If3 (If)

- **Node Details:**  
  - *Send a message5*  
    - Type: Gmail  
    - Role: Sends an email containing a request for approval to the configured user email, with instructions to review the Google Sheet.  
    - Configuration: Gmail OAuth2 credentials required, subject and body instructing manual review, uses "sendAndWait" operation to wait for user response (approval).  
    - Inputs: Lead batch or individual lead data.  
    - Outputs: Approval status.  
    - Edge Cases: OAuth failure, email sending errors, no user response.  
  - *If3*  
    - Type: If  
    - Role: Checks if the user approved sending messages (`data.approved == true`).  
    - Configuration: Loose type validation, condition on JSON field from Gmail approval response.  
    - Inputs: Approval response JSON.  
    - Outputs: Proceeds only if approved; otherwise, halts.  
    - Edge Cases: Incorrect response format, false negatives.

---

#### 2.5 Multi-Channel Sending

- **Overview:**  
  Sends approved personalized messages through LinkedIn (via Phantombuster API), Gmail (email), WhatsApp, and sends Telegram notifications for monitoring.

- **Nodes Involved:**  
  - LinkedIn Requests3 (HTTP Request)  
  - Get row(s) in sheet5 (Google Sheets)  
  - Loop Over Items5 (SplitInBatches)  
  - Send a message6 (Gmail)  
  - Send message (WhatsApp)  
  - Send a text message2 (Telegram)

- **Node Details:**  
  - *LinkedIn Requests3*  
    - Type: HTTP Request  
    - Role: Launches a Phantombuster LinkedIn automation agent to send connection requests with personalized messages.  
    - Configuration: POST to Phantombuster API with API key and agent ID headers; user must replace placeholders YOUR_PHANTOMBUSTER_API_KEY and YOUR_AGENT_ID.  
    - Inputs: Triggered after approval.  
    - Outputs: API response from Phantombuster.  
    - Edge Cases: Phantombuster API limits, expired API keys, LinkedIn account issues.  
  - *Get row(s) in sheet5*  
    - Type: Google Sheets  
    - Role: Retrieves updated rows from Google Sheet post-approval for sending emails and messages.  
    - Configuration: Same as earlier Google Sheets nodes, requires credentials and sheet details.  
    - Inputs: Triggered after LinkedIn request node.  
    - Outputs: Lead rows with AI-generated messages.  
    - Edge Cases: Same as previous Google Sheet nodes.  
  - *Loop Over Items5*  
    - Type: SplitInBatches  
    - Role: Processes approved leads individually for sending email and notifications sequentially.  
    - Inputs: Lead rows from Google Sheets.  
    - Outputs: Single lead per iteration.  
    - Edge Cases: Large data sets, rate limits.  
  - *Send a message6*  
    - Type: Gmail  
    - Role: Sends personalized AI-generated HTML email to the lead's personal email address.  
    - Configuration: Uses Gmail OAuth2, subject dynamically includes company name, body is AI-generated HTML content.  
    - Inputs: Lead data including "AI Email".  
    - Outputs: Confirmation of email sent.  
    - Edge Cases: Email quota exceeded, invalid email addresses.  
  - *Send message*  
    - Type: WhatsApp  
    - Role: Sends personalized WhatsApp message using WhatsApp Business API credentials.  
    - Configuration: Requires WhatsApp API credentials configured in n8n.  
    - Inputs: Lead data including WhatsApp message content.  
    - Outputs: Confirmation of message sent.  
    - Edge Cases: WhatsApp API limits, invalid phone numbers.  
  - *Send a text message2*  
    - Type: Telegram  
    - Role: Sends connection messages as Telegram notifications for monitoring/logging purposes.  
    - Configuration: Requires Telegram Bot token and chat ID; sends message from AI connection message field.  
    - Inputs: Lead data with connection message.  
    - Outputs: Confirmation of Telegram message sent.  
    - Edge Cases: Invalid chat ID, bot token issues.

---

#### 2.6 Supporting Setup and Guidance

- **Overview:**  
  Sticky notes throughout the workflow provide detailed instructions on setup, configuration, best practices, customization ideas, and rate limit considerations to assist users in effectively deploying and maintaining the workflow.

- **Nodes Involved:**  
  - Sticky Note - Title  
  - Sticky Note 1 through Sticky Note 10

- **Node Details:**  
  - These are n8n Sticky Note nodes containing markdown-formatted guidance, checklists, and warnings.  
  - They do not process data but are essential for user onboarding and maintenance.  
  - Include links (e.g., Google Sheet URLs) and placeholders to be replaced by users.  
  - Cover steps such as Google Sheet setup, credentials, AI prompt customization, sending limits, and troubleshooting.

---

### 3. Summary Table

| Node Name                          | Node Type               | Functional Role                                       | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                                     |
|-----------------------------------|-------------------------|------------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Title                | Sticky Note             | Workflow title and branding                           |                             |                             | ## Personalised AI Sales Automation **LinkedIn, Email & Whatsapp**                                                             |
| Sticky Note 1                     | Sticky Note             | Instructions for Google Sheet setup                   |                             |                             | ## STEP 1: Setup Your Google Sheet ...                                                                                          |
| Sticky Note 2                     | Sticky Note             | Explanation of batching and data prep                 |                             |                             | ## STEP 2: Process Data in Batches ...                                                                                          |
| Sticky Note 3                     | Sticky Note             | AI prompt customization instructions                   |                             |                             | ## STEP 3: AI Personalization Engine ...                                                                                        |
| Sticky Note 4                     | Sticky Note             | Instructions on cleaning AI output and saving          |                             |                             | ## STEP 4: Clean & Save AI Output ...                                                                                           |
| Sticky Note 5                     | Sticky Note             | Manual approval process instructions                    |                             |                             | ## STEP 5: Manual Approval Gate ...                                                                                             |
| Sticky Note 6                     | Sticky Note             | LinkedIn sending setup instructions                     |                             |                             | ## STEP 6: Send LinkedIn Requests ...                                                                                           |
| Sticky Note 7                     | Sticky Note             | Email and notifications sending instructions            |                             |                             | ## STEP 7: Send Emails & Notifications ...                                                                                      |
| Sticky Note 8                     | Sticky Note             | Quick start checklist                                   |                             |                             | ## QUICK START CHECKLIST ...                                                                                                    |
| Sticky Note 9                     | Sticky Note             | Important notes on rate limits and best practices      |                             |                             | ## IMPORTANT NOTES ...                                                                                                          |
| Sticky Note 10                    | Sticky Note             | Customization ideas and alternative tools              |                             |                             | ## CUSTOMIZATION IDEAS ...                                                                                                      |
| Get row(s) in sheet4              | Google Sheets           | Fetch lead data from Google Sheet                       |                             | Loop Over Items4             |                                                                                                                                |
| Loop Over Items4                  | SplitInBatches          | Batch processing of leads                               | Get row(s) in sheet4         | Send a message5, Edit Fields2 |                                                                                                                                |
| Edit Fields2                     | Set                     | Add full lead row JSON for AI input                     | Loop Over Items4             | AI Agent (Data Analysis & Personalisation)2 |                                                                                                                                |
| AI Agent (Data Analysis & Personalisation)2 | LangChain Agent         | Generate personalized LinkedIn, Email, WhatsApp messages | Edit Fields2                | Cleans up2                  |                                                                                                                                |
| OpenAI Chat Model3               | OpenAI Chat Model       | GPT-4.1-mini language model used by AI Agent           | AI Agent                    | AI Agent                    |                                                                                                                                |
| Cleans up2                      | Code                    | Parse AI JSON string output into structured JSON       | AI Agent                    | Update row in sheet2        |                                                                                                                                |
| Update row in sheet2             | Google Sheets           | Write AI-generated messages back to Google Sheet       | Cleans up2                  | Loop Over Items4            |                                                                                                                                |
| Send a message5                 | Gmail                   | Send approval request email                             | Loop Over Items4             | If3                        |                                                                                                                                |
| If3                            | If                      | Check if user approved sending messages                 | Send a message5              | LinkedIn Requests3          |                                                                                                                                |
| LinkedIn Requests3               | HTTP Request            | Trigger Phantombuster LinkedIn connection requests     | If3                         | Get row(s) in sheet5        |                                                                                                                                |
| Get row(s) in sheet5            | Google Sheets           | Retrieve updated rows for sending emails/notifications | LinkedIn Requests3           | Loop Over Items5            |                                                                                                                                |
| Loop Over Items5                | SplitInBatches          | Batch process approved leads for sending                | Get row(s) in sheet5         | Send a message6, (empty)    |                                                                                                                                |
| Send a message6                 | Gmail                   | Send personalized AI-generated email                    | Loop Over Items5             | Send message                |                                                                                                                                |
| Send message                   | WhatsApp                | Send personalized WhatsApp message                       | Send a message6              | Send a text message2        |                                                                                                                                |
| Send a text message2            | Telegram                | Send message to Telegram for monitoring                  | Send message                 | Loop Over Items5            |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Credentials:**  
   - Setup OAuth2 credentials for Google Sheets in n8n.

2. **Create Gmail Credentials:**  
   - Setup OAuth2 credentials for Gmail in n8n.

3. **Create OpenAI Credentials:**  
   - Add OpenAI API key credential with access to GPT-4.1-mini or equivalent.

4. **Create Telegram Bot Credentials (optional):**  
   - Create Telegram bot with @BotFather and get bot token.  
   - Add Telegram credentials in n8n.

5. **Create WhatsApp API Credentials (optional):**  
   - Configure WhatsApp Business API credentials in n8n.

6. **Create HTTP Request Credentials for Phantombuster (optional):**  
   - Obtain Phantombuster API key and agent ID; no specific credential node needed but store securely.

7. **Add Google Sheets Node ("Get row(s) in sheet4"):**  
   - Operation: Read rows from your Google Sheet with lead data.  
   - Configure documentId and sheetName with your sheet ID and tab (gid).

8. **Add SplitInBatches Node ("Loop Over Items4"):**  
   - Connect from Google Sheets node.  
   - No batch size set explicitly (defaults to 1).  
   - Purpose: Process leads one by one.

9. **Add Set Node ("Edit Fields2"):**  
   - Connect from SplitInBatches.  
   - Add field "Fullrow" with value set to the entire JSON (`{{$json}}`).

10. **Add LangChain AI Agent Node ("AI Agent (Data Analysis & Personalisation)2"):**  
    - Connect from Set node.  
    - Configure prompt with the detailed personalization instructions.  
    - Replace placeholders [YOUR_NAME], [YOUR_TITLE], [YOUR_EMAIL], [YOUR_LINKEDIN_URL] with your own details.  
    - Link to OpenAI Chat Model Node.

11. **Add OpenAI Chat Model Node ("OpenAI Chat Model3"):**  
    - Select model "gpt-4.1-mini" or preferred GPT model.  
    - Ensure OpenAI credentials are linked.

12. **Add Code Node ("Cleans up2"):**  
    - Connect from AI Agent node.  
    - Paste JavaScript code to parse AI output JSON string into structured fields.

13. **Add Google Sheets Node ("Update row in sheet2"):**  
    - Connect from Code node.  
    - Operation: Update row in Google Sheet.  
    - Map "Connection", "AI Email", "AI Whatsapp Message" columns from parsed AI output.  
    - Use "row_number" to identify the row to update.

14. **Connect "Update row in sheet2" back to "Loop Over Items4":**  
    - Completes the batch processing loop.

15. **Add Gmail Node ("Send a message5") for Approval Request:**  
    - Connect from "Loop Over Items4" initial output (parallel branch).  
    - Configure recipient email, subject, and message content asking for manual approval.  
    - Use "Send and Wait" operation to capture approval response.

16. **Add If Node ("If3") to check approval:**  
    - Connect from Gmail approval node.  
    - Condition: Proceed only if `data.approved == true`.

17. **Add HTTP Request Node ("LinkedIn Requests3"):**  
    - Connect from If node "true" branch.  
    - Configure POST request to Phantombuster API with headers: API key and agent ID.  
    - Add query parameters as needed.

18. **Add Google Sheets Node ("Get row(s) in sheet5"):**  
    - Connect from LinkedIn Requests node.  
    - Reads updated sheet data to proceed with email and messaging.

19. **Add SplitInBatches Node ("Loop Over Items5"):**  
    - Connect from Google Sheets node.  
    - Processes each approved lead individually.

20. **Add Gmail Node ("Send a message6") to send emails:**  
    - Connect from SplitInBatches node.  
    - Configure recipient as lead’s personal email.  
    - Set subject dynamically including company name.  
    - Use AI-generated HTML email content as body.

21. **Add WhatsApp Node ("Send message"):**  
    - Connect from Gmail send node.  
    - Configure with WhatsApp Business API credentials.  
    - Send WhatsApp message from AI output.

22. **Add Telegram Node ("Send a text message2") for notifications:**  
    - Connect from WhatsApp send node.  
    - Configure with Telegram bot token and chat ID.  
    - Send connection message for monitoring.

23. **Add Sticky Note nodes throughout:**  
    - Add as documentation and setup guidance, including steps for Sheet setup, credential configuration, AI prompt customization, and limits.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Google Sheet must include columns: First Name, Last Name, Title, Company Name, Personal Email, etc. | Setup instructions in Sticky Note 1                     |
| AI prompt placeholders: replace [YOUR_NAME], [YOUR_TITLE], [YOUR_EMAIL], [YOUR_LINKEDIN_URL]         | Sticky Note 3                                           |
| Phantombuster API requires signup and setup for LinkedIn automation                                 | Sticky Note 6                                           |
| Gmail API has sending limits (100-500 emails/day)                                                   | Sticky Note 9                                           |
| OpenAI API rate limits vary (~3-10 requests/min)                                                   | Sticky Note 9                                           |
| Start with small batches (5-10 leads) to test and avoid rate limits                                 | Sticky Note 9                                           |
| Telegram bot setup requires @BotFather and chat ID                                                  | Sticky Note 7                                           |
| WhatsApp Business API credentials required for WhatsApp node                                       | Configuration note in WhatsApp node                     |
| Customization suggestions include A/B testing, CRM integration, delays, and alternate sending tools | Sticky Note 10                                          |
| Troubleshooting tips: verify credentials, permissions, and test each node individually              | Sticky Note 9                                           |

---

**Disclaimer:** The text above is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.