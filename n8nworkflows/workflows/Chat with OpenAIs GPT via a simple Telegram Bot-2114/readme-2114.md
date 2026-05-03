Chat with OpenAIs GPT via a simple Telegram Bot

https://n8nworkflows.xyz/workflows/chat-with-openais-gpt-via-a-simple-telegram-bot-2114


# Chat with OpenAIs GPT via a simple Telegram Bot

### 1. Workflow Overview

This workflow enables a seamless chat interface between Telegram users and OpenAI's GPT language model (GPT-3.5 or GPT-4o-mini variant) through a Telegram bot. It is designed to provide AI-generated responses enriched with emojis, making conversations more engaging and user-friendly.

The workflow is structured into three main logical blocks:

- **1.1 Input Reception:** Captures incoming messages from Telegram users via a Telegram trigger node.
- **1.2 AI Processing:** Processes the incoming text using OpenAI’s GPT model through a LangChain AI Agent node which handles prompt formulation and response generation.
- **1.3 Output Delivery:** Sends the AI-generated response back to the user via the Telegram node.

This design leverages n8n’s LangChain integration to manage the AI interaction and Telegram API nodes to handle messaging.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for new Telegram messages sent to the bot and fires the workflow when a new message arrives.
- **Nodes Involved:**
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**
    - Type: Trigger node for Telegram messaging platform.
    - Role: Captures incoming Telegram messages (specifically "message" updates).
    - Configuration:
      - Listens for "message" updates only.
      - Uses a Telegram API credential linked to the bot (`jimleuk_handoff_bot`).
      - Webhook ID is auto-generated for receiving Telegram updates.
    - Inputs: External webhook trigger from Telegram servers.
    - Outputs: Emits JSON containing the incoming message details, including chat id and message text.
    - Edge Cases:
      - Possible failures include invalid Telegram API key or revoked bot token.
      - Network or webhook connection failures.
      - Message format variations that might lack expected fields.
    - Version: 1.1
    - Sub-workflow: None

#### 2.2 AI Processing

- **Overview:** This block formulates a prompt based on the user’s message and sends it to OpenAI’s GPT model via the LangChain AI Agent. It enriches the response by instructing the model to include emojis.
- **Nodes Involved:**
  - AI Agent
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**
    - Type: LangChain Agent node in n8n.
    - Role: Constructs and manages AI prompt/response flow using LangChain.
    - Configuration:
      - Prompt text: `"Respond to this as a helpful assistant with emojis:  {{ $json.message.text }}"` — dynamically injects user message text.
      - Prompt type: "define," meaning a custom prompt template is used.
      - Connects to OpenAI Chat Model as the underlying language model.
    - Inputs: Receives JSON from Telegram Trigger containing user message.
    - Outputs: Produces AI-generated text response.
    - Edge Cases:
      - Failures if prompt expression (`{{ $json.message.text }}`) references missing or malformed data.
      - Timeout or quota issues from OpenAI.
      - Unexpected AI output formats.
    - Version: 1.8
    - Sub-workflow: None

  - **OpenAI Chat Model**
    - Type: LangChain OpenAI chat language model node.
    - Role: Executes the actual call to OpenAI’s GPT API.
    - Configuration:
      - Model: Uses `"gpt-4o-mini"` variant, a lightweight GPT-4 option.
      - No additional options configured.
      - Credentials: OpenAI API key (`OpenAi account`).
    - Inputs: Connected as the language model for the AI Agent node.
    - Outputs: Provides AI textual responses back to the AI Agent.
    - Edge Cases:
      - Potential API key authentication errors.
      - Model availability or rate limiting.
      - Network timeouts.
    - Version: 1.2
    - Sub-workflow: None

#### 2.3 Output Delivery

- **Overview:** This block sends the AI-generated response back to the Telegram user in the appropriate chat.
- **Nodes Involved:**
  - Telegram

