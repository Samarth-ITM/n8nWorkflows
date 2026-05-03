Automated FAQ Generator from WhatsApp Groups using GPT-4 and Google Docs

https://n8nworkflows.xyz/workflows/automated-faq-generator-from-whatsapp-groups-using-gpt-4-and-google-docs-8096


# Automated FAQ Generator from WhatsApp Groups using GPT-4 and Google Docs

---

### 1. Workflow Overview

This workflow automates the generation of Frequently Asked Questions (FAQ) from WhatsApp group conversations using GPT-4 and Google Docs. It is designed to assist educational or community managers in extracting recurring questions and concerns from chat logs, summarizing them with AI, and documenting the results in a Google Doc for easy sharing and reference.

**Target Use Cases:**  
- Educational communities seeking to identify common student questions.  
- Community managers wanting to automate FAQ updates from chat data.  
- Teams using WhatsApp groups for communication who want structured insights.  

**Logical Blocks:**

- **1.1 Scheduled Data Retrieval:** Triggered weekly to fetch WhatsApp message logs from Google Sheets filtered by specific group IDs.  
- **1.2 Message Grouping and Preprocessing:** Using JavaScript code, messages are parsed and grouped by ISO week number, creating message blocks per week.  
- **1.3 AI Analysis:** An AI Agent (LangChain) processes each weekly message block, identifying recurring questions and generating a structured FAQ summary.  
- **1.4 Document Preparation:** A Google Docs template is copied to create a new document for the current FAQ.  
- **1.5 FAQ Document Update:** The AI-generated FAQ content is inserted into the new Google Doc, finalizing the report.  

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:**  
This block triggers the workflow every Monday at 6 AM and retrieves WhatsApp group message data from a Google Sheets document filtered by group identifiers.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet  

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: n8n Schedule Trigger  
    - Configuration: Runs weekly on Monday at 6:00 AM.  
    - Inputs: None (start node).  
    - Outputs: Triggers the next node.  
    - Edge Cases: Timezone misconfigurations could cause unexpected trigger times. Ensure server timezone matches expectations.  

  - **Get row(s) in sheet**  
    - Type: n8n Google Sheets node  
    - Configuration: Reads rows from a Google Sheet identified by document ID "1WNX4TNB0AEFXGnBbGYJqz0by9TcHcUd5Z5cH4GsGZ6g" (sheet "gid=0").  
    - Filters: Retrieves rows where "RemoteJid Grupo" column matches either "120363418264921236@g.us" OR "120363401761323889@g.us" (two WhatsApp group IDs).  
    - Credentials: Uses Google Sheets OAuth2 credential "Google Sheets - Luís".  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: List of message rows matching the filter.  
    - Edge Cases:  
      - API rate limits or OAuth token expiration may cause failures.  
      - Empty or malformed data rows may propagate errors downstream.  

---

#### 2.2 Message Grouping and Preprocessing

- **Overview:**  
Transforms raw messages into weekly grouped blocks by parsing message dates and aggregating messages per week. Prepares data for AI processing.

- **Nodes Involved:**  
  - Code  

- **Node Details:**  

  - **Code**  
    - Type: n8n Code node (JavaScript)  
    - Configuration:  
      - Parses dates from "Data" column (format dd/MM/yyyy).  
      - Computes ISO week number for each message date.  
      - Groups messages by week, concatenates all messages into a single text block per week.  
      - Outputs an array of objects with properties: "semana" (week string) and "bloco" (concatenated messages).  
    - Input: Rows from Google Sheets.  
    - Output: Weekly grouped message blocks (one per week).  
    - Edge Cases:  
      - Missing or malformed dates ("Data") are skipped.  
      - Incorrect date formats lead to skipped messages.  
      - Very large message blocks may exceed downstream API limits.  
    - Notes: The code uses ISO week calculation to standardize weeks.  

---

#### 2.3 AI Analysis

