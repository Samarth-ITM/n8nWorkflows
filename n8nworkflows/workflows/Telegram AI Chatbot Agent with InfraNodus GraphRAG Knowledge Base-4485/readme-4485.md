Telegram AI Chatbot Agent with InfraNodus GraphRAG Knowledge Base

https://n8nworkflows.xyz/workflows/telegram-ai-chatbot-agent-with-infranodus-graphrag-knowledge-base-4485


# Telegram AI Chatbot Agent with InfraNodus GraphRAG Knowledge Base

### 1. Workflow Overview

This workflow implements a Telegram AI Chatbot Agent integrated with InfraNodus GraphRAG knowledge bases to provide expert-level, context-aware answers. It is designed for conversational support, ideation, and writing assistance by leveraging multiple specialized knowledge graphs as “experts.” The chatbot intelligently selects which InfraNodus knowledge graph to query based on the user’s input, synthesizes responses from these sources, and maintains conversational context for coherent interactions.

Logical blocks:

- **1.1 Telegram Input Reception**: Listens to incoming Telegram messages and provides immediate feedback to users.
- **1.2 Chat Memory Management**: Tracks conversation context with a memory buffer to maintain coherent dialogue.
- **1.3 AI Agent & OpenAI Language Model**: Uses OpenAI GPT-4o as the language model backend and an AI Agent node to coordinate tool (expert) usage.
- **1.4 InfraNodus Expert Integrations**: Four HTTP Request nodes representing distinct InfraNodus knowledge graphs, serving as expert tools queried dynamically.
- **1.5 Response Delivery**: Sends the synthesized AI-generated reply back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

- **Overview:** This block triggers the workflow upon receiving a Telegram message, then immediately sends a “typing...” action to indicate the bot is working, improving user experience.
- **Nodes Involved:**  
  - Receive a Message on Telegram  
  - Send "Typing..." message to User  
- **Node Details:**

  - **Receive a Message on Telegram**  
    - Type: Telegram Trigger  
    - Role: Entry point listening to all incoming Telegram messages.  
    - Configuration: Listens for "message" updates only; uses Telegram API credentials linked to the bot.  
    - Input: Telegram webhook events.  
    - Output: Message JSON payload including chat and user info.  
    - Edge Cases: Potential webhook misconfiguration, Telegram API token invalid or revoked, message format issues.

  - **Send "Typing..." message to User**  
    - Type: Telegram node  
    - Role: Sends “typing” chat action to Telegram chat to signal bot activity.  
    - Configuration: Uses chat ID extracted from the incoming message JSON; operation set to sendChatAction.  
    - Input: Message JSON from previous node.  
    - Output: Confirmation of chat action sent.  
    - Edge Cases: Telegram API rate limits, chat ID invalid if message malformed.

#### 2.2 Chat Memory Management

- **Overview:** Maintains conversational context by storing recent chat history keyed by the Telegram chat ID, enabling context-aware AI responses.
- **Nodes Involved:**  
  - Simple Memory  
- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain memoryBufferWindow node  
    - Role: Stores and retrieves conversation history for the ongoing chat session.  
    - Configuration: Uses Telegram chat ID as session key (custom key type).  
    - Key Expressions: `sessionKey` = `={{ $('Receive a Message on Telegram').item.json.message.chat.id }}`  
    - Input: Incoming message JSON.  
    - Output: Memory object containing dialogue history.  
    - Edge Cases: Memory overflow if conversation is very long; session key collision or changes in chat ID format.

#### 2.3 AI Agent & OpenAI Language Model

- **Overview:** Core AI processing block that uses GPT-4o to interpret user queries, dynamically selects which InfraNodus expert tools to query, and synthesizes their responses into a coherent answer.
- **Nodes Involved:**  
  - OpenAI Model  
  - AI Agent  
- **Node Details:**

  - **OpenAI Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides GPT-4o language generation capabilities.  
    - Configuration: Model set to "gpt-4o" with default options; OpenAI API credentials configured.  
    - Input: Prompt from AI Agent node.  
    - Output: Generated text completion used by AI Agent.  
    - Edge Cases: API rate limits, invalid API key, timeout or network errors.

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Coordinates tools (InfraNodus experts), processes user input, modifies queries per tool context, and synthesizes final responses.  
    - Configuration:  
      - Input text: user message text from Telegram.  
      - System prompt: Describes expertise on Dmitry Paranyushkin’s books and instructs on tool usage and combining perspectives.  
      - Prompt type: "define" to specify system prompt explicitly.  
      - Tools: Connected to all InfraNodus HTTP Request nodes (experts).  
    - Key Expressions:  
      - User message: `={{ $('Receive a Message on Telegram').item.json.message.text }}`  
    - Input: User text, session memory, language model output, plus tool responses.  
    - Output: Final synthesized answer text.  
    - Edge Cases: Misalignment between tool descriptions and actual tool output, failure to pick tools, unexpected AI outputs.

