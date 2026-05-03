Create an AI Shopping Assistant on Telegram with GPT-4.1, Voice Recognition & Google Sheets

https://n8nworkflows.xyz/workflows/create-an-ai-shopping-assistant-on-telegram-with-gpt-4-1--voice-recognition---google-sheets-11238


# Create an AI Shopping Assistant on Telegram with GPT-4.1, Voice Recognition & Google Sheets

---
### 1. Workflow Overview

This workflow implements an AI-powered shopping assistant on Telegram, designed specifically for a men’s clothing store. It handles both text and voice messages, transcribes voice inputs, maintains conversation context, searches a product knowledge base using Retrieval-Augmented Generation (RAG) with Pinecone vector search, and records customer orders into Google Sheets. The workflow is structured into the following logical blocks:

- **1.1 Capturing Telegram Messages:** Receives incoming messages from Telegram users (text or voice).
- **1.2 Message Type Routing:** Differentiates between text and voice messages to route them appropriately.
- **1.3 Voice Message Processing:** Downloads and transcribes voice messages into text.
- **1.4 AI Agent Core:** Processes user queries, maintains conversation memory, and generates replies using GPT-4.1-nano with RAG support.
- **1.5 Store Knowledge Base Integration:** Retrieves relevant store product information using Pinecone vector search and OpenAI embeddings.
- **1.6 Order Recording:** Collects complete customer order details and appends them to a Google Sheet.
- **1.7 Response Delivery:** Sends generated responses back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Capturing Telegram Messages

- **Overview:** Listens for incoming Telegram messages (text or voice) from users via a webhook.
- **Nodes Involved:** Telegram Trigger
- **Node Details:**
  - **Telegram Trigger**
    - Type: Telegram Trigger node (Webhook)
    - Configuration: Listens for "message" updates only.
    - Credentials: Uses a Telegram bot API token ("RAG BOT").
    - Inputs: External webhook call from Telegram.
    - Outputs: Emits raw Telegram message JSON.
    - Edge Cases: Webhook misconfiguration, bot token invalid or revoked.
    - Version: 1.1

#### 2.2 Message Type Routing

- **Overview:** Determines if the incoming message is a voice note or text message, routing accordingly.
- **Nodes Involved:** Switch, Set 'Text'
- **Node Details:**
  - **Switch**
    - Type: Switch node
    - Configuration: Checks presence of `message.voice.file_id` for voice messages or `message.text` for text.
    - Inputs: Telegram Trigger output.
    - Outputs: Two branches — "Voice" for voice messages, "Text" for text.
    - Edge Cases: Messages without text or voice (e.g., stickers) will not match either condition.
  - **Set 'Text'**
    - Type: Set node
    - Configuration: Extracts and assigns `text` from `message.text` and `chat_ID` from `message.chat.id`.
    - Inputs: Switch node "Text" output.
    - Outputs: Structured JSON with `text` and `chat_ID`.
    - Edge Cases: Missing or empty text during assignment.

#### 2.3 Voice Message Processing

- **Overview:** Downloads voice files from Telegram and transcribes them into text using OpenAI's audio transcription.
- **Nodes Involved:** Download Voice File, Transcribe Audio
- **Node Details:**
  - **Download Voice File**
    - Type: Telegram node
    - Configuration: Downloads the voice file using `file_id` from the incoming message.
    - Credentials: Same Telegram bot API as trigger.
    - Inputs: Switch node "Voice" output.
    - Outputs: Binary audio file data.
    - Edge Cases: File download failure, invalid file ID, Telegram API rate limits.
  - **Transcribe Audio**
    - Type: OpenAI audio transcription node (LangChain)
    - Configuration: Transcribes audio to text.
    - Credentials: OpenAI API (account "nagorik").
    - Inputs: Downloaded audio file.
    - Outputs: Transcribed text.
    - Edge Cases: Audio format issues, API timeouts, transcription inaccuracies.

#### 2.4 AI Agent Core

