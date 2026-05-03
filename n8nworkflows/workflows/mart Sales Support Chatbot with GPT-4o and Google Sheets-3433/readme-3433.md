mart Sales Support Chatbot with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/mart-sales-support-chatbot-with-gpt-4o-and-google-sheets-3433


# mart Sales Support Chatbot with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow implements a **customer and sales support chatbot** tailored for a webshop selling Apple phone cases. It is designed primarily for solopreneurs who want to automate customer interactions without complex or costly support systems. The chatbot handles user queries such as checking product availability, placing orders, and updating stock, all managed through a single Google Sheet that tracks inventory and orders.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users.
- **1.2 AI Processing & Memory:** Uses OpenAI GPT-4 to process messages and maintain conversation context.
- **1.3 Support Agent Logic:** Implements the chatbot’s decision-making and tool-calling logic, including stock queries, order placement, and stock updates.
- **1.4 Google Sheets Integration:** Reads stock data, appends new orders, and updates inventory in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from customers and initiates the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* Chat trigger node from LangChain, acts as webhook entry point for chat messages.  
    - *Configuration:*  
      - Public webhook enabled for external access.  
      - Response mode set to "lastNode" to send the final node’s output back to the user.  
      - Loads previous session memory from "memory" to maintain context.  
      - Initial greeting message: "Hi! I’m Babish from Apple Case. How can I help?”  
    - *Input/Output:*  
      - Input: External chat message via webhook.  
      - Output: Passes message data to the Support Agent node.  
    - *Edge Cases:*  
      - Webhook downtime or network issues may cause message loss.  
      - Session memory loading failure could cause loss of conversation context.  
    - *Version:* 1.1

---

#### 2.2 AI Processing & Memory

- **Overview:**  
  This block processes user input using OpenAI GPT-4 and maintains conversation memory to enable context-aware responses.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node using OpenAI GPT-4.  
    - *Configuration:*  
      - Model: GPT-4 (version 4.1).  
      - Max tokens: 1024 to limit response length.  
      - Temperature: 0.3 for controlled creativity.  
      - Credentials: Uses configured OpenAI API key.  
    - *Input/Output:*  
      - Input: Text prompt from Support Agent node.  
      - Output: Generated chat response passed back to Support Agent.  
    - *Edge Cases:*  
      - API rate limits or key expiration may cause failures.  
      - Timeout or network errors.  
    - *Version:* 1.2

  - **Simple Memory**  
    - *Type & Role:* Memory buffer node to store recent conversation history.  
    - *Configuration:* Default settings, stores conversation context in a sliding window.  
    - *Input/Output:*  
      - Input: Chat messages and AI responses.  
      - Output: Provides context to Support Agent for follow-up messages.  
    - *Edge Cases:*  
      - Memory overflow or corruption could lose context.  
    - *Version:* 1.3

---

#### 2.3 Support Agent Logic

- **Overview:**  
  This is the core decision-making block that interprets user intents, calls appropriate tools (Google Sheets nodes), and manages the chatbot’s business logic.

- **Nodes Involved:**  
  - Support Agent

- **Node Details:**

  - **Support Agent**  
    - *Type & Role:* LangChain Agent node orchestrating tool usage and conversation flow.  
    - *Configuration:*  
      - System message defines the chatbot persona and operational rules.  
      - Tools defined: GetStock, PlaceOrder, UpdateStock with detailed input/output schemas.  
      - Rules enforce single tool call per turn, exact matching of case IDs, and response formatting.  
      - Supports multi-language replies (English and Roman-Nepali).  
      - Embeds product images in Markdown if available.  
      - Maintains internal state for stock data and user selections.  
      - Returns intermediate steps for debugging or transparency.  
    - *Input/Output:*  
      - Input: User message and memory context.  
      - Output: Next action (tool call or reply) to OpenAI Chat Model or Google Sheets nodes.  
    - *Edge Cases:*  
      - Incorrect or missing data from Google Sheets could cause logic errors.  
      - User ambiguity in product selection handled by asking clarifying questions.  
      - Tool call failures (e.g., Google Sheets API errors) must be handled gracefully.  
    - *Version:* 1.8

