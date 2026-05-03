Send Personalized Occasion Wishes with AI, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-occasion-wishes-with-ai--google-sheets-and-gmail-11143


# Send Personalized Occasion Wishes with AI, Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates sending personalized occasion wishes (e.g., birthdays, anniversaries) daily by integrating Google Sheets, AI, and Gmail. It targets users who want to maintain close relationships by never missing special dates of friends, family, or professional contacts. The workflow runs every day at 8 AM, checks a Google Sheet for matching occasions based on the current date (month and day), generates heartfelt personalized messages using an AI agent, and sends emails accordingly.

Logical blocks include:

- **1.1 Scheduled Trigger & Date Retrieval:** Runs daily and obtains the current date.
- **1.2 AI-Based Occasion Matching & Message Generation:** Uses an AI agent with access to the Google Sheet to find today's occasions and generate personalized messages.
- **1.3 Output Formatting & Email Sending:** Parses AI output, filters occasions, and sends personalized emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Date Retrieval

**Overview:**  
This block triggers the workflow daily at 8 AM and fetches the current date for further processing.

**Nodes Involved:**  
- Every Day at 8 AM  
- Date & Time

**Node Details:**

- **Every Day at 8 AM**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 24 hours at approximately 8 AM.  
  - Configuration: Interval set to 24 hours; no complex schedule expressions.  
  - Inputs: None (trigger node).  
  - Outputs: Triggers the "Date & Time" node.  
  - Edge Cases: If n8n server downtime occurs at the scheduled time, the trigger may be missed; consider enabling "Execute on missed executions" if critical.  

- **Date & Time**  
  - Type: Date & Time  
  - Role: Retrieves current date information for use in subsequent nodes.  
  - Configuration: Default settings to output current date/time.  
  - Inputs: Trigger from "Every Day at 8 AM."  
  - Outputs: Passes current date to "AI Agent" node.  
  - Edge Cases: None significant; timezone settings on n8n instance affect output date/time.  

---

#### 1.2 AI-Based Occasion Matching & Message Generation

**Overview:**  
This block leverages an AI agent powered by LangChain and OpenAI GPT-4 to read data from a Google Sheet, identify if today matches any occasion (based on month and day), and generate personalized, relationship-aware messages with proper formatting.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Get row(s) in sheet in Google Sheets  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Central AI component that receives current date and Google Sheets data to identify occasions and generate personalized messages.  
  - Configuration:  
    - Input text prompt includes today's date and instructions for strict JSON output.  
    - System message defines agent identity, capabilities, responsibilities, message tone rules, and formatting constraints.  
    - Output parser enabled to parse structured JSON responses.  
    - Uses OpenAI Chat Model as language model.  
    - Uses Google Sheets node as external tool to read data.  
  - Inputs: Current date from "Date & Time" node, Google Sheets data via "Get row(s) in sheet in Google Sheets."  
  - Outputs: JSON containing list of occasions with fields like hasOccasion, email, subject, and message.  
  - Edge Cases:  
    - AI output may fail to parse due to unexpected formatting; mitigated by output parser and subsequent parsing code node.  
    - No occasions found returns a specific JSON flag to prevent unnecessary email sending.  
    - Rate limits or API errors from OpenAI or Google Sheets may cause failures.  
  - Version-Specific: Requires LangChain integration and OpenAI GPT-4 support (model "gpt-4.1-mini").  

- **OpenAI Chat Model**  
  - Type: Language Model (OpenAI)  
  - Role: Provides GPT-4 based chat completions to the AI Agent node.  
  - Configuration: Model set to "gpt-4.1-mini".  
  - Inputs: Receives AI prompt from AI Agent.  
  - Outputs: Returns AI-generated text to AI Agent.  
  - Credentials: Requires valid OpenAI API key with GPT-4 access.  
  - Edge Cases: API key issues, quota limits, or network errors can interrupt processing.  

