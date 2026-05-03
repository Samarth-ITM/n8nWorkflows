Cold Lead Re-Engagement Email Generator with GPT-4o-mini, Outlook, and Sheets

https://n8nworkflows.xyz/workflows/cold-lead-re-engagement-email-generator-with-gpt-4o-mini--outlook--and-sheets-6919


# Cold Lead Re-Engagement Email Generator with GPT-4o-mini, Outlook, and Sheets

### 1. Workflow Overview

This workflow automates the generation of personalized re-engagement emails for cold leads by leveraging data from Google Sheets, historical email conversations from Microsoft Outlook, and AI-powered content creation using OpenAI's GPT-4o-mini model. It is designed for sales or marketing teams aiming to revive inactive contacts through strategic, AI-crafted outreach.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Retrieve a list of stale leads (emails) from a Google Sheets spreadsheet.
- **1.2 Historical Email Retrieval**: For each lead, query Microsoft Outlook to collect all past emails sent by that lead since a specified date.
- **1.3 Data Aggregation & Transformation**: Aggregate and convert the historical emails into a single text string suitable for AI processing.
- **1.4 AI Processing**: Use OpenAI's GPT-4o-mini model to analyze the email history, summarize past communications, suggest re-engagement strategies, and draft a personalized email.
- **1.5 Output Handling**: Save the AI-generated summary and email drafts back to Google Sheets and create draft emails in Outlook for manual review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block fetches the list of stale leads from a Google Sheets document. It serves as the source data for the workflow.

**Nodes Involved:**  
- Start Workflow (Manual Trigger)  
- Get Stale Leads (Google Sheets)

**Node Details:**

- **Start Workflow**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow  
  - Configuration: No parameters; triggers workflow execution  
  - Inputs: None  
  - Outputs: Triggers the next node ("Get Stale Leads")  
  - Edge Cases: User must manually trigger; no automatic scheduling  
  - Version: 1  

- **Get Stale Leads**  
  - Type: Google Sheets  
  - Role: Reads lead email addresses from a specified Google Sheets document  
  - Configuration:  
    - Document ID: Points to a lead re-engagement spreadsheet  
    - Sheet Name: "Sheet1" (default tab)  
    - OAuth2 credential configured for Google Sheets access  
  - Key Expressions: None specifically; reads raw rows  
  - Inputs: Triggered by "Start Workflow"  
  - Outputs: Emits each lead’s data (must include an `Email` field)  
  - Edge Cases:  
    - Requires valid Google Sheets OAuth2 credentials  
    - Spreadsheet must contain an `Email` column  
    - If sheet is empty or missing, workflow fails or returns no leads  
  - Version: 4.6  

---

#### 2.2 Historical Email Retrieval

**Overview:**  
For each lead obtained, this block queries Microsoft Outlook to retrieve all emails sent by the lead since January 1, 2025.

**Nodes Involved:**  
- Get all emails from the lead (Microsoft Outlook)

**Node Details:**

- **Get all emails from the lead**  
  - Type: Microsoft Outlook  
  - Role: Fetches all emails received from the lead since 2025-01-01  
  - Configuration:  
    - Operation: `getAll`  
    - Filters:  
      - `receivedDateTime ge 2025-01-01T00:00:00Z` (emails from 2025 onwards)  
      - `sender = {{ $json.Email }}` (dynamic filter using lead email)  
    - Return All: true  
    - Output: raw email data  
    - OAuth2 credential for Microsoft Outlook configured  
  - Key Expressions: Uses dynamic filter to inject lead email from Google Sheets node  
  - Inputs: Receives lead data from "Get Stale Leads"  
  - Outputs: Emits emails per lead  
  - Edge Cases:  
    - Authentication errors if Outlook OAuth2 token expired or invalid  
    - API rate limits or timeouts possible if many emails  
    - If no emails found, output empty, which must be handled downstream  
  - Version: 2  

---

#### 2.3 Data Aggregation & Transformation

**Overview:**  
Aggregates the relevant email fields and converts the aggregated data into a single text string for AI consumption.

**Nodes Involved:**  
- Combine into one field (Aggregate)  
- Convert object fields to text (Code)

**Node Details:**

- **Combine into one field**  
  - Type: Aggregate  
  - Role: Combines all email records’ fields into a single aggregated data set  
  - Configuration:  
    - Aggregate Mode: `aggregateAllItemData`  
    - Fields Included: `subject`, `body`, `createdDateTime`  
  - Inputs: Emails from "Get all emails from the lead"  
  - Outputs: Aggregated email data  
  - Edge Cases: If input items are empty, aggregation outputs empty result  
  - Version: 1  

