Create a Slack AI Chatbot with Threads & Thinking UI using OpenRouter & Postgres

https://n8nworkflows.xyz/workflows/create-a-slack-ai-chatbot-with-threads---thinking-ui-using-openrouter---postgres-5749


# Create a Slack AI Chatbot with Threads & Thinking UI using OpenRouter & Postgres

---

### 1. Workflow Overview

This workflow implements a Slack AI chatbot that interacts with users through threaded conversations, providing contextual, AI-generated replies while displaying a "thinking" UI during processing. It leverages OpenRouter as the AI language model provider and stores chat history in a Postgres database for memory retention across threads.

**Target Use Cases:**  
- Slack bots that manage threaded conversations smoothly  
- AI assistants that maintain context over multiple message exchanges  
- Chatbots requiring visual feedback ("thinking" indicator) during response generation  

**Logical Blocks:**  
- **1.1 Input Reception:** Slack event trigger capturing incoming user messages in a specific channel  
- **1.2 Message Filtering:** Filtering out bot messages and non-message events to process only valid user messages  
- **1.3 Thinking UI Activation:** Sending an HTTP request to Slack API to display a "thinking..." status in the conversation thread  
- **1.4 AI Processing:** Passing the user message to a customized AI agent using OpenRouter language model and Postgres chat memory for context  
- **1.5 Response Delivery:** Sending the AI-generated reply back to Slack as a threaded message  

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  Listens for new messages in a specific Slack channel (typically a direct message or app-specific channel). Acts as the workflow’s entry point.

- **Nodes Involved:**  
  - On Message Received  
  - Sticky Note (instructional)

- **Node Details:**  

  - **On Message Received**  
    - Type: Slack Trigger  
    - Role: Listens for Slack `message` events in the configured channel  
    - Configuration:  
      - Trigger event: `message`  
      - Channel to watch: User must set the Slack app’s channel ID (`YOUR_APPS_CHANNEL_ID`)  
      - Resolving IDs: Disabled (raw IDs used)  
    - Inputs: None (Webhook trigger)  
    - Outputs: JSON with Slack message event data (including text, channel, thread timestamp, bot_id)  
    - Failure Modes: Incorrect channel ID, Slack API auth errors, webhook misconfiguration  
    - Sticky Note Content:  
      > "## Listen for a DM  
      > **IMPORTANT**: Enter your Slack app's ID in the \"Channel to Watch\" field."

---

#### 2.2 Message Filtering

- **Overview:**  
  Filters incoming messages to ensure only user-originated, textual messages of type `message` are processed. This removes bot messages or other event types that are not relevant.

- **Nodes Involved:**  
  - Check If User  
  - NoOp (No Operation fallback)  
  - Sticky Note (instructional)

- **Node Details:**  

  - **Check If User**  
    - Type: If Node  
    - Role: Checks conditions on incoming message JSON to validate it is a user message  
    - Configuration: Conditions combined with AND:  
      - `bot_id` does not exist (message is not from a bot)  
      - `text` exists (message contains text)  
      - `type` is exactly `"message"`  
    - Inputs: From "On Message Received"  
    - Outputs:  
      - True branch: Message passes filters  
      - False branch: Message ignored, routed to NoOp  
    - Edge Cases: Messages missing expected fields, false positives if bot_id is present but message is valid  
    - Sticky Note Content:  
      > "## Filter out noise  
      > We only care about user messages."

  - **NoOp**  
    - Type: No Operation  
    - Role: Ends processing for non-user or irrelevant messages  
    - Inputs: False branch from "Check If User"  
    - Outputs: None

---

#### 2.3 Thinking UI Activation

- **Overview:**  
  Sends an HTTP POST request to Slack’s API endpoint `assistant.threads.setStatus` to display a "is thinking..." indicator in the conversation thread. This visually informs users that the bot is processing their message.

- **Nodes Involved:**  
  - Set Thinking Status  
  - Sticky Note (instructional)

- **Node Details:**  

  - **Set Thinking Status**  
    - Type: HTTP Request  
    - Role: Calls Slack API to set thread status  
    - Configuration:  
      - Method: POST  
      - URL: `https://slack.com/api/assistant.threads.setStatus`  
      - Authentication: Bearer token via "Jina Bearer Token | Paper Jam" credentials  
      - Headers: Content-Type: application/json  
      - Body (JSON):  
        - `channel_id`: from incoming JSON `channel` field  
        - `thread_ts`: from incoming JSON `thread_ts` field  
        - `status`: fixed string "is thinking..."  
    - Inputs: True branch from "Check If User"  
    - Outputs: Passes data forward to AI Agent  
    - Failure Modes: Token expiry, Slack API downtime, invalid thread_ts or channel_id  
    - Sticky Note Content:  
      > "## Activate loading UI  
      > This HTTP POST request will trigger the \"three dots\" thinking UI in the message thread. Slack automatically removes this UI when our app responds."

