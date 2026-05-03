Score Website Leads with GPT-4o, Respond via Gmail, and Notify via Telegram

https://n8nworkflows.xyz/workflows/score-website-leads-with-gpt-4o--respond-via-gmail--and-notify-via-telegram-10586


# Score Website Leads with GPT-4o, Respond via Gmail, and Notify via Telegram

### 1. Workflow Overview

This workflow automates lead scoring and response for website form submissions using AI (GPT-4o-mini), Google Sheets, Gmail, and Telegram. It targets sales and marketing teams who want to efficiently qualify and engage inbound leads by automating data extraction, AI-driven lead scoring, database storage, team notifications, and personalized auto-reply emails.

**Logical Blocks:**

- **1.1 Input Reception**: Receive and parse website form submissions via webhook.
- **1.2 Data Extraction**: Extract individual form fields and priority score from submission.
- **1.3 AI Lead Scoring**: Use GPT-4o-mini to assign a priority emoji score to the lead based on extracted data.
- **1.4 Data Aggregation**: Merge form data with AI-generated lead score.
- **1.5 Data Persistence**: Save complete lead data into Google Sheets.
- **1.6 Notifications**: Notify the sales team on Telegram with lead details.
- **1.7 Auto-Response**: Send personalized confirmation emails to prospects via Gmail.
- **1.8 Supporting Utilities**: Include memory node for AI context and sticky notes documenting steps and configuration.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives incoming lead data via HTTP POST webhook from the website form.

- **Nodes Involved:**  
  - Website Form Submission

- **Node Details:**  
  - **Website Form Submission**  
    - Type: Webhook  
    - Configuration: HTTP POST path `website-form-submissions` to receive form submission payloads.  
    - Key Expression: Payload accessed in downstream nodes via `$json.body.data.fields`.  
    - Input: External POST Requests  
    - Output: JSON payload of form data  
    - Edge Cases: Missing fields, malformed POST, webhook URL misconfiguration, unauthorized requests (security is optional).  
    - Sticky Note: "Step 1 Details" explains webhook setup and integration tips.

#### 1.2 Data Extraction

- **Overview:**  
Extracts all relevant form fields and the priority score from the raw webhook payload into structured variables for further processing.

- **Nodes Involved:**  
  - Extract Form Fields  
  - Extract Priority Score

- **Node Details:**  
  - **Extract Form Fields**  
    - Type: Set Node  
    - Configuration: Maps each form field from the webhook JSON array to named variables (e.g., First Name, Last Name, Company Name, Industry, Phone, Email, Request Type, Specific Request).  
    - Expressions: Uses `$json.body.data.fields[index].value` to extract each field.  
    - Input: Website Form Submission node output  
    - Output: Structured form data JSON  
    - Edge Cases: Index out-of-bounds if form structure changes, missing or empty fields.  
    - Sticky Note: "Step 2 Details" explains extraction logic and field mapping.  

  - **Extract Priority Score**  
    - Type: Set Node  
    - Configuration: Extracts the lead priority scoring value specifically from field index 8 (assumed to be a scoring field).  
    - Expression: `$json.body.data.fields[8].value[0]` for the score value.  
    - Input: Website Form Submission node output  
    - Output: JSON with only the "Scoring" field  
    - Edge Cases: Missing or malformed scoring field, unexpected value type.  
    - Sticky Note: "Step 3 Details" explains the reason for separating score extraction.

#### 1.3 AI Lead Scoring

- **Overview:**  
Assigns a lead priority level emoji based on the extracted scoring value using GPT-4o-mini AI model with a structured prompt.

- **Nodes Involved:**  
  - Window Buffer Memory  
  - OpenAI Chat Model  
  - AI Lead Scoring Agent