- **Get row(s) in sheet in Google Sheets**  
  - Type: Google Sheets Tool  
  - Role: Fetches all rows from the configured Google Sheet containing occasions data.  
  - Configuration:  
    - Document ID set to specific Google Sheet.  
    - Sheet ID set to "gid=0" (first sheet).  
    - No filtering; retrieves all rows for AI to analyze.  
  - Inputs: None directly; used as an external tool by AI Agent node.  
  - Outputs: Provides sheet data to AI Agent.  
  - Credentials: Requires OAuth2 credentials linked to Google Sheets account.  
  - Edge Cases: Google API rate limits, permissions errors if OAuth token invalid or sheet access revoked.  

---

#### 1.3 Output Formatting & Email Sending

**Overview:**  
This block processes the AI agent’s JSON output, filters out cases where no occasion is found, and sends personalized emails via Gmail for each occasion identified.

**Nodes Involved:**  
- Code in JavaScript  
- Send a message  

**Node Details:**

- **Code in JavaScript**  
  - Type: Code Node (JavaScript)  
  - Role: Parses AI output JSON string, handles potential double-encoded JSON, filters out "no occasion" results, and prepares separate items for each occasion to send emails.  
  - Configuration:  
    - Custom JS code handling multiple parsing attempts for robustness.  
    - Returns empty array if no occasions found to stop workflow elegantly.  
    - Maps each occasion to individual n8n item with email, subject, and message JSON fields.  
  - Inputs: AI Agent output JSON string with occasion data.  
  - Outputs: One item per occasion to "Send a message" node.  
  - Edge Cases:  
    - Parsing errors if AI output is malformed or incomplete.  
    - Empty output leads to zero downstream emails, which is the intended behavior.  

- **Send a message**  
  - Type: Gmail Node  
  - Role: Sends personalized occasion emails to recipients.  
  - Configuration:  
    - Sends to recipient email from JSON output.  
    - Uses subject and message from AI-generated content.  
    - No additional options enabled (e.g., attachments).  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: Formatted occasion messages from "Code in JavaScript" node.  
  - Outputs: None (final node in flow).  
  - Credentials: Requires Gmail account OAuth2 credentials configured.  
  - Edge Cases:  
    - Email sending failures due to invalid addresses, quota exceeded, or OAuth token expiry.  
    - Ensure correct email formatting to avoid spam filters.  

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                                  | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                            |
|-------------------------------|-------------------------------|-------------------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Every Day at 8 AM              | Schedule Trigger              | Daily trigger at 8 AM                            | None                        | Date & Time                | ## 1. Trigger Everyday and fetch current date                                                                        |
| Date & Time                   | Date & Time                   | Fetches current date/time                        | Every Day at 8 AM           | AI Agent                   | ## 1. Trigger Everyday and fetch current date                                                                        |
| AI Agent                      | LangChain Agent               | Checks Google Sheet for occasions & generates messages | Date & Time, Google Sheets  | Code in JavaScript         | ## 2. AI Agent with Google Sheets resource                                                                             |
| OpenAI Chat Model             | Language Model (OpenAI)       | Provides GPT-4 based AI completions              | AI Agent                    | AI Agent                   | ## 2. AI Agent with Google Sheets resource                                                                             |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool           | Fetches all rows from Google Sheet               | None (used by AI Agent)     | AI Agent                   | ## 2. AI Agent with Google Sheets resource                                                                             |
| Code in JavaScript            | Code (JavaScript)             | Parses AI output and prepares email items        | AI Agent                    | Send a message             | ## 3. Format output from AI Agent and send email if found any event                                                   |
| Send a message                | Gmail                        | Sends personalized occasion emails               | Code in JavaScript          | None                       | ## 3. Format output from AI Agent and send email if found any event                                                   |
| Sticky Note                   | Sticky Note                  | Overview and setup instructions                   | None                        | None                       | ## Overview: Automated Occasion Wisher ...                                                                            |
| Sticky Note1                  | Sticky Note                  | Notes on trigger and date retrieval block         | None                        | None                       | ## 1. Trigger Everyday and fetch current date                                                                        |
| Sticky Note2                  | Sticky Note                  | Notes on AI agent and Google Sheets integration   | None                        | None                       | ## 2. AI Agent with Google Sheets resource                                                                             |
| Sticky Note3                  | Sticky Note                  | Notes on output formatting and email sending block | None                        | None                       | ## 3. Format output from AI Agent and send email if found any event                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Schedule Trigger" node:**
   - Name: "Every Day at 8 AM"
   - Type: Schedule Trigger
   - Parameters: Set interval to 24 hours (runs once per day)
   - Position: Leftmost node
   - No credentials needed.