---

#### 2.4 AI Processing

- **Overview:**  
  The core AI logic that generates replies. This block consists of an AI Agent powered by a Langchain OpenRouter chat model and remembers conversation history stored in Postgres.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  
  - Postgres Chat Memory  
  - Sticky Note (instructional)

- **Node Details:**  

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Orchestrates AI prompt handling, memory, and model calls  
    - Configuration:  
      - Input text: dynamically from incoming JSON `text`  
      - System message prompt: "You are a helpful, friendly, assistant. You always respond only nicely formatted markdown where appropriate."  
      - Prompt type: define (custom prompt)  
    - Inputs:  
      - Main input from "Set Thinking Status"  
      - AI language model input from "OpenRouter Chat Model"  
      - AI memory input from "Postgres Chat Memory"  
    - Outputs: AI-generated response JSON forwarded to "Send Reply"  
    - Edge Cases: Model API errors, memory retrieval failures, malformed input text  
    - Sticky Note Content:  
      > "## Customize your agent here  
      > The agent is the brains of your chatbot. Customize the prompts and tooling however you need."

  - **OpenRouter Chat Model**  
    - Type: Langchain OpenRouter Chat Model Node  
    - Role: Provides the AI language model interface for the agent  
    - Configuration: Uses OpenRouter API credentials ("OpenRouter | Paper Jam")  
    - Inputs: Connected to AI Agent node as the language model source  
    - Outputs: Chat completions to AI Agent  
    - Edge Cases: API quota limits, authentication errors

  - **Postgres Chat Memory**  
    - Type: Langchain Postgres Chat Memory Node  
    - Role: Stores and retrieves chat history for context awareness  
    - Configuration:  
      - Postgres credentials ("Postgres | Paper Jam n8n")  
      - Table name: "chat_histories"  
      - Session key: dynamically set via Slack thread timestamp (`thread_ts`) to keep context per thread  
      - Session ID type: custom key  
    - Inputs: Stores and fetches memory for AI Agent  
    - Outputs: Feeds memory into AI Agent  
    - Edge Cases: Database connection failures, missing session keys, data corruption

---

#### 2.5 Response Delivery

- **Overview:**  
  Sends the AI-generated reply back to Slack as a threaded message, preserving conversation context and formatting.

- **Nodes Involved:**  
  - Send Reply

- **Node Details:**  

  - **Send Reply**  
    - Type: Slack Node (Message)  
    - Role: Posts the AI output as a Slack message reply in the thread  
    - Configuration:  
      - Text: from AI Agent output field `output`  
      - Channel: dynamically from original message’s channel ID  
      - Message type: block (Slack Block Kit used for formatting)  
      - Blocks UI: Markdown block containing AI output text serialized as JSON string  
      - Thread timestamp: references original message `ts` to reply in thread  
      - Other options: markdown enabled, no workflow link included  
      - Credentials: Slack API ("Slack | Paper Jam")  
    - Inputs: From AI Agent node  
    - Outputs: None (end of response chain)  
    - Edge Cases: Slack API rate limiting, invalid thread timestamp, message formatting errors

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                                  | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                   |
|----------------------|----------------------------------|-------------------------------------------------|-----------------------|----------------------|---------------------------------------------------------------------------------------------------------------|
| On Message Received   | Slack Trigger                    | Entry trigger capturing Slack messages          | None                  | Check If User         | ## Listen for a DM  **IMPORTANT**: Enter your Slack app's ID in the "Channel to Watch" field.                 |
| Check If User         | If Node                         | Filters only user messages                        | On Message Received    | Set Thinking Status, NoOp | ## Filter out noise  We only care about user messages.                                                       |
| NoOp                 | No Operation                    | Ends processing for irrelevant messages          | Check If User (False)  | None                 |                                                                                                               |
| Set Thinking Status   | HTTP Request                   | Displays "thinking..." UI in Slack thread         | Check If User (True)   | AI Agent              | ## Activate loading UI  This HTTP POST request will trigger the "three dots" thinking UI in the message thread. Slack automatically removes this UI when our app responds. |
| AI Agent             | Langchain Agent                 | Processes message with AI using prompt and memory| Set Thinking Status, OpenRouter Chat Model, Postgres Chat Memory | Send Reply            | ## Customize your agent here  The agent is the brains of your chatbot. Customize the prompts and tooling however you need. |
| OpenRouter Chat Model | Langchain OpenRouter Language Model | Provides OpenRouter AI completions               | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                                               |
| Postgres Chat Memory  | Langchain Postgres Chat Memory  | Fetches and stores chat history for context      | AI Agent (ai_memory)   | AI Agent              |                                                                                                               |
| Send Reply           | Slack Node (Message)             | Sends AI reply message back to Slack thread      | AI Agent              | None                  |                                                                                                               |
| Sticky Note          | Sticky Note                    | Instructional comments                            | None                  | None                  | See respective node rows                                                                                      |
| Sticky Note1         | Sticky Note                    | Instructional comments                            | None                  | None                  |                                                                                                               |
| Sticky Note2         | Sticky Note                    | Instructional comments                            | None                  | None                  |                                                                                                               |
| Sticky Note3         | Sticky Note                    | Instructional comments                            | None                  | None                  |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create the Slack Trigger node  
- Type: Slack Trigger  
- Configure to listen on event type `message`  
- Set the channel ID to your Slack app’s channel (replace `YOUR_APPS_CHANNEL_ID`)  
- Set "Resolve IDs" to false  
- Connect this node as the workflow’s starting point

