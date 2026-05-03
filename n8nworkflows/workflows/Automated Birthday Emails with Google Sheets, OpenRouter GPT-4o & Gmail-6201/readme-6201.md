Automated Birthday Emails with Google Sheets, OpenRouter GPT-4o & Gmail

https://n8nworkflows.xyz/workflows/automated-birthday-emails-with-google-sheets--openrouter-gpt-4o---gmail-6201


# Automated Birthday Emails with Google Sheets, OpenRouter GPT-4o & Gmail

### 1. Workflow Overview

This workflow automates sending personalized birthday emails by integrating Google Sheets, OpenRouter GPT-4o language model, and Gmail. It is designed to run daily at a specific hour, fetch birthday data from a Google Sheet, filter contacts whose birthday matches the current date, generate a warm, personalized birthday email using AI, and send the email through Gmail.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 9 AM.
- **1.2 Data Retrieval:** Reads birthday and contact data from a Google Sheet.
- **1.3 Filtering:** Selects only those rows where the date of birth matches today’s date.
- **1.4 AI Email Composition:** Uses OpenRouter GPT-4o via LangChain agent to compose a personalized birthday message.
- **1.5 Email Sending:** Sends the generated email via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Starts the workflow automatically every day at 9 AM to check for birthdays.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**
  - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)
  - Configuration: Executes once daily at 9:00 AM (triggerAtHour: 9).
  - Inputs: None (trigger node).
  - Outputs: Triggers the "Get row(s) in sheet" node.
  - Edge Cases: Workflow will not run outside scheduled time; if the node fails, no daily execution will occur.
  - Version: 1.2
  - No sub-workflows.

#### 1.2 Data Retrieval

- **Overview:** Fetches all rows from the specified Google Sheet containing birthday data.
- **Nodes Involved:** Get row(s) in sheet
- **Node Details:**
  - Type: Google Sheets node (n8n-nodes-base.googleSheets)
  - Configuration:
    - Document ID: Google Sheet with birthday data (ID: `1fTGWkzIMdq1G3v2R3tRsRWIgEByPBeKsKXIJTcsxWwk`).
    - Sheet Name: Default first sheet (`gid=0`).
    - Operation: Get rows (default, all rows).
  - Credentials: Google Sheets OAuth2 credential connected.
  - Inputs: Triggered by Schedule Trigger.
  - Outputs: Rows passed to Filter node.
  - Edge Cases:
    - Authentication failure with Google OAuth.
    - Sheet ID or sheet name mismatch.
    - Empty or malformed rows.
  - Version: 4.6
  - No sub-workflows.

#### 1.3 Filtering

- **Overview:** Filters rows to keep only those whose date of birth matches today’s date (day and month).
- **Nodes Involved:** Filter
- **Node Details:**
  - Type: Filter node (n8n-nodes-base.filter)
  - Configuration:
    - Condition: Compares the date of birth (DOB) field in each row to current date.
    - DOB extraction:  
      Expression: `{{$json.DOB.split('/').slice(0,2).join('-')}}`  
      This transforms a DOB from `dd/mm/yyyy` format into `dd-mm` string.
    - Current date formatted as `dd-MM`: `{{$now.toFormat("dd-MM")}}`.
    - The filter matches rows where DOB day and month equal today’s day and month.
  - Inputs: Rows from Google Sheets.
  - Outputs: Only matching rows passed to AI Agent.
  - Edge Cases:
    - Missing or malformed DOB field.
    - Date format assumptions (expects `dd/mm/yyyy`).
    - Timezone issues affecting `$now`.
  - Version: 2.2
  - No sub-workflows.

#### 1.4 AI Email Composition

- **Overview:** Generates a personalized and warm birthday email using OpenRouter’s GPT-4o model via LangChain agent.
- **Nodes Involved:** AI Agent, OpenRouter Chat Model, Structured Output Parser
- **Node Details:**

  - **OpenRouter Chat Model**
    - Type: Language Model Chat Node (@n8n/n8n-nodes-langchain.lmChatOpenRouter)
    - Configuration:
      - Model: `openai/gpt-4o-mini` (OpenRouter GPT-4o variant)
      - No additional options set.
    - Credentials: OpenRouter API.
    - Input: Connected as the language model for AI Agent.
    - Output: Language model response to AI Agent.

  - **Structured Output Parser**
    - Type: LangChain Output Parser (@n8n/n8n-nodes-langchain.outputParserStructured)
    - Configuration:
      - JSON Schema Example specifying the expected output format:
        ```json
        {
          "subject": "Happy Birthday, Keyur! 🎉",
          "body": "Dear Keyur,\n\nHappy Birthday! I hope your special day ... \nParth"
        }
        ```
      - Ensures the AI output is structured with `subject` and `body` fields.
    - Input: Connected as output parser to AI Agent.
    - Output: Structured data passed to AI Agent node.

  - **AI Agent**
    - Type: LangChain Agent Node (@n8n/n8n-nodes-langchain.agent)
    - Configuration:
      - Prompt text:
        ```
        use below name for birthday wishes.

        name:{{ $json.name }}
        dob:{{ $json.DOB }}

        write email according to you and best regards by parth.
        ```
      - System Message:
        ```
        You are an assistant that writes warm and personalized birthday emails. Use the user’s name to begin the email. Keep it short, friendly, and positive. Add one emoji. End with best wishes and the sender’s name.
        ```
      - Prompt Type: Define (custom prompt).
      - Output Parser: Enabled; uses Structured Output Parser.
    - Inputs: Receives filtered birthday row.
    - Outputs: Generates structured email subject and body.
    - Dependencies:
      - Requires OpenRouter Chat Model node as language model.
      - Requires Structured Output Parser node for parsing.
    - Edge Cases:
      - AI response timeouts or failures.
      - Unexpected output format from AI.
      - Missing `name` or `DOB` fields in input JSON.
    - Version: 2

