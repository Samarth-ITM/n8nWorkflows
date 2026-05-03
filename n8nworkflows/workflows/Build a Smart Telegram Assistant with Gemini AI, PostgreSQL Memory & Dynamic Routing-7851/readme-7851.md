Build a Smart Telegram Assistant with Gemini AI, PostgreSQL Memory & Dynamic Routing

https://n8nworkflows.xyz/workflows/build-a-smart-telegram-assistant-with-gemini-ai--postgresql-memory---dynamic-routing-7851


# Build a Smart Telegram Assistant with Gemini AI, PostgreSQL Memory & Dynamic Routing

### 1. Workflow Overview

This workflow implements a **Smart Telegram Assistant** that leverages Google Gemini AI models, PostgreSQL for chat memory, and dynamic routing to handle different message types and complexities. Its goal is to provide intelligent, context-aware, and efficient responses to Telegram users by summarizing conversation history, dynamically selecting AI models based on task difficulty, and maintaining conversation context in a database.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & Message Type Routing:** Receives Telegram messages, identifies message type (text, voice, unsupported), and preprocesses accordingly.
- **1.2 Voice Message Handling:** Downloads, normalizes, and transcribes voice messages via Google Gemini AI.
- **1.3 Chat Memory Management & Context Summarization:** Retrieves past conversation history from PostgreSQL, aggregates, and summarizes it with difficulty classification.
- **1.4 AI Agent Processing & Dynamic Model Selection:** Routes the query and context to an AI agent, selects an appropriate Gemini model based on difficulty, formats output, sends Telegram messages, and updates chat memory.
- **1.5 Database Initialization & Setup:** Creates the PostgreSQL table for chat memory if not present.
- **1.6 Utility & Error Handling:** Includes MIME type fixes and error message generation for unsupported inputs.
- **1.7 User Experience Enhancements:** Shows typing status and formats messages for Telegram MarkdownV2 compatibility.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Message Type Routing

- **Overview:** Handles incoming Telegram messages, detects their type, and routes them to the appropriate processing path (text, voice, or error for unsupported media).
- **Nodes Involved:**  
  - Telegram Trigger  
  - Input Message Router1  
  - get_message (text)  
  - get_error_message  

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point listening for new Telegram messages.  
    - *Config:* Monitors "message" updates.  
    - *Connections:* Outputs to Input Message Router1 and Typing… node.  
    - *Edge Cases:* Telegram API connection issues; webhook misconfiguration.

  - **Input Message Router1**  
    - *Type:* Switch  
    - *Role:* Classifies message as Text, Voice Message, or unsupported (extra).  
    - *Config:* Checks existence of `message.text` for Text, `message.voice` for Voice Message.  
    - *Connections:* Routes to get_message (text), Download Voice Message, or get_error_message.  
    - *Edge Cases:* Messages without text or voice fields; unexpected input types.

  - **get_message (text)**  
    - *Type:* Set  
    - *Role:* Extracts and sets normalized text message and chat_id.  
    - *Config:* Assigns `message` from Telegram text, `chat_id` from chat.id.  
    - *Connections:* Routes to Normalize input node.  
    - *Edge Cases:* Missing text field; malformed message JSON.

  - **get_error_message**  
    - *Type:* Set  
    - *Role:* Sets error message and chat_id for unsupported inputs.  
    - *Config:* Returns fixed error string indicating unsupported file type.  
    - *Connections:* Routes to Normalize input node.  
    - *Edge Cases:* Proper error messaging on unsupported media inputs.

---

#### 2.2 Voice Message Handling

- **Overview:** Processes voice messages by downloading the audio file, fixing MIME types, transcribing with AI, and extracting transcript text.
- **Nodes Involved:**  
  - Download Voice Message  
  - Fix mime  
  - Analyze voice message  
  - get_message (Audio/Video message)  