- **Convert object fields to text**  
  - Type: Code (JavaScript)  
  - Role: Converts aggregated email objects into a single concatenated JSON string  
  - Code Summary:  
    - Maps each item’s JSON to stringified JSON  
    - Joins with newline characters  
  - Inputs: Aggregated data from "Combine into one field"  
  - Outputs: Single JSON object with `text` property containing all email info as string  
  - Edge Cases:  
    - If input is empty, output text will be empty string  
    - Potential JSON stringify errors if data malformed (rare)  
  - Version: 2  

---

#### 2.4 AI Processing

**Overview:**  
Uses OpenAI GPT-4o-mini to analyze the email history, summarize, suggest re-engagement strategies, and draft an email. This block parses the structured JSON response from the AI.

**Nodes Involved:**  
- OpenAI Chat Model (OpenAI GPT-4o-mini)  
- AI Agent - Re-Engage Lead (Langchain Agent)  
- Structured Output Parser (Langchain Output Parser Structured)

**Node Details:**

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (via Langchain integration)  
  - Role: Provides the language model endpoint for AI processing  
  - Configuration:  
    - Model: `gpt-4o-mini` (lightweight GPT-4 variant)  
    - No additional model options specified  
    - OpenAI API credentials configured (valid API key)  
  - Inputs: Connected to AI Agent node as language model provider  
  - Outputs: Provides AI-generated text to agent node  
  - Edge Cases:  
    - API quota limits or key invalidation may cause failures  
    - Model output latency or timeouts possible  
  - Version: 1.2  

- **AI Agent - Re-Engage Lead**  
  - Type: Langchain Agent Node  
  - Role: Receives aggregated email text and runs the prompt to:  
    1. Summarize prior communications  
    2. Suggest 2–3 re-engagement strategies  
    3. Draft a professional re-engagement email  
  - Configuration:  
    - Prompt includes system message specifying detailed instructions and JSON output format with keys: `summary`, `how to re-engage`, `email subject`, `email body`  
    - Input text dynamically injected as recent emails from lead  
    - Output parsing enabled to expect structured JSON  
  - Inputs: Receives combined text from "Convert object fields to text"  
  - Outputs: Structured JSON output with summary, suggestions, and email draft  
  - Edge Cases:  
    - AI may produce malformed JSON if prompt misunderstood; structured parser may fail  
    - Model errors or rate limits  
  - Version: 2  

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI agent JSON output into structured data fields for downstream nodes  
  - Configuration:  
    - JSON schema example provided to guide parsing for keys: `summary`, `how to re-engage`, `email subject`, `email body`  
  - Inputs: Connected to output of OpenAI Chat Model node  
  - Outputs: Parsed structured JSON to AI Agent node input (feedback loop for parsing)  
  - Edge Cases: If AI output does not match schema, parsing errors may occur  
  - Version: 1.2  

---

#### 2.5 Output Handling

**Overview:**  
Saves the AI-generated data back to Google Sheets and creates draft emails in Outlook for each lead.

**Nodes Involved:**  
- Save data to Sheets (Google Sheets)  
- Create draft email (Microsoft Outlook)

**Node Details:**

- **Save data to Sheets**  
  - Type: Google Sheets  
  - Role: Writes AI outputs (summary, re-engagement tips, email subject/body) back to the spreadsheet, updating rows by lead email  
  - Configuration:  
    - Operation: Append or update existing rows based on matching `Email` column  
    - Columns mapped:  
      - `Email` (from original lead)  
      - `Summary` (AI summary)  
      - `How Can I Re-Engage` (AI suggestions)  
      - `Email Subject` (draft subject)  
      - `Email Body` (draft body)  
    - Sheet: "Sheet1" in the same spreadsheet as input  
    - Uses Google Sheets OAuth2 credential  
  - Inputs: AI Agent output with enriched data and original lead email from "Get Stale Leads"  
  - Outputs: None (final)  
  - Edge Cases:  
    - Credential issues or spreadsheet access problems  
    - Data overwrite conflicts if multiple workflow runs overlap  
  - Version: 4.6  

