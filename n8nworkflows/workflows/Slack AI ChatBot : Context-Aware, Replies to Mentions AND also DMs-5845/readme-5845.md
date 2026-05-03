Slack AI ChatBot : Context-Aware, Replies to Mentions AND also DMs

https://n8nworkflows.xyz/workflows/slack-ai-chatbot---context-aware--replies-to-mentions-and-also-dms-5845


# Slack AI ChatBot : Context-Aware, Replies to Mentions AND also DMs

### 1. Workflow Overview

This workflow implements a **Slack AI ChatBot** that is **context-aware** and capable of replying to both Slack mentions in public channels and direct messages (DMs). It is designed to provide intelligent, conversational replies by leveraging AI models, memory buffers, and Slack channel history, ensuring responses are relevant to the ongoing conversation context.

The workflow is logically divided into the following functional blocks:

- **1.1 Slack Event Reception**  
  Captures incoming Slack events, specifically mentions and DMs, to trigger the chatbot.

- **1.2 Data Preparation and Context Mapping**  
  Extracts and maps Slack event data to a format suitable for AI processing.

- **1.3 AI Processing and Contextualization**  
  Uses OpenAI chat models and Langchain tools (memory buffers, AI agents, and optionally vector stores) to generate context-aware responses.

- **1.4 Response Routing**  
  Determines whether to reply publicly in a channel or privately via DM based on the event type.

- **1.5 Response Dispatch to Slack**  
  Sends the generated response back to Slack via appropriate channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Event Reception

- **Overview:**  
  This block listens for Slack events where the bot is either mentioned in a public channel or receives a direct message, triggering the workflow.

- **Nodes Involved:**  
  - Slack Trigger

- **Node Details:**

  - **Slack Trigger**  
    - Type: Slack Trigger node  
    - Role: Entry point that listens for Slack events such as mentions and DMs.  
    - Configuration: Connected to a specific Slack workspace via OAuth2 credentials; configured to receive message events.  
    - Inputs: External Slack events  
    - Outputs: Event data payload forwarded to the next node for processing  
    - Edge Cases: Possible failures include expired OAuth tokens, Slack API rate limits, or misconfigured event subscriptions.

---

#### 2.2 Data Preparation and Context Mapping

- **Overview:**  
  This block processes the incoming Slack event data, mapping it into a suitable format for AI consumption and decision-making.

- **Nodes Involved:**  
  - Mapping data for the Agent

- **Node Details:**

  - **Mapping data for the Agent**  
    - Type: Set node  
    - Role: Extracts relevant fields from Slack event payload (e.g., message text, user info, channel id) and structures them for AI input.  
    - Configuration: Uses expressions to map Slack event fields to standardized variables for downstream nodes.  
    - Inputs: Slack event data from Slack Trigger  
    - Outputs: Mapped data forwarded to AI Agent  
    - Edge Cases: Expression evaluation errors if expected fields are missing or malformed Slack payloads.

---

#### 2.3 AI Processing and Contextualization

- **Overview:**  
  This block integrates multiple AI-related nodes to generate context-aware responses by combining Slack channel history, AI memory, and language models.

- **Nodes Involved:**  
  - AI Agent1  
  - Simple Memory1  
  - OpenAI Chat Model1  
  - Get the history of a channel in Slack  
  - Think  
  - Pinecone Vector Store (disabled)  
  - Embeddings OpenAI (disabled)

- **Node Details:**

  - **AI Agent1**  
    - Type: Langchain Agent node  
    - Role: Central AI logic node orchestrating memory, tools, and language model to generate the chatbot response.  
    - Configuration: Connected with ai_memory (Simple Memory1), ai_languageModel (OpenAI Chat Model1), ai_tool (Think, Get the history of a channel in Slack) inputs.  
    - Inputs: Mapped data, memory buffer, AI language model, and tools outputs.  
    - Outputs: Generated AI response text to routing decision node.  
    - Edge Cases: API key limits, AI model timeouts, malformed input, or memory sync issues.

  - **Simple Memory1**  
    - Type: Memory Buffer Window (Langchain)  
    - Role: Maintains a sliding window of recent conversation history for context continuity.  
    - Configuration: Default window size, stores recent messages to feed AI Agent.  
    - Inputs: Event data and AI responses  
    - Outputs: Context memory to AI Agent  
    - Edge Cases: Memory overflow, data loss on restart.

  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides the core natural language understanding and generation capabilities.  
    - Configuration: Uses OpenAI API via configured credentials; model parameters can be adjusted (e.g., temperature).  
    - Inputs: Prompt from AI Agent  
    - Outputs: Generated text response  
    - Edge Cases: API rate limits, authentication errors, invalid prompts.

  - **Get the history of a channel in Slack**  
    - Type: Slack Tool node  
    - Role: Retrieves recent messages from the Slack channel to enrich conversation context for AI.  
    - Configuration: Uses Slack API with OAuth2 credentials; configured to fetch recent channel messages.  
    - Inputs: Slack event info from AI Agent tool input  
    - Outputs: Channel history as tool output to AI Agent  
    - Edge Cases: Slack API rate limits, permission errors, empty or inaccessible channels.

  - **Think**  
    - Type: Langchain Tool Think node  
    - Role: Auxiliary AI tool to assist AI Agent in reasoning or intermediate thought processing.  
    - Configuration: Connected as ai_tool input to AI Agent  
    - Inputs: Data from AI Agent or other nodes  
    - Outputs: Reasoning results to AI Agent  
    - Edge Cases: Timeouts or failures in tool processing.

  - **Pinecone Vector Store** *(disabled)*  
    - Type: Langchain Pinecone Vector Store node  
    - Role: Intended for semantic search or vector similarity search in AI workflows (disabled here).  
    - Configuration: None active  
    - Edge Cases: Not active; no effect.

  - **Embeddings OpenAI** *(disabled)*  
    - Type: Langchain OpenAI Embeddings node  
    - Role: Generates embeddings for semantic search or vector similarity (disabled here).  
    - Edge Cases: Not active; no effect.

---

#### 2.4 Response Routing

- **Overview:**  
  This block decides whether the chatbot reply should be sent as a direct message or a public reply in the Slack channel based on the original event context.

- **Nodes Involved:**  
  - Either the bot should reply in dm or in public channel (If node)

- **Node Details:**

  - **Either the bot should reply in dm or in public channel**  
    - Type: If node  
    - Role: Checks event metadata to route the AI-generated response either to DM replying node or public channel reply node.  
    - Configuration: Condition based on Slack event type or presence of DM metadata.  
    - Inputs: AI Agent response  
    - Outputs: Two branches: DM reply and public channel reply  
    - Edge Cases: Incorrect routing if event data is ambiguous or missing fields.

---

#### 2.5 Response Dispatch to Slack

- **Overview:**  
  This block sends the AI-generated reply back to Slack, either as a direct message or as a reply in a public channel.

- **Nodes Involved:**  
  - Reply to DM  
  - Reply to public mention

- **Node Details:**

  - **Reply to DM**  
    - Type: Slack node  
    - Role: Sends a direct message reply to the user who initiated the DM.  
    - Configuration: Uses Slack OAuth2 credentials; message content is AI-generated response.  
    - Inputs: Routed from If node DM branch  
    - Outputs: Confirmation of message sent  
    - Edge Cases: Permission errors, user blocking bot, API limits.

  - **Reply to public mention**  
    - Type: Slack node  
    - Role: Posts a reply message in the public Slack channel where the bot was mentioned.  
    - Configuration: Uses Slack OAuth2 credentials; posts in thread or channel as reply.  
    - Inputs: Routed from If node public branch  
    - Outputs: Confirmation of message sent  
    - Edge Cases: Channel posting restrictions, API rate limits.

---

### 3. Summary Table

| Node Name                                | Node Type                           | Functional Role                         | Input Node(s)                 | Output Node(s)                              | Sticky Note                            |
|-----------------------------------------|-----------------------------------|---------------------------------------|------------------------------|---------------------------------------------|--------------------------------------|
| Slack Trigger                           | Slack Trigger                     | Entry point for Slack events          | -                            | Mapping data for the Agent                   |                                      |
| Mapping data for the Agent              | Set                               | Data extraction and formatting        | Slack Trigger                | AI Agent1                                  |                                      |
| AI Agent1                              | Langchain Agent                   | Core AI processing and response gen   | Simple Memory1, OpenAI Chat Model1, Think, Get the history of a channel in Slack, Mapping data for the Agent | Either the bot should reply in dm or in public channel |                                      |
| Simple Memory1                         | Memory Buffer Window (Langchain) | Maintains conversation history        | -                            | AI Agent1                                  |                                      |
| OpenAI Chat Model1                     | Langchain OpenAI Chat Model       | Natural language generation            | AI Agent1                    | AI Agent1                                  |                                      |
| Get the history of a channel in Slack | Slack Tool                       | Retrieves Slack channel history       | AI Agent1                    | AI Agent1                                  |                                      |
| Think                                 | Langchain Tool Think              | Auxiliary reasoning tool               | AI Agent1                    | AI Agent1                                  |                                      |
| Either the bot should reply in dm or in public channel | If                                | Routes response to DM or public reply | AI Agent1                    | Reply to DM, Reply to public mention         |                                      |
| Reply to DM                           | Slack node                       | Sends DM reply                        | Either the bot should reply… | -                                           |                                      |
| Reply to public mention               | Slack node                       | Sends public channel reply             | Either the bot should reply… | -                                           |                                      |
| Pinecone Vector Store                 | Langchain Vector Store (Pinecone) | Disabled vector store node             | Embeddings OpenAI            | -                                           | Disabled                            |
| Embeddings OpenAI                    | Langchain Embeddings              | Disabled embedding generation          | -                            | Pinecone Vector Store                        | Disabled                            |
| Sticky Note4                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note5                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note6                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note7                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note8                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note9                         | Sticky Note                      | -                                     | -                            | -                                           |                                      |
| Sticky Note10                        | Sticky Note                      | -                                     | -                            | -                                           |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node:**  
   - Type: Slack Trigger  
   - Configure OAuth2 credentials for your Slack workspace.  
   - Set event subscriptions to listen for messages where the bot is mentioned and for direct messages.  
   - Position: Entry node.

2. **Add Set node "Mapping data for the Agent":**  
   - Use expressions to extract relevant Slack event fields: message text, user ID, channel ID, timestamp, etc.  
   - This node prepares structured data for AI consumption.  
   - Connect Slack Trigger → Mapping data for the Agent.

3. **Add Langchain OpenAI Chat Model node "OpenAI Chat Model1":**  
   - Configure with OpenAI API credentials.  
   - Set model parameters such as model name (e.g., gpt-4 or gpt-3.5-turbo), temperature, max tokens.  
   - Position accordingly.

4. **Add Langchain Memory Buffer Window node "Simple Memory1":**  
   - Configure sliding window size for conversation history (default or as per requirement).  
   - Connect inputs to receive conversation exchanges for context retention.

5. **Add Langchain Tool node "Get the history of a channel in Slack":**  
   - Configure Slack OAuth2 credentials.  
   - Set to fetch recent message history from the Slack channel based on incoming event data.  
   - Connect as ai_tool input to AI Agent.

6. **Add Langchain Tool Think node "Think":**  
   - No special configuration needed; acts as an auxiliary AI tool for internal reasoning.  
   - Connect as ai_tool input to AI Agent.

7. **Add Langchain Agent node "AI Agent1":**  
   - Connect inputs:  
     - ai_memory → Simple Memory1  
     - ai_languageModel → OpenAI Chat Model1  
     - ai_tool → Think, Get the history of a channel in Slack  
     - main → Mapping data for the Agent output  
   - This node orchestrates generation of AI responses using memory, language model, and tools.

8. **Add If node "Either the bot should reply in dm or in public channel":**  
   - Configure condition to check if the Slack event is a DM or a public mention.  
   - Example condition: check if the Slack event channel type is “im” (DM) or not.  
   - Connect AI Agent1 main output → If node.

9. **Add Slack node "Reply to DM":**  
   - Configure with Slack OAuth2 credentials.  
   - Set to send message to the user via DM using the AI Agent response text.  
   - Connect If node “true” branch (DM condition) → Reply to DM.

10. **Add Slack node "Reply to public mention":**  
    - Configure with Slack OAuth2 credentials.  
    - Set to post reply message in the public channel where the bot was mentioned, ideally as a thread reply.  
    - Connect If node “false” branch (public mention) → Reply to public mention.

11. **(Optional) Add Langchain Embeddings OpenAI and Pinecone Vector Store nodes:**  
    - These nodes are disabled in the current workflow but can be configured for advanced semantic search or vector-based context retrieval.  
    - Configure API keys and connection parameters if used.

12. **Add Sticky Notes as needed for documentation or reminders.**

13. **Test the workflow:**  
    - Ensure Slack app is installed with correct permissions.  
    - Validate credentials, API keys, and event subscriptions.  
    - Trigger mentions and DMs to verify replies.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow enables AI chatbot to reply contextually in Slack, covering both DMs and public mentions. | General workflow purpose.                                                                            |
| Uses Langchain integration nodes for AI agent, memory, and tools.                                | n8n Langchain nodes documentation: https://docs.n8n.io/integrations/built-in/n8n-nodes-langchain/   |
| Slack OAuth2 credentials require proper bot scopes: chat:write, channels:history, im:history, etc. | Slack API Bot Token Scopes: https://api.slack.com/scopes                                                 |
| OpenAI API key required with access to chat models (e.g., GPT-4 or GPT-3.5)                      | OpenAI API Documentation: https://platform.openai.com/docs/api-reference/chat                         |
| Disabled Pinecone Vector Store and Embeddings nodes suggest possible future semantic search expansion. | Pinecone Vector Store: https://www.pinecone.io/docs/                                                  |
| Slack event types to monitor: message.im (DMs) and message.channels (mentions)                   | Slack Events API: https://api.slack.com/events/message.im, message.channels                             |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.