---

#### 2.4 Google Sheets Integration

- **Overview:**  
  These nodes interact with Google Sheets to fetch stock data, append new orders, and update inventory after sales.

- **Nodes Involved:**  
  - GetStock  
  - Place order  
  - Update Stock

- **Node Details:**

  - **GetStock**  
    - *Type & Role:* Google Sheets Tool node to read stock inventory.  
    - *Configuration:*  
      - Filters rows by "Phone Model" column matching user input.  
      - Reads from the "Inventory" sheet of the specified Google Sheet document.  
      - Combines filters with OR logic (though only one filter used).  
      - Credentials: Google Sheets OAuth2 account connected.  
    - *Input/Output:*  
      - Input: Phone model string from Support Agent.  
      - Output: List of matching stock items with fields like case_id, case_name, quantity_available, sold, image_url.  
    - *Edge Cases:*  
      - No matching rows found returns empty list; agent must handle gracefully.  
      - Google Sheets API quota or auth errors.  
    - *Version:* 4.5

  - **Place order**  
    - *Type & Role:* Google Sheets Tool node to append new order entries.  
    - *Configuration:*  
      - Appends rows to the "Order placed" sheet in the Google Sheet document.  
      - Columns mapped manually with AI overrides for fields like Case ID, Customer Name, Quantity, Address, Timestamp, etc.  
      - Credentials: Google Sheets OAuth2 account connected.  
    - *Input/Output:*  
      - Input: Order details from Support Agent.  
      - Output: Confirmation of append operation.  
    - *Edge Cases:*  
      - Invalid or missing order data could cause append failure.  
      - Google Sheets API errors or permission issues.  
    - *Version:* 4.5

  - **Update Stock**  
    - *Type & Role:* Google Sheets Tool node to update stock quantities after order.  
    - *Configuration:*  
      - Updates rows in the "Inventory" sheet matching on "Case ID".  
      - Updates columns: Quantity Available, Sold, and Updated ISO timestamp.  
      - Columns mapped manually.  
      - Credentials: Google Sheets OAuth2 account connected.  
    - *Input/Output:*  
      - Input: Updated stock numbers from Support Agent.  
      - Output: Confirmation of update operation.  
    - *Edge Cases:*  
      - Mismatched Case ID or concurrency issues could cause update failures.  
      - Google Sheets API errors or permission issues.  
    - *Version:* 4.5

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                      | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                           |
|-------------------------|------------------------------------|------------------------------------|---------------------------|---------------------------|-----------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat messages       | (Webhook external)         | Support Agent             | Initial greeting: "Hi! I’m Babish from Apple Case. How can I help?”                                 |
| Support Agent           | @n8n/n8n-nodes-langchain.agent     | Core chatbot logic and tool calls  | When chat message received, Simple Memory, OpenAI Chat Model | OpenAI Chat Model, GetStock, Place order, Update Stock | Detailed system message with rules, tools, and examples for customer support chatbot                |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context     | When chat message received | Support Agent             |                                                                                                     |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi | Processes chat messages with GPT-4 | Support Agent             | Support Agent             | Model: GPT-4.1, maxTokens=1024, temperature=0.3                                                     |
| GetStock                | n8n-nodes-base.googleSheetsTool    | Reads stock data from Google Sheets | Support Agent             | Support Agent             | Filters on Phone Model column, reads Inventory sheet                                               |
| Place order             | n8n-nodes-base.googleSheetsTool    | Appends new order to Google Sheets  | Support Agent             | Support Agent             | Appends to Order placed sheet, manual column mapping                                               |
| Update Stock            | n8n-nodes-base.googleSheetsTool    | Updates stock quantities in Google Sheets | Support Agent             | Support Agent             | Updates Inventory sheet matching on Case ID, updates Quantity Available and Sold                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a **LangChain Chat Trigger** node named "When chat message received".  
   - Set it as a public webhook.  
   - Configure response mode to "lastNode".  
   - Enable loading previous session memory ("memory").  
   - Set initial message: "Hi! I’m Babish from Apple Case. How can I help?”  

