Update Grocery Lists in Asana via Telegram Voice & Text using GPT-4o mini

https://n8nworkflows.xyz/workflows/update-grocery-lists-in-asana-via-telegram-voice---text-using-gpt-4o-mini-7845


# Update Grocery Lists in Asana via Telegram Voice & Text using GPT-4o mini

### 1. Workflow Overview

This workflow, titled **"Update Grocery Lists in Asana via Telegram Voice & Text using GPT-4o mini"**, automates managing grocery lists stored in Asana by leveraging Telegram for user interaction and AI for natural language processing. Users can communicate with the grocery list bot via Telegram either by text or voice messages, instructing it to add, update, or check items with details like quantity and expiry date. The workflow integrates several services—Telegram, OpenAI for transcription and AI processing, OpenRouter, and Asana API—to deliver a seamless, conversational grocery management experience.

**Logical Blocks:**

- **1.1 Input Reception (Telegram Trigger & Message Routing):** Accepts incoming Telegram messages, distinguishes between voice and text input, and prepares text for AI processing.
- **1.2 Voice Transcription:** Downloads and transcribes voice messages into text using OpenAI.
- **1.3 AI Processing (Grocery Agent):** Uses a LangChain agent with GPT-4o mini to interpret user instructions, decide actions on grocery items, and orchestrate further API calls.
- **1.4 Asana Integration (Search, Update, Check/Uncheck):** Searches grocery items in Asana, modifies their status, and updates quantities and expiry dates using custom workflows and Asana API nodes.
- **1.5 Response Generation:** Sends a summary reply back to the Telegram user, including a hyperlink to the Asana grocery list.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (Telegram Trigger & Message Routing)

- **Overview:** Listens for new messages on Telegram, routes input between text and voice paths.
- **Nodes Involved:** Telegram Trigger, Route Text or Voice, Get Text Field, Download Voice File
- **Node Details:**
  - **Telegram Trigger**
    - *Type:* Telegram Trigger Node (Webhook-based)
    - *Role:* Entry point capturing all new Telegram messages.
    - *Config:* Triggers on message updates; uses Telegram API credentials.
    - *Input:* Telegram messages (voice or text)
    - *Output:* Passes message JSON downstream.
    - *Potential failures:* Telegram API errors, webhook misconfiguration.
  - **Route Text or Voice**
    - *Type:* Switch Node
    - *Role:* Determines if incoming message contains text or voice.
    - *Config:* Checks existence of `message.text` or `message.voice.file_id`.
    - *Input:* Telegram Trigger output
    - *Output:* Routes to "Text" or "Voice" branches.
    - *Edge cases:* Messages with neither text nor voice; unexpected message formats.
  - **Get Text Field**
    - *Type:* Set Node
    - *Role:* Extracts text from text messages.
    - *Config:* Assigns `text` variable from `message.text`.
    - *Input:* Route Text or Voice (Text branch)
    - *Output:* Passes text field to AI agent.
  - **Download Voice File**
    - *Type:* Telegram Node (File download)
    - *Role:* Downloads voice message file from Telegram servers.
    - *Config:* Uses file ID from `message.voice.file_id`.
    - *Input:* Route Text or Voice (Voice branch)
    - *Output:* File binary output for transcription.
    - *Potential failures:* File download errors, invalid file ID.

#### 2.2 Voice Transcription

- **Overview:** Converts voice recordings into text for AI processing.
- **Nodes Involved:** Transcribe a recording
- **Node Details:**
  - **Transcribe a recording**
    - *Type:* LangChain OpenAI Audio Node
    - *Role:* Uses OpenAI’s audio transcription to convert voice to text.
    - *Config:* Transcribe operation on audio resource.
    - *Input:* Download Voice File binary data
    - *Output:* Text transcription passed to AI agent.
    - *Credentials:* OpenAI API key required.
    - *Failures:* API quota exceeded, transcription errors, unsupported audio format.

#### 2.3 AI Processing (Grocery Agent)

- **Overview:** Core natural language processor that interprets user commands, decides actions, and calls supporting tools.
- **Nodes Involved:** Grocery Agent, GPT 4o mini
- **Node Details:**
  - **GPT 4o mini**
    - *Type:* LangChain LM Chat Node (OpenRouter API)
    - *Role:* Provides GPT-4o mini model for conversational AI.
    - *Config:* Model set to `openai/gpt-4o-mini`.
    - *Input:* Passed from Grocery Agent's prompt.
    - *Output:* Language model predictions.
    - *Credentials:* OpenRouter API credentials.
  - **Grocery Agent**
    - *Type:* LangChain Agent Node
    - *Role:* Implements rules and instructions to manage grocery items.
    - *Config:* 
      - System message defines the agent’s role, tool usage, and rules.
      - Tools: Search, Check/Uncheck, Update Expiry & Quantity.
      - Instructions on how to interpret phrases like "need," "bought," etc.
      - Uses AI tools for stepwise grocery list management.
    - *Input:* Text from Get Text Field or Transcribe a recording.
    - *Output:* Final summary text to reply in Telegram.
    - *Edge Cases:* Ambiguous commands, missing item info, incomplete instructions.
    - *Version:* Requires LangChain agent features available in n8n version supporting `@n8n/n8n-nodes-langchain.agent` v2.2+.

