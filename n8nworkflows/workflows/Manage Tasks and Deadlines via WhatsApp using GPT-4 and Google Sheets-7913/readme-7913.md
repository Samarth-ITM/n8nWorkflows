Manage Tasks and Deadlines via WhatsApp using GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/manage-tasks-and-deadlines-via-whatsapp-using-gpt-4-and-google-sheets-7913


# Manage Tasks and Deadlines via WhatsApp using GPT-4 and Google Sheets

### 1. Workflow Overview

This workflow implements an automated WhatsApp chatbot that manages task tracking and deadlines by interfacing with Google Sheets and GPT-4 via OpenAI. It targets users who want to query and update task-related information directly through WhatsApp, leveraging AI to provide concise, actionable responses and maintain conversation context.

**Logical Blocks:**

- **1.1 Input Reception:** Receives incoming WhatsApp messages via webhook.
- **1.2 AI Processing:** Uses a LangChain AI agent powered by GPT-4, with conversation memory, to interpret user queries and interact with task data.
- **1.3 Google Sheets Integration:** Retrieves and updates task data from a Google Sheet.
- **1.4 Messaging Response:** Sends AI-generated replies back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming WhatsApp messages to trigger the workflow.

- **Nodes Involved:**  
  - WhatsApp Trigger1

- **Node Details:**  
  - **WhatsApp Trigger1**  
    - *Type:* WhatsApp Trigger  
    - *Role:* Listens for incoming WhatsApp messages to start the workflow.  
    - *Configuration:*  
      - Listens for "messages" events only.  
      - Uses OAuth credentials for WhatsApp API authentication.  
    - *Expressions/Variables:* None directly; outputs incoming message JSON.  
    - *Inputs:* External webhook calls (WhatsApp message).  
    - *Outputs:* Sends message JSON to “AI Agent1”.  
    - *Version:* 1  
    - *Failures:* Possible webhook misconfiguration, OAuth token expiry, message format errors.  
    - *Sub-workflow:* None.

#### 2.2 AI Processing

- **Overview:**  
  Processes user messages through a LangChain AI agent that uses GPT-4, maintaining conversation context and integrating data from Google Sheets.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenAI Chat Model1  
  - Simple Memory1  

- **Node Details:**

  - **AI Agent1**  
    - *Type:* LangChain Agent  
    - *Role:* Core AI logic interpreting user queries, accessing task data, and generating responses.  
    - *Configuration:*  
      - Prompt defines an AI assistant with responsibilities: concise answers, data access to Google Sheets, task status identification, and deadline highlighting.  
      - Inputs user text: `{{ $json.messages[0].text.body }}` and current date `{{ $now }}`.  
      - Connects to OpenAI Chat Model and Simple Memory nodes for language model and conversational memory.  
    - *Expressions:* Uses expressions to extract WhatsApp message text and current timestamp.  
    - *Inputs:* Receives messages from WhatsApp Trigger1, Google Sheets data from “Get row(s)” and “Update Row”, memory from Simple Memory1.  
    - *Outputs:* Generates text response fed into “Send message1”.  
    - *Version:* 2.2  
    - *Failures:* AI API rate limits, prompt syntax errors, memory session key mismatches, connectivity issues.  
    - *Sub-workflow:* None.

  - **OpenAI Chat Model1**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Provides GPT-4.1-mini model responses for AI Agent1.  
    - *Configuration:*  
      - Model: GPT-4.1-mini.  
      - Uses OpenAI API credentials.  
    - *Inputs:* Receives prompt and context from AI Agent1.  
    - *Outputs:* Returns AI-generated text to AI Agent1.  
    - *Version:* 1.2  
    - *Failures:* API key issues, network errors, rate limiting.  
    - *Sub-workflow:* None.

  - **Simple Memory1**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains conversational context using the WhatsApp sender’s ID as session key.  
    - *Configuration:*  
      - Session key: `={{ $json.contacts[0].wa_id }}` to tie memory to user.  
      - Session type: customKey.  
    - *Inputs:* Stores conversation history from AI Agent1.  
    - *Outputs:* Supplies memory context back to AI Agent1.  
    - *Version:* 1.3  
    - *Failures:* Memory overflow or corruption, session key extraction errors.  
    - *Sub-workflow:* None.