- **Overview:**  
Uses a LangChain AI Agent configured with a system prompt to identify recurring doubts and questions from message blocks. Produces a structured FAQ in text format.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Configuration:  
      - Receives input: text containing the week ID and concatenated message block.  
      - System Message describes the task: identify recurring questions, explanations, and suggested answers for FAQs.  
      - Output: organized, clear list of recurring questions with explanations and optional suggested responses.  
    - Input: Weekly message blocks from Code node.  
    - Output: FAQ text summary per week.  
    - Edge Cases:  
      - Overly long message blocks might cause token limit errors.  
      - Misinterpretation of ambiguous messages may lead to incomplete or incorrect FAQs.  
      - AI service availability and authentication issues are possible.  
    - Sub-Workflow: Utilizes the OpenAI Chat Model node as the language model backend.  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Configuration: Uses GPT-4.1-mini model (GPT-4 variant).  
    - Credentials: OpenAI API key "OpenAi account".  
    - Input: Receives prompt from AI Agent.  
    - Output: AI-generated text passed back to AI Agent.  
    - Edge Cases:  
      - API rate limits, quota, or network errors may interrupt processing.  
      - Model version must support GPT-4.  

---

#### 2.4 Document Preparation

- **Overview:**  
Copies a Google Docs template document to create a new document where the FAQ will be written.

- **Nodes Involved:**  
  - Copy file  

- **Node Details:**  

  - **Copy file**  
    - Type: n8n Google Drive node  
    - Operation: Copy a file.  
    - Configuration: Copies the template document with ID "1hAStPci4ZgaXMPbpLRh4U8MxeJUIkul_iYwYzhQVaus".  
    - New file named: "Extraction FAQ - Whatsapp Group".  
    - Credentials: Google Drive OAuth2 "Google Drive account".  
    - Input: Triggered after AI Agent produces output.  
    - Output: Metadata of the new copied Google Doc, including its URL and ID.  
    - Edge Cases:  
      - Google Drive API errors or permission issues.  
      - Template document changes would affect output format.  

---

#### 2.5 FAQ Document Update

- **Overview:**  
Inserts the AI-generated FAQ content into the newly created Google Doc.

- **Nodes Involved:**  
  - Update a document  

- **Node Details:**  

  - **Update a document**  
    - Type: n8n Google Docs node  
    - Operation: Update document by inserting text.  
    - Configuration: Inserts the text output from the AI Agent node into the new Google Doc identified by its document URL (from Copy file output).  
    - Credentials: Google Docs OAuth2 "Google Docs account".  
    - Input: Receives new document ID and AI-generated FAQ text.  
    - Edge Cases:  
      - Document permission or API quota issues.  
      - Large text insertion might cause timeout or truncation.  

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                               | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                 |
|----------------------|--------------------------------|----------------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger     | n8n-nodes-base.scheduleTrigger | Triggers workflow weekly                      | None                   | Get row(s) in sheet      | Run Every Week: Every Monday 6am                                                            |
| Get row(s) in sheet  | n8n-nodes-base.googleSheets    | Retrieves WhatsApp messages from Google Sheets | Schedule Trigger       | Code                     | Find all messages of that last week                                                        |
| Code                 | n8n-nodes-base.code            | Groups messages by ISO week                    | Get row(s) in sheet    | AI Agent                 | Do "messages blocks" and identify the week of those messages                                |
| AI Agent             | @n8n/n8n-nodes-langchain.agent| Analyzes messages block and generates FAQ text | Code                   | Copy file                | The AI Agent separates the questions and the context about these questions                  |
| OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 language model for AI Agent      | AI Agent (ai_languageModel) | AI Agent                |                                                                                             |
| Copy file            | n8n-nodes-base.googleDrive     | Copies FAQ template document                   | AI Agent                | Update a document        | Copy the template doc. file and create new after that                                      |
| Update a document    | n8n-nodes-base.googleDocs      | Inserts AI-generated FAQ text into Google Doc | Copy file               | None                     |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run weekly: every Monday at 6:00 AM.  