- **Node Details:**

  - **Download Voice Message**  
    - *Type:* Telegram  
    - *Role:* Downloads the voice message file by file_id.  
    - *Config:* Uses `message.voice.file_id` from Telegram update.  
    - *Connections:* Routes to Fix mime node.  
    - *Edge Cases:* File download failures; invalid file_id.

  - **Fix mime**  
    - *Type:* Code  
    - *Role:* Corrects MIME types based on file extensions for better compatibility.  
    - *Config:* Uses a comprehensive extension-to-MIME map for common media formats.  
    - *Connections:* Routes to Analyze voice message node.  
    - *Edge Cases:* Unknown extensions; missing file names.

  - **Analyze voice message**  
    - *Type:* Google Gemini Audio Analysis  
    - *Role:* Uses Gemini AI to transcribe/analyze the audio message.  
    - *Config:* Model "models/gemini-2.5-pro"; resource type "audio"; input type binary.  
    - *Connections:* Routes to get_message (Audio/Video message).  
    - *Edge Cases:* API errors; transcription inaccuracies; timeout.

  - **get_message (Audio/Video message)**  
    - *Type:* Set  
    - *Role:* Extracts transcription text from Gemini output and sets chat_id.  
    - *Config:* Sets `message` from transcription content; sets `chat_id` from Telegram.  
    - *Connections:* Routes to Normalize input node.  
    - *Edge Cases:* Missing transcription data; partial results.

---

#### 2.3 Chat Memory Management & Context Summarization

- **Overview:** Retrieves previous chat messages from PostgreSQL, aggregates them, summarizes relevant context, and classifies difficulty for model selection.
- **Nodes Involved:**  
  - Normalize input  
  - Get Chat Memory  
  - Aggregate  
  - Summarize & Categorize  
  - Structured Output Parser  
  - Google Gemini 2.5 Flash Lite  

- **Node Details:**

  - **Normalize input**  
    - *Type:* Set  
    - *Role:* Normalizes incoming message object with standardized keys `message` and `chat_id`.  
    - *Connections:* Routes to Get Chat Memory node.

  - **Get Chat Memory**  
    - *Type:* PostgreSQL Select  
    - *Role:* Retrieves last 25 messages for current `session_id` (chat_id).  
    - *Config:* Selects from `chat_memory` table, schema `public`.  
    - *Connections:* Routes to Aggregate node.  
    - *Edge Cases:* DB connection failures; empty history.

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Concatenates retrieved messages into a single aggregated string.  
    - *Connections:* Routes to Summarize & Categorize node.

  - **Summarize & Categorize**  
    - *Type:* Chain LLM (Google Gemini)  
    - *Role:* Summarizes chat history and classifies request difficulty (1 to 3) with structured JSON output.  
    - *Config:* Uses Gemini 2.5 Flash Lite model (low cost, low latency).  
    - *Prompt:* Includes chat memory and current user request.  
    - *Connections:* Output parsed by Structured Output Parser; then routes to Agent node.  
    - *Edge Cases:* Parsing errors; model response inconsistencies.

  - **Structured Output Parser**  
    - *Type:* Output Parser (Structured)  
    - *Role:* Enforces JSON schema on Summarize & Categorize output with fields `difficulty` (enum 1-3) and `context` (string).  
    - *Connections:* Routes to Summarize & Categorize node.

---

#### 2.4 AI Agent Processing & Dynamic Model Selection

- **Overview:** Uses the summarized context and user message to generate a response with the appropriate Gemini AI model based on difficulty; formats result for Telegram and updates chat memory.
- **Nodes Involved:**  
  - Agent  
  - Model Selector  
  - Gemini 2.5 Flash Lite  
  - Gemini 2.5 Flash  
  - Gemini 2.5 Pro  
  - MarkdownV2  
  - Send a text message  
  - Update Chat Memory (User and Agent)  

- **Node Details:**

  - **Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Generates AI response using user message and context.  
    - *Config:* Prompt includes context and user message; returns intermediate steps.  
    - *Connections:* Routes to MarkdownV2 node.

  - **Model Selector**  
    - *Type:* LangChain Model Selector  
    - *Role:* Selects Gemini model based on difficulty:  
      - 1 → Gemini 2.5 Flash Lite (fastest, cheapest)  
      - 2 → Gemini 2.5 Flash  
      - 3 → Gemini 2.5 Pro (advanced reasoning)  
    - *Connections:* Routes to Agent node.

  - **Gemini 2.5 Flash Lite / Flash / Pro**  
    - *Type:* Google Gemini AI LLM (Chat)  
    - *Role:* Execute chat completions per selected model.  
    - *Config:* Credentials use Google Palm API account.  
    - *Connections:* Outputs to Model Selector or Summarize & Categorize as applicable.

  - **MarkdownV2**  
    - *Type:* Code  
    - *Role:* Converts AI output to Telegram-safe MarkdownV2, chunking messages to fit Telegram limits.  
    - *Connections:* Routes to Send a text message.

  - **Send a text message**  
    - *Type:* Telegram  
    - *Role:* Sends formatted message back to Telegram chat.  
    - *Config:* Uses MarkdownV2 parse mode, disables attribution, sends to current chat_id.  
    - *Connections:* None (end of message sending).

  - **Update Chat Memory (User and Agent)**  
    - *Type:* PostgreSQL Insert  
    - *Role:* Inserts combined user and AI agent message as one row to chat_memory table for session persistence.  
    - *Connections:* None (end of memory update).  
    - *Edge Cases:* DB errors, message encoding issues.