#### 2.3 Google Sheets Integration

- **Overview:**  
  Retrieves existing tasks and updates task data in Google Sheets as requested by the user.

- **Nodes Involved:**  
  - Get row(s)  
  - Update Row  

- **Node Details:**

  - **Get row(s)**  
    - *Type:* Google Sheets Tool (Read)  
    - *Role:* Fetches all rows from the “Tasks” sheet to provide task data to the AI Agent.  
    - *Configuration:*  
      - Document: Google Sheet URL with ID `1T9Xhmu_xr-mPXcOIPr0MbvMiKOT6jIhQWYxhETi9X8k`  
      - Sheet ID: 569756928 (“Tasks” sheet)  
      - No filters; retrieves all rows.  
      - OAuth2 credentials.  
    - *Inputs:* No inputs; triggered by workflow start.  
    - *Outputs:* Sends rows to AI Agent1 as input data.  
    - *Version:* 4.7  
    - *Failures:* OAuth token expiry, sheet access permissions, sheet not found, API quota limits.  
    - *Sub-workflow:* None.

  - **Update Row**  
    - *Type:* Google Sheets Tool (Append or Update)  
    - *Role:* Modifies or appends task rows in the “Tasks” sheet based on AI instructions.  
    - *Configuration:*  
      - Matches rows by “Task” column.  
      - Maps “Task”, “Notes”, “Status”, and “Due date” from AI output fields.  
      - Uses same Google Sheet and sheet as Get row(s).  
      - OAuth2 credentials.  
    - *Inputs:* Receives AI-generated data from AI Agent1.  
    - *Outputs:* Sends confirmation or updated data back to AI Agent1.  
    - *Version:* 4.7  
    - *Failures:* Data mapping errors, sheet concurrency conflicts, malformed data, OAuth issues.  
    - *Sub-workflow:* None.

#### 2.4 Messaging Response

- **Overview:**  
  Sends the AI-generated response text back to the user on WhatsApp.

- **Nodes Involved:**  
  - Send message1

- **Node Details:**  
  - **Send message1**  
    - *Type:* WhatsApp Node  
    - *Role:* Sends text messages to the user’s WhatsApp number.  
    - *Configuration:*  
      - Text body set dynamically to AI Agent1’s output: `={{ $json.output }}`  
      - Recipient phone number extracted from WhatsApp Trigger1: `={{ $('WhatsApp Trigger1').item.json.contacts[0].wa_id }}`  
      - Uses WhatsApp API OAuth credentials.  
    - *Inputs:* Receives AI text from AI Agent1.  
    - *Outputs:* None (end of workflow).  
    - *Version:* 1  
    - *Failures:* Message send failures, invalid phone number, OAuth expiry.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name         | Node Type                            | Functional Role                     | Input Node(s)          | Output Node(s)      | Sticky Note                       |
|-------------------|------------------------------------|-----------------------------------|-----------------------|---------------------|---------------------------------|
| WhatsApp Trigger1  | WhatsApp Trigger                   | Receive incoming WhatsApp messages| None                  | AI Agent1           |                                 |
| AI Agent1         | LangChain Agent                    | Core AI processing and response   | WhatsApp Trigger1, Get row(s), Update Row, Simple Memory1 | Send message1       |                                 |
| OpenAI Chat Model1 | LangChain OpenAI Chat Model        | GPT-4 model for AI Agent          | AI Agent1 (ai_languageModel) | AI Agent1           |                                 |
| Simple Memory1    | LangChain Memory Buffer Window      | Maintain conversation memory      | AI Agent1 (ai_memory)  | AI Agent1           |                                 |
| Get row(s)        | Google Sheets Tool (Read)           | Retrieve task data from Google Sheet | None                  | AI Agent1           |                                 |
| Update Row        | Google Sheets Tool (AppendOrUpdate) | Update or append task data in sheet | AI Agent1             | AI Agent1           |                                 |
| Send message1     | WhatsApp Node                      | Send message back to WhatsApp user| AI Agent1              | None                |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node:**  
   - Type: WhatsApp Trigger  
   - Configure to listen for “messages” events only.  
   - Add OAuth2 credentials for WhatsApp API.  
   - Position: Start node.