#### 1.5 Email Sending

- **Overview:** Sends the personalized birthday email via Gmail to the recipient.
- **Nodes Involved:** Send a message
- **Node Details:**
  - Type: Gmail node (n8n-nodes-base.gmail)
  - Configuration:
    - Recipient email: `{{$node["Filter"].item.json.EMail}}` — dynamically takes the email from the filtered birthday record.
    - Subject: `{{$json.output.subject}}` — from AI Agent output.
    - Message body: `{{$json.output.body}}` — from AI Agent output.
    - Email type: Text (plain text email).
    - Option: Attribution disabled.
  - Credentials: Gmail OAuth2 account connected.
  - Input: Receives output from AI Agent.
  - Output: Sends the email; no further output.
  - Edge Cases:
    - Gmail API quota limits or auth errors.
    - Invalid recipient email address.
    - Failures in AI output leading to empty subject/body.
  - Version: 2.1
  - No sub-workflows.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                     | Input Node(s)           | Output Node(s)        | Sticky Note                                      |
|------------------------|----------------------------------|-----------------------------------|------------------------|-----------------------|-------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                  | Initiates daily workflow at 9 AM  | None                   | Get row(s) in sheet   |                                                 |
| Get row(s) in sheet    | Google Sheets                    | Retrieves birthday data rows      | Schedule Trigger       | Filter                |                                                 |
| Filter                 | Filter                          | Filters rows for today’s birthdays| Get row(s) in sheet    | AI Agent              |                                                 |
| AI Agent               | LangChain Agent                  | Generates personalized birthday email | Filter               | Send a message        |                                                 |
| OpenRouter Chat Model  | LangChain LM Chat (OpenRouter)  | Provides GPT-4o model for AI Agent| AI Agent (ai_languageModel) | AI Agent           |                                                 |
| Structured Output Parser | LangChain Output Parser         | Parses AI response into subject/body | AI Agent (ai_outputParser) | AI Agent           |                                                 |
| Send a message         | Gmail                           | Sends birthday email to recipient | AI Agent               | None                  |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**
   - Type: Schedule Trigger
   - Set to trigger daily at 9:00 AM (`triggerAtHour`: 9).
   - No credentials required.
   - Connect its output to the next node.

2. **Create Google Sheets node:**
   - Type: Google Sheets
   - Credentials: Connect your Google Sheets OAuth2 credential.
   - Operation: Get rows.
   - Document ID: Use your Google Sheet ID containing birthdays (`1fTGWkzIMdq1G3v2R3tRsRWIgEByPBeKsKXIJTcsxWwk` or your own).
   - Sheet Name: Use the first sheet or specify the gid (`gid=0`).
   - Connect Schedule Trigger output to this node’s input.

3. **Create Filter node:**
   - Type: Filter
   - Condition:
     - Version 2.
     - Use a DateTime equals comparison.
     - Left value expression: `{{$json.DOB.split('/').slice(0,2).join('-')}}`
     - Right value expression: `{{$now.toFormat("dd-MM")}}`
   - Connect Google Sheets node output to Filter node input.

4. **Create LangChain OpenRouter Chat Model node:**
   - Type: LangChain LM Chat OpenRouter
   - Credentials: Connect OpenRouter API key.
   - Model: `openai/gpt-4o-mini`.
   - Leave other options default.
   - Do not connect yet; will be referenced by AI Agent.

5. **Create LangChain Structured Output Parser node:**
   - Type: LangChain Output Parser Structured
   - JSON Schema Example:
     ```json
     {
       "subject": "Happy Birthday, Keyur! 🎉",
       "body" : "Dear Keyur,\n\nHappy Birthday! I hope your special day ... \nParth"
     }
     ```
   - No direct input connection needed; referenced by AI Agent.

6. **Create LangChain AI Agent node:**
   - Type: LangChain Agent
   - Text prompt:
     ```
     use below name for birthday wishes.

     name:{{ $json.name }}
     dob:{{ $json.DOB }}

     write email according to you and best regards by parth.
     ```
   - System message:
     ```
     You are an assistant that writes warm and personalized birthday emails. Use the user’s name to begin the email. Keep it short, friendly, and positive. Add one emoji. End with best wishes and the sender’s name.
     ```
   - Prompt Type: Define.
   - Enable Output Parser.
   - Assign OpenRouter Chat Model node as language model input.
   - Assign Structured Output Parser node as output parser.
   - Connect Filter node output to AI Agent input.

7. **Create Gmail node:**
   - Type: Gmail
   - Credentials: Connect Gmail OAuth2 account.
   - Send To: Use expression to get email from filtered data: `{{$node["Filter"].item.json.EMail}}`
   - Subject: `{{$json.output.subject}}`
   - Message: `{{$json.output.body}}`
   - Email Type: Text.
   - Option: Disable append attribution.
   - Connect AI Agent output to Gmail node input.

8. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses OpenRouter GPT-4o-mini for AI message generation, which requires an OpenRouter API key.    | https://openrouter.ai/                                                                         |
| Google Sheets OAuth2 credentials must have read access to the birthday spreadsheet.                           | https://developers.google.com/sheets/api/guides/authorizing                                       |
| Gmail OAuth2 credentials require permission to send emails on behalf of the user.                            | https://developers.google.com/gmail/api/auth/about-auth                                       |
| Date format in Google Sheets must be `dd/mm/yyyy` in the DOB column for filter expression to work correctly.| Important to maintain consistent date formatting for filter logic accuracy.                    |
| The AI Agent uses LangChain integration nodes available in n8n version supporting LangChain nodes (>v0.200).| https://docs.n8n.io/integrations/agents/langchain/                                              |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.