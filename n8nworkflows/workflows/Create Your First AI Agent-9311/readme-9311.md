Create Your First AI Agent

https://n8nworkflows.xyz/workflows/create-your-first-ai-agent-9311


# Create Your First AI Agent

### 1. Workflow Overview

This workflow, titled **"Build Your First Conversational AI Chatbot Agent,"** is designed to create an interactive conversational AI agent using **Google Gemini (PaLM/Gemini Pro)** via **LangChain** integration within n8n. It targets users seeking to build a customizable AI chatbot that can:

- Receive chat messages from external chat platforms (via webhook)
- Process these messages using Google Gemini AI models
- Optionally fetch and simplify Google Docs content referenced in the chat
- Log the conversation (user inputs and AI responses) to a Google Sheet for persistent storage and later analysis
- Optionally send AI responses back to a Telegram chat for notification or direct user feedback

**Logical Blocks:**

- **1.1 Input Reception & Trigger:** Listens and receives incoming chat messages.
- **1.2 AI Processing:** Uses LangChain-connected Google Gemini AI to generate contextual replies.
- **1.3 Optional Google Docs Handling:** Retrieves and simplifies Google Docs documents if referenced.
- **1.4 Logging & Output:** Stores conversation data in Google Sheets and optionally sends replies to Telegram.
- **1.5 Auxiliary Nodes:** Sticky notes for documentation, instructions, and optional configurations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Trigger

- **Overview:** This block captures incoming chat messages from external sources via a webhook and triggers the workflow.
- **Nodes Involved:** `Chat message`
- **Node Details:**

  - **Chat message**
    - *Type & Role:* LangChain Chat Trigger node; acts as a webhook listener for incoming chat messages.
    - *Configuration:* 
      - Public webhook enabled, allowing external chat systems to send messages.
      - Initial message configured as "Hi Nani! 👋" (sent to users when the chat starts).
    - *Expressions/Variables:* Captures user chat input as `chatInput`.
    - *Inputs:* External webhook (HTTP request).
    - *Outputs:* Passes user message JSON to the AI Agent node.
    - *Edge cases:* Webhook URL must be publicly accessible; malformed or missing chat input could cause failures.
    - *Version:* 1.1

#### 1.2 AI Processing

- **Overview:** Uses LangChain’s AI Agent node integrated with Google Gemini AI to generate intelligent, user-friendly replies to chat messages.
- **Nodes Involved:** `AI Agent`, `Gemini Chat`, `Gemini`, `Request`
- **Node Details:**

  - **AI Agent**
    - *Type & Role:* LangChain Agent node responsible for orchestrating AI processing and tool usage.
    - *Configuration:* Uses default options; connects to AI language model and external tools.
    - *Inputs:* Receives message from `Chat message` node and tool outputs (`Gemini`, `Docs`, `Request` nodes).
    - *Outputs:* Sends AI-generated output to logging and Telegram nodes.
    - *Edge cases:* Requires valid API credentials; failure in any connected tool may affect output.
    - *Version:* 2.1

  - **Gemini Chat**
    - *Type & Role:* LangChain Google Gemini Chat language model node.
    - *Configuration:* Uses Google Gemini API credentials; no additional options set.
    - *Inputs:* Connected as AI language model for `AI Agent`.
    - *Outputs:* Provides AI-generated chat responses.
    - *Edge cases:* API quota limits, network errors, or invalid modelId could cause failures.
    - *Version:* 1

  - **Gemini**
    - *Type & Role:* LangChain Google Gemini tool node.
    - *Configuration:* Model ID set to `models/gemini-2.5-flash`; pre-message instructs the model to deliver user-friendly replies avoiding robotic language.
    - *Inputs:* Used as a tool by `AI Agent`.
    - *Outputs:* Provides processed text to `AI Agent`.
    - *Edge cases:* Same as Gemini Chat; API credential or usage limits.
    - *Version:* 1

  - **Request**
    - *Type & Role:* HTTP Request node.
    - *Configuration:* Set to call URL `https://google.cm/` (likely placeholder or test).
    - *Inputs:* Used as a tool by `AI Agent`.
    - *Outputs:* Provides HTTP response to `AI Agent`.
    - *Edge cases:* Network failure, invalid URL.
    - *Version:* 4.2

#### 1.3 Optional Google Docs Handling

- **Overview:** Retrieves and simplifies Google Docs content if the user's message references a Google Docs document.
- **Nodes Involved:** `Docs`
- **Node Details:**

  - **Docs**
    - *Type & Role:* Google Docs node to fetch and simplify document content.
    - *Configuration:* 
      - Operation: `get`
      - Document URL or ID dynamically set using AI override expression.
      - Simplify option also controlled via AI override.
    - *Inputs:* Used as a tool by `AI Agent`.
    - *Outputs:* Supplies simplified document text for AI processing.
    - *Credentials:* Requires OAuth2 Google Docs API credentials.
    - *Edge cases:* Invalid document URL/ID, permission errors, token expiration.
    - *Version:* 2

#### 1.4 Logging & Output