#### 2.4 Asana Integration (Search, Update, Check/Uncheck)

- **Overview:** Interacts with Asana API to search for grocery items, update their completion status, and modify quantity or expiry date.
- **Nodes Involved:** Search for Grocery Item, Check or Uncheck Grocery Items, Update Expiry Date & Qty
- **Node Details:**
  - **Search for Grocery Item**
    - *Type:* Asana Tool Node
    - *Role:* Searches grocery list tasks in Asana workspace.
    - *Config:* Uses OAuth2 authentication; searches tasks matching AI-provided text.
    - *Input:* AI tool input from Grocery Agent.
    - *Output:* Task details if found.
    - *Failure modes:* OAuth token expiration, Asana API limits.
  - **Check or Uncheck Grocery Items**
    - *Type:* Asana Tool Node
    - *Role:* Updates task completion status (checked/unchecked).
    - *Config:* Uses task ID and completion boolean from AI.
    - *Input:* AI tool input from Grocery Agent.
    - *Output:* Confirmation of update.
    - *Failures:* Invalid task ID, permission denied.
  - **Update Expiry Date & Qty**
    - *Type:* LangChain Tool Workflow Node
    - *Role:* Calls a sub-workflow to update custom fields (expiry date, quantity).
    - *Config:* Passes parameters TaskID, QuantityValue, ExpiryDateValue.
    - *Input:* From Grocery Agent AI tool.
    - *Output:* Update confirmation.
    - *Sub-workflow:* Workflow ID "6xj0T4s40s9NgWjC" titled "🍎 Update Expiry & Quantity".
    - *Failures:* Sub-workflow errors, invalid field values.

#### 2.5 Response Generation

- **Overview:** Sends a tailored response back to the Telegram user summarizing the updates and providing an Asana link.
- **Nodes Involved:** Reply in Chat
- **Node Details:**
  - **Reply in Chat**
    - *Type:* Telegram Node (Send message)
    - *Role:* Sends a reply message to the Telegram chat.
    - *Config:* Text content from Grocery Agent output; uses original chat ID.
    - *Input:* Grocery Agent output text.
    - *Output:* Telegram message sent confirmation.
    - *Failures:* Telegram API errors, invalid chat ID.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                       | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                 |