- **Node Details:**  
  - **Window Buffer Memory**  
    - Type: Memory Buffer Window (Langchain)  
    - Configuration: Uses the form submission eventId as a session key to maintain context per lead.  
    - Input: None (receives from webhook indirectly)  
    - Output: Memory context passed to AI Agent  
    - Edge Cases: Session key missing or duplicated, memory overflow in long sessions.  

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model (gpt-4o-mini)  
    - Configuration: Model set to "gpt-4o-mini" for cost-effective AI inference.  
    - Input: Connected to AI Lead Scoring Agent node.  
    - Output: AI generated text (emoji).  
    - Edge Cases: API key issues, rate limits, model unavailability.  

  - **AI Lead Scoring Agent**  
    - Type: Langchain AI Agent  
    - Configuration:  
      - Task: Analyze lead priority based on scoring value and return a single emoji representing priority level (⚪️, 🟢, 🔵, 🟣, 🔴).  
      - Input Data: Passes extracted scoring value.  
      - Scoring System: Matches scoring value to one of five priority levels with exact ID matching from form options to ensure accuracy.  
      - Output: Single emoji only, no extra text or formatting.  
    - Input: Priority score from "Extract Priority Score" node and memory context.  
    - Output: Emoji string as lead score.  
    - Edge Cases: Unexpected scoring values, AI returning malformed output, latency or timeout issues.  
    - Sticky Note: "Step 4 Details" explains AI scoring logic and prompt design.

#### 1.4 Data Aggregation

- **Overview:**  
Combines the extracted form data with the AI-generated lead score into a single data object for downstream use.

- **Nodes Involved:**  
  - Combine Form Data with Score

- **Node Details:**  
  - **Combine Form Data with Score**  
    - Type: Merge Node  
    - Configuration: "Combine" mode, merging data from two branches: form fields and AI score.  
    - Input: Outputs from "Extract Form Fields" and "AI Lead Scoring Agent"  
    - Output: Complete lead profile including contact info, request details, and lead score emoji.  
    - Edge Cases: Data mismatch if one branch fails or delays, merge conflicts.  
    - Sticky Note: "Step 5 Details" explains merge purpose and output structure.

#### 1.5 Data Persistence

- **Overview:**  
Stores the enriched lead data, including AI score, into a Google Sheets document as a lead database.

- **Nodes Involved:**  
  - Save to Lead Database

- **Node Details:**  
  - **Save to Lead Database**  
    - Type: Google Sheets node  
    - Configuration: Appends new rows to "Sheet1" in specified Google Sheet (replace `YOUR_GOOGLE_SHEET_ID`).  
    - Columns mapped explicitly: Email, Phone, Company, Industry, Last Name, First Name, Lead Score (emoji), Request Type, Specific Request.  
    - Input: Combined lead data from previous merge node.  
    - Output: Confirmation of append operation.  
    - Edge Cases: Google API authentication errors, quota limits, incorrect Sheet ID, data type mismatches, duplicate entries (matching on Email column).  
    - Sticky Note: "Step 6 Details" gives setup instructions and best practices.

#### 1.6 Notifications

- **Overview:**  
Sends an instant Telegram message to the sales team with full lead details including AI lead score.

- **Nodes Involved:**  
  - Notify Sales Team

- **Node Details:**  
  - **Notify Sales Team**  
    - Type: Telegram node  
    - Configuration: Sends message to specified Telegram chat ID (`YOUR_TELEGRAM_CHAT_ID`), with a formatted message containing name, company, industry, contact info, request details, and lead score emoji.  
    - Input: Output from the merged lead data node.  
    - Output: Telegram message send status.  
    - Edge Cases: Invalid chat ID, Telegram bot token issues, network errors, message length limits.  
    - Sticky Note: "Step 7 Details" includes setup steps for Telegram bot and chat ID retrieval.

#### 1.7 Auto-Response

- **Overview:**  
Delivers a personalized email reply to the lead acknowledging receipt of their inquiry and setting expectations.

- **Nodes Involved:**  
  - Prepare Email Data  
  - Send Auto-Reply Email

- **Node Details:**  
  - **Prepare Email Data**  
    - Type: Set Node  
    - Configuration: Extracts and assigns variables needed for email: First Name, Email, Request Type, Company Name.  
    - Input: Output of Google Sheets node (lead data saved).  
    - Output: Cleaned variables for email node.  
    - Edge Cases: Missing email or name fields, data propagation delays.

  - **Send Auto-Reply Email**  
    - Type: Gmail node  
    - Configuration:  
      - Sends email to the extracted Email address.  
      - Subject: "Thank you for your inquiry!"  
      - Message: Personalized text with first name, request type, company name, and 24h response promise.  
      - Email type: plain text.  
      - Credential: Gmail OAuth2 configured for sending.  
    - Input: Data from "Prepare Email Data" node.  
    - Output: Email sent confirmation.  
    - Edge Cases: Gmail API limits, invalid email addresses, auth failures.  
    - Sticky Note: "Step 8 Details" explains email personalization and alternatives.

