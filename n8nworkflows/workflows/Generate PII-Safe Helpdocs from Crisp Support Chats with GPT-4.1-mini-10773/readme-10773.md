Generate PII-Safe Helpdocs from Crisp Support Chats with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/generate-pii-safe-helpdocs-from-crisp-support-chats-with-gpt-4-1-mini-10773


# Generate PII-Safe Helpdocs from Crisp Support Chats with GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the generation of PII-safe (Personally Identifiable Information-free) help documentation articles from Crisp support chat sessions using GPT-4.1-mini. It targets customer support teams who want to convert resolved chat conversations into structured, sanitized help articles automatically.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception:** Captures incoming Crisp chat events via webhook.
- **1.2 Session Resolution Check:** Filters events to trigger only when a chat session is marked as resolved.
- **1.3 Raw Chat Storage:** Stores incoming chat messages into a raw chat datatable to preserve the full session transcript.
- **1.4 Data Formatting:** Retrieves all messages for the session and formats them into chronological Q&A pairs.
- **1.5 AI Processing:** Uses GPT-4.1-mini to generate a comprehensive help document from the formatted conversation, removing all PII with placeholders.
- **1.6 Helpdoc Storage:** Saves the generated help documentation into a dedicated datatable for publishing or further use.

The workflow is designed for easy integration with Crisp’s webhook system and requires OpenAI credentials for AI processing. It also uses n8n datatables for storing raw chats and final help documents.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives Crisp chat event data via an HTTP POST webhook. This node is the entry point for all Crisp chat events sent to the workflow.

- **Nodes Involved:**  
  - `crisp` (Webhook)

- **Node Details:**  
  - Type: Webhook  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Unique identifier path (e.g., `b16f5841-783c-47de-a4ae-b11a81be200c`)  
  - Inputs: External HTTP POST requests from Crisp  
  - Outputs: Raw event payload to next node  
  - Edge cases:  
    - Missing or malformed webhook data  
    - Unauthorized or unexpected source requests (should be limited by Crisp webhook setup)  
  - Sticky Note:  
    - “Use this in Crisp/Settings/Workspace settings/Advanced” – reminder to configure Crisp webhook with this URL

---

#### 2.2 Session Resolution Check

- **Overview:**  
  Filters incoming events to proceed only when the chat session state changes to “resolved” (indicating the conversation is finished).

- **Nodes Involved:**  
  - `closed` (If node)

- **Node Details:**  
  - Type: If (conditional logic)  
  - Configuration:  
    - Condition checks if `body.data.content.namespace` equals the string `state:resolved` strictly and case sensitively  
  - Inputs: Output from `crisp` webhook  
  - Outputs:  
    - True branch: session resolved → proceed with data processing  
    - False branch: ignored  
  - Edge cases:  
    - Events without expected namespace field  
    - Different namespaces or states requiring workflow adjustment  
  - Sticky Note:  
    - “Check if resolved”  

---

#### 2.3 Raw Chat Storage

- **Overview:**  
  Stores each individual chat message from the Crisp event into a raw chats datatable, preserving all messages for the session.

- **Nodes Involved:**  
  - `insert` (DataTable node)

- **Node Details:**  
  - Type: DataTable (Insert operation)  
  - Configuration:  
    - Inserts columns:  
      - `text`: message content from `body.data.content`  
      - `type`: event type from `body.event` (e.g., message:send, message:received)  
      - `session_id`: session identifier from `body.data.session_id`  
    - DataTable ID: points to raw chats table (ID `UTnFJYqs2RBUfKbl`)  
  - Inputs: From `closed` node (triggered only on resolved session events)  
  - Outputs: None (endpoint)  
  - Edge cases:  
    - Datatable connectivity or permission errors  
    - Missing or malformed message data  
  - Sticky Note:  
    - “Add chat to table”  

---

#### 2.4 Data Formatting

- **Overview:**  
  Retrieves all stored messages for the resolved session and processes them into chronological Q&A pairs matching user questions with operator answers.

- **Nodes Involved:**  
  - `get` (DataTable get operation)  
  - `format` (Code node)

- **Node Details:**  
  - `get` Node:  
    - Type: DataTable (Get operation)  
    - Configuration:  
      - Filter: session_id equals incoming event’s session_id  
      - DataTable ID: raw chats table ID (`UTnFJYqs2RBUfKbl`)  
    - Input: True branch of `closed` node  
    - Output: all messages for the session

  - `format` Node:  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Sorts messages by message ID to maintain chronological order  
      - Separates messages into user questions (`message:send`) and operator answers (`message:received`)  
      - Pairs questions and answers in order, allowing for potential missing pairs  
      - Outputs:  
        - `qaPairs`: array of question-answer objects  
        - `formattedConversation`: multi-line text with numbered Q&A pairs  
        - `totalQuestions`, `totalAnswers`: counts for analytics  
        - `conversationSummary`: summary string  
    - Input: messages from `get` node  
    - Output: formatted conversation to next node  
    - Edge cases:  
      - Missing or inconsistent message types  
      - Sessions with unbalanced Q&A pairs  
      - Empty or partial conversations  
  - Sticky Note:  
    - “Chat finished, format the chats into a Q+A”