2. **Create Google Sheets node ("Get row(s) in sheet"):**  
   - Type: Google Sheets node (read operation)  
   - Connect input from Schedule Trigger.  
   - Set Document ID to your WhatsApp messages spreadsheet ID.  
   - Set Sheet Name to the correct sheet (gid=0).  
   - Add filter to "RemoteJid Grupo" column: include rows where value equals either "120363418264921236@g.us" OR "120363401761323889@g.us".  
   - Use your Google Sheets OAuth2 credentials.  

3. **Create Code node:**  
   - Connect input from Google Sheets node.  
   - Paste JavaScript code that:  
     - Parses message dates in dd/MM/yyyy format.  
     - Converts dates to ISO weeks.  
     - Groups messages by week and concatenates them into a single string per week.  
   - Output: JSON objects with "semana" (week) and "bloco" (message block).  

4. **Create LangChain AI Agent node:**  
   - Connect input from Code node.  
   - Configure prompt inputs to include:  
     - "Semana que está sendo analisada: {{ $json.semana }}"  
     - "Bloco de Mensagens a ser tratado e analisado: {{ $json.bloco }}"  
   - Set System Message to instruct the AI to identify recurring questions, explain them, and provide suggested answers, as per the detailed prompt in the original workflow.  
   - Set to use "define" prompt type.  

5. **Create OpenAI Chat Model node:**  
   - Use GPT-4.1-mini model.  
   - Use your OpenAI API credentials.  
   - Connect as AI language model backend of the AI Agent node.  

6. **Create Google Drive node ("Copy file"):**  
   - Connect input from AI Agent output.  
   - Operation: Copy a file.  
   - Set File ID to your FAQ template Google Doc ID.  
   - Name the new file appropriately (e.g., "Extraction FAQ - Whatsapp Group").  
   - Use Google Drive OAuth2 credentials.  

7. **Create Google Docs node ("Update a document"):**  
   - Connect input from Copy file node.  
   - Operation: Update document by inserting text.  
   - Insert text from AI Agent's output JSON (FAQ content).  
   - Document URL set dynamically from Copy file node output.  
   - Use Google Docs OAuth2 credentials.  

8. **Connect nodes in order:**  
   Schedule Trigger → Get row(s) in sheet → Code → AI Agent → Copy file → Update a document.  

9. **Test the flow:**  
   - Run manually or wait for scheduled trigger.  
   - Verify Google Docs file creation and content correctness.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow runs every Monday at 6 AM to produce weekly FAQ updates from WhatsApp group chats.         | Time scheduling information from sticky notes.                                                                         |
| Source WhatsApp messages are stored in a Google Sheet filtered by group IDs (WhatsApp group JIDs).  | Group IDs filter: "120363418264921236@g.us", "120363401761323889@g.us".                                                 |
| AI Agent prompt is carefully designed to extract recurring questions with explanations and suggestions. | Detailed system prompt included within AI Agent node parameters.                                                        |
| Google Docs template document is copied to ensure consistent formatting and to keep a master template intact. | Template Doc ID: "1hAStPci4ZgaXMPbpLRh4U8MxeJUIkul_iYwYzhQVaus".                                                      |
| OpenAI GPT-4 model used via LangChain integration requires valid OpenAI API credentials.             | Ensure API key with GPT-4 access and sufficient quota.                                                                   |
| Google OAuth2 credentials for Sheets, Drive, and Docs must have proper scopes and permissions set.  | Credentials setup required for Google APIs: Sheets (read), Drive (copy), Docs (update).                                   |
| Edge cases include handling missing dates, API rate limits, AI prompt length limits, and document write failures. | Plan for retries, error handling, and message size limitations.                                                         |
| The workflow is inactive by default (active=false) and should be activated after credential verification and testing. | Activation step required in n8n UI before scheduling begins.                                                            |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---