- **Node Details:**

  - **Telegram**
    - Type: Telegram node for sending messages.
    - Role: Sends AI response text back to the user’s chat.
    - Configuration:
      - Text: Dynamically set to AI Agent’s output (`{{ $json.output }}`).
      - Chat ID: Dynamically set to the chat id from the Telegram Trigger (`{{ $('Telegram Trigger').item.json.message.chat.id }}`).
      - Uses same Telegram API credential as the trigger node.
    - Inputs: Receives AI response from AI Agent.
    - Outputs: Sends message to Telegram user; no further output nodes.
    - Edge Cases:
      - Errors if chat id is missing or invalid.
      - Telegram API limits or downtime.
      - Message size limits or formatting errors.
    - Version: 1.2
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name        | Node Type                                | Functional Role          | Input Node(s)      | Output Node(s)    | Sticky Note                                    |
|------------------|-----------------------------------------|-------------------------|--------------------|-------------------|-----------------------------------------------|
| Telegram Trigger  | n8n-nodes-base.telegramTrigger           | Input Reception         | -                  | AI Agent          |                                               |
| AI Agent         | @n8n/n8n-nodes-langchain.agent          | AI Processing           | Telegram Trigger, OpenAI Chat Model | Telegram           |                                               |
| OpenAI Chat Model| @n8n/n8n-nodes-langchain.lmChatOpenAi   | AI Language Model       | AI Agent (ai_languageModel input) | AI Agent          |                                               |
| Telegram         | n8n-nodes-base.telegram                  | Output Delivery         | AI Agent           | -                 |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Add a new node: Type `Telegram Trigger`.
   - Configure to listen to `message` updates only.
   - Set webhook ID (auto-generated or leave default).
   - Add Telegram API credentials:
     - Create or select a Telegram API credential using your Telegram bot token from BotFather.
   - Position this node as the workflow start.

2. **Create OpenAI Chat Model Node**
   - Add a new node: Type `LangChain OpenAI Chat Model`.
   - Set the model to `gpt-4o-mini` (or your preferred GPT-3.5/4 variant).
   - Assign OpenAI API credentials:
     - Use your OpenAI API key credential.
   - Leave options blank unless customization is needed.

3. **Create AI Agent Node**
   - Add a new node: Type `LangChain Agent`.
   - Set `Prompt Type` to `define` (custom prompt).
   - In the `text` parameter, enter:  
     `=Respond to this as a helpful assistant with emojis:  {{ $json.message.text }}`
   - Connect the `ai_languageModel` input of this node to the output of the OpenAI Chat Model node.
   - Connect the main input of this node to the output of the Telegram Trigger node.

4. **Create Telegram Node**
   - Add a new node: Type `Telegram`.
   - Set the `text` parameter to:  
     `={{ $json.output }}`
   - Set the `chatId` parameter to:  
     `={{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Use the same Telegram API credentials as the trigger node.
   - Connect the main input of this node to the output of the AI Agent node.

5. **Connect Nodes**
   - Telegram Trigger → AI Agent (main)
   - OpenAI Chat Model → AI Agent (ai_languageModel input)
   - AI Agent → Telegram

6. **Save and Activate Workflow**
   - Verify Telegram webhook is registered correctly.
   - Test by sending messages to your Telegram bot.
   - Adjust OpenAI model or prompt text as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                   |
|----------------------------------------------------------------------------------------------|--------------------------------------------------|
| Add your Telegram API key and OpenAI API key in the respective credentials sections.         | Instruction from workflow description             |
| The prompt instructs GPT to respond "as a helpful assistant with emojis" for engaging replies.| Custom prompt design                               |
| This workflow uses n8n LangChain integration for better AI prompt management and modularity. | n8n documentation on LangChain nodes              |
| Telegram bots require webhook setup via BotFather and proper bot token for API access.       | Telegram Bot API documentation                     |
| OpenAI model `gpt-4o-mini` is a lightweight GPT-4 variant; change model if you want GPT-3.5. | OpenAI API model list and descriptions             |

---

This documentation fully describes the workflow and should enable developers or AI agents to understand, reproduce, and maintain the Telegram-to-OpenAI chat integration efficiently.