- **Overview:** Uses GPT-4.1-nano based AI agent to process user messages (transcribed or text), maintain conversation context, query knowledge base, and handle order taking.
- **Nodes Involved:** AI Agent1, Simple Memory, OpenAI Chat Model, OpenAI Chat Model4, Answer questions with a vector store1, Embeddings OpenAI1, Pinecone Vector Store
- **Node Details:**
  - **AI Agent1**
    - Type: LangChain Agent node
    - Configuration: Uses GPT-4.1-nano model; system message defines the agent’s role as a friendly men’s clothing store support agent who handles customer support, order taking, and uses RAG for product info.
    - Inputs: Text from Set or transcribed voice.
    - Outputs: Agent’s text reply.
    - Edge Cases: Model API limits, misinterpretation of instructions, incomplete order data.
  - **Simple Memory**
    - Type: Memory buffer node (LangChain)
    - Configuration: Keeps last 8 messages per chat using `chat_ID` as session key.
    - Inputs: AI Agent1 (ai_memory).
    - Outputs: Maintains conversation context for AI Agent.
    - Edge Cases: Memory overflow, session key mismatch.
  - **OpenAI Chat Model**
    - Type: GPT-4.1-nano language model node (LangChain)
    - Role: Provides core language model responses to AI Agent1.
  - **OpenAI Chat Model4**
    - Type: GPT-4.1-nano language model node (LangChain)
    - Role: Used by the vector store tool for enhanced query answering.
  - **Answer questions with a vector store1**
    - Type: Tool node connected to vector store
    - Configuration: Uses Pinecone-powered vector search to answer queries about men’s clothing.
  - **Embeddings OpenAI1**
    - Type: OpenAI embeddings node (LangChain)
    - Role: Generates embeddings from queries.
  - **Pinecone Vector Store**
    - Type: Pinecone vector store node
    - Configuration: Uses pre-existing "mens-collection" index and namespace.
    - Credentials: Pinecone API account.
    - Role: Retrieves relevant product info for RAG.

#### 2.5 Order Recording

- **Overview:** Once all required customer order details (Name, Phone number, Address, Category) are collected, the order is appended as a new row in a Google Sheet.
- **Nodes Involved:** Append row in sheet in Google Sheets
- **Node Details:**
  - **Append row in sheet in Google Sheets**
    - Type: Google Sheets Tool node
    - Configuration: Appends new rows with columns: Name, Phone number, Address, Category.
    - Credentials: Google Sheets OAuth2 API.
    - Inputs: Order details from AI Agent1.
    - Spreadsheet: Document ID and Sheet name specified.
    - Edge Cases: Invalid/missing credentials, API rate limits, spreadsheet permission issues.

#### 2.6 Response Delivery

- **Overview:** Sends the AI-generated reply back to the user on Telegram.
- **Nodes Involved:** Response
- **Node Details:**
  - **Response**
    - Type: Telegram node
    - Configuration: Sends text messages to the chat ID.
    - Credentials: Telegram bot API.
    - Inputs: AI Agent1 output.
    - Edge Cases: Telegram API errors, message sending failures.

---

### 3. Summary Table