#### 1.8 Supporting Utilities and Documentation

- **Overview:**  
Includes sticky notes documenting workflow overview, each step details, configuration checklist, benefits, and customization ideas.

- **Nodes Involved:**  
  - Workflow Overview (sticky note)  
  - Step 1 to Step 8 Details (sticky notes)  
  - Configuration Checklist (sticky note)  
  - Workflow Benefits (sticky note)  
  - Customization Ideas (sticky note)

- **Node Details:**  
  - All are `stickyNote` nodes purely for documentation and guidance within the n8n editor.  
  - Positioned logically near their relevant nodes.  
  - Contain links to official n8n docs for each node type.  
  - Provide setup tips, best practices, and advanced ideas.  
  - No inputs or outputs; no runtime effect.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role               | Input Node(s)             | Output Node(s)                    | Sticky Note                                                                                               |
|-------------------------|---------------------------------|------------------------------|---------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------|
| Website Form Submission  | Webhook                         | Receive form submission       | External POST request      | Extract Form Fields, Extract Priority Score | Step 1 Details: Webhook setup and integration tips [https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/] |
| Extract Form Fields      | Set                             | Parse form fields             | Website Form Submission    | Combine Form Data with Score      | Step 2 Details: Extract field mappings [https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/] |
| Extract Priority Score   | Set                             | Parse lead scoring field      | Website Form Submission    | AI Lead Scoring Agent             | Step 3 Details: Separate scoring extraction for clarity                                                  |
| Window Buffer Memory     | Langchain Memory Buffer Window  | Maintain AI session context   | -                         | AI Lead Scoring Agent             |                                                                                                          |
| OpenAI Chat Model        | Langchain OpenAI Chat Model     | GPT-4o-mini AI inference      | AI Lead Scoring Agent      | AI Lead Scoring Agent             |                                                                                                          |
| AI Lead Scoring Agent    | Langchain AI Agent              | Generate lead score emoji     | Extract Priority Score, Window Buffer Memory | Combine Form Data with Score      | Step 4 Details: AI scoring logic and prompt design [https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/] |
| Combine Form Data with Score | Merge                       | Merge form data & AI score    | Extract Form Fields, AI Lead Scoring Agent | Notify Sales Team, Save to Lead Database | Step 5 Details: Merge result for full lead profile                                                        |
| Save to Lead Database    | Google Sheets                   | Store lead data               | Combine Form Data with Score | Prepare Email Data               | Step 6 Details: Google Sheets setup and mapping [https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/] |
| Prepare Email Data       | Set                             | Prepare variables for email   | Save to Lead Database      | Send Auto-Reply Email             |                                                                                                          |
| Send Auto-Reply Email    | Gmail                           | Send personalized confirmation email | Prepare Email Data         | -                               | Step 8 Details: Gmail node email customization [https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/] |
| Notify Sales Team        | Telegram                        | Send Telegram notification    | Combine Form Data with Score | -                               | Step 7 Details: Telegram bot setup and message details [https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/] |
| Workflow Overview        | Sticky Note                    | Documentation overview        | -                         | -                               | Overview of complete workflow and use cases                                                             |
| Step 1 Details           | Sticky Note                    | Webhook documentation         | -                         | -                               | Webhook node explanation                                                                                  |
| Step 2 Details           | Sticky Note                    | Field extraction documentation | -                         | -                               | Set node field extraction details                                                                         |
| Step 3 Details           | Sticky Note                    | Priority score extraction docs | -                         | -                               | Explains separate score extraction                                                                        |
| Step 4 Details           | Sticky Note                    | AI scoring documentation      | -                         | -                               | Details on AI agent and scoring logic                                                                     |
| Step 5 Details           | Sticky Note                    | Data merge explanation        | -                         | -                               | Merge node purpose                                                                                        |
| Step 6 Details           | Sticky Note                    | Google Sheets setup           | -                         | -                               | Google Sheets node configuration tips                                                                     |
| Step 7 Details           | Sticky Note                    | Telegram notification setup   | -                         | -                               | Telegram bot and chat ID instructions                                                                     |
| Step 8 Details           | Sticky Note                    | Gmail auto-reply setup        | -                         | -                               | Email personalization and alternatives                                                                    |
| Configuration Checklist  | Sticky Note                    | Setup checklist               | -                         | -                               | Checklist to replace placeholders and configure credentials                                               |
| Workflow Benefits        | Sticky Note                    | Workflow advantages           | -                         | -                               | Benefits and ROI summary                                                                                   |
| Customization Ideas      | Sticky Note                    | Advanced options and integrations | -                         | -                               | Suggestions for extending and customizing workflow                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Website Form Submission":**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `website-form-submissions`  
   - No authentication (optional to secure)  
   - Purpose: Receive incoming form submissions from website.