#### 2.4 InfraNodus Expert Integrations

- **Overview:** Four HTTP Request Tool nodes query InfraNodus API endpoints representing different knowledge graphs (“experts”) specialized on various themes from Dmitry Paranyushkin’s books. Each receives a context-adapted prompt from the AI Agent.
- **Nodes Involved:**  
  - Special Agent's Manual Book Expert  
  - Waves into Patterns Book Expert  
  - The Flow and the Notion Book  
  - The Polysingularity Letters Book  
- **Node Details (commonalities):**

  - Type: HTTP Request Tool  
  - Role: Send POST requests to InfraNodus API `/graphAndAdvice` endpoint with parameters including:  
    - `name`: InfraNodus graph identifier  
    - `prompt`: User query adjusted for context (auto-overridden by AI Agent)  
    - `requestMode`: "response"  
    - `aiTopics`: true (enables topic extraction)  
  - Authentication: HTTP Bearer with InfraNodus API keys.  
  - Output: JSON containing graph summaries and advice statements used by AI Agent.  
  - Edge Cases: HTTP errors (timeout, 401 unauthorized), malformed responses, API quota limits.

  - **Special Agent's Manual Book Expert**  
    - Graph name: `special_agents_manual`  
    - Description: Expert on human agency, infiltration, tension dynamics, and identity.

  - **Waves into Patterns Book Expert**  
    - Graph name: `special_agents_manual` (appears same as above, possibly reuse or typo)  
    - Description: Expert on natural cycles, fractal dynamics, variability.

  - **The Flow and the Notion Book**  
    - Graph name: `the_flow_and_notion`  
    - Description: Expert on creativity, dreaming, art, dissipating ideas.

  - **The Polysingularity Letters Book**  
    - Graph name: `polysingularity_letters`  
    - Description: Expert on networks, multiplicity, consciousness, meaning creation.

#### 2.5 Response Delivery

- **Overview:** After the AI Agent generates the final response, this block sends that message back to the user’s Telegram chat.
- **Nodes Involved:**  
  - Send Telegram Message to User  
- **Node Details:**

  - **Send Telegram Message to User**  
    - Type: Telegram node  
    - Role: Sends the AI-generated answer text to the Telegram chat.  
    - Configuration:  
      - `text`: Output from AI Agent (`$json.output`)  
      - `chatId`: Extracted from original Telegram message  
      - `appendAttribution`: false (does not append bot attribution)  
    - Input: AI Agent's output.  
    - Output: Confirmation of message sent.  
    - Edge Cases: Invalid chat ID, Telegram API rate limits, message length exceeding Telegram limits.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                                 | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                                               |