| Node Name                        | Node Type                         | Functional Role                           | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                             |
|---------------------------------|----------------------------------|-----------------------------------------|------------------------------|----------------------------|-------------------------------------------------------------------------------------------------------|
| Telegram Trigger                | Telegram Trigger (Webhook)        | Captures incoming Telegram messages     | N/A                          | Switch                     |                                                                                                       |
| Switch                         | Switch                           | Routes message based on type (voice/text)| Telegram Trigger             | Download Voice File, Set 'Text'|                                                                                                       |
| Download Voice File            | Telegram (file download)          | Downloads voice message file             | Switch (Voice branch)         | Transcribe Audio           | ## Voice Message Processing\nProcesses the voice message to extract the queries                      |
| Transcribe Audio               | OpenAI Audio Transcription        | Transcribes audio to text                 | Download Voice File           | AI Agent1                 |                                                                                                       |
| Set 'Text'                    | Set                             | Extracts text and chat ID from message  | Switch (Text branch)          | AI Agent1                 |                                                                                                       |
| AI Agent1                     | LangChain Agent                  | Core AI agent for answering and order taking| Set 'Text', Transcribe Audio | Response, Append row in sheet in Google Sheets| ## RAG and Order Taking Agent                                                                       |
| Simple Memory                 | LangChain Memory Buffer          | Maintains conversation context           | AI Agent1 (ai_memory)         | AI Agent1 (ai_memory)      | ## Agent's Brain\nAgent uses these components to think and keep converstation in memory               |
| OpenAI Chat Model             | LangChain LM Chat OpenAI         | Language model for AI Agent              | AI Agent1 (ai_languageModel) | AI Agent1                 |                                                                                                       |
| OpenAI Chat Model4            | LangChain LM Chat OpenAI         | Language model for vector store tool    |                            | Answer questions with a vector store1 |                                                                                                       |
| Answer questions with a vector store1 | LangChain Tool Vector Store        | Retrieves store info via Pinecone        | OpenAI Chat Model4            | AI Agent1 (ai_tool)       | ## Agent's Store Knowledge Base\nDescription: Retrives store data from the existing knowledge base.   |
| Embeddings OpenAI1            | LangChain Embeddings OpenAI      | Generates embeddings for vector search  |                              | Pinecone Vector Store      |                                                                                                       |
| Pinecone Vector Store         | LangChain Vector Store Pinecone  | Vector search on store data              | Embeddings OpenAI1            | Answer questions with a vector store1 |                                                                                                       |
| Append row in sheet in Google Sheets | Google Sheets Tool                | Records customer order details           | AI Agent1 (ai_tool)           | AI Agent1 (ai_tool)        | ## Order Registry                                                                                      |
| Response                      | Telegram                         | Sends reply message back to Telegram     | AI Agent1                     |                            | ## Bot Response Point                                                                                  |
| Sticky Note                   | Sticky Note                     | Annotation                              |                              |                            | ## Capturing Telegram Message                                                                          |
| Sticky Note1                  | Sticky Note                     | Annotation                              |                              |                            | ## Text/Voice Message Router                                                                           |
| Sticky Note2                  | Sticky Note                     | Annotation                              |                              |                            | ## Voice Message Processing                                                                            |
| Sticky Note3                  | Sticky Note                     | Annotation                              |                              |                            | ## RAG and Order Taking Agent                                                                          |
| Sticky Note4                  | Sticky Note                     | Annotation                              |                              |                            | ## Agent's Brain                                                                                       |
| Sticky Note5                  | Sticky Note                     | Annotation                              |                              |                            | ## Agent's Store Knowledge Base                                                                        |
| Sticky Note6                  | Sticky Note                     | Annotation                              |                              |                            | ## Order Registry                                                                                      |
| Sticky Note7                  | Sticky Note                     | Annotation                              |                              |                            | ## Bot Response Point                                                                                  |
| Sticky Note8                  | Sticky Note                     | Workflow overview and setup instructions|                              |                            | # Overview\n## How it works: * Captures all incoming text and voice messages through a Telegram Trigger.\n* Detects message type using a Switch node and routes text and voice notes to the correct processing path.\n* Transcribes voice messages using OpenAI and extracts clean message content with a Set node.\n* Uses an AI Agent (GPT-4.1-nano) to generate friendly, accurate replies while maintaining conversation context with an 8-message memory window.\n* Searches your product catalog and information through a Pinecone-powered RAG system to provide prices, features, and other details with high accuracy.\n* Automatically collects customer order details and records complete orders into Google Sheets.\n* Sends final responses and confirmations back to customers through Telegram.\n\n## Setup steps: * Import JSON to n8n; create Telegram bot; add credentials; set Pinecone; prepare Google Sheet; customize AI Agent system message; test and activate. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Parameters: Listen to "message" updates
   - Credentials: Add Telegram bot API token (create bot via @BotFather)
   - Connect it as the entry point.

2. **Add Switch Node for Message Type Routing**
   - Type: Switch
   - Conditions:
     - Output 1 ("Voice"): Check if `message.voice.file_id` exists
     - Output 2 ("Text"): Check if `message.text` exists
   - Connect Telegram Trigger output to Switch input.

3. **Set Up Voice Message Processing Branch**
   - Add Telegram node "Download Voice File"
     - Parameter: Use `file_id` from `message.voice.file_id`
     - Credentials: Same Telegram bot token
   - Add OpenAI Audio Transcription node "Transcribe Audio"
     - Operation: Transcribe audio to text
     - Credentials: OpenAI API account
   - Connect Switch "Voice" output → Download Voice File → Transcribe Audio

