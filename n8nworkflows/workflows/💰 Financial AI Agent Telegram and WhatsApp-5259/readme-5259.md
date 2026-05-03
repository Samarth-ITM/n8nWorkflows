💰 Financial AI Agent Telegram and WhatsApp

https://n8nworkflows.xyz/workflows/---financial-ai-agent-telegram-and-whatsapp-5259


# 💰 Financial AI Agent Telegram and WhatsApp

### 1. Workflow Overview

This workflow, titled **"Financial AI Agent Telegram and WhatsApp"**, is designed to serve as an intelligent financial assistant that interacts with users via Telegram (WhatsApp integration is implied by the title but not explicitly configured). It leverages AI capabilities to understand user queries related to financial transactions, balance reports, and other related functions by integrating with OpenAI's language models and leveraging PostgreSQL for chat memory persistence. The workflow is structured into several logical blocks that handle input reception, message normalization, AI processing with memory and tools, and output delivery.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming messages from Telegram and a dedicated chat trigger.
- **1.2 Message Normalization:** Prepares and standardizes the incoming message data for AI processing.
- **1.3 AI Processing:** Uses LangChain AI Agent with OpenAI Chat Model and PostgreSQL memory for conversational context.
- **1.4 Tool Invocation:** Executes specific workflows as AI tools for registering transactions and generating balance reports.
- **1.5 Output Delivery:** Sends the AI-generated responses back to Telegram users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for incoming user messages from Telegram and a generic chat trigger webhook, initiating the workflow process.
- **Nodes Involved:** `Telegram Trigger`, `Chat`

**Node Details:**

- **Telegram Trigger**
  - Type: Telegram Trigger node
  - Role: Listens for new messages sent to the Telegram bot.
  - Configuration: Uses webhook with ID `c9065337-71a9-4dd3-9a5a-23978ae4a8df`.
  - Inputs: External Telegram messages.
  - Outputs: Passes data to `Normalize` node.
  - Edge Cases: Telegram API rate limits, webhook downtime, malformed messages.
  - Credentials: Requires Telegram Bot OAuth2 credentials.

- **Chat**
  - Type: LangChain Chat Trigger
  - Role: Alternative chat trigger (likely for other chat platforms or manual triggers).
  - Configuration: Webhook with ID `30dc5ca2-b504-4fff-81a9-3675abbc9fd9`.
  - Inputs: External chat messages.
  - Outputs: Currently no connected downstream node; might represent an inactive or placeholder entry point.
  - Edge Cases: Webhook unavailability, incomplete message data.

#### 2.2 Message Normalization

- **Overview:** Standardizes the incoming message format to ensure consistent data structure for AI processing.
- **Nodes Involved:** `Normalize`

**Node Details:**

- **Normalize**
  - Type: Set Node
  - Role: Prepares and normalizes incoming Telegram message data.
  - Configuration: Likely sets or renames fields to match AI Agent expectations.
  - Inputs: From `Telegram Trigger`.
  - Outputs: To `AI Agent`.
  - Expressions: Could include extraction of message text, user ID, chat ID.
  - Edge Cases: Missing or malformed message content, unexpected payload changes in Telegram API.

#### 2.3 AI Processing

