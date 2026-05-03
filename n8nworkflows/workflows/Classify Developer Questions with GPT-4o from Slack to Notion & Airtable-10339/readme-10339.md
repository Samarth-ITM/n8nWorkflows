Classify Developer Questions with GPT-4o from Slack to Notion & Airtable

https://n8nworkflows.xyz/workflows/classify-developer-questions-with-gpt-4o-from-slack-to-notion---airtable-10339


# Classify Developer Questions with GPT-4o from Slack to Notion & Airtable

### 1. Workflow Overview

This workflow automates the classification and handling of developer questions posted in a specific Slack channel. It leverages the GPT-4o language model via Azure OpenAI to determine if a question matches an internal FAQ. Based on the classification, it either logs the answered questions into a Notion database or records unanswered questions into Airtable for follow-up. Additionally, it includes error handling by logging issues into a Google Sheets document.

The workflow is logically organized into the following blocks:

- **1.1 Slack Input Reception and Validation**: Captures new Slack messages in a developer Q&A channel and validates the payload.
- **1.2 Question Data Extraction**: Cleans and structures the Slack message content into a standardized format.
- **1.3 AI-Based Question Classification**: Uses GPT-4o to classify questions as answered or unanswered by comparing them to an internal FAQ.
- **1.4 AI Output Parsing and Decision Making**: Parses AI JSON output and routes the flow based on classification.
- **1.5 Data Logging and Storage**: Saves answered questions to Notion and unanswered questions to Airtable.
- **1.6 Error Handling**: Logs any workflow errors or invalid payloads into a Google Sheets error log.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Input Reception and Validation

- **Overview**: This block triggers on new Slack messages posted in a specified channel and validates the message payload to ensure it contains a user ID and text.
- **Nodes Involved**:
  - Slack Channel Trigger – Developer Q&A
  - Validate Slack Message Payload
  - Log Workflow Errors to Google Sheets

**Node Details**:

- **Slack Channel Trigger – Developer Q&A**
  - **Type**: Slack Trigger  
  - **Role**: Listens for new messages in Slack channel ID `C09CVLMSF3R` (developer Q&A channel).
  - **Config**: Trigger on message events; credentials use a Slack account.
  - **Input/Output**: Initiates workflow; outputs Slack event JSON.
  - **Edge Cases**: Slack API rate limits, missing or malformed events.
  - **Sticky Note Content**: Explained as the workflow entry point capturing developer questions.

- **Validate Slack Message Payload**
  - **Type**: IF node (boolean condition)
  - **Role**: Ensures incoming Slack message has non-empty `user` field.
  - **Config**: Condition checks `$json.user` is not empty.
  - **Input**: Output from Slack trigger node.
  - **Output**: True path proceeds; false path triggers error logging.
  - **Edge Cases**: System messages without user or empty messages.
  - **Sticky Note Content**: Prevents processing invalid or system-generated messages.

- **Log Workflow Errors to Google Sheets**
  - **Type**: Google Sheets Append
  - **Role**: Logs error entries when validation fails or other errors occur.
  - **Config**: Appends rows to "error log sheet" within a specified Google Sheets document.
  - **Input**: Receives error details from failed validation or error paths.
  - **Edge Cases**: Google API quota limits, credential expiration.
  - **Sticky Note Content**: Ensures transparency on workflow errors.

---

#### 2.2 Question Data Extraction

- **Overview**: Cleans and formats Slack message content into a structured JSON object for AI processing.
- **Nodes Involved**:
  - Extract Question Metadata (JavaScript)

**Node Details**:

- **Extract Question Metadata (JavaScript)**
  - **Type**: Code (JavaScript)
  - **Role**: Removes unwanted characters (`>`, newlines, quotes) and extracts key message data: question text, user ID, timestamp, and channel.
  - **Config**: Uses regex replace and trim on `$json.text`.
  - **Input**: Output from Validate Slack Message Payload.
  - **Output**: JSON object with cleaned question metadata.
  - **Edge Cases**: Unexpected message formatting, empty text after cleaning.
  - **Sticky Note Content**: Prepares clean “question object” for AI classification.

---

#### 2.3 AI-Based Question Classification