- **Overview:** Logs the conversation (user question and AI reply) into Google Sheets and optionally sends the AI response back to Telegram.
- **Nodes Involved:** `Store in sheet`, `Crypto`, `Store in Your Chat`
- **Node Details:**

  - **Store in sheet**
    - *Type & Role:* Google Sheets node appending conversation data.
    - *Configuration:* 
      - Operation: append
      - Document ID and sheet name (gid=0) fixed to a specific Google Sheet.
      - Columns mapped: "replay message" ← AI output, "output data (Crypto)" ← processed crypto output.
    - *Inputs:* Receives processed data from `Crypto` node.
    - *Outputs:* None (final storage).
    - *Credentials:* Requires OAuth2 Google Sheets credentials.
    - *Edge cases:* Sheet permissions, rate limits, malformed data.
    - *Version:* 4.6

  - **Crypto**
    - *Type & Role:* Crypto node that processes AI output (could be for encryption or hashing).
    - *Configuration:* Input value set to AI generated output.
    - *Inputs:* Receives AI output from `AI Agent`.
    - *Outputs:* Sends processed data to `Store in sheet`.
    - *Edge cases:* Invalid input data may cause errors.
    - *Version:* 1

  - **Store in Your Chat**
    - *Type & Role:* Telegram node sending messages to a Telegram chat.
    - *Configuration:*
      - Sends formatted message containing user question and AI reply.
      - Chat ID must be specified (placeholder "your chat ID").
    - *Inputs:* Receives AI output from `AI Agent`.
    - *Outputs:* None.
    - *Credentials:* Requires Telegram API credentials.
    - *Edge cases:* Invalid chat ID, Telegram API limits, network errors.
    - *Version:* 1.2

#### 1.5 Auxiliary Nodes (Sticky Notes)