---

#### 2.5 Database Initialization & Setup

- **Overview:** Creates the `chat_memory` table and necessary indexes in PostgreSQL if not already present to store conversation history.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Create Chat Memory Table  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual execution for initial setup.  
    - *Connections:* Routes to Create Chat Memory Table node.

  - **Create Chat Memory Table**  
    - *Type:* PostgreSQL Execute Query  
    - *Role:* Runs SQL to create table and index if missing.  
    - *SQL:* Creates `chat_memory` table with fields: id (serial primary key), session_id (varchar), message (text). Index on session_id for performance.  
    - *Connections:* None.  
    - *Edge Cases:* Permissions or connection issues.

---

#### 2.6 Utility & Error Handling

- **Overview:** Handles MIME type normalization for media files and graceful error messaging for unsupported inputs.
- **Nodes Involved:**  
  - Fix mime  
  - get_error_message  

- Details covered in previous blocks.

---

#### 2.7 User Experience Enhancements

- **Overview:** Improves user experience by showing the “typing…” indicator in Telegram and ensuring message formatting compatibility.
- **Nodes Involved:**  
  - Typing…  
  - MarkdownV2  

- **Node Details:**

  - **Typing…**  
    - *Type:* Telegram  
    - *Role:* Sends chat action "typing" to indicate bot is processing.  
    - *Config:* Uses chat_id from incoming message.  
    - *Edge Cases:* Telegram API limits or errors.

  - **MarkdownV2**  
    - See above for detailed description.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                                      | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                               |
|-------------------------------|--------------------------------------|-----------------------------------------------------|-----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger                     | Entry point for Telegram messages                    |                             | Input Message Router1, Typing…    | ## 🔵 Input Handling (Telegram Trigger & Preprocessing) ... [Includes link to multimodal extension template]              |
| Input Message Router1         | Switch                              | Routes messages by type: Text / Voice / Unsupported | Telegram Trigger            | get_message (text), Download Voice Message, get_error_message |                                                                                                                           |
| get_message (text)            | Set                                 | Extracts text message and chat_id                     | Input Message Router1       | Normalize input                  |                                                                                                                           |
| get_error_message             | Set                                 | Sets error message for unsupported inputs            | Input Message Router1       | Normalize input                  |                                                                                                                           |
| Download Voice Message        | Telegram                            | Downloads voice audio file from Telegram              | Input Message Router1       | Fix mime                        |                                                                                                                           |
| Fix mime                     | Code                                | Corrects MIME types based on file extension           | Download Voice Message      | Analyze voice message            |                                                                                                                           |
| Analyze voice message         | Google Gemini Audio Analysis        | Transcribes voice message using Gemini AI             | Fix mime                    | get_message (Audio/Video message)|                                                                                                                           |
| get_message (Audio/Video message) | Set                             | Extracts transcription text and chat_id               | Analyze voice message       | Normalize input                  |                                                                                                                           |
| Normalize input              | Set                                 | Standardizes input with `message` and `chat_id`       | get_message (text), get_message (Audio/Video message), get_error_message | Get Chat Memory                  |                                                                                                                           |
| Get Chat Memory              | PostgreSQL Select                   | Retrieves last 25 chat messages for session           | Normalize input             | Aggregate                      |                                                                                                                           |
| Aggregate                   | Aggregate                           | Concatenates retrieved chat messages                   | Get Chat Memory             | Summarize & Categorize          | ## 🔴 Chat Memory Retrieval & Context Optimization ...                                                                        |
| Summarize & Categorize       | Chain LLM (Google Gemini)           | Summarizes chat history & classifies difficulty        | Aggregate                   | Agent                         |                                                                                                                           |
| Structured Output Parser     | Output Parser (Structured)           | Ensures Summarize & Categorize outputs valid JSON     | Summarize & Categorize      | Summarize & Categorize          |                                                                                                                           |
| Agent                       | LangChain Agent                     | Generates AI response with user message and context    | Summarize & Categorize      | MarkdownV2                    | ## 🟣 Agent Processing & Response Delivery ...                                                                              |
| Model Selector              | LangChain Model Selector             | Selects Gemini model based on difficulty                | Gemini 2.5 Flash Lite, Gemini 2.5 Flash, Gemini 2.5 Pro | Agent                         |                                                                                                                           |
| Gemini 2.5 Flash Lite        | Google Gemini LLM                    | Low-cost summarization and difficulty classification   |                             | Model Selector, Summarize & Categorize |                                                                                                                           |
| Gemini 2.5 Flash             | Google Gemini LLM                    | Mid-level model for moderate difficulty tasks          |                             | Model Selector                  |                                                                                                                           |
| Gemini 2.5 Pro               | Google Gemini LLM                    | Advanced model for complex queries                      |                             | Model Selector                  |                                                                                                                           |
| MarkdownV2                  | Code                                | Formats AI output to Telegram MarkdownV2 with chunking | Agent                       | Send a text message            |                                                                                                                           |
| Send a text message         | Telegram                            | Sends the formatted response message to Telegram chat  | MarkdownV2                  |                              |                                                                                                                           |
| Update Chat Memory (User and Agent) | PostgreSQL Insert               | Stores combined user and agent messages into database | MarkdownV2                  |                              |                                                                                                                           |
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual start for DB setup                               |                             | Create Chat Memory Table        | ## ⚙️ Database Initialization (Chat Memory Table) ...                                                                       |
| Create Chat Memory Table     | PostgreSQL Execute Query            | Creates chat_memory table and index if absent          | When clicking ‘Execute workflow’ |                              |                                                                                                                           |
| Typing…                    | Telegram                            | Sends “typing” status to Telegram                        | Telegram Trigger            |                              |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Config: Listen to "message" updates  
   - Credential: Telegram API account  
   - Connect output to Input Message Router1 and Typing… node.

