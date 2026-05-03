Manage KlickTipp Contacts via Telegram Bot with GPT-4o Agent

https://n8nworkflows.xyz/workflows/manage-klicktipp-contacts-via-telegram-bot-with-gpt-4o-agent-5747


# Manage KlickTipp Contacts via Telegram Bot with GPT-4o Agent

---

### 1. Workflow Overview

This workflow enables managing KlickTipp contacts through a Telegram bot interface, leveraging an AI agent powered by GPT-4o to interpret natural language commands. It facilitates intuitive contact management by translating user queries into KlickTipp API calls.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures incoming messages from Telegram or n8n chat interface, distinguishing between text and voice.

- **1.2 Audio Processing:** Downloads and transcribes voice messages to text for further processing.

- **1.3 Text Preparation:** Extracts and sets the user message text uniformly from Telegram or n8n chat sources.

- **1.4 AI Agent Interaction:** Uses GPT-4o to interpret user input, maintain conversation memory, and route commands to KlickTipp API tools.

- **1.5 KlickTipp API Operations:** A comprehensive set of nodes handling contacts, tags, opt-in processes, data fields, and redirects.

- **1.6 Output Handling:** Sends processed results back to the Telegram user or performs no operation if the source is not Telegram.

- **1.7 Utility Tools:** Helper nodes for time conversion and message type/source checks.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
Receives user messages from either Telegram bot or n8n chat interface, detecting message type to route accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- When chat message received  
- Check the message type  
- Check the source  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages (text or voice).  
  - Config: Watches for "message" updates; linked to Telegram bot credentials.  
  - Inputs: External Telegram messages.  
  - Outputs: Passes message JSON downstream.  
  - Failure modes: Telegram API downtime, invalid webhook setup.

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Receives messages from n8n chat interface alternative to Telegram.  
  - Config: Webhook mode, public with n8n user authentication.  
  - Inputs: External webhook calls.  
  - Outputs: Passes chat input downstream.  
  - Failure modes: Webhook misconfiguration, auth failures.

- **Check the message type**  
  - Type: Switch  
  - Role: Detects if incoming Telegram message is voice or text.  
  - Config: Conditions check existence of `message.voice` or `message.text`.  
  - Inputs: Telegram Trigger output.  
  - Outputs: Branches to voice or text processing nodes.  
  - Failure modes: Unexpected message formats.

- **Check the source**  
  - Type: Switch  
  - Role: Determines if input originated from Telegram or n8n chat.  
  - Config: Checks if Telegram Trigger executed.  
  - Inputs: KlickTipp Agent output (interpreted message).  
  - Outputs: Routes to Telegram response or no-operation node.  
  - Failure modes: Misrouting if node execution flags are inaccurate.

---

#### 1.2 Audio Processing

**Overview:**  
Handles Telegram voice messages by downloading the file and transcribing audio to text for AI processing.

**Nodes Involved:**  
- Download voice file  
- Transcribe audio  

**Node Details:**

- **Download voice file**  
  - Type: Telegram  
  - Role: Downloads the voice message file from Telegram servers.  
  - Config: Uses `file_id` from incoming Telegram voice message.  
  - Inputs: Telegram Trigger (voice message branch).  
  - Outputs: Binary audio data to transcription node.  
  - Failure modes: Telegram file download errors, missing file ID.

- **Transcribe audio**  
  - Type: Langchain OpenAI audio operation  
  - Role: Converts audio to text using OpenAI Whisper transcription.  
  - Config: Uses OpenAI API credentials, transcribes binary property "data".  
  - Inputs: Binary audio from Download voice file.  
  - Outputs: Transcribed text passed to KlickTipp Agent.  
  - Failure modes: OpenAI API errors, unsupported audio format.

---

#### 1.3 Text Preparation

**Overview:**  
Standardizes the message text from either Telegram or n8n chat input for AI agent consumption.

**Nodes Involved:**  
- Set text from Telegram  
- Set text from the n8n chat  

**Node Details:**