2. **Add Set Node "Extract Form Fields":**  
   - Extract and assign variables from webhook payload:  
     - First Name: `{{$json.body.data.fields[0].value}}`  
     - Last Name: `{{$json.body.data.fields[1].value}}`  
     - Company Name: `{{$json.body.data.fields[2].value}}`  
     - Industry: `{{$json.body.data.fields[3].value}}`  
     - Phone: `{{$json.body.data.fields[4].value}}`  
     - Email: `{{$json.body.data.fields[5].value}}`  
     - Request Type: `{{$json.body.data.fields[6].value}}`  
     - Specific Request: `{{$json.body.data.fields[7].value}}`  
   - Connect input from "Website Form Submission".

3. **Add Set Node "Extract Priority Score":**  
   - Extract scoring value:  
     - Scoring: `{{$json.body.data.fields[8].value[0]}}`  
   - Connect input from "Website Form Submission".

4. **Add Langchain Node "Window Buffer Memory":**  
   - Session Key: `{{$json.body.eventId}}` (custom key)  
   - Purpose: Maintain per-lead AI session context.

5. **Add Langchain Node "OpenAI Chat Model":**  
   - Model: `gpt-4o-mini`  
   - Connect memory output to AI agent.

6. **Add Langchain Node "AI Lead Scoring Agent":**  
   - Task: Analyze scoring value and return exactly one emoji representing the priority.  
   - Input variables: Scoring value extracted.  
   - Use prompt with defined scoring criteria and ID matching (see logic in original node).  
   - Connect inputs from "Extract Priority Score" and "Window Buffer Memory".  
   - Connect output to "OpenAI Chat Model".

7. **Add Merge Node "Combine Form Data with Score":**  
   - Mode: Combine  
   - Input 1: "Extract Form Fields" output  
   - Input 2: "AI Lead Scoring Agent" output  
   - Output: Single combined JSON with all form data plus lead score emoji.

8. **Add Google Sheets Node "Save to Lead Database":**  
   - Operation: Append  
   - Sheet Name: `Sheet1`  
   - Map columns: Email, Phone, Company, Industry, Last Name, First Name, Lead Score (emoji), Request Type, Specific Request  
   - Document ID: Replace with your Google Sheet ID  
   - Credentials: Google Sheets OAuth2  
   - Input: "Combine Form Data with Score" output.

9. **Add Set Node "Prepare Email Data":**  
   - Assign:  
     - First Name: `{{$json['First Name']}}`  
     - Email: `{{$json.Email}}`  
     - Request Type: `{{$json['Request Type']}}`  
     - Company Name: `{{$json['Company Name']}}`  
   - Input: "Save to Lead Database" output.

10. **Add Gmail Node "Send Auto-Reply Email":**  
    - Send To: `{{$json.Email}}`  
    - Subject: "Thank you for your inquiry!"  
    - Message (plain text):  
      ```
      Hello {{$json['First Name']}},

      Thank you for reaching out to us!

      We've received your inquiry regarding:
      "{{$json['Request Type']}}"

      Our team is already reviewing your request and one of our experts will contact you within 24 hours.

      We appreciate {{$json['Company Name']}}'s interest in our AI automation services.

      Best regards,
      The Team
      ```
    - Credential: Gmail OAuth2 configured for sending emails  
    - Input: "Prepare Email Data" output.