- **Create draft email**  
  - Type: Microsoft Outlook  
  - Role: Creates a draft email in Outlook for manual review and sending  
  - Configuration:  
    - Resource: `draft` (email draft creation)  
    - To Recipients: Lead’s email from Google Sheets node  
    - Subject: AI generated email subject  
    - Body Content: AI generated email body  
    - OAuth2 credential for Outlook configured  
  - Inputs: AI Agent output with email content and lead email  
  - Outputs: None (side-effect action)  
  - Edge Cases:  
    - OAuth2 token expiration or API rate limits  
    - Draft creation failure due to malformed email body or subject from AI  
  - Version: 2  

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                                     | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
|----------------------------|---------------------------------------|----------------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start Workflow             | Manual Trigger                        | Manual start trigger of the workflow                | None                       | Get Stale Leads               |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Get Stale Leads            | Google Sheets                        | Fetch stale lead emails from Google Sheets          | Start Workflow             | Get all emails from the lead  | ### 📥 Step 1 — Pull Email List from Google Sheets<br>1. Copy the sample spreadsheet [Click to copy](https://docs.google.com/spreadsheets/d/1rQD493GNtTWms6GF0Wracu9Yrm0AR0jxwaWdv8eJbUM/copy). It must include a column named `Email`.<br>2. Configure Google Sheets OAuth2 credentials.<br>3. Select the sheet and sheet name in the node.<br>                                                                                                                                                                                        |
| Get all emails from the lead | Microsoft Outlook                  | Retrieve all emails sent by the lead since 2025     | Get Stale Leads            | Combine into one field         | ### 📬 Step 2 — Search Outlook for Past Conversations<br>1. Create Microsoft Outlook OAuth2 credential.<br>2. Set operation to `getAll` with filters for date and sender.<br>3. Return all emails sent by each lead since 2025.<br>---<br>### 📊 Step 3 — Aggregate Email Content<br>Use an Aggregate node and Code node to combine email content.<br>                                                                                                                                                                                |
| Combine into one field     | Aggregate                           | Aggregate email fields (subject, body, date)         | Get all emails from the lead | Convert object fields to text  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Convert object fields to text | Code                              | Convert aggregated email objects into single text    | Combine into one field      | AI Agent - Re-Engage Lead      | ### 🧠 Step 4 — Use OpenAI to Summarize & Suggest Re-Engagement Strategy<br>1. Add OpenAI API key and use the `OpenAI Chat Model` node with `gpt-4o-mini`.<br>2. Use a Code node to prepare aggregated content as single text block.<br>                                                                                                                                                                                                                                             |
| OpenAI Chat Model          | OpenAI Chat Model (Langchain LM)    | Provide GPT-4o-mini model interface for AI processing | Connected internally to AI Agent node | AI Agent - Re-Engage Lead (via langchain interface) |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Structured Output Parser   | Langchain Output Parser Structured  | Parse AI JSON output into structured fields           | OpenAI Chat Model          | AI Agent - Re-Engage Lead      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| AI Agent - Re-Engage Lead | Langchain Agent                    | Analyze emails, summarize, suggest strategy, draft email | Convert object fields to text, Structured Output Parser, OpenAI Chat Model | Create draft email, Save data to Sheets |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Save data to Sheets        | Google Sheets                      | Append or update AI-generated data back to Sheets    | AI Agent - Re-Engage Lead  | None                         |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Create draft email         | Microsoft Outlook                  | Create draft re-engagement email in Outlook          | AI Agent - Re-Engage Lead  | None                         | ### 📨 Step 6 — Create a Draft Email in Outlook<br>1. Add Microsoft Outlook node.<br>2. Set Resource to `draft`.<br>3. Map To, Subject, Body from AI output and lead email.<br>4. Connect Outlook OAuth2 credential.<br>💡 Creates draft email ready for manual review and sending.                                                                                                                                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node to serve as the workflow entry point.

2. **Add Google Sheets Node "Get Stale Leads"**  
   - Type: Google Sheets  
   - Operation: Read rows from spreadsheet  
   - Configure Credentials: Google Sheets OAuth2 with access to your leads spreadsheet  
   - Set Document ID: Link to your copied lead re-engagement Google Sheet  
   - Set Sheet Name: `Sheet1` (or your target sheet)  
   - Connect output of "Manual Trigger" to this node.

3. **Add Microsoft Outlook Node "Get all emails from the lead"**  
   - Type: Microsoft Outlook  
   - Operation: `getAll`  
   - Filters:  
     - Custom filter: `receivedDateTime ge 2025-01-01T00:00:00Z`  
     - Sender filter: `={{ $json.Email }}` (inject lead email dynamically)  
   - Return All: True  
   - Configure Credentials: Microsoft Outlook OAuth2  
   - Connect output of "Get Stale Leads" to this node.

4. **Add Aggregate Node "Combine into one field"**  
   - Type: Aggregate  
   - Aggregate Mode: `aggregateAllItemData`  
   - Fields to Include: `subject`, `body`, `createdDateTime`  
   - Connect output of "Get all emails from the lead" here.

