Telegram Chat Summarizer with AI - using @telepilotco/n8n-nodes-telepilot

https://n8nworkflows.xyz/workflows/telegram-chat-summarizer-with-ai---using--telepilotco-n8n-nodes-telepilot-4461


# Telegram Chat Summarizer with AI - using @telepilotco/n8n-nodes-telepilot

### 1. Workflow Overview

This workflow is designed to summarize Telegram chat messages using AI. It listens for incoming chat messages or can be manually triggered, collects recent messages from a specified chat, filters and aggregates them, then uses an AI agent powered by Langchain and OpenRouter to generate a summary. Finally, it sends the summarized content back to the Telegram chat. The workflow integrates the Telepilot nodes for Telegram API interaction and Langchain nodes for AI processing with MongoDB as chat memory.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception**: Triggers the workflow either manually or automatically on receiving new Telegram chat messages.
- **1.2 Telegram Chat Data Retrieval**: Retrieves the target chat ID and fetches the recent chat messages.
- **1.3 Message Filtering and Preparation**: Extracts message text, filters messages by recency and emptiness, and aggregates them for processing.
- **1.4 AI Processing**: Uses Langchain’s AI Agent with OpenRouter Chat Model and MongoDB chat memory to generate a chat summary.
- **1.5 Output Sending**: Sends the AI-generated summary back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow either manually via an execution trigger or automatically upon receiving a Telegram chat message.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Schedule Trigger  
- When chat message received  
- Telegram Login  
- Try this for check Login (testing node)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually for testing or forced execution  
  - Connections: Outputs to “Get Chat Id By Name”  
  - Edge Cases: Manual trigger bypasses normal Telegram webhook, so no new messages fetched.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Can be used to run the workflow on a schedule (though no cron or interval parameters set here)  
  - Connections: Outputs to “Get Chat Id By Name”  
  - Edge Cases: No schedule parameters configured; may not trigger automatically.

- **When chat message received**  
  - Type: Langchain Chat Trigger (Telegram webhook)  
  - Role: Automatically triggers when new Telegram chat messages arrive  
  - Configuration: Uses a webhook with ID "b0697338-a2a6-4f38-b78b-970e420b6947"  
  - Connections: Outputs to “Telegram Login”  
  - Edge Cases: Requires proper webhook setup in Telegram API; failure if webhook not registered or if Telegram API down.

- **Telegram Login**  
  - Type: Telepilot (Telegram API node)  
  - Role: Authenticates and maintains session with Telegram to enable API calls  
  - Connections: Outputs to “Try this for check Login”  
  - Edge Cases: Authentication failure, session expiration.

- **Try this for check Login**  
  - Type: Telepilot (Telegram API node)  
  - Role: Test node to verify Telegram login/session  
  - Position: Connected after Telegram Login, but no downstream connections  
  - Edge Cases: Used for debugging; no effect on main workflow.

---

#### 1.2 Telegram Chat Data Retrieval

**Overview:**  
This block obtains the Telegram chat ID by name and retrieves the last messages from that chat for summarization.

**Nodes Involved:**  
- Get Chat Id By Name  
- Last Messages History  
- Split Out

**Node Details:**

- **Get Chat Id By Name**  
  - Type: Telepilot node  
  - Role: Resolves a Telegram chat’s name/username to its unique chat ID  
  - Configuration: Likely configured with chat name input parameter  
  - Connections: Outputs to “Last Messages History”  
  - Edge Cases: Chat name not found, or permission denied to access chat info.

- **Last Messages History**  
  - Type: Telepilot node  
  - Role: Fetches the recent message history from the provided chat ID  
  - Connections: Outputs to “Split Out”  
  - Edge Cases: API rate limits, empty chat history, or chat access restrictions.

- **Split Out**  
  - Type: Split Out node  
  - Role: Splits the array of messages into single message items for individual processing  
  - Connections: Outputs to “Get message text”  
  - Edge Cases: Empty input array results in no downstream processing.

---

#### 1.3 Message Filtering and Preparation

**Overview:**  
Processes individual messages: extracts text, filters by recency (last 2 hours), removes empty messages, and aggregates the filtered messages for AI input.

**Nodes Involved:**  
- Get message text  
- Filter Last 2 hours  
- Skip Empty Message  
- Aggregate

**Node Details:**

- **Get message text**  
  - Type: Set node  
  - Role: Extracts the actual text content from each message object  
  - Configuration: Likely sets a field with the message text for filtering and aggregation  
  - Connections: Outputs to “Filter Last 2 hours”  
  - Edge Cases: Messages without text (e.g., media only) may cause empty string extraction.

- **Filter Last 2 hours**  
  - Type: Filter node  
  - Role: Allows only messages sent in the last 2 hours to continue  
  - Configuration: Uses a timestamp comparison expression against current time minus 2 hours  
  - Connections: Outputs to “Skip Empty Message”  
  - Edge Cases: Timezone inconsistencies, messages without timestamps.

- **Skip Empty Message**  
  - Type: Filter node  
  - Role: Filters out any messages with empty or missing text content  
  - Connections: Outputs to “Aggregate”  
  - Edge Cases: Messages with whitespace-only text may still pass if not trimmed.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Collects filtered messages back into an array for AI processing  
  - Connections: Outputs to “AI Agent”  
  - Edge Cases: No messages after filtering leads to empty aggregation, possibly causing AI input failure.

---

#### 1.4 AI Processing

**Overview:**  
Generates the summary using Langchain’s AI Agent with OpenRouter language model and MongoDB-based chat memory for context.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- MongoDB Chat Memory

**Node Details:**

- **AI Agent**  
  - Type: Langchain Agent node  
  - Role: Orchestrates AI summarization logic using the language model and memory  
  - Connections: Inputs from “Aggregate” (messages), AI memory from “MongoDB Chat Memory”, language model from “OpenRouter Chat Model”  
  - Outputs to “Send Summary To Chat”  
  - Edge Cases: AI errors, invalid or empty input, memory query failures.

- **OpenRouter Chat Model**  
  - Type: Langchain OpenRouter Chat Model node  
  - Role: Provides the language model API interface (e.g., OpenAI-compatible)  
  - Connections: Connected as language model input to “AI Agent”  
  - Configuration: Requires OpenRouter API credentials  
  - Edge Cases: API key/authentication errors, rate limits, timeouts.

- **MongoDB Chat Memory**  
  - Type: Langchain MongoDB memory node  
  - Role: Stores and retrieves conversational context to enhance AI responses  
  - Connections: Connected as memory input to “AI Agent”  
  - Configuration: Requires MongoDB credentials and connection details  
  - Edge Cases: Database connectivity issues, query failures.

---

#### 1.5 Output Sending

**Overview:**  
Sends the AI-generated chat summary back to the Telegram chat.

**Nodes Involved:**  
- Send Summary To Chat

**Node Details:**

- **Send Summary To Chat**  
  - Type: Telepilot node  
  - Role: Posts the summarized text message into the Telegram chat  
  - Connections: Input from “AI Agent”  
  - Configuration: Requires Telegram credentials and chat ID  
  - Edge Cases: Message send failure, chat permissions, API rate limits.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                       | Input Node(s)                   | Output Node(s)              | Sticky Note                         |
|---------------------------|-----------------------------------------|------------------------------------|--------------------------------|-----------------------------|-----------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                         | Manual workflow start trigger       |                                | Get Chat Id By Name          |                                   |
| Schedule Trigger          | Schedule Trigger                         | Scheduled workflow start trigger    |                                | Get Chat Id By Name          |                                   |
| When chat message received| Langchain Chat Trigger                   | Trigger on Telegram chat message    |                                | Telegram Login               |                                   |
| Telegram Login            | Telepilot (Telegram API)                 | Authenticate Telegram API session   | When chat message received      | Try this for check Login     |                                   |
| Try this for check Login  | Telepilot (Telegram API)                 | Test Telegram login/session         | Telegram Login                 |                             |                                   |
| Get Chat Id By Name       | Telepilot (Telegram API)                 | Get chat ID from chat name          | When clicking ‘Execute workflow’, Schedule Trigger | Last Messages History         |                                   |
| Last Messages History     | Telepilot (Telegram API)                 | Fetch recent messages from chat     | Get Chat Id By Name            | Split Out                   |                                   |
| Split Out                 | Split Out                               | Split message array into single items | Last Messages History          | Get message text            |                                   |
| Get message text          | Set                                    | Extract message text content        | Split Out                     | Filter Last 2 hours         |                                   |
| Filter Last 2 hours       | Filter                                 | Filter messages from last 2 hours  | Get message text              | Skip Empty Message          |                                   |
| Skip Empty Message        | Filter                                 | Remove empty messages               | Filter Last 2 hours           | Aggregate                  |                                   |
| Aggregate                | Aggregate                              | Collect filtered messages for AI    | Skip Empty Message            | AI Agent                   |                                   |
| AI Agent                 | Langchain Agent                        | Generate chat summary using AI      | Aggregate, MongoDB Chat Memory, OpenRouter Chat Model | Send Summary To Chat         |                                   |
| OpenRouter Chat Model    | Langchain OpenRouter Language Model    | Provide AI language model           |                              | AI Agent (as AI language model) |                                   |
| MongoDB Chat Memory       | Langchain MongoDB Memory                | Store/retrieve chat memory context  |                              | AI Agent (as AI memory)     |                                   |
| Send Summary To Chat      | Telepilot (Telegram API)                 | Send AI summary to Telegram chat    | AI Agent                     |                             |                                   |
| Sticky Note3              | Sticky Note                            | (Empty content)                     |                              |                             |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Allows manual execution  
   - Connect its output to “Get Chat Id By Name”.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - (Optionally configure schedule parameters if desired)  
   - Connect its output to “Get Chat Id By Name”.

3. **Create Telegram Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook URL and set up Telegram webhook accordingly  
   - Connect output to “Telegram Login”.

4. **Create Telegram Login Node**  
   - Type: Telepilot  
   - Configure with Telegram OAuth2 or login credentials  
   - Connect output to “Try this for check Login”.

5. **Create Test Login Node**  
   - Type: Telepilot  
   - Purpose: to verify login/session (optional)  
   - No outputs needed.

6. **Create Get Chat Id By Name Node**  
   - Type: Telepilot  
   - Configure to input Telegram chat name or username  
   - Connect outputs from Manual Trigger and Schedule Trigger to this node  
   - Connect output to “Last Messages History”.

7. **Create Last Messages History Node**  
   - Type: Telepilot  
   - Configure to fetch messages from chat ID given by “Get Chat Id By Name”  
   - Connect output to “Split Out”.

8. **Create Split Out Node**  
   - Type: Split Out  
   - No special configuration  
   - Connect output to “Get message text”.

9. **Create Get message text Node**  
   - Type: Set  
   - Configure to extract the message text field (e.g., from Telegram message object)  
   - Connect output to “Filter Last 2 hours”.

10. **Create Filter Last 2 hours Node**  
    - Type: Filter  
    - Configure condition to keep messages with timestamp within the last 2 hours, e.g., using an expression comparing message timestamp to current time minus 2 hours  
    - Connect output to “Skip Empty Message”.

11. **Create Skip Empty Message Node**  
    - Type: Filter  
    - Condition to exclude messages with empty or null text fields (trim whitespace if needed)  
    - Connect output to “Aggregate”.

12. **Create Aggregate Node**  
    - Type: Aggregate  
    - Configure to collect all filtered message texts into an array or combined string suitable for AI input  
    - Connect output to “AI Agent”.

13. **Create OpenRouter Chat Model Node**  
    - Type: Langchain OpenRouter Chat Model  
    - Configure with OpenRouter API credentials  
    - Connect output to AI Agent as language model input.

14. **Create MongoDB Chat Memory Node**  
    - Type: Langchain MongoDB Memory  
    - Configure MongoDB connection credentials and collection details for chat memory  
    - Connect output to AI Agent as AI memory input.

15. **Create AI Agent Node**  
    - Type: Langchain Agent  
    - Configure to accept aggregated messages as input  
    - Connect language model input from “OpenRouter Chat Model” and memory input from “MongoDB Chat Memory”  
    - Connect output to “Send Summary To Chat”.

16. **Create Send Summary To Chat Node**  
    - Type: Telepilot  
    - Configure to send messages to the Telegram chat using chat ID  
    - Connect input from “AI Agent”.

17. **Connect all nodes as per above connections**, ensuring data flows from triggers through to output.

18. **Credential Setup**  
    - Configure Telepilot nodes with valid Telegram credentials (OAuth2 or API keys).  
    - Configure OpenRouter Chat Model node with valid OpenRouter API key.  
    - Configure MongoDB Chat Memory node with valid MongoDB credentials and accessible database.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Telepilot nodes used to interact with Telegram API, providing enhanced Telegram integration.    | https://github.com/telepilotco/n8n-nodes-telepilot                                                    |
| Langchain nodes enable AI processing with memory and language models, facilitating complex AI workflows. | https://docs.n8n.io/integrations/builtin/nodes/langchain/                                             |
| MongoDB memory node stores conversational context, improving AI response relevance over time.   | Requires accessible MongoDB instance                                                                  |
| OpenRouter Chat Model node requires API credentials; compatible with OpenAI API and alternatives.| https://openrouter.ai                                                                                   |
| The workflow uses webhook ID "b0697338-a2a6-4f38-b78b-970e420b6947" for Telegram chat trigger; ensure webhook is properly registered with Telegram Bot API. | Telegram Bot API documentation: https://core.telegram.org/bots/api                                     |

---

This analysis and reconstruction document provides a complete and detailed understanding of the “Telegram Chat Summarizer with AI” workflow, its components, configuration, and connections, enabling reproduction or modification by developers or AI agents.