3. **Add a "Date & Time" node:**
   - Name: "Date & Time"
   - Type: Date & Time
   - Parameters: Default (outputs current date/time)
   - Connect "Every Day at 8 AM" → "Date & Time"

4. **Add a "Google Sheets" node to fetch occasion data:**
   - Name: "Get row(s) in sheet in Google Sheets"
   - Type: Google Sheets Tool
   - Parameters:
     - Document ID: Your Google Sheet ID containing occasions
     - Sheet Name: Sheet with occasion data (e.g., "Sheet1", gid=0)
     - Retrieve all rows (no filter)
   - Credentials: Create or select Google Sheets OAuth2 credentials with read access to the sheet.
   - No direct connection; this node is configured as a tool for the AI Agent (explained next).

5. **Add an "OpenAI Chat Model" node:**
   - Name: "OpenAI Chat Model"
   - Type: LangChain OpenAI Chat Model
   - Parameters:
     - Model: Select "gpt-4.1-mini" or equivalent GPT-4 model
   - Credentials: Provide OpenAI API key with GPT-4 access.

6. **Add an "AI Agent" node (LangChain Agent):**
   - Name: "AI Agent"
   - Type: LangChain Agent
   - Parameters:
     - Text prompt: Include today's date variable and instructions for JSON output (use example from original workflow).
     - System message: Define agent's identity, responsibilities, output format, and tone rules (copy from original).
     - Enable Output Parser for JSON.
     - Set AI language model to "OpenAI Chat Model" node.
     - Set AI tool for Google Sheets node ("Get row(s) in sheet in Google Sheets").
   - Connect "Date & Time" → "AI Agent"
   - Connect "OpenAI Chat Model" → "AI Agent" (ai_languageModel)
   - Connect "Get row(s) in sheet in Google Sheets" → "AI Agent" (ai_tool)

7. **Add a "Code" node to parse AI output:**
   - Name: "Code in JavaScript"
   - Type: Code (JavaScript)
   - Parameters: Use the provided JS code that:
     - Parses AI JSON output
     - Handles potential double parsing
     - Filters out cases with no occasions
     - Outputs one item per occasion with email, subject, message fields.
   - Connect "AI Agent" → "Code in JavaScript"

8. **Add a "Gmail" node to send emails:**
   - Name: "Send a message"
   - Type: Gmail
   - Parameters:
     - Send To: Expression `{{$json["email"]}}`
     - Subject: Expression `{{$json["subject"]}}`
     - Message: Expression `{{$json["message"]}}`
   - Credentials: Configure Gmail OAuth2 credentials with send email permission.
   - Connect "Code in JavaScript" → "Send a message"

9. **Optionally add Sticky Notes for documentation clarity.**

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow uses advanced AI prompt engineering to guarantee strict JSON output for reliable parsing. | Prompt content is embedded in the AI Agent node’s system and user messages.                                        |
| Google Sheet must have columns: Name, Occasion_Date, Email, Occasion_Type, Relationship, Personal_Note. | Sheet ID and name configured in the Google Sheets node.                                                           |
| Personalization includes relationship-aware tone and ordinal suffixes (1st, 2nd, 3rd, etc.).            | Ensures messages feel authentic and suited to the recipient.                                                       |
| Workflow can be extended with logging, Slack alerts, or saving sent email history for auditing.          | Recommended for production environments.                                                                           |
| AI model version “gpt-4.1-mini” requires OpenAI GPT-4 API access; confirm quota and billing are set up.  | https://platform.openai.com/docs/models/gpt-4                                                                              |
| Gmail node requires OAuth2 credentials with Gmail API enabled and authorized scopes to send emails.     | https://developers.google.com/gmail/api/quickstart/nodejs                                                          |
| The workflow respects privacy by only sending emails when occasions exactly match the current date (MM-DD). | No bulk or generic messaging is sent.                                                                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.