2. **Create AI Agent Node:**  
   - Type: LangChain Agent (from n8n-nodes-langchain.agent)  
   - Parameters:  
     - Set prompt text: Describe AI assistant role with instructions to access Google Sheets, identify tasks by status/due date, highlight overdue/urgent tasks, provide concise answers.  
     - Use expressions to insert user message: `{{ $json.messages[0].text.body }}`  
     - Insert current date: `{{ $now }}`  
   - Connect input to WhatsApp Trigger node.  
   - Connect outputs to Send message node.  
   - Version: 2.2.

3. **Create OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4.1-mini  
   - Attach OpenAI API credentials.  
   - Connect as language model input to AI Agent node.  
   - Version: 1.2.

4. **Create Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Session key expression: `={{ $json.contacts[0].wa_id }}` (to tie memory to WhatsApp user)  
   - Session type: customKey  
   - Connect as AI memory input to AI Agent node.  
   - Version: 1.3.

5. **Create Get row(s) Node:**  
   - Type: Google Sheets Tool (Read)  
   - Document: Paste Google Sheet URL (for example, https://docs.google.com/spreadsheets/d/1T9Xhmu_xr-mPXcOIPr0MbvMiKOT6jIhQWYxhETi9X8k)  
   - Sheet: Choose “Tasks” by sheet ID 569756928.  
   - OAuth2 credentials for Google Sheets.  
   - Connect output to AI Agent node (ai_tool input).  
   - Version: 4.7.

6. **Create Update Row Node:**  
   - Type: Google Sheets Tool (Append or Update)  
   - Document and sheet same as Get row(s).  
   - Matching column: “Task”  
   - Map columns: “Task”, “Notes”, “Status”, “Due date” using AI Agent output fields.  
   - OAuth2 credentials.  
   - Connect output to AI Agent node (ai_tool input).  
   - Version: 4.7.

7. **Create Send message Node:**  
   - Type: WhatsApp Node  
   - Operation: send  
   - Text body: Set expression from AI Agent output `={{ $json.output }}`  
   - Recipient phone number: Expression from WhatsApp Trigger node `={{ $('WhatsApp Trigger1').item.json.contacts[0].wa_id }}`  
   - OAuth2 credentials for WhatsApp API.  
   - Connect input from AI Agent node.  
   - Version: 1.

8. **Connect nodes in order:**  
   - WhatsApp Trigger1 → AI Agent1  
   - Get row(s) → AI Agent1 (ai_tool)  
   - Update Row → AI Agent1 (ai_tool)  
   - Simple Memory1 → AI Agent1 (ai_memory)  
   - OpenAI Chat Model1 → AI Agent1 (ai_languageModel)  
   - AI Agent1 → Send message1

9. **Set workflow settings:**  
   - Execution order: v1 (default)  
   - Activate workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| WhatsApp OAuth credentials require prior app registration and approval through Facebook Business Manager.                      | Official WhatsApp Business API docs: https://developers.facebook.com/docs/whatsapp/overview                    |
| OpenAI GPT-4.1-mini model provides cost-effective GPT-4 variant for conversational AI.                                         | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                                  |
| Google Sheets OAuth2 requires enabling Sheets API and configuring OAuth consent screen in Google Cloud Console.                | Google Sheets API docs: https://developers.google.com/sheets/api                                                 |
| LangChain integration in n8n requires node versions supporting LangChain nodes (minimum n8n 0.211.0 recommended).              | n8n LangChain node docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                     |
| For WhatsApp phone number format, use E.164 international format without spaces or symbols.                                     | WhatsApp API best practices: https://developers.facebook.com/docs/whatsapp/guides/send-message                   |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.