- **Overview:** Core AI logic combining a LangChain AI Agent with OpenAI language model and PostgreSQL database to maintain conversational memory. It also integrates tool workflows for extended functionalities.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`, `Postgres Chat Memory`, `Register Transaction`, `Balance report`

**Node Details:**

- **AI Agent**
  - Type: LangChain Agent Node
  - Role: Orchestrates AI-driven responses, decides when to invoke tools.
  - Configuration: References auxiliary nodes for language model (`OpenAI Chat Model`), memory (`Postgres Chat Memory`), and tools (`Register Transaction`, `Balance report`).
  - Inputs: From `Normalize` (normalized user input).
  - Outputs: Sends AI-generated responses to `Telegram`.
  - Expression Use: Dynamically selects tools based on user intent.
  - Edge Cases: AI service downtime, unexpected user queries, tool invocation failures.
  - Version: 1.9.

- **OpenAI Chat Model**
  - Type: LangChain OpenAI Chat Model Node
  - Role: Provides natural language understanding and generation.
  - Configuration: Uses OpenAI API with proper credentials.
  - Inputs: Connected to AI Agent as language model.
  - Outputs: AI-generated text responses.
  - Edge Cases: API rate limits, authentication errors, input size limits.
  - Version: 1.2.

- **Postgres Chat Memory**
  - Type: LangChain PostgreSQL Chat Memory Node
  - Role: Stores conversation history in PostgreSQL for context retention.
  - Configuration: Requires PostgreSQL credentials, database, and table setup.
  - Inputs: Connected to AI Agent as memory store.
  - Outputs: Provides conversational context to AI Agent.
  - Edge Cases: Database connectivity issues, query errors, data corruption.
  - Version: 1.3.

- **Register Transaction**
  - Type: LangChain Tool Workflow Node
  - Role: Sub-workflow to register financial transactions as requested by users.
  - Configuration: Invoked as an AI tool by AI Agent.
  - Inputs: From AI Agent tool invocation.
  - Outputs: Returns operation status or confirmation to AI Agent.
  - Edge Cases: Sub-workflow errors, database write failures, validation errors.
  - Version: 2.2.

- **Balance report**
  - Type: LangChain Tool Workflow Node
  - Role: Sub-workflow to generate and provide financial balance reports.
  - Configuration: Invoked as an AI tool by AI Agent.
  - Inputs: From AI Agent tool invocation.
  - Outputs: Provides balance information to AI Agent.
  - Edge Cases: Data retrieval errors, report generation failures.
  - Version: 2.2.

#### 2.4 Output Delivery

- **Overview:** Sends the AI-generated response messages back to Telegram users.
- **Nodes Involved:** `Telegram`

**Node Details:**

- **Telegram**
  - Type: Telegram node (message sender)
  - Role: Delivers AI responses back to Telegram users.
  - Configuration: Uses webhook ID `91b61951-671c-456c-a470-9cd41266f123`.
  - Inputs: Receives messages from `AI Agent`.
  - Outputs: None (end node).
  - Edge Cases: Telegram API errors, message size limits, user blocked bot.
  - Credentials: Telegram Bot OAuth2 credentials.
  - Version: 1.2.

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                  | Input Node(s)         | Output Node(s)       | Sticky Note                       |
|---------------------|------------------------------------|--------------------------------|-----------------------|----------------------|---------------------------------|
| Chat                | LangChain Chat Trigger              | Alternative chat message input | -                     | -                    |                                 |
| Telegram Trigger    | Telegram Trigger                    | Capture Telegram messages       | -                     | Normalize            |                                 |
| Normalize           | Set                                | Normalize incoming messages     | Telegram Trigger      | AI Agent             |                                 |
| AI Agent            | LangChain Agent                    | Core AI processing and tool orchestration | Normalize           | Telegram, Register Transaction, Balance report |                                 |
| OpenAI Chat Model   | LangChain OpenAI Chat Model        | Language model for AI Agent     | - (connected to AI Agent) | AI Agent             |                                 |
| Postgres Chat Memory| LangChain PostgreSQL Chat Memory   | Persist conversation context    | - (connected to AI Agent) | AI Agent             |                                 |
| Register Transaction| LangChain Tool Workflow            | Sub-workflow to register transactions | AI Agent (tool input) | AI Agent             |                                 |
| Balance report      | LangChain Tool Workflow            | Sub-workflow for balance reports | AI Agent (tool input) | AI Agent             |                                 |
| Telegram            | Telegram                           | Send AI responses to users      | AI Agent              | -                    |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Set node type to `Telegram Trigger`.
   - Configure with your Telegram Bot OAuth2 credentials.
   - Enable webhook to receive incoming messages from Telegram users.
   - Position as entry node.

2. **Create Chat Trigger Node (Optional)**
   - Add `LangChain Chat Trigger` node.
   - Configure webhook for alternative chat inputs.
   - (Note: No downstream connections by default.)

3. **Create Normalize Node**
   - Add a `Set` node named `Normalize`.
   - Configure to extract and standardize fields from Telegram Trigger:
     - Extract message text.
     - Extract chat ID and user ID.
   - Connect `Telegram Trigger` output to `Normalize` input.

4. **Create OpenAI Chat Model Node**
   - Add `LangChain OpenAI Chat Model` node.
   - Configure with OpenAI API credentials.
   - Set model parameters as needed (temperature, max tokens, etc.).
   - No direct input; this will be connected to AI Agent.

5. **Create PostgreSQL Chat Memory Node**
   - Add `LangChain PostgreSQL Chat Memory` node.
   - Configure PostgreSQL credentials, database, and table for conversation storage.
   - No direct input; connected to AI Agent.

6. **Create Register Transaction Tool Workflow Node**
   - Add `LangChain Tool Workflow` node named `Register Transaction`.
   - Link to or create a sub-workflow that processes transaction registrations.
   - Ensure sub-workflow accepts parameters like transaction details and returns status.
   - No direct input; connected as AI tool in AI Agent.

7. **Create Balance Report Tool Workflow Node**
   - Add `LangChain Tool Workflow` node named `Balance report`.
   - Link to or create a sub-workflow generating financial balance reports.
   - Ensure sub-workflow returns formatted balance data.
   - No direct input; connected as AI tool in AI Agent.

8. **Create AI Agent Node**
   - Add `LangChain Agent` node named `AI Agent`.
   - Configure to use:
     - OpenAI Chat Model node as language model.
     - PostgreSQL Chat Memory as memory backend.
     - Register Transaction and Balance Report nodes as AI tools.
   - Connect input from `Normalize` node.
   - Connect outputs to `Telegram` node and tools as appropriate.

9. **Create Telegram Node**
   - Add `Telegram` node to send messages.
   - Configure with Telegram Bot OAuth2 credentials.
   - Connect input from `AI Agent`.
   - This node sends responses back to users.

10. **Connect Nodes**
    - `Telegram Trigger` → `Normalize`
    - `Normalize` → `AI Agent`
    - `OpenAI Chat Model` → AI Agent (language model input)
    - `Postgres Chat Memory` → AI Agent (memory input)
    - `Register Transaction` → AI Agent (tool input)
    - `Balance report` → AI Agent (tool input)
    - `AI Agent` → `Telegram`

11. **Set Workflow Settings**
    - Enable execution timeout (e.g., 3600 seconds).
    - Set execution order to "v1".
    - Save data for all successful executions.
    - Ensure credentials for Telegram and OpenAI are correctly set and tested.

12. **Sub-workflow Setup**
    - For `Register Transaction` and `Balance report` nodes:
      - Create workflows accepting input parameters from AI Agent.
      - Implement database interactions or API calls as needed.
      - Return JSON with status or report data for AI Agent consumption.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow is designed specifically for Telegram integration; WhatsApp is not explicitly configured. | Workflow title references WhatsApp, but no WhatsApp nodes are present. |
| Ensure PostgreSQL database is properly configured with tables to store chat memory for conversation context. | Database setup is critical to maintain conversational history. |
| Use valid Telegram Bot credentials and ensure webhook URLs are properly registered with Telegram.       | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| OpenAI API credentials must have sufficient quota and access to chat models (e.g., GPT-4 or GPT-3.5).   | OpenAI API docs: https://platform.openai.com/docs/api-reference/chat |
| Consider implementing error handling sub-workflows or nodes to capture and log failures in tool workflows.| Important to prevent silent failures in transaction registration or report generation. |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.