2. **Create Input Message Router1 node**  
   - Type: Switch  
   - Rules:  
     - Output "Text": if `message.text` exists  
     - Output "Voice Message": if `message.voice` exists  
     - Output "extra": fallback  
   - Connect "Text" output to get_message (text) node  
   - Connect "Voice Message" output to Download Voice Message node  
   - Connect "extra" output to get_error_message node.

3. **Create get_message (text) node**  
   - Type: Set  
   - Assign:  
     - `message` = `{{$json["message"]["text"]}}` from Telegram Trigger  
     - `chat_id` = `{{$json["message"]["chat"]["id"]}}`  
   - Connect to Normalize input node.

4. **Create get_error_message node**  
   - Type: Set  
   - Assign:  
     - `message` = "It was not possible to process the file.File type not supported."  
     - `chat_id` = same as Telegram Trigger chat id  
   - Connect to Normalize input node.

5. **Create Download Voice Message node**  
   - Type: Telegram  
   - Config: Download file using `message.voice.file_id`  
   - Connect to Fix mime node.

6. **Create Fix mime node**  
   - Type: Code  
   - Paste comprehensive MIME type mapping JS code for file extension correction  
   - Connect to Analyze voice message node.

7. **Create Analyze voice message node**  
   - Type: Google Gemini (Audio analysis)  
   - Model: "models/gemini-2.5-pro"  
   - Input type: binary (audio)  
   - Credential: Google Palm API account  
   - Connect to get_message (Audio/Video message) node.

8. **Create get_message (Audio/Video message) node**  
   - Type: Set  
   - Assign:  
     - `message` = transcription text from Gemini output  
     - `chat_id` = Telegram chat id  
   - Connect to Normalize input node.

9. **Create Normalize input node**  
   - Type: Set  
   - Assign standardized keys:  
     - `message` from previous node  
     - `chat_id` from previous node  
   - Connect to Get Chat Memory node.

10. **Create Get Chat Memory node**  
    - Type: PostgreSQL Select  
    - Config:  
      - Table: chat_memory  
      - Schema: public  
      - Limit: 25  
      - Filter by `session_id` = `chat_id`  
    - Credential: PostgreSQL account  
    - Connect to Aggregate node.

11. **Create Aggregate node**  
    - Type: Aggregate  
    - Config: Concatenate field `message` from DB rows into single string  
    - Connect to Summarize & Categorize node.

12. **Create Summarize & Categorize node**  
    - Type: Chain LLM (Google Gemini)  
    - Model: Gemini 2.5 Flash Lite (low cost)  
    - Prompt: Includes chat memory and user request, instructing to summarize and classify difficulty (1-3)  
    - Connect to Structured Output Parser node.