|----------------------------|---------------------------------|------------------------------------|-------------------------------|----------------------------------|----------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                | Entry point for Telegram messages  | -                             | Route Text or Voice              | Setup Telegram bot - details [here](https://core.telegram.org/bots/tutorial).|
| Route Text or Voice         | Switch                         | Route between text or voice input  | Telegram Trigger              | Get Text Field, Download Voice File | ## 2. Voice/Text: Set fields for AI processing.                            |
| Get Text Field              | Set                            | Extract text from Telegram message | Route Text or Voice (Text)    | Grocery Agent                   | ## 2. Voice/Text: Set fields for AI processing.                            |
| Download Voice File         | Telegram (File download)        | Download voice message file        | Route Text or Voice (Voice)   | Transcribe a recording          | ## 2. Voice/Text: Set fields for AI processing.                            |
| Transcribe a recording      | LangChain OpenAI Audio          | Transcribe voice to text           | Download Voice File           | Grocery Agent                   |                                                                            |
| GPT 4o mini                | LangChain LM Chat OpenRouter    | Provide GPT-4o-mini language model | Grocery Agent (AI languageModel) | Grocery Agent                  |                                                                            |
| Grocery Agent              | LangChain Agent                 | Core AI logic for grocery updates | Get Text Field, Transcribe a recording, GPT 4o mini | Reply in Chat, Search for Grocery Item, Check or Uncheck Grocery Items, Update Expiry Date & Qty | ## 3. Grocery Updates: Search, check/uncheck, and update expiry/quantity.  |
| Search for Grocery Item     | Asana Tool                     | Search groceries in Asana          | Grocery Agent (ai_tool)       | Grocery Agent (ai_tool)          | ## 3. Grocery Updates: Search, check/uncheck, and update expiry/quantity.  |
| Check or Uncheck Grocery Items | Asana Tool                 | Update task completion status      | Grocery Agent (ai_tool)       | Grocery Agent (ai_tool)          | ## 3. Grocery Updates: Search, check/uncheck, and update expiry/quantity.  |
| Update Expiry Date & Qty    | LangChain Tool Workflow         | Update expiry date and quantity via sub-workflow | Grocery Agent (ai_tool)       | Grocery Agent (ai_tool)          | ## 3. Grocery Updates: Search, check/uncheck, and update expiry/quantity.  |
| Reply in Chat              | Telegram                        | Reply to user with summary         | Grocery Agent                 | -                              | ## 4. Response: Summarises updates with Asana link.                        |
| Sticky Note                | Sticky Note                    | Documentation                     | -                             | -                              | ## Update Grocery Lists in Asana via Telegram using Natural Language AI...  |
| Sticky Note1               | Sticky Note                    | Documentation                     | -                             | -                              | Setup Telegram bot - details [here](https://core.telegram.org/bots/tutorial).|
| Sticky Note2               | Sticky Note                    | Documentation                     | -                             | -                              | ## 2. Voice/Text: Set fields that Agent will take in.                       |
| Sticky Note3               | Sticky Note                    | Documentation                     | -                             | -                              | ## 3. Grocery Updates: Agent instructions for updating grocery items.      |
| Sticky Note4               | Sticky Note                    | Documentation                     | -                             | -                              | ## 4. Response: Reply summarising updates with Asana hyperlink.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Set to trigger on "message" updates.
   - Configure Telegram API credentials (Telegram bot token).
   - Position as entry node.

2. **Add Switch Node (Route Text or Voice)**
   - Type: Switch
   - Add two outputs: "Text" and "Voice".
   - Configure rules:
     - Text: Exists `message.text`
     - Voice: Exists `message.voice.file_id`
   - Connect Telegram Trigger output to this node.

3. **Add Set Node (Get Text Field)**
   - Type: Set
   - Assign field `text` with expression `{{$json.message.text}}`.
   - Connect "Text" output from Switch node here.

4. **Add Telegram Node (Download Voice File)**
   - Type: Telegram
   - Operation: download file
   - File ID: `{{$json.message.voice.file_id}}`
   - Connect "Voice" output from Switch node here.
   - Use same Telegram API credentials.

5. **Add LangChain OpenAI Audio Node (Transcribe a recording)**
   - Type: LangChain OpenAI
   - Resource: Audio
   - Operation: Transcribe
   - Connect Download Voice File output here.
   - Configure OpenAI credentials with your API key.

6. **Add LangChain Agent Node (Grocery Agent)**
   - Type: LangChain Agent
   - Add system message defining:
     - Role as grocery agent
     - Tool usage instructions (Search, Check/Uncheck, Update Expiry & Quantity)
     - Rules on interpreting user phrases
     - Example queries
   - Set input text for the agent from either Get Text Field OR Transcribe a recording output.
   - Configure prompt type: define.
   - Connect GPT 4o mini node as LM provider for this agent.

7. **Add LangChain LM Chat Node (GPT 4o mini)**
   - Type: LangChain LM Chat OpenRouter
   - Model: openai/gpt-4o-mini
   - Configure OpenRouter API credentials.
   - Connect to Grocery Agent node as language model.

8. **Add Asana Tool Nodes**
   - **Search for Grocery Item**
     - Operation: search tasks
     - Workspace ID: your Asana workspace ID
     - Authentication: OAuth2 with Asana credentials
     - Search text: from AI tool override (`$fromAI('Text')`)
   - **Check or Uncheck Grocery Items**
     - Operation: update task completion
     - Task ID: from AI tool override (`$fromAI('Task_ID')`)
     - Completed: boolean from AI (`$fromAI('Completed')`)
   - Connect both nodes as AI tools callable by Grocery Agent.

9. **Add LangChain Tool Workflow Node (Update Expiry Date & Qty)**
   - Configure to call sub-workflow for updating expiry and quantity.
   - Pass parameters:
     - TaskID (number)
     - QuantityValue (number, optional)
     - ExpiryDateValue (string, optional)
   - Connect as AI tool callable by Grocery Agent.

10. **Add Telegram Node (Reply in Chat)**
    - Operation: Send message
    - Text: `{{$json.output}}` from Grocery Agent output
    - Chat ID: `{{$node["Telegram Trigger"].json.message.chat.id}}`
    - Connect Grocery Agent output here.

11. **Set up credentials:**
    - Telegram Bot API token (Botfather)
    - OpenAI API key (for transcription)
    - OpenRouter API key (for GPT 4o mini)
    - Asana OAuth2 credentials (with access to grocery list workspace)

12. **Optional:** Add sticky notes documenting each block as in the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Setup Telegram bot via Botfather. Detailed instructions available.                                                    | https://core.telegram.org/bots/tutorial                                                                 |
| OpenAI API for transcription services requires credits.                                                              | https://openai.com/index/openai-api/                                                                     |
| OpenRouter account needed for GPT 4o mini usage.                                                                      | https://openrouter.ai/docs/api-reference/authentication                                                  |
| Asana API setup needed with OAuth2 credentials.                                                                       | https://developers.asana.com/docs/api-explorer                                                          |
| Workflow supports customization of tracked fields (expiry date, quantity, food type, purchase date, etc.)             | User-configurable based on Asana custom fields                                                          |
| Example command usage patterns included in Grocery Agent system prompt for robust natural language handling           | Internal system message in Grocery Agent node                                                            |
| The link included in Telegram responses points users directly to their Asana grocery list for quick reference         | Insert your Asana grocery page URL in the agent's final response template                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.