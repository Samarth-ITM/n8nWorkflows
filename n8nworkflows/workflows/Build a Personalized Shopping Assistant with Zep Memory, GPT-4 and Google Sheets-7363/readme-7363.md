Build a Personalized Shopping Assistant with Zep Memory, GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/build-a-personalized-shopping-assistant-with-zep-memory--gpt-4-and-google-sheets-7363


# Build a Personalized Shopping Assistant with Zep Memory, GPT-4 and Google Sheets

### 1. Workflow Overview

This workflow implements a **Personalized Shopping Assistant** for Infystore, designed to interact with customers via chat, providing real-time and context-aware answers about product inventory, order tracking, and return policies. It leverages Zep Memory for conversation context, GPT-4 via OpenAI for natural language understanding and generation, and Google Sheets as a backend database for inventory, orders, and policy data.

**Use Cases:**
- Checking product availability and details
- Tracking order status by phone number
- Providing return policy information
- Guiding customers through return processes

**Logical Blocks:**

- **1.1 Input Reception:** Captures chat messages from users via a chat trigger.
- **1.2 AI Processing and Memory:** Processes user input with GPT-4 and LangChain Agent, manages conversation memory with Zep.
- **1.3 Data Lookup:** Queries Google Sheets for inventory, order, and return policy data as requested.
- **1.4 Response Generation:** Uses the AI Agent node to generate context-aware, concise, polite responses based on retrieved data and conversation history.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming user chat messages through a webhook and passes them to the AI processing block.
- **Nodes Involved:** 
  - When chat message received
- **Node Details:**

| Node Name             | Details |
|-----------------------|---------|
| When chat message received | - Type: LangChain Chat Trigger<br>- Role: Entry point, webhook listener for incoming chat messages.<br>- Configuration: Default webhook with no additional parameters.<br>- Input: External chat client message.<br>- Output: Passes user message payload (in `$json.chatInput`) to next node.<br>- Edge cases: Webhook failures, message format errors, concurrency issues.<br>- Version: 1.1 |

#### 1.2 AI Processing and Memory

- **Overview:** Uses GPT-4 model with LangChain Agent node, enriched by conversation memory stored/retrieved via Zep Memory node, to understand and generate responses to user queries.
- **Nodes Involved:** 
  - AI Agent
  - OpenAI Chat Model
  - Zep
- **Node Details:**

| Node Name      | Details |
|----------------|---------|
| AI Agent       | - Type: LangChain Agent Node<br>- Role: Core conversational AI logic processing user input with system prompts and chaining retrieval tools.<br>- Configuration: Uses system messages defining assistant behavior and dialogue flows for inventory, orders, returns, and general rules.<br>- Key expressions: Uses `{{$json.chatInput}}` to feed user text.<br>- Inputs: Receives user message from "When chat message received".<br>- Outputs: Sends response downstream.<br>- Integrations: Calls Google Sheets nodes as AI tools, uses Zep as AI memory.<br>- Edge cases: Misinterpretation of user input, incomplete extraction of entities, memory sync issues.<br>- Version: 2<br>- Sub-workflow: None |
| OpenAI Chat Model | - Type: LM Chat OpenAI<br>- Role: Language model provider (GPT-4.1-mini).<br>- Configuration: Model set to "gpt-4.1-mini", linked with OpenAI credentials.<br>- Input: Text from AI Agent.<br>- Output: Language generation responses.<br>- Edge cases: API rate limits, authentication errors, latency.<br>- Version: 1.2 |
| Zep            | - Type: LangChain Memory Zep<br>- Role: Persistent conversation memory storage and retrieval.<br>- Configuration: Uses Zep API credentials.<br>- Input: Feeds conversation history to AI Agent.<br>- Output: Updated memory state.<br>- Edge cases: API connectivity, data consistency.<br>- Version: 1.3 |

#### 1.3 Data Lookup

- **Overview:** Queries Google Sheets for dynamic data regarding inventory, orders, and return policies as required by the conversation context.
- **Nodes Involved:** 
  - Get_Inventory
  - Get_Orders
  - Get_ReturnPolicy
- **Node Details:**

| Node Name      | Details |
|----------------|---------|
| Get_Inventory  | - Type: Google Sheets Tool<br>- Role: Retrieve product information (stock level, restock ETA, description) from "Product Details" sheet.<br>- Configuration: Reads from Sheet with gid=0 in specified Google Sheet document.<br>- Credentials: Google Sheets OAuth2.<br>- Input: Triggered by AI Agent to fetch product details.<br>- Output: Returns product availability data.<br>- Edge cases: Sheet access permissions, empty or missing data, query errors.<br>- Version: 4.6 |
| Get_Orders     | - Type: Google Sheets Tool<br>- Role: Fetch order details using phone number from "Order Tracking" sheet.<br>- Configuration: Reads from Sheet with gid=2103540895.<br>- Credentials: Google Sheets OAuth2.<br>- Input: Triggered by AI Agent to fetch user orders.<br>- Output: Order status, delivery ETA.<br>- Edge cases: Phone number mismatches, no orders found, data format issues.<br>- Version: 4.6 |
| Get_ReturnPolicy | - Type: Google Sheets Tool<br>- Role: Retrieve return policy details.<br>- Configuration: Reads from Sheet with gid=1762722848.<br>- Credentials: Google Sheets OAuth2.<br>- Input: Triggered by AI Agent upon return policy inquiries.<br>- Output: Return window days, conditions.<br>- Edge cases: Missing policy data, sheet access failures.<br>- Version: 4.6 |

#### 1.4 Response Generation