2. **Add Simple Memory Node**  
   - Add a **LangChain Memory Buffer Window** node named "Simple Memory".  
   - Use default settings to maintain recent conversation context.  

3. **Add OpenAI Chat Model Node**  
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Select model "gpt-4.1".  
   - Set max tokens to 1024.  
   - Set temperature to 0.3.  
   - Attach your OpenAI API credentials (OAuth or API key).  

4. **Add Support Agent Node**  
   - Add a **LangChain Agent** node named "Support Agent".  
   - Paste the provided system message defining the chatbot persona, tools, rules, and examples.  
   - Enable "return intermediate steps" for debugging.  

5. **Add Google Sheets Tool Node for GetStock**  
   - Add a **Google Sheets Tool** node named "GetStock".  
   - Set operation to "read".  
   - Select your Google Sheets OAuth2 credentials.  
   - Set Document ID to your stock/order spreadsheet ID.  
   - Set Sheet Name to the inventory sheet (e.g., "Inventory").  
   - Configure filter to lookup rows where "Phone Model" equals the input value from AI (`{{$fromAI('Value')}}`).  
   - Set combine filters to "OR".  

6. **Add Google Sheets Tool Node for Place order**  
   - Add a **Google Sheets Tool** node named "Place order".  
   - Set operation to "append".  
   - Select your Google Sheets OAuth2 credentials.  
   - Set Document ID to your spreadsheet ID.  
   - Set Sheet Name to the order sheet (e.g., "Order placed").  
   - Manually map columns for order details: Case ID, Case Name, Phone Model, Customer Name, Phone Number, Address, Quantity, Timestamp.  
   - Use AI overrides to populate these fields from the chatbot input.  

7. **Add Google Sheets Tool Node for Update Stock**  
   - Add a **Google Sheets Tool** node named "Update Stock".  
   - Set operation to "update".  
   - Select your Google Sheets OAuth2 credentials.  
   - Set Document ID to your spreadsheet ID.  
   - Set Sheet Name to the inventory sheet (e.g., "Inventory").  
   - Set "Column to match on" as "Case ID".  
   - Map columns to update: Quantity Available, Sold, Updated ISO (timestamp).  
   - Use AI overrides to populate these fields from the chatbot input.  

8. **Connect Nodes**  
   - Connect "When chat message received" → "Support Agent" (main).  
   - Connect "Support Agent" → "OpenAI Chat Model" (ai_languageModel).  
   - Connect "Support Agent" → "GetStock" (ai_tool).  
   - Connect "Support Agent" → "Place order" (ai_tool).  
   - Connect "Support Agent" → "Update Stock" (ai_tool).  
   - Connect "When chat message received" → "Simple Memory" (ai_memory).  
   - Connect "Simple Memory" → "Support Agent" (ai_memory).  

9. **Credentials Setup**  
   - Configure Google Sheets OAuth2 credentials with access to your spreadsheet.  
   - Configure OpenAI API credentials with valid API key and permissions.  

10. **Test and Validate**  
    - Test the chatbot by sending messages to the webhook URL.  
    - Validate stock queries, order placement, and stock updates reflect correctly in Google Sheets.  
    - Adjust system message or prompts in Support Agent for custom behavior.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Google Sheets credentials before running the workflow.                                       | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/#finish-your-n8n-credential |
| Google Sheet template includes two sheets: "Inventory" for stock and "Order placed" for orders.    | Example stock sheet and order sheet formats provided in the description.                            |
| The chatbot supports English and Roman-Nepali languages, adapting responses accordingly.           | Defined in Support Agent system message.                                                           |
| The workflow uses GPT-4.1 model with controlled temperature for reliable responses.                 | OpenAI Chat Model node configuration.                                                              |
| The Support Agent node enforces strict rules to avoid guessing case IDs and ensures data integrity.| System message rules section.                                                                       |
| Embeds product images in Markdown format if available from stock data.                             | Markdown syntax: `![<case_name>](<image_url>)`                                                     |

---

This documentation provides a complete, structured reference to understand, reproduce, and modify the "Customer and Sales Support" chatbot workflow using n8n, OpenAI GPT-4, and Google Sheets integration.