11. **Add Telegram Node "Notify Sales Team":**  
    - Chat ID: Replace `YOUR_TELEGRAM_CHAT_ID` with your Telegram chat ID  
    - Text:  
      ```
      🔔 New Lead Submission!

      👤 Name: {{$json['First Name']}} {{$json['Last Name']}}
      🏢 Company: {{$json['Company Name']}}
      🏭 Industry: {{$json['Industry']}}
      📞 Phone: {{$json['Phone']}}
      📧 Email: {{$json['Email']}}

      📝 Request Type:
      {{$json['Request Type']}}

      💬 Specific Request:
      {{$json['Specific Request']}}

      ⭐ Lead Score: {{$json.output}}

      Time to follow up! 🚀
      ```
    - Credential: Telegram bot API token  
    - Input: "Combine Form Data with Score" output.

12. **Connect Nodes in Order:**  
    - "Website Form Submission" → "Extract Form Fields"  
    - "Website Form Submission" → "Extract Priority Score"  
    - "Extract Priority Score" → "AI Lead Scoring Agent" (with memory node in parallel)  
    - "Window Buffer Memory" → "AI Lead Scoring Agent"  
    - "AI Lead Scoring Agent" → "OpenAI Chat Model"  
    - "Extract Form Fields" + "AI Lead Scoring Agent" → "Combine Form Data with Score"  
    - "Combine Form Data with Score" → "Notify Sales Team" and "Save to Lead Database"  
    - "Save to Lead Database" → "Prepare Email Data" → "Send Auto-Reply Email"

13. **Configure Credentials:**  
    - OpenAI API key with GPT-4o-mini access  
    - Google Sheets OAuth2 credentials for your spreadsheet  
    - Gmail OAuth2 credentials with send email permissions  
    - Telegram Bot token and chat ID for notifications

14. **Final Steps:**  
    - Replace all placeholders (`YOUR_GOOGLE_SHEET_ID`, `YOUR_TELEGRAM_CHAT_ID`) with real values  
    - Adjust field indices if your form structure differs  
    - Test with sample submissions and monitor logs for errors  
    - Activate the workflow

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                      | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates lead qualification with AI scoring and multi-channel notifications, saving 2-3 hours daily on manual lead processing.                                                                                                                                                   | Workflow Benefits sticky note                                                                    |
| Setup requires Google Sheet with proper columns, Telegram bot via @BotFather, Gmail credentials, and OpenAI API key with GPT-4o-mini model access.                                                                                                                                               | Configuration Checklist sticky note                                                             |
| Learn more about webhook node and payload handling: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/                                                                                                                                                                 | Step 1 Details sticky note                                                                       |
| How to extract and map fields using Set node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/                                                                                                                                                                           | Step 2 Details sticky note                                                                       |
| AI agent prompt design and Langchain AI integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/                                                                                                                                               | Step 4 Details sticky note                                                                       |
| Google Sheets node configuration and best practices: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                                                                                                            | Step 6 Details sticky note                                                                       |
| Telegram node setup and bot creation guide: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/                                                                                                                                                                         | Step 7 Details sticky note                                                                       |
| Gmail node for sending emails with OAuth2: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                                                                                                             | Step 8 Details sticky note                                                                       |
| Advanced customization ideas include routing VIP leads to different workflows, CRM integrations, SMS alerts, automated meeting booking, and lead enrichment APIs.                                                                                                                               | Customization Ideas sticky note                                                                 |
| Prompt structure enforces returning only the emoji for scoring to avoid AI output ambiguity—key for stable automation.                                                                                                                                                                          | AI Lead Scoring Agent node description and Step 4 Details sticky note                            |
| Workflow is scalable and designed for unlimited form submissions with consistent AI scoring, immediate engagement, and audit trail storage.                                                                                                                                                     | Workflow Benefits sticky note                                                                    |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._