- **Set text from Telegram**  
  - Type: Set  
  - Role: Extracts `message.text` from Telegram payload and sets it as `text` variable.  
  - Config: Assigns `text` = `{{$json.message.text}}`.  
  - Inputs: From Check the message type (text branch).  
  - Outputs: Passes text to KlickTipp Agent.  
  - Failure modes: Missing text field.

- **Set text from the n8n chat**  
  - Type: Set  
  - Role: Extracts `chatInput` from n8n chat webhook payload and sets `text`.  
  - Config: Assigns `text` = `{{$json.chatInput}}`.  
  - Inputs: From When chat message received.  
  - Outputs: Passes text to KlickTipp Agent.  
  - Failure modes: Missing chatInput field.

---

#### 1.4 AI Agent Interaction

**Overview:**  
Central AI node interprets user requests, maintains session memory, and calls appropriate KlickTipp API tools based on intent.

**Nodes Involved:**  
- KlickTipp Agent  
- OpenAI Chat Model  
- Simple Memory  

**Node Details:**

- **KlickTipp Agent**  
  - Type: Langchain Agent  
  - Role: Core AI processing node that interprets messages, manages dialogue, and orchestrates KlickTipp API calls.  
  - Config:  
    - Uses system prompt defining role as KlickTipp AI assistant with language detection and multi-playbook logic.  
    - Enforces strict rules: no data invention, always call API tools, multi-language support (German/English).  
    - Handles contact data enrichment, detailed playbooks for contact/tag management, and error handling.  
  - Inputs: Receives `text` from Set nodes or transcribed audio.  
  - Outputs: Calls multiple KlickTipp API nodes as tools, then outputs processed results.  
  - Utilizes Simple Memory for session continuity keyed by Telegram user ID or chat session.  
  - Failure modes: AI model API issues, prompt failures, session key errors, ambiguous user input requiring human clarification.

- **OpenAI Chat Model**  
  - Type: Langchain LLM Chat Model  
  - Role: Provides GPT-4o model backend for the KlickTipp Agent.  
  - Config: Model set explicitly to "gpt-4o" with OpenAI API credentials.  
  - Inputs: Receives prompt instructions from KlickTipp Agent.  
  - Outputs: AI-generated text/commands for KlickTipp Agent.  
  - Failure modes: API throttling, credential errors.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Stores recent conversation history indexed by user session.  
  - Config: Session key dynamically set from Telegram user ID or n8n chat session ID.  
  - Inputs: Conversation data from KlickTipp Agent.  
  - Outputs: Provides memory context for subsequent AI processing.  
  - Failure modes: Memory store corruption or key mismatch.

---

#### 1.5 KlickTipp API Operations

**Overview:**  
A comprehensive suite of nodes performing all KlickTipp platform operations related to contacts, tags, opt-ins, data fields, and redirects.

**Nodes Involved:**  
- Contact Management  
  - List Contacts  
  - Get Contact  
  - Get Contact ID  
  - Add or Update Contact  
  - Update Contact  
  - Delete Contact  
  - Unsubscribe Contact  

- Tag Operations  
  - List Tags  
  - Get Tag  
  - Create Tag  
  - Update Tag  
  - Delete Tag  
  - Tag Contact  
  - Untag Contact  
  - List Tagged Contacts  

- Opt-in Processes  
  - List Opt-in Processes  
  - Get Opt-in Process  
  - Get Redirect URL  

- Data Fields  
  - List Data Fields  
  - Get Data Field  

**Node Details (common for KlickTipp nodes):**

- Type: KlickTipp Tool node (custom n8n node for KlickTipp API)  
- Role: Interact with KlickTipp API endpoints to manage subscribers, tags, opt-ins, and data fields.  
- Config:  
  - Each node specifies resource and operation (e.g., subscriber/get, contact-tagging/tag).  
  - Parameters often use AI-extracted variables via `$fromAI()` expressions to dynamically pass required data.  
  - Authenticated with KlickTipp API credentials configured per node.  
- Inputs: Invoked by KlickTipp Agent as AI tools.  
- Outputs: Return API responses for downstream processing or AI agent consumption.  
- Failure modes: API authentication errors, invalid parameters, network timeouts, rate limits.