- **Overview:** Provides documentation, usage instructions, reminders, and optional configurations visually within the workflow.
- **Nodes Involved:** All sticky notes (`Sticky Note`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note12`, `Sticky Note13`)
- **Node Details:**

  - **Sticky Note** (large overview note)
    - Contains detailed overview, features, requirements, usage instructions, and helpful links.
  - **Sticky Note2 & Sticky Note3**
    - Notes about tools and chat models customization.
  - **Sticky Note4**
    - Describes optional Telegram notification and Google Sheets logging.
  - **Sticky Note12**
    - Encourages activation and sharing of the public chat URL.
  - **Sticky Note13**
    - Explains AI agent capabilities and system message tuning.

---

### 3. Summary Table

| Node Name         | Node Type                            | Functional Role                     | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                      |
|-------------------|------------------------------------|-----------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Chat message      | @n8n/n8n-nodes-langchain.chatTrigger | Receives incoming chat messages   | External webhook               | AI Agent                      | Activate this workflow and share the public chat URL to let others talk to your AI Agent!                       |
| AI Agent          | @n8n/n8n-nodes-langchain.agent     | Orchestrates AI processing        | Chat message, Gemini, Docs, Request | Store in Your Chat, Crypto    | Your AI agent can: Receive messages, select tools, respond with live answers. Adjust system message behavior.   |
| Gemini Chat       | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI language model for chat        | AI Agent (ai_languageModel)    | AI Agent                      | Chat modal can be changed; requires API key.                                                                    |
| Gemini            | @n8n/n8n-nodes-langchain.googleGeminiTool | AI tool for friendly replies      | AI Agent (ai_tool)             | AI Agent                      | Chat modal can be changed; requires API key.                                                                    |
| Request           | n8n-nodes-base.httpRequestTool     | HTTP call tool                   | AI Agent (ai_tool)             | AI Agent                      | Chat modal can be changed; requires API key.                                                                    |
| Docs              | n8n-nodes-base.googleDocsTool       | Fetches and simplifies Google Docs| AI Agent (ai_tool)             | AI Agent                      | Tool can be added/changed; requires OAuth2 Google Docs API.                                                     |
| Store in sheet    | n8n-nodes-base.googleSheets         | Logs conversation in Google Sheets| Crypto                        | None                         | Optional: User can get notifications on Telegram and store in Google Sheets.                                    |
| Crypto            | n8n-nodes-base.crypto               | Processes AI output (crypto)       | AI Agent                      | Store in sheet                |                                                                                                                 |
| Store in Your Chat| n8n-nodes-base.telegram             | Sends AI replies to Telegram chat  | AI Agent                      | None                         | Optional: User can get notifications on Telegram and store in Google Sheets.                                    |
| Sticky Note       | n8n-nodes-base.stickyNote           | Documentation & instructions       | None                         | None                         | Contains workflow overview, features, requirements, usage instructions, and helpful community links.           |
| Sticky Note2      | n8n-nodes-base.stickyNote           | Notes on tool customization        | None                         | None                         | Tools can be changed but require API keys and data for replies.                                                |
| Sticky Note3      | n8n-nodes-base.stickyNote           | Notes on chat model customization  | None                         | None                         | Chat modal can be changed but requires API keys.                                                                |
| Sticky Note4      | n8n-nodes-base.stickyNote           | Optional Telegram notification info| None                         | None                         | Optional user notification on Telegram and Google Sheets logging.                                              |
| Sticky Note12     | n8n-nodes-base.stickyNote           | Activation and sharing reminder    | None                         | None                         | Activate workflow and share public chat URL.                                                                    |
| Sticky Note13     | n8n-nodes-base.stickyNote           | AI agent capability explanation    | None                         | None                         | Describes AI agent's receiving, tool selecting, and responding capabilities along with system message tuning. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**
   - Add **LangChain Chat Trigger** node.
   - Configure webhook as **public**.
   - Set initial message: `"Hi Nani! 👋"`.
   - This node receives chat inputs from users.

2. **Add LangChain AI Agent Node**
   - Add **LangChain Agent** node.
   - Connect output of **Chat Trigger** node to input of **AI Agent** node.
   - Leave options default; this node coordinates AI processing.

3. **Add Gemini Chat Model Node**
   - Add **LangChain Google Gemini Chat** node.
   - Configure credentials with Google Gemini (PaLM) API OAuth2 credentials.
   - Connect this node as the AI language model input to the **AI Agent** node.

4. **Add Gemini AI Tool Node**
   - Add **LangChain Google Gemini Tool** node.
   - Set `modelId` to `"models/gemini-2.5-flash"`.
   - Add pre-message instruction: `"Give me user user-friendly reply. Don't give me a robotic type relay."`
   - Configure credentials with Google Gemini (PaLM) API OAuth2 credentials.
   - Connect this node as an AI tool input to the **AI Agent** node.

5. **Add HTTP Request Tool Node**
   - Add **HTTP Request** node.
   - Set URL to `"https://google.cm/"` (can be customized).
   - Connect this node as an AI tool input to the **AI Agent** node.

6. **Add Google Docs Node**
   - Add **Google Docs** node.
   - Operation: `get`.
   - Set **Document URL** dynamically to an expression that can be overridden via AI input.
   - Set **Simplify** option dynamically (boolean) via AI override.
   - Connect this node as an AI tool input to the **AI Agent** node.
   - Configure credentials with Google Docs OAuth2 credentials.

7. **Add Telegram Node for Chat Notification**
   - Add **Telegram** node.
   - Configure credentials with Telegram API OAuth2.
   - Set message text to include user question and AI reply using expressions.
   - Specify the `chatId` for the Telegram chat to send messages.
   - Connect output of **AI Agent** node to this node.

8. **Add Crypto Node**
   - Add **Crypto** node.
   - Set input value dynamically to AI output (`{{ $json.output }}`).
   - Connect output of **AI Agent** node to this node.

9. **Add Google Sheets Node**
   - Add **Google Sheets** node.
   - Operation: `append`.
   - Set Spreadsheet ID to desired Google Sheet document ID.
   - Set Sheet name (e.g., `gid=0`).
   - Define columns mapping:
     - `"replay message"` ← AI output
     - `"output data (Crypto)"` ← Crypto node output
   - Connect output of **Crypto** node to this node.
   - Configure credentials with Google Sheets OAuth2 credentials.

10. **Connect AI Agent Outputs**
    - Connect main output of **AI Agent** node to **Telegram** node and **Crypto** node.
    - Connect output of **Crypto** node to **Google Sheets** node.

11. **Add Sticky Notes (Optional)**
    - For documentation and instructions, add sticky notes with relevant content to guide users.

12. **Activate Workflow**
    - Ensure all credentials are valid and workflow is activated.
    - Share the public webhook URL from **Chat message** node for integration with chat platforms.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow listens for chat messages, uses Google Gemini AI (PaLM) via LangChain, fetches Google Docs, logs to Sheets | Overview and key features as documented in the main sticky note.                                  |
| Requires OAuth2 credentials for Google APIs (Gemini, Docs, Sheets) and Telegram API credentials                      | Credential setup essential for functionality.                                                     |
| Telegram chat ID must be set to receive AI notification messages                                                  | Telegram node configuration requirement.                                                         |
| Public webhook URL from Chat Trigger node allows integration with multiple chat platforms (Telegram, Slack, etc.)  | Enables external chat system connection.                                                         |
| Helpful community & author links:                                                                                  | [devcodejourney.com](https://devcodejourney.com), [LinkedIn Shakil](https://www.linkedin.com/in/shakilpg/), WhatsApp Channel, Direct Chat |
| AI Agent can be extended with custom tools and models by modifying LangChain Agent's tool and model inputs          | Flexible modular design for enhancements.                                                        |
| Google Docs document URL and simplify setting are dynamically handled via AI overrides                              | Allows dynamic document retrieval per user input.                                                |
| Crypto node can be customized for encryption or hashing of outputs before storing                                  | Security or integrity enhancement optional.                                                      |
| Initial greeting message can be customized in Chat Trigger node                                                   | Personalizes user experience.                                                                     |

---

This detailed reference fully describes the workflow structure, node configurations, connections, and usage instructions, enabling advanced users or automation agents to understand, reproduce, or modify it effectively.