---

#### 2.5 AI Processing

- **Overview:**  
  Uses the GPT-4.1-mini language model to generate a clean, PII-safe help documentation article from the formatted conversation Q&A pairs.

- **Nodes Involved:**  
  - `Gen Help` (Chain LLM node)  
  - `OpenAI Chat Model` (Language model node)

- **Node Details:**  
  - `OpenAI Chat Model`:  
    - Type: Langchain OpenAI LLM node  
    - Configuration:  
      - Model: `gpt-4.1-mini`  
      - Credentials: OpenAI API key (configured in credentials)  
    - Output: LLM response to `Gen Help`

  - `Gen Help`:  
    - Type: Chain LLM  
    - Configuration:  
      - Prompt:  
        - Instruction to create a help article from conversation Q&A  
        - Explicit requirement to remove or replace all PII (emails, names, org codes, passwords, phone numbers, account numbers, etc.) with generic placeholders  
        - Input variable: `formattedConversation` from previous node  
    - Inputs: LLM response from `OpenAI Chat Model`  
    - Outputs: sanitized help documentation text  
  - Edge cases:  
    - API quota or authentication errors with OpenAI  
    - Model latency or timeout issues  
    - Failure to detect or sanitize all PII (depends on model accuracy)  
  - Sticky Note:  
    - “Create the Helpdoc and store it. You could add an email node or Google docs node after here too.”

---

#### 2.6 Helpdoc Storage

- **Overview:**  
  Saves the generated, sanitized help documentation text into a dedicated helpdocs datatable for storage and later use.

- **Nodes Involved:**  
  - `store-doc` (DataTable insert operation)

- **Node Details:**  
  - Type: DataTable (Insert)  
  - Configuration:  
    - Columns:  
      - `doc`: the help documentation text from AI output  
      - `publish`: boolean field defaulting to false for manual review/publishing  
    - DataTable ID: helpdocs table ID (`cMiRhzsbTqFjiXXW`)  
  - Input: from `Gen Help` node  
  - Outputs: None (endpoint)  
  - Edge cases:  
    - Datatable write failures or permission issues  
    - Empty or incomplete documentation text  
  - Sticky Note: (none)

---

### 3. Summary Table

| Node Name         | Node Type                            | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                     |
|-------------------|------------------------------------|-------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| crisp             | Webhook                            | Receive Crisp chat events           | External HTTP POST      | closed                  | Use this in Crisp/Settings/Workspace settings/Advanced                                          |
| closed            | If                                 | Check if session is resolved        | crisp                  | get, insert              | Check if resolved                                                                              |
| get               | DataTable (Get)                    | Retrieve all messages for session   | closed                  | format                  |                                                                                                |
| insert            | DataTable (Insert)                 | Store raw chat message              | closed                  |                         | Add chat to table                                                                             |
| format            | Code                              | Format messages into Q&A pairs      | get                     | Gen Help                | Chat finished, format the chats into a Q+A                                                    |
| OpenAI Chat Model | Langchain OpenAI LLM               | AI model to generate helpdoc        |                         | Gen Help (ai_languageModel) |                                                                                                |
| Gen Help          | Chain LLM                         | Generate PII-safe help documentation| OpenAI Chat Model, format| store-doc                | Create the Helpdoc and store it. You could add an email node or Google docs node after here too.|
| store-doc         | DataTable (Insert)                 | Store sanitized help documentation  | Gen Help                |                         |                                                                                                |
| Sticky Note       | StickyNote                        | Documentation / setup instructions  |                         |                         | Crisp chat → Helpdoc generator; setup instructions; see sticky note content                   |
| Sticky Note1      | StickyNote                        | Documentation                       |                         |                         | Check if resolved                                                                             |
| Sticky Note2      | StickyNote                        | Documentation                       |                         |                         | Add chat to table                                                                            |
| Sticky Note3      | StickyNote                        | Documentation                       |                         |                         | Chat finished, format the chats into a Q+A                                                   |
| Sticky Note4      | StickyNote                        | Documentation                       |                         |                         | Create the Helpdoc and store it. You could add an email node or Google docs node after here too.|
| Sticky Note5      | StickyNote                        | Documentation / setup instructions  |                         |                         | Use this in Crisp/Settings/Workspace settings/Advanced                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (`crisp`):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Set a unique URL path (e.g., a UUID or recognizable string)  
   - Purpose: Receive Crisp chat event payloads  
   - Save and activate the webhook URL in Crisp under Workspace Settings → Advanced → Webhooks.