---

#### 1.6 Output Handling

**Overview:**  
Sends the AI agent’s response back to the Telegram user or performs no operation for non-Telegram sources.

**Nodes Involved:**  
- Telegram  
- No Operation, do nothing  

**Node Details:**

- **Telegram**  
  - Type: Telegram node  
  - Role: Sends text messages to Telegram users with AI response output.  
  - Config: Uses chatId from Telegram message sender and text from AI output property.  
  - Inputs: From Check the source node (Telegram branch).  
  - Outputs: Final message sent to user.  
  - Failure modes: Telegram API errors, invalid chatId.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Placeholder node for non-Telegram sources to prevent unwanted actions.  
  - Config: No parameters.  
  - Inputs: From Check the source (non-Telegram branch).  
  - Outputs: Terminates workflow silently.  
  - Failure modes: None.

---

#### 1.7 Utility Tools

**Overview:**  
Additional helper nodes for date/time conversions and interface indications helping the AI agent with date fields.

**Nodes Involved:**  
- Timestamp to date  
- Date to timestamp  

**Node Details:**

- **Timestamp to date**  
  - Type: Langchain ToolCode  
  - Role: Converts Unix timestamp input (seconds) to formatted date string in Europe/Berlin timezone.  
  - Config: JavaScript code parses and formats timestamp or returns "N/A" for invalid input.  
  - Inputs: Timestamp field from AI or KlickTipp contact data.  
  - Outputs: Formatted date string.  
  - Failure modes: Invalid or missing timestamp inputs.

- **Date to timestamp**  
  - Type: Langchain ToolCode  
  - Role: Parses various human-readable date formats and converts to Unix timestamp in seconds.  
  - Config: Tries multiple date formats, falls back to ISO/SQL parsing in Europe/Berlin timezone.  
  - Inputs: Date string from AI input.  
  - Outputs: Unix timestamp or "N/A".  
  - Failure modes: Unparseable dates.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                            | Input Node(s)                  | Output Node(s)                           | Sticky Note                                                                                         |