|-----------------------------------|----------------------------------|------------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Receive a Message on Telegram      | Telegram Trigger                 | Entry point: receives Telegram messages         | —                             | Send "Typing..." message to User | ## 1. Telegram Trigger<br>The conversation will be triggered when you send a message to your Telegram bot. To set up a bot, simply message `/newbot` to `@botfather` in Telegram. It takes 30 seconds. |
| Send "Typing..." message to User   | Telegram                        | Sends typing action to Telegram chat            | Receive a Message on Telegram  | AI Agent                     | ## 2. Give Feedback to User<br>This node gives feedback to the user that the Agent is working.                                              |
| Simple Memory                     | LangChain memoryBufferWindow     | Maintains chat memory for context                | —                             | AI Agent                     | ## Chat Memory<br>We use the Simple Memory node to track the conversation's context so that the user can refer to previous messages.       |
| OpenAI Model                     | LangChain LM Chat OpenAI         | Provides GPT-4o language generation              | AI Agent                      | AI Agent                     |                                                                                                                                           |
| AI Agent                        | LangChain Agent                  | Coordinates tools, processes query, synthesizes response | Send "Typing..." message to User, Simple Memory, OpenAI Model, InfraNodus Experts | Send Telegram Message to User | ## 3. AI Agent Workflow<br>Chooses which tool (expert) to use, depending on the user's message. Then receives the responses and synthesizes the final answer.<br>Make sure you describe the tools available well both in the Agent's System Prompt and in the tools' descriptions. |
| Special Agent's Manual Book Expert | HTTP Request Tool (InfraNodus)  | InfraNodus knowledge graph expert #1             | AI Agent                      | AI Agent                     | ## Expert #1<br>Add your InfraNodus graph here via the HTTP node using its name in the `body.name` field. Describe what the expert does in the Description of the tool.<br>![Book Screenshot](https://i.ibb.co/rfxsJ4MV/circadian-special-agents-manual.png) |
| Waves into Patterns Book Expert    | HTTP Request Tool (InfraNodus)  | InfraNodus knowledge graph expert #2             | AI Agent                      | AI Agent                     | ## Expert #2<br>Add your InfraNodus graph here via the HTTP node using its name in the `body.name` field. Describe what the expert does in the Description of the tool.<br>![waves into patterns screen](https://i.ibb.co/1tDJSgVq/circadian-waves-into-patterns.png) |
| The Flow and the Notion Book       | HTTP Request Tool (InfraNodus)  | InfraNodus knowledge graph expert #3             | AI Agent                      | AI Agent                     | ## Expert #3<br>You can add more experts here. Just make to give them descriptive names, so the agent knows which one to connect to when it has a question.<br>![flow and notion image](https://i.ibb.co/prLbFG0w/circadian-the-flow-and-notion.png) |
| The Polysingularity Letters Book   | HTTP Request Tool (InfraNodus)  | InfraNodus knowledge graph expert #4             | AI Agent                      | AI Agent                     | ## Expert #4<br>You can add more experts here. Just make to give them descriptive names, so the agent knows which one to connect to when it has a question.<br>![infranodus graph](https://i.ibb.co/hRqxn8JN/circadian-conversation-book.png) |
| Send Telegram Message to User      | Telegram                        | Sends AI-generated response back to Telegram user | AI Agent                      | —                            | ## 4. Respond to the User<br>Once the response is generated, send it back to the user's chat in Telegram.                                    |
| Sticky Note                       | Sticky Note                     | Describes purpose and InfraNodus integration     | —                             | —                            | ## AI Chatbot Agent with Experts<br>Uses InfraNodus knowledge graphs and Graph RAG to retrieve relevant information.<br>InfraNodus API key info and video demo included. |
| Sticky Note4                      | Sticky Note                     | Chat memory explanation                           | —                             | —                            | ## Chat Memory<br>We use the Simple Memory node to track the conversation's context so that the user can refer to previous messages.       |
| Sticky Note5                      | Sticky Note                     | Describes AI Agent workflow                       | —                             | —                            | ## 3. AI Agent Workflow<br>Chooses which tool (expert) to use, depending on the user's message. Then receives the responses and synthesizes the final answer. |
| Sticky Note6                      | Sticky Note                     | Telegram trigger explanation                       | —                             | —                            | ## 1. Telegram Trigger<br>The conversation will be triggered when you send a message to your Telegram bot. To set up a bot, message `/newbot` to `@botfather`. |
| Sticky Note7                      | Sticky Note                     | Feedback to user explanation                       | —                             | —                            | ## 2. Give Feedback to User<br>This node gives feedback to the user that the Agent is working.                                                |
| Sticky Note8                      | Sticky Note                     | Response delivery explanation                      | —                             | —                            | ## 4. Respond to the User<br>Once the response is generated, send it back to the user's chat in Telegram.                                    |
| Sticky Note9                      | Sticky Note                     | Expert #4 description                              | —                             | —                            | ## Expert #4<br>You can add more experts here. Just make to give them descriptive names, so the agent knows which one to connect to when it has a question. |
| Sticky Note10                     | Sticky Note                     | Telegram bot setup instructions                    | —                             | —                            | ## Setting up a Telegram bot<br>Step-by-step instructions with @botfather to create the bot and configure access tokens and group permissions. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Credentials**  
   - Use Telegram Bot API token obtained from `@botfather`.  
   - Name the credential appropriately (e.g., “Dmitry's Books Bot”).

2. **Add Telegram Trigger node**  
   - Node type: Telegram Trigger  
   - Configure for update type: `message` only.  
   - Set Telegram credentials.  
   - This node will be the workflow entry point.

3. **Add Telegram node "Send 'Typing...' message to User"**  
   - Node type: Telegram  
   - Operation: `sendChatAction`  
   - Chat ID: `={{ $json.message.chat.id }}` (from Telegram Trigger output)  
   - Set same Telegram credentials.  
   - Connect Telegram Trigger main output to this node’s input.

4. **Add Simple Memory node**  
   - Node type: LangChain memoryBufferWindow  
   - Set `sessionKey` to `={{ $('Receive a Message on Telegram').item.json.message.chat.id }}`  
   - Session ID type: customKey  
   - This node stores conversation history.

5. **Add OpenAI Model node**  
   - Node type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o`  
   - Configure OpenAI API credentials.  
   - No special options needed.

6. **Add AI Agent node**  
   - Node type: LangChain Agent  
   - Text input: `={{ $('Receive a Message on Telegram').item.json.message.text }}`  
   - System message:  
     ```
     You are well-versed on Dmitry Paranyushkin's books through the tools you have access to. When you receive a user's query, you can modify it to suit better each tool's context. Always access at least one of the tools and deliver an augmented response.

     When you're generating a response, attempt to combine perspectives where they fit and point out discrepancies when they exist.
     ```
   - Prompt type: define  
   - Connect inputs from:  
     - “Send 'Typing...' message to User” (main)  
     - Simple Memory (ai_memory)  
     - OpenAI Model (ai_languageModel)  
   - Add tools (ai_tool) connections to InfraNodus HTTP Request nodes (next step).

7. **Add InfraNodus HTTP Request Tool nodes (4 experts)**  
   - Node type: HTTP Request Tool  
   - Method: POST  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeGraph=false&includeGraphSummary=true&includeStatements=true`  
   - Authentication: HTTP Bearer with InfraNodus API key credentials.  
   - Body parameters:  
     - `name`: InfraNodus graph identifier (e.g., `special_agents_manual`, `the_flow_and_notion`, `polysingularity_letters`)  
     - `prompt`: `={{ $fromAI('parameters1_Value', 'User\'s request adjusted to suit this context', 'string') }}` (auto-overridden by AI Agent)  
     - `requestMode`: `response`  
     - `aiTopics`: `true`  
   - Tool Description: Provide descriptive text on the expert’s domain and topics.  
   - Connect each node as an `ai_tool` input into the AI Agent node.

8. **Connect AI Agent node output to Telegram node "Send Telegram Message to User"**  
   - Node type: Telegram  
   - Operation: sendMessage  
   - Chat ID: `={{ $('Receive a Message on Telegram').item.json.message.chat.id }}`  
   - Text: `={{ $json.output }}` (AI Agent response)  
   - Credentials: Telegram bot credentials.

9. **Workflow Connections Summary**  
   - Telegram Trigger → Send "Typing..." message to User → AI Agent  
   - Simple Memory → AI Agent (ai_memory input)  
   - OpenAI Model → AI Agent (ai_languageModel input)  
   - Each InfraNodus HTTP Request Tool → AI Agent (ai_tool inputs)  
   - AI Agent → Send Telegram Message to User  

10. **Test workflow**  
    - Deploy.  
    - Send messages to Telegram bot and verify responses.  
    - Monitor logs for errors (auth, API limits, etc.).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Use your InfraNodus graph as the knowledge base for your AI chatbot. Upload any data, generate knowledge graphs, then connect them as tools to the AI agent for expert responses. InfraNodus API key can be obtained at https://infranodus.com/use-case/ai-knowledge-graphs | InfraNodus integration overview and API key source.                                                                                        |
| Video demo of this workflow: https://www.youtube.com/watch?v=kS0QTUvcH6E                                                                                                                                                              | Video tutorial on the InfraNodus AI Chatbot Agent with GraphRAG knowledge base.                                                             |
| Detailed description and setup instructions available at Nodus Labs support portal: https://support.noduslabs.com/hc/en-us/articles/20174217658396-Using-InfraNodus-Knowledge-Graphs-as-Experts-for-AI-Chatbot-Agents-in-n8n          | Official documentation for configuring InfraNodus knowledge graphs as AI experts in n8n.                                                    |
| Telegram bot setup requires creating a bot with @botfather and configuring credentials in n8n. For group usage, enable group permissions and disable group privacy for the bot to read group messages. Bots cannot respond to each other due to Telegram limits. | Telegram setup detailed instructions (Sticky Note10).                                                                                       |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.