4. **Set Up Text Message Processing Branch**
   - Add Set node "Set 'Text'"
     - Assign variables:
       - `text` = `{{ $json.message.text }}`
       - `chat_ID` = `{{ $json.message.chat.id }}`
   - Connect Switch "Text" output → Set 'Text'

5. **Create AI Agent Node**
   - Type: LangChain Agent
   - Parameters:
     - Text input: `{{ $json.text }}`
     - System message: Define agent role for men’s clothing store support, order handling, and RAG instructions as per original.
     - Model: GPT-4.1-nano
   - Credentials: OpenAI API
   - Connect both Set 'Text' output and Transcribe Audio output to AI Agent input.

6. **Set Up Conversation Memory**
   - Add Simple Memory node (Memory Buffer Window)
     - Session key: `{{ $json.chat_ID }}`
     - Context window length: 8 messages
   - Connect AI Agent memory input/output accordingly.

7. **Set Up Language Models for AI Agent**
   - Add OpenAI Chat Model nodes for AI Agent (with GPT-4.1-nano)
   - Connect these nodes to AI Agent’s language model inputs.

8. **Set Up Store Knowledge Base Integration**
   - Add Embeddings OpenAI node for query embeddings
   - Add Pinecone Vector Store node
     - Configure with your Pinecone API credentials
     - Use your Pinecone index "mens-collection" and namespace
   - Add "Answer questions with a vector store" tool node linked to Pinecone and OpenAI Chat Model4
   - Connect these nodes as per AI Agent’s tool inputs.

9. **Configure Order Recording Node**
   - Add Google Sheets Tool node "Append row in sheet in Google Sheets"
     - Operation: Append
     - Columns: Name, Phone number, Address, Category (mapped from AI Agent output)
     - Set your Google Sheets document ID and sheet name
     - Credentials: Google Sheets OAuth2
   - Connect AI Agent tool output to this node.

10. **Add Response Node**
    - Type: Telegram node
    - Parameters:
      - Text: `{{ $json.output }}`
      - Chat ID: `{{ $json.chat_ID }}`
    - Credentials: Telegram bot API
    - Connect AI Agent output to Response node.

11. **Add Sticky Notes (Optional)**
    - Add sticky notes for documentation and clarity at each logical block.

12. **Test Workflow**
    - Send both text and voice messages to your Telegram bot.
    - Verify transcription, AI responses, order recording, and message delivery.
    - Troubleshoot as needed.

13. **Activate Workflow**
    - Once verified, activate workflow for continuous operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow includes detailed sticky notes explaining each block for easier maintenance and understanding.                                                                                                                        | Visible in workflow as Sticky Note nodes.                                                                       |
| Telegram bot creation via [@BotFather](https://telegram.me/BotFather) is required to obtain API token.                                                                                                                         | Telegram bot setup prerequisite.                                                                                 |
| Pinecone index "mens-collection" must be pre-populated with men’s clothing product data for effective RAG retrieval.                                                                                                           | Pinecone setup instructions: https://www.pinecone.io/docs/                                                     |
| Google Sheet must have columns: Name, Phone number, Address, Category. Update Google Sheets node with your document ID and sheet name accordingly.                                                                             | Google Sheets setup.                                                                                             |
| AI Agent system message is carefully crafted to limit conversation to men’s clothing support and order taking, improving relevance and safety. Customize as necessary for your store voice and policies.                         | Located in AI Agent1 node parameters.                                                                           |
| Voice messages are transcribed using OpenAI's Whisper model integrated via LangChain nodes, ensuring accurate text extraction.                                                                                                  | Requires OpenAI Audio transcription credentials.                                                                |
| Conversation memory maintains last 8 messages per user to keep context during interactions, enhancing user experience.                                                                                                          | Implemented via Simple Memory node.                                                                              |
| The workflow respects Telegram API rate limits and handles potential failures such as missing message content, file download errors, and API timeouts by design, but further custom error handling can be added by users.          | Recommended best practice for production environments.                                                          |
| The workflow can be extended to other product categories by adjusting the knowledge base, Pinecone namespace, and AI Agent instructions accordingly.                                                                            | Modular design supports easy customization.                                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. The processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.