- **Overview:** The AI Agent node composes replies based on the structured system instructions, data fetched from sheets, and conversation memory, ensuring concise, polite, and contextually relevant answers.
- **Nodes Involved:** 
  - AI Agent (same as in 1.2)
- **Node Details:** Already covered in 1.2

**Sticky Note:**

- Attached near Google Sheets nodes, contains a link to the sample Google Sheet used for data storage:  
  https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                            | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                     |
|--------------------------|--------------------------------|--------------------------------------------|----------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger          | Entry point, receives chat messages        | (External webhook)          | AI Agent                  |                                                                                                |
| AI Agent                 | LangChain Agent Node            | Processes input, manages dialogue & logic  | When chat message received  | Zep, Get_Orders, Get_Inventory, Get_ReturnPolicy |                                                                                                |
| OpenAI Chat Model        | LM Chat OpenAI                  | Provides GPT-4 language model backend       | AI Agent                   | AI Agent                  |                                                                                                |
| Zep                      | LangChain Memory Zep            | Stores and retrieves conversation memory   | AI Agent                   | AI Agent                  |                                                                                                |
| Get_Orders               | Google Sheets Tool              | Fetches order details from Google Sheets   | AI Agent                   | AI Agent                  | Sample Google Sheet link: https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing |
| Get_Inventory            | Google Sheets Tool              | Fetches product inventory details           | AI Agent                   | AI Agent                  | Sample Google Sheet link: https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing |
| Get_ReturnPolicy         | Google Sheets Tool              | Retrieves return policy information          | AI Agent                   | AI Agent                  | Sample Google Sheet link: https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing |
| Sticky Note              | Sticky Note                    | Provides sample Google Sheets link           |                            |                           | https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "When chat message received" node:**
   - Type: LangChain Chat Trigger
   - Purpose: Receive incoming chat messages via webhook.
   - Configuration: Default settings; no extra parameters.
   - Position: Leftmost, start of workflow.

2. **Create the "AI Agent" node:**
   - Type: LangChain Agent
   - Connect input to "When chat message received" node main output.
   - Parameter “text”: Set to `={{ $json.chatInput }}`
   - System message: Insert comprehensive instructions defining assistant role, dialogue flows for inventory, orders, returns, and general rules as per overview.
   - Set prompt type to "define".
   - Connect AI Agent outputs to Zep and the three Google Sheets nodes as AI tools.

3. **Set up the "OpenAI Chat Model" node:**
   - Type: LM Chat OpenAI
   - Model: Select "gpt-4.1-mini".
   - Credentials: Configure with valid OpenAI API credentials.
   - Connect output to AI Agent’s language model input.

4. **Create the "Zep" memory node:**
   - Type: LangChain Memory Zep
   - Credentials: Configure with valid Zep API credentials.
   - Connect input and output to AI Agent’s memory interface.

5. **Create three Google Sheets nodes for data retrieval:**

   a. **"Get_Inventory":**
   - Type: Google Sheets Tool
   - Document ID: Use the Google Sheet ID `17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo`
   - Sheet Name: Use gid `0` (Product Details sheet)
   - Credentials: Configure with Google Sheets OAuth2 credentials.
   - Connect input from AI Agent’s AI tool output.

   b. **"Get_Orders":**
   - Type: Google Sheets Tool
   - Document ID: Same as above.
   - Sheet Name: Use gid `2103540895` (Order Tracking sheet)
   - Credentials: Same Google OAuth2.
   - Connect input from AI Agent’s AI tool output.

   c. **"Get_ReturnPolicy":**
   - Type: Google Sheets Tool
   - Document ID: Same as above.
   - Sheet Name: Use gid `1762722848` (Return Policy sheet)
   - Credentials: Same Google OAuth2.
   - Connect input from AI Agent’s AI tool output.

6. **Arrange connections:**
   - "When chat message received" → "AI Agent"
   - "AI Agent" → "Zep" (ai_memory)
   - "AI Agent" → "Get_Orders" (ai_tool)
   - "AI Agent" → "Get_Inventory" (ai_tool)
   - "AI Agent" → "Get_ReturnPolicy" (ai_tool)
   - "OpenAI Chat Model" → "AI Agent" (ai_languageModel)

7. **Credentials setup:**
   - Obtain and configure:
     - OpenAI API credentials (for GPT-4)
     - Zep API credentials (for memory)
     - Google Sheets OAuth2 credentials with access to the specified Google Sheet

8. **System message configuration:**
   - Copy the detailed system prompt provided in AI Agent node parameters, describing assistant behavior, dialogue flows for inventory, order tracking, return policies, and general response rules.

9. **Testing and validation:**
   - Send test chat messages via webhook.
   - Verify responses align with inventory data, order tracking info, and return policies.
   - Validate conversation memory persistence across multiple messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Sample Google Sheet URL containing inventory, order tracking, and return policy sheets:                           | https://docs.google.com/spreadsheets/d/17PsTWr5shCgA2RnHuj1xwqSRX7uBcb-psRYZA2jFOTo/edit?usp=sharing                          |
| System prompt includes strict rules: no markdown, no images, no repeated questions, concise and polite replies. | Embedded in AI Agent node’s systemMessage parameter.                                                                            |
| Workflow uses LangChain Agent pattern integrating multiple AI tools and memory node for contextual assistance.  | n8n LangChain nodes documentation: https://docs.n8n.io/integrations/builtin/core-nodes/langchain-agent/                        |

---

**Disclaimer:** The provided text and documentation exclusively originate from an automated workflow created with n8n, adhering strictly to content policies, handling only legal and public data.