- **Overview**: Uses GPT-4o model to classify the developer question as answered or unanswered based on internal FAQ.
- **Nodes Involved**:
  - Configure GPT-4o Model
  - Classify Developer Question (AI)

**Node Details**:

- **Configure GPT-4o Model**
  - **Type**: Azure OpenAI Chat Model Node
  - **Role**: Sets GPT-4o as the active AI model.
  - **Config**: Uses Azure OpenAI credentials; model set to `gpt-4o`.
  - **Input**: None explicitly; invoked before AI classification node.
  - **Output**: Provides AI model context for classification node.
  - **Edge Cases**: Azure API limits, authentication errors.
  - **Sticky Note Content**: Configures GPT-4o for semantic understanding.

- **Classify Developer Question (AI)**
  - **Type**: Langchain Agent Node (AI Agent)
  - **Role**: Receives cleaned question text and compares it against three predefined FAQ entries.
  - **Config**: Prompt defines FAQ baseline questions and instructs GPT-4o to return JSON with keys: `status`, `answer_quality`, `canonical_answer`.
  - **Key Expressions**: Uses expression `=You are a Slack Q&A classifier...` with system message dynamically set to current question.
  - **Input**: Cleaned question JSON from extraction node.
  - **Output**: Text response formatted as JSON string.
  - **Edge Cases**: AI response malformed, timeout, API throttling.
  - **Sticky Note Content**: AI classification and similarity matching.

---

#### 2.4 AI Output Parsing and Decision Making

- **Overview**: Parses the AI output string into JSON and routes workflow based on whether question is answered.
- **Nodes Involved**:
  - Parse AI JSON Output
  - Check If Question Was Answered

**Node Details**:

- **Parse AI JSON Output**
  - **Type**: Code (JavaScript)
  - **Role**: Parses AI string output into JSON object for downstream nodes.
  - **Config**: Uses `JSON.parse($json.output)`.
  - **Input**: AI text output from Classify Developer Question node.
  - **Output**: Parsed JSON with `status`, `answer_quality`, `canonical_answer`.
  - **Edge Cases**: JSON parse errors if AI response is malformed.
  - **Sticky Note Content**: Bridges AI output to workflow logic.

- **Check If Question Was Answered**
  - **Type**: IF node
  - **Role**: Checks if `status` field equals `"answered"`.
  - **Config**: Condition checks `$json.status == "answered"`.
  - **Input**: Parsed AI JSON.
  - **Output**: True path for answered questions; false path for unanswered.
  - **Edge Cases**: Missing `status` key, unexpected status values.
  - **Sticky Note Content**: Decision point for routing to Notion or Airtable.

---

#### 2.5 Data Logging and Storage

- **Overview**: Stores answered questions into Notion FAQ database; logs unanswered questions into Airtable for manual follow-up.
- **Nodes Involved**:
  - Save Answered Question to Notion FAQ
  - Log Unanswered Question to Airtable

**Node Details**:

- **Save Answered Question to Notion FAQ**
  - **Type**: Notion Database Page Create
  - **Role**: Creates a new database entry recording the answered question.
  - **Config**: Maps Slack question text as page title; stores `Status`, `Priority` (answer quality), and `FAQ Content` (canonical answer).
  - **Credentials**: Notion API account.
  - **Input**: AI JSON output and original Slack message text.
  - **Edge Cases**: Notion API limits, database schema mismatches.
  - **Sticky Note Content**: Maintains internal knowledge base automatically.

- **Log Unanswered Question to Airtable**
  - **Type**: Airtable Create Record
  - **Role**: Adds new record for unanswered questions to Airtable table.
  - **Config**: Stores Slack question text, user ID, message ID, and AI classification status.
  - **Credentials**: Airtable Personal Access Token.
  - **Input**: AI JSON output and original Slack message.
  - **Edge Cases**: Airtable API rate limits, schema changes.
  - **Sticky Note Content**: Tracks unresolved questions for manual review and FAQ training.

---

#### 2.6 Error Handling

- **Overview**: Captures and logs workflow errors or invalid payloads to Google Sheets for monitoring.
- **Nodes Involved**:
  - Log Workflow Errors to Google Sheets (also used in validation block)

**Node Details**:

- **Log Workflow Errors to Google Sheets**
  - See details in 2.1 above.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                                      | Input Node(s)                          | Output Node(s)                             | Sticky Note                                                   |
|-----------------------------------|-----------------------------------|-----------------------------------------------------|--------------------------------------|-------------------------------------------|---------------------------------------------------------------|
| Slack Channel Trigger – Developer Q&A | Slack Trigger                    | Entry point; listens for new Slack channel messages | None                                 | Validate Slack Message Payload             | Triggers workflow on new Slack messages in developer Q&A channel |
| Validate Slack Message Payload     | IF Node                           | Validates message contains user and text             | Slack Channel Trigger – Developer Q&A | Extract Question Metadata (JS), Log Workflow Errors to Google Sheets | Checks for valid user and text; prevents empty/system messages  |
| Extract Question Metadata (JavaScript) | Code (JavaScript)               | Cleans and structures Slack message data             | Validate Slack Message Payload        | Classify Developer Question (AI)           | Cleans message text and creates structured question object      |
| Configure GPT-4o Model             | Azure OpenAI Chat Model           | Configures GPT-4o model for AI classification         | None                                 | Classify Developer Question (AI)            | Sets GPT-4o as AI model for semantic understanding             |
| Classify Developer Question (AI)  | Langchain Agent (AI)              | Classifies question against internal FAQ              | Extract Question Metadata (JS), Configure GPT-4o Model | Parse AI JSON Output                        | Uses GPT-4o to classify question as answered/unanswered         |
| Parse AI JSON Output               | Code (JavaScript)                 | Parses AI output string into JSON                      | Classify Developer Question (AI)      | Check If Question Was Answered              | Parses AI JSON output for workflow logic                        |
| Check If Question Was Answered    | IF Node                          | Routes flow based on AI classification status         | Parse AI JSON Output                  | Save Answered Question to Notion FAQ, Log Unanswered Question to Airtable | Checks if question was answered; routes to Notion or Airtable   |
| Save Answered Question to Notion FAQ | Notion Database Page Create     | Logs answered questions to Notion FAQ database        | Check If Question Was Answered        | None                                      | Stores answered question and metadata in Notion                |
| Log Unanswered Question to Airtable | Airtable Create Record           | Logs unanswered questions for manual follow-up        | Check If Question Was Answered        | None                                      | Records unanswered questions in Airtable for review             |
| Log Workflow Errors to Google Sheets | Google Sheets Append             | Logs errors or invalid payloads for monitoring        | Validate Slack Message Payload (false path), other error paths | None                                      | Logs workflow errors ensuring transparency                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Channel Trigger – Developer Q&A**  
   - Node Type: Slack Trigger  
   - Configuration: Trigger on `message` events in channel ID `C09CVLMSF3R`  
   - Credentials: Slack API credentials with access to the target channel  
   - Connect output to: Validate Slack Message Payload node

2. **Create Validate Slack Message Payload**  
   - Node Type: IF Node (Version 2.2)  
   - Condition: Check `$json.user` is not empty (string not empty)  
   - True output: Connect to Extract Question Metadata (JS) node  
   - False output: Connect to Log Workflow Errors to Google Sheets node

3. **Create Log Workflow Errors to Google Sheets**  
   - Node Type: Google Sheets Append  
   - Document ID: `1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y`  
   - Sheet Name: `error log sheet` (ID 1338537721)  
   - Columns: `error_id`, `error` (both string)  
   - Credentials: Google OAuth2 with write access  
   - Input: Error details from validation false output or other error paths

4. **Create Extract Question Metadata (JavaScript)**  
   - Node Type: Code (JavaScript)  
   - Code:  
     ```javascript
     return [{
       question: $json.text.replace(/&gt;|[\n"]/g, '').trim(),
       user: $json.user,
       timestamp: $json.ts,
       channel: $json.channel
     }];
     ```  
   - Input: True output from Validate Slack Message Payload  
   - Connect output to: Configure GPT-4o Model node

5. **Create Configure GPT-4o Model**  
   - Node Type: Azure OpenAI Chat Model  
   - Model: `gpt-4o`  
   - Credentials: Azure OpenAI API credentials  
   - Connect output to: Classify Developer Question (AI) node