5. **Add Code Node "Convert object fields to text"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return [{
       json: {
         text: items.map(item => JSON.stringify(item.json)).join('\n'),
       },
     }];
     ```  
   - Connect output of "Combine into one field" to this node.

6. **Add OpenAI Chat Model Node "OpenAI Chat Model"**  
   - Type: OpenAI Chat Model via Langchain integration  
   - Model: `gpt-4o-mini`  
   - Configure OpenAI API Key credential  
   - Connect this node internally to the "AI Agent - Re-Engage Lead" node (see next step).

7. **Add Langchain Agent Node "AI Agent - Re-Engage Lead"**  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     You are a helpful assistant trained to analyze prior email conversations for lead re-engagement.

     Given a list of recent emails from a contact, perform the following:

     1. Summarize what the person has communicated in the past. Focus on the main topics, interests, concerns, or any open threads.
     2. Suggest 2–3 strategic ideas for how I can re-engage this lead. Format your suggestions as bullet points.
     3. Write a professional, friendly re-engagement email I can send them. Make the tone approachable but business-relevant.

     Return your output in the following JSON structure:

     {
       "summary": "Concise summary of their past emails",
       "how to re-engage": "• First suggestion\n• Second suggestion\n• Third suggestion",
       "email subject": "Suggested subject line",
       "email body": "Complete draft email body"
     }
     ```
   - Enable Output Parsing with a structured output parser node (add next step).  
   - Connect input of this node to output of "Convert object fields to text".

8. **Add Langchain Structured Output Parser Node "Structured Output Parser"**  
   - Type: Langchain Output Parser Structured  
   - JSON Schema Example (to parse AI output):  
     ```json
     {
       "summary": "summary",
       "how to re-engage": "how to re-engage", 
       "email subject": "email subject", 
       "email body": "Email Body"
     }
     ```  
   - Connect output of "OpenAI Chat Model" to this node.  
   - Connect output of this parser back to the "AI Agent - Re-Engage Lead" node’s input parser.

9. **Add Google Sheets Node "Save data to Sheets"**  
   - Type: Google Sheets  
   - Operation: Append or update row  
   - Mapping Columns:  
     - `Email` → `={{ $('Get Stale Leads').item.json.Email }}`  
     - `Summary` → `={{ $json.output.summary }}`  
     - `How Can I Re-Engage` → `={{ $json.output['how to re-engage'] }}`  
     - `Email Subject` → `={{ $json.output['email subject'] }}`  
     - `Email Body` → `={{ $json.output['email body'] }}`  
   - Configure Google Sheets OAuth2 credentials  
   - Connect output of "AI Agent - Re-Engage Lead" to this node.

10. **Add Microsoft Outlook Node "Create draft email"**  
    - Type: Microsoft Outlook  
    - Resource: `draft`  
    - Subject: `={{ $json.output['email subject'] }}`  
    - Body Content: `={{ $json.output['email body'] }}`  
    - To Recipients: `={{ $('Get Stale Leads').item.json.Email }}`  
    - Configure Microsoft Outlook OAuth2 credentials  
    - Connect output of "AI Agent - Re-Engage Lead" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| The Google Sheets sample spreadsheet can be copied from: [Lead Re-Engagement Sample Sheet](https://docs.google.com/spreadsheets/d/1rQD493GNtTWms6GF0Wracu9Yrm0AR0jxwaWdv8eJbUM/copy) — must contain an `Email` column for workflow input.                                                                                                                                                                                                                  | Input spreadsheet source for lead emails                                                                       |
| For Outlook, create and configure Microsoft Outlook OAuth2 credentials in n8n before running the workflow.                                                                                                                                                                                                                                                                                                                                                                                  | Microsoft Outlook integration                                                                                   |
| Use OpenAI GPT-4o-mini or higher models with valid API keys for effective summarization and email generation.                                                                                                                                                                                                                                                                                                                                                                               | OpenAI API usage                                                                                                |
| The workflow creates draft emails in Outlook for manual review before sending, supporting a human-in-the-loop approach to outbound email campaigns.                                                                                                                                                                                                                                                                                                                                      | Draft email creation step                                                                                        |
| Contact for workflow support and consulting: Robert Breen — [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/) — robert@ynteractive.com                                                                                                                                                                                                                                                                                                                                        | Author contact and professional support                                                                         |

---

This completes the comprehensive reference for the "Cold Lead Re-Engagement Email Generator with GPT-4o-mini, Outlook, and Sheets" n8n workflow.