**Step 2:** Add an If node named "Check If User"  
- Add three conditions combined with AND:  
  - `bot_id` does not exist (string operation: notExists)  
  - `text` exists (string operation: exists)  
  - `type` equals `message` (string equals)  
- Input: connect from Slack Trigger  
- True output: continue processing  
- False output: route to NoOp

**Step 3:** Add a No Operation node named "NoOp"  
- No configuration needed  
- Connect from the false branch of "Check If User"  
- Ends processing for ignored messages

**Step 4:** Add an HTTP Request node named "Set Thinking Status"  
- Method: POST  
- URL: `https://slack.com/api/assistant.threads.setStatus`  
- Authentication: HTTP Bearer Auth (configure with your Slack Bearer token)  
- Headers: Content-Type: application/json  
- Body parameters (JSON):  
  - `channel_id`: from incoming JSON `channel` field  
  - `status`: fixed string `"is thinking..."`  
  - `thread_ts`: from incoming JSON `thread_ts` field  
- Connect from the true output of "Check If User"

**Step 5:** Add Langchain nodes for AI processing  
- Add "OpenRouter Chat Model" node  
  - Configure with OpenRouter API credentials  
- Add "Postgres Chat Memory" node  
  - Configure with Postgres credentials  
  - Set table name to `chat_histories`  
  - Session key expression: `{{$('On Message Received').item.json.thread_ts}}` (or from the incoming message thread timestamp)  
  - Session ID type: customKey  
- Add "AI Agent" node  
  - Text input: `{{$json.text}}` (the user message)  
  - System message: "You are a helpful, friendly, assistant. You always respond only nicely formatted markdown where appropriate."  
  - Prompt type: define  
  - Connect `ai_languageModel` input to "OpenRouter Chat Model"  
  - Connect `ai_memory` input to "Postgres Chat Memory"  
  - Connect main input from "Set Thinking Status" node  
- Connect the output of AI Agent to the next node for sending reply

**Step 6:** Add Slack node "Send Reply"  
- Type: Slack Message node  
- Text: `{{$json.output}}` (the AI agent’s output)  
- Channel: from original Slack message channel (`{{$('On Message Received').item.json.channel}}`)  
- Message type: block  
- Blocks UI: a markdown block containing the AI output serialized as JSON string  
- Set threaded reply timestamp to original message `ts` to reply in thread  
- Enable markdown  
- Connect input from AI Agent output

**Step 7:** Set credentials in n8n for:  
- Slack API (with permissions to read messages and post replies)  
- OpenRouter API (OpenRouter API key)  
- Postgres database connection for chat memory  
- HTTP Bearer Auth for Slack thread status API

**Step 8 (Optional):** Add sticky notes for documentation and guidance as per original positions

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The Slack app’s channel ID must be set manually in the Slack Trigger node to ensure the workflow listens to the correct channel. | Critical configuration step to avoid missing events                                                |
| The Slack API endpoint `assistant.threads.setStatus` is used to trigger the "thinking..." UI in threads. This is a Slack Labs API and may require special permissions. | Slack Threads API documentation (may require Slack Labs access)                                   |
| The AI Agent node uses Langchain’s OpenRouter model integration, allowing customizable prompt templates and memory tools. | n8n Langchain AI nodes documentation                                                               |
| Postgres is used to keep chat histories keyed by Slack thread timestamp, enabling contextual AI responses in threads.      | Requires a properly configured Postgres database and table `chat_histories`                         |
| Slack message replies use Block Kit with markdown to format AI responses nicely within threads.                            | Slack Block Kit Builder: https://app.slack.com/block-kit-builder                                   |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---