6. **Create Classify Developer Question (AI)**  
   - Node Type: Langchain Agent (AI Agent)  
   - Parameters:  
     - Text input: Prompt defines internal FAQ knowledge with three Q&A examples and instructions to output JSON with `status`, `answer_quality`, `canonical_answer`.  
     - System message: Dynamic, set to current question with expression: `=Question: {{$json.question}}`  
   - Credentials: Uses the GPT-4o configured in previous node  
   - Input: From Extract Question Metadata and Configure GPT-4o Model  
   - Connect output to: Parse AI JSON Output node

7. **Create Parse AI JSON Output**  
   - Node Type: Code (JavaScript)  
   - Code: `return [JSON.parse($json.output)];`  
   - Input: Output from AI Classification node  
   - Connect output to: Check If Question Was Answered node

8. **Create Check If Question Was Answered**  
   - Node Type: IF Node (Version 2.2)  
   - Condition: Check if `$json.status` equals `"answered"`  
   - True output: Connect to Save Answered Question to Notion FAQ node  
   - False output: Connect to Log Unanswered Question to Airtable node

9. **Create Save Answered Question to Notion FAQ**  
   - Node Type: Notion Database Page Create  
   - Database ID: `29a802b9-1fa0-804a-b406-e078961e0659` (Release Notes)  
   - Properties Mapping:  
     - Title: Slack message text from `Slack Channel Trigger – Developer Q&A` node  
     - Status (rich_text): `$json.status`  
     - Priority (rich_text): `$json.answer_quality`  
     - FAQ Content (rich_text): `$json.canonical_answer`  
   - Credentials: Notion API account with write access  
   - Input: True branch of Check If Question Was Answered

10. **Create Log Unanswered Question to Airtable**  
    - Node Type: Airtable Create Record  
    - Base ID: `appsZ3Uuh5PnD215s`  
    - Table ID: `tblZvyR7J8hndLlUZ` (Table 1)  
    - Columns Mapping:  
      - Name: Slack message text from `Slack Channel Trigger – Developer Q&A`  
      - Status: `Todo` (static string)  
      - FAQ Match: `$json.answer_quality`  
      - Error Code: `$json.status`  
      - Issue Number: Slack message client_msg_id  
    - Credentials: Airtable Personal Access Token  
    - Input: False branch of Check If Question Was Answered

11. **Configure Error Handling**  
    - Ensure that any errors or invalid payloads from nodes (especially Slack trigger and AI calls) are routed to the Log Workflow Errors to Google Sheets node for monitoring.

12. **Link All Nodes According to the described connections**  
    - Slack Trigger → Validate Payload  
    - Validate Payload (true) → Extract Metadata → Configure GPT-4o → AI Classification → Parse AI Output → Check If Answered → (True) Notion, (False) Airtable  
    - Validate Payload (false) → Log Errors  
    - AI or other node errors → Log Errors

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4o model from Azure OpenAI to perform semantic similarity matching against an internal FAQ.                    | Azure OpenAI GPT-4o model                                                                              |
| Slack channel ID `C09CVLMSF3R` is configured as the source for developer Q&A messages.                                               | Slack channel configuration                                                                            |
| Notion database used is named "Release Notes" with database ID `29a802b9-1fa0-804a-b406-e078961e0659`.                              | Notion database for FAQ storage                                                                        |
| Airtable base and table configured to track unresolved questions for developer follow-up and continual FAQ improvement.              | Airtable base `appsZ3Uuh5PnD215s`, Table `tblZvyR7J8hndLlUZ`                                           |
| Google Sheets document used for error logging enables monitoring silent failures or missing payloads.                                | Google Sheets ID: `1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y`                                      |
| The workflow prevents processing empty or system Slack messages by validating presence of user ID and text before AI calls.          | Workflow robustness and cost-saving measure                                                           |
| The AI prompt includes three specific FAQ Q&A examples, which can be extended for better coverage.                                  | AI prompt design for classification logic                                                             |
| If the AI returns malformed JSON, the workflow may fail at the JSON parsing node, which should be caught by error handling.          | Potential failure point at JSON parsing                                                                |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. The processing respects current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.