13. **Create Structured Output Parser node**  
    - Type: Output Parser (Structured)  
    - Schema: JSON with `difficulty` (integer enum 1-3) and `context` (string)  
    - Connect output back to Summarize & Categorize node (to parse output).

14. **Create Model Selector node**  
    - Type: LangChain Model Selector  
    - Rules:  
      - If difficulty=1 → Gemini 2.5 Flash Lite  
      - If difficulty=2 → Gemini 2.5 Flash  
      - If difficulty=3 → Gemini 2.5 Pro  
    - Connect selected model output to Agent node.

15. **Create Agent node**  
    - Type: LangChain Agent  
    - Input prompt includes: context from summarization and user message  
    - Config: Return intermediate steps enabled  
    - Connect output to MarkdownV2 node.

16. **Create Gemini 2.5 Flash Lite, Flash, and Pro nodes**  
    - Type: Google Gemini LLM (Chat)  
    - Model names as per above  
    - Credentials: Google Palm API account  
    - Connect to Model Selector or Summarize & Categorize as per logic.

17. **Create MarkdownV2 node**  
    - Type: Code  
    - Paste provided JS code to format output for Telegram MarkdownV2 and chunk messages <=4096 chars  
    - Connect to Send a text message node.

18. **Create Send a text message node**  
    - Type: Telegram  
    - Config:  
      - Text = formatted message from MarkdownV2 node  
      - ChatId from Telegram Trigger  
      - Parse mode: MarkdownV2  
      - Disable attribution  
    - No output connection (end of message sending).

19. **Create Update Chat Memory (User and Agent) node**  
    - Type: PostgreSQL Insert  
    - Insert combined message:  
      ```
      User:  {{normalized input message}}
      Agent: {{AI response output}}
      ```
    - Session_id = chat_id  
    - Credential: PostgreSQL account  
    - No output connection.

20. **Create Typing… node**  
    - Type: Telegram  
    - Operation: sendChatAction "typing"  
    - ChatId from Telegram Trigger  
    - Connect from Telegram Trigger.

21. **Create When clicking ‘Execute workflow’ node**  
    - Type: Manual Trigger  
    - Connect to Create Chat Memory Table node.

22. **Create Create Chat Memory Table node**  
    - Type: PostgreSQL Execute Query  
    - SQL:  
      ```sql
      CREATE TABLE IF NOT EXISTS public.chat_memory (
        id SERIAL PRIMARY KEY,
        session_id VARCHAR(255) NOT NULL,
        message TEXT DEFAULT 'Could not get data'
      );
      CREATE INDEX IF NOT EXISTS chat_memory_session_id_idx ON public.chat_memory (session_id);
      ```
    - Credential: PostgreSQL account.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This architecture prioritizes cost efficiency by using cheaper LLMs for summarization and simple queries, and reserves expensive models for complex tasks only.      | Sticky Note8: Key Benefits of This Architecture                                                                                                           |
| Summarization reduces token usage, speeds up responses, and prevents irrelevant chat history from cluttering model attention.                                         | Sticky Note6: Chat Memory Retrieval & Context Optimization                                                                                                |
| Dynamic model selection optimizes resource usage and response time, balancing cost and capability depending on task complexity.                                       | Sticky Note7: Agent Processing & Response Delivery                                                                                                        |
| Input handling is extensible for multimodal inputs such as images, videos, or multiple files via a linked template workflow.                                         | Sticky Note5: Input Handling (Telegram Trigger & Preprocessing) [Link to https://n8n.io/workflows/7455...]                                                 |
| Acknowledgment to Davide for inspiration from the AI Orchestrator workflow that dynamically selects models based on input type.                                        | Sticky Note9 with link: https://n8n.io/workflows/7004-ai-orchestrator-dynamically-selects-models-based-on-input-type/                                      |
| For assistance customizing or extending the workflow, contact John Alejandro Silva Rodríguez via email or LinkedIn.                                                  | Sticky Note10: Contact info: johnsilva11031@gmail.com, https://www.linkedin.com/in/john-alejandro-silva-rodriguez-48093526b/                              |
| This workflow respects content policies strictly and handles only legal public data with no illegal or offensive content.                                             | Disclaimer (from user prompt)                                                                                                                              |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow built with n8n, adhering strictly to content policies, and contains no illegal, offensive, or protected elements. All processed data is legal and public.