2. **Create If Node (`closed`):**  
   - Type: If  
   - Condition: Check if `{{$json["body"]["data"]["content"]["namespace"]}}` equals `"state:resolved"` (case sensitive, strict)  
   - Connect `crisp` node output to this node input.

3. **Create DataTable Get Node (`get`):**  
   - Type: DataTable (Get)  
   - DataTable: Select or create a datatable to store raw chat messages (e.g., "crisp")  
   - Filter: `session_id` equals `{{$json["body"]["data"]["session_id"]}}`  
   - Connect the True output of `closed` node to this node.

4. **Create DataTable Insert Node (`insert`):**  
   - Type: DataTable (Insert)  
   - DataTable: Same raw chat messages datatable as in `get`  
   - Columns:  
     - `text`: `{{$json["body"]["data"]["content"]}}`  
     - `type`: `{{$json["body"]["event"]}}`  
     - `session_id`: `{{$json["body"]["data"]["session_id"]}}`  
   - Connect the True output of `closed` node to this node (parallel to `get`).

5. **Create Code Node (`format`):**  
   - Type: Code (JavaScript)  
   - Input: Connect output of `get` node  
   - Code logic:  
     - Sort messages by `id` ascending  
     - Separate into user questions (`type === "message:send"`) and operator answers (`type === "message:received"`)  
     - Pair questions and answers in chronological order  
     - Build a formatted conversation string with "Q1:", "A1:" style numbering  
     - Return JSON with `qaPairs`, `formattedConversation`, counts, and summary  
   - Example: Use the provided JavaScript logic in the original workflow.

6. **Add OpenAI Chat Model Node (`OpenAI Chat Model`):**  
   - Type: Langchain OpenAI LLM node  
   - Model: `gpt-4.1-mini`  
   - Credentials: Configure with your OpenAI API key (create credentials in n8n)  
   - No inputs; will be triggered by the chain node.

7. **Create Chain LLM Node (`Gen Help`):**  
   - Type: Chain LLM  
   - Input: Connect from `format` node output to this node input  
   - AI Model: Connect the `OpenAI Chat Model` node to the `Gen Help` node’s `ai_languageModel` input  
   - Prompt:  
     ```
     Based on the following customer support conversation Q&A pairs, create a comprehensive help documentation article. Extract the key problem, solution, and any important details. Format it as a clear, reusable help doc entry.

     IMPORTANT: Remove all personal identifying information (PII) from the help documentation including:
     - Email addresses
     - Names of people or organizations
     - Organization codes or IDs
     - Passwords or credentials
     - Phone numbers
     - Account numbers
     - Any other sensitive personal data

     Replace PII with generic placeholders like [USER_EMAIL], [ORGANIZATION_NAME], [ACCOUNT_ID], etc.

     Conversation: {{ $json.formattedConversation }}
     ```
   - No batching needed.

8. **Create DataTable Insert Node (`store-doc`):**  
   - Type: DataTable (Insert)  
   - DataTable: Create/select a datatable for storing sanitized helpdocs (e.g., "crisphelp")  
   - Columns:  
     - `doc`: `{{$json["text"]}}` (text output from `Gen Help`)  
     - `publish`: false (boolean, default)  
   - Connect output of `Gen Help` to this node.

9. **Connect all nodes in this sequence:**  
   `crisp` (Webhook) → `closed` (If)  
   - True branch of `closed` → parallel to `insert` (store raw chat) and `get` (retrieve all messages)  
   - `get` → `format` → `Gen Help` → `store-doc`  
   - `Gen Help` also connects to `OpenAI Chat Model` as AI backend.

10. **Configure Credentials:**  
    - Add OpenAI API credentials in n8n credentials manager for GPT-4.1-mini usage.

11. **Test the workflow:**  
    - Send a resolved Crisp chat event through webhook  
    - Verify raw messages are stored  
    - Confirm helpdoc generation without PII  
    - Check helpdoc is saved in the correct datatable

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Crisp chat → Helpdoc generator: workflow receives Crisp chat events via webhook, triggers on session resolved, stores raw chats, formats Q&A, generates PII-safe helpdoc, and stores it. Setup includes webhook configuration, OpenAI credentials, and datatable setup. Detailed instructions in sticky note.         | Sticky Note at top of workflow                                                                     |
| Crisp webhook URL must be added in Crisp Workspace Settings → Advanced → Webhooks to enable event delivery.                                                                                                                                                                                                 | Sticky Note5                                                                                       |
| You can extend this workflow by adding nodes after helpdoc generation to email the document, post it to Google Docs, or publish directly.                                                                                                                                                                  | Sticky Note4                                                                                       |

---

**Disclaimer:** The text provided is generated exclusively from an n8n automated workflow. The process complies fully with content policies and contains no illegal, offensive, or protected material. All data handled are legal and publicly accessible.