|---------------------------|-------------------------------------|--------------------------------------------|-------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger              | Receives messages from n8n chat interface  | External webhook              | Set text from the n8n chat               |                                                                                                   |
| OpenAI Chat Model          | Langchain LLM Chat Model            | LLM backend for AI agent                    | KlickTipp Agent (lmChatOpenAi) | KlickTipp Agent                         |                                                                                                   |
| Sticky Note4              | Sticky Note                         | Notes on connecting agents with KlickTipp  | -                             | -                                       | ## 2. Connect any Agent with a KlickTipp tools                                                     |
| Simple Memory              | Langchain Memory Buffer Window      | Stores conversation memory by user session | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Sticky Note               | Sticky Note                         | Community node disclaimer and workflow description | -                             | -                                       | Community Node Disclaimer: As this workflow relies on a community node, it is limited to self-hosted environments. ... |
| List Contacts             | KlickTipp Tool                     | Lists all contacts                          | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Get Contact              | KlickTipp Tool                     | Gets detailed contact data by ID            | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Get Contact ID           | KlickTipp Tool                     | Searches contact ID by email                 | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| List Tagged Contacts     | KlickTipp Tool                     | Lists contacts tagged with specific tag     | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| List Tags                | KlickTipp Tool                     | Lists all tags                               | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Update Contact           | KlickTipp Tool                     | Updates a contact by ID                      | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Delete Contact           | KlickTipp Tool                     | Deletes a contact                            | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Unsubscribe Contact      | KlickTipp Tool                     | Unsubscribes a contact                       | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Create Tag               | KlickTipp Tool                     | Creates a new tag                            | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Get Tag                  | KlickTipp Tool                     | Gets tag details by ID                      | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Delete Tag               | KlickTipp Tool                     | Deletes a tag                               | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Update Tag               | KlickTipp Tool                     | Updates a tag                               | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| List Opt-in Processes    | KlickTipp Tool                     | Lists all opt-in processes                   | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Get Opt-in Process       | KlickTipp Tool                     | Gets opt-in process details by ID           | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| List Data Fields         | KlickTipp Tool                     | Lists all data fields                        | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Get Data Field           | KlickTipp Tool                     | Gets data field info by ID                   | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Untag Contact            | KlickTipp Tool                     | Removes a tag from a contact                 | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Tag Contact              | KlickTipp Tool                     | Adds one or more tags to a contact           | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Download voice file      | Telegram                           | Downloads voice message audio file           | Telegram Trigger (voice)      | Transcribe audio                        |                                                                                                   |
| Transcribe audio         | Langchain OpenAI Audio             | Transcribes audio to text                     | Download voice file           | KlickTipp Agent                         |                                                                                                   |
| Telegram                 | Telegram                           | Sends reply messages to Telegram users       | Check the source (Telegram)   | -                                       |                                                                                                   |
| No Operation, do nothing | NoOp                              | Ends workflow silently for non-Telegram inputs | Check the source (non-Telegram) | -                                       |                                                                                                   |
| Timestamp to date        | Langchain ToolCode                 | Converts Unix timestamp to date string       | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Date to timestamp        | Langchain ToolCode                 | Converts date string to Unix timestamp        | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |
| Check the message type   | Switch                            | Determines if message is voice or text        | Telegram Trigger             | Download voice file / Set text from Telegram |                                                                                                   |
| Set text from Telegram   | Set                               | Sets text variable from Telegram message      | Check the message type (text) | KlickTipp Agent                         |                                                                                                   |
| Set text from the n8n chat | Set                             | Sets text variable from n8n chat input        | When chat message received    | KlickTipp Agent                         |                                                                                                   |
| Check the source         | Switch                            | Routes response to Telegram or no-op           | KlickTipp Agent               | Telegram / No Operation                 |                                                                                                   |
| KlickTipp Agent          | Langchain Agent                  | AI assistant interpreting commands and routing | Set text nodes / Transcribe audio | Check the source                       |                                                                                                   |
| Add or Update Contact   | KlickTipp Tool                     | Adds or updates contact (subscribe operation) | KlickTipp Agent               | KlickTipp Agent                         |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials (bot token).  
   - Set updates to listen for "message".  
   - Position as input entry for Telegram messages.

2. **Create When chat message received node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook mode with public access and n8n user authentication.  
   - Acts as alternative input source for chat messages.

3. **Create Check the message type switch node**  
   - Type: Switch  
   - Input: Telegram Trigger output.  
   - Add two rules:  
     - "Is voice message?": Check if `message.voice` exists.  
     - "Is text message?": Check if `message.text` exists.

4. **Create Download voice file node**  
   - Type: Telegram  
   - Input: Connect from "Is voice message?" output of the previous node.  
   - Configure `fileId` to `{{$json.message.voice.file_id}}`.  
   - Use Telegram bot credentials.  

5. **Create Transcribe audio node**  
   - Type: Langchain OpenAI (audio transcription)  
   - Input: Connect from Download voice file.  
   - Configure binary property name as "data".  
   - Use OpenAI API credentials.

6. **Create Set text from Telegram node**  
   - Type: Set  
   - Input: Connect from "Is text message?" output of Check the message type.  
   - Define assignment: `text = {{$json.message.text}}`.

7. **Create Set text from the n8n chat node**  
   - Type: Set  
   - Input: Connect from When chat message received node.  
   - Define assignment: `text = {{$json.chatInput}}`.

8. **Create OpenAI Chat Model node**  
   - Type: Langchain LLM Chat Model  
   - Configure model to "gpt-4o".  
   - Use OpenAI API credentials.

9. **Create Simple Memory node**  
   - Type: Langchain Memory Buffer Window  
   - Set session key expression: `={{ $('Telegram Trigger').isExecuted ? $('Telegram Trigger').item.json.message.from.id : $('When chat message received').item.json.sessionId }}`.

10. **Create KlickTipp Agent node**  
    - Type: Langchain Agent  
    - Connect inputs from Set text from Telegram, Set text from n8n chat, and Transcribe audio.  
    - Configure with system message prompt defining AI role, language protocol, playbooks, and critical execution rules as detailed in the overview.  
    - Set AI language model input to OpenAI Chat Model node.  
    - Connect memory input to Simple Memory node.

11. **Connect KlickTipp Agent outputs to KlickTipp API nodes:**  
    - Create KlickTipp Tool nodes for all required resources and operations:  
      - Contact management (list, get, update, delete, unsubscribe, add/update).  
      - Tag operations (list, create, get, update, delete, tag, untag, list tagged).  
      - Opt-in processes (list, get, redirect URL).  
      - Data fields (list, get).  
    - Use AI variable expressions with `$fromAI()` to pass parameters dynamically.  
    - Authenticate each with your KlickTipp API credentials.

12. **Create Check the source switch node**  
    - Input: Connect from KlickTipp Agent output.  
    - Rule 1: "Is Telegram" if Telegram Trigger executed.  
    - Rule 2: "Is n8n chat" if Telegram Trigger not executed.

13. **Create Telegram node (send message)**  
    - Input: Connect from "Is Telegram" output of Check the source.  
    - Configure to send text from AI output (`{{$json.output}}`) to the Telegram user (`{{$json.message.from.id}}`).  
    - Use Telegram bot credentials.

14. **Create No Operation node**  
    - Input: Connect from "Is n8n chat" output of Check the source.  
    - No parameters; acts as silent end.

15. **Create Utility nodes:**  
    - Timestamp to date (Langchain ToolCode) to convert Unix timestamps to formatted date strings.  
    - Date to timestamp (Langchain ToolCode) to convert date strings to Unix timestamps.

16. **Add Sticky Notes** for documentation blocks in workflow canvas to clarify sections for users.

17. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Community Node Disclaimer: As this workflow relies on a community node, it is limited to self-hosted environments.                                                                                                                                                                                                                                                                                                                                                                                                             | Workflow description sticky note.                                                                |
| Telegram & LLM Interaction Setup: Captures messages via Telegram bot, maintains conversation state, interprets input with LLM, routes commands to KlickTipp API, sends responses back.                                                                                                                                                                                                                                                                                                                                       | Workflow overview in sticky notes.                                                              |
| KlickTipp Integration: Full API coverage for contact, tag, opt-in, data fields, and redirects management.                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow overview in sticky notes.                                                              |
| Use Cases Supported: Query contacts, segment by attributes, create/update contacts, manage tags, automate campaigns via Telegram bot.                                                                                                                                                                                                                                                                                                                                                                                         | Workflow overview in sticky notes.                                                              |
| Setup Instructions: Create Telegram bot with BotFather, configure Telegram Trigger, set up OpenAI and KlickTipp credentials, activate workflow.                                                                                                                                                                                                                                                                                                                                                                               | Workflow overview in sticky notes with setup steps.                                             |
| AI Agent Operational Principles: Customer obsession, understand first, AI first, focus on impact, own mistakes, follow strict rules to only output fetched data, multi-language support, and detailed playbooks for contact/tag workflows.                                                                                                                                                                                                                                                                                   | KlickTipp Agent node system prompt content.                                                    |
| Testing and Deployment: Use example Telegram queries like “Tell me something about the contact with email X”, “Tag all contacts from region Y”, “Send campaign Z to customers in area A”. Validate KlickTipp results and Telegram responses.                                                                                                                                                                                                                                                                                 | Workflow overview in sticky notes.                                                              |
| Support Links: For KlickTipp issues, refer users to https://www.klicktipp.com/de/support/                                                                                                                                                                                                                                                                                                                                                                                                                                      | KlickTipp Agent node system prompt and sticky notes.                                           |
| Date Conversion Utilities use Europe/Berlin timezone and multiple date formats to ensure robust parsing and formatting.                                                                                                                                                                                                                                                                                                                                                                                                       | Date to timestamp and timestamp to date nodes.                                                 |

---

**Disclaimer:** The provided text is generated from an n8n workflow automation and adheres to all content policies. It contains no illegal or offensive elements and only processes legal, public data.

---