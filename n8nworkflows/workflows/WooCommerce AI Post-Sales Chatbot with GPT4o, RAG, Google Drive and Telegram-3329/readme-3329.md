WooCommerce AI Post-Sales Chatbot with GPT4o, RAG, Google Drive and Telegram

https://n8nworkflows.xyz/workflows/woocommerce-ai-post-sales-chatbot-with-gpt4o--rag--google-drive-and-telegram-3329


# WooCommerce AI Post-Sales Chatbot with GPT4o, RAG, Google Drive and Telegram

### 1. Workflow Overview

This workflow implements a **WooCommerce AI Post-Sales Chatbot** designed to automate and enhance customer support after purchase. It integrates WooCommerce order data, AI-powered natural language understanding, a vectorized knowledge base for policies, and human escalation via Telegram. The chatbot verifies customer identity strictly by matching email and order number before sharing sensitive order details, ensuring privacy and security.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Conversation Memory**: Receives customer chat messages and maintains conversation context.
- **1.2 AI Agent & Tool Integration**: The core AI agent processes requests, uses multiple tools to fetch order, user, and tracking data, and consults a vector store for policy FAQs.
- **1.3 WooCommerce Data Retrieval**: Nodes that interact with WooCommerce API to get order details, customer info, and multiple orders.
- **1.4 Tracking Information Retrieval**: Fetches shipment tracking data from WooCommerce tracking plugin and processes it.
- **1.5 Policy & FAQ Handling via Vector Store**: Uses Qdrant vector database and Google Drive documents to answer policy-related questions.
- **1.6 Human Escalation via Telegram**: Escalates complex or unresolved queries to human agents through Telegram.
- **1.7 Setup & Maintenance Utilities**: Nodes for creating and refreshing Qdrant collections, downloading documents, and embedding them.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Conversation Memory

**Overview:**  
This block receives incoming chat messages from customers and maintains a memory buffer to preserve conversation context for coherent multi-turn dialogues.

**Nodes Involved:**  
- When chat message received  
- Simple Memory

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger (Webhook)  
  - Role: Entry point for customer chat messages via webhook, triggers the workflow on new messages.  
  - Config: Public webhook with response mode set to respond via a node.  
  - Inputs: External chat message webhook  
  - Outputs: Connects to the AI agent node  
  - Edge Cases: Webhook failures, malformed input, network issues.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Stores recent conversation history to provide context for AI responses.  
  - Config: Default buffer window (no custom parameters)  
  - Inputs: From AI agent node (for storing context)  
  - Outputs: Back to AI agent node (for context injection)  
  - Edge Cases: Memory overflow, context loss if buffer size is too small.

---

#### 2.2 AI Agent & Tool Integration

**Overview:**  
The AI agent node orchestrates the conversation, uses various tools to fetch data, verifies identity, and decides when to escalate to humans.

**Nodes Involved:**  
- Post-Sales Agent  
- Calculator (tool for calculations)  
- ToS (Terms of Service vector store tool)  
- get_order (WooCommerce order fetch tool)  
- get_orders (WooCommerce multiple orders fetch tool)  
- get_user (WooCommerce customer fetch tool)  
- get_tracking (tracking info tool)  
- human_assistence (Telegram escalation tool)  
- GPT 4o-mini (OpenAI GPT-4o-mini language model)

**Node Details:**

- **Post-Sales Agent**  
  - Type: Langchain Agent  
  - Role: Central AI agent managing conversation flow, tool usage, and verification logic.  
  - Config:  
    - System message defines role as post-sales support AI for an online clothing store.  
    - Strict email verification rules before sharing order info.  
    - Uses tools: get_order, get_orders, get_user, get_tracking, ToS, human_assistence.  
    - Communication style: professional, precise, privacy-focused.  
  - Inputs: From chat trigger and memory nodes  
  - Outputs: To memory and tools  
  - Edge Cases: Expression evaluation errors, tool failures, incorrect email verification.

- **Calculator**  
  - Type: Langchain Calculator Tool  
  - Role: Performs any necessary calculations during conversation (e.g., dates, totals).  
  - Config: Default, no parameters.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: Calculation errors or invalid inputs.

- **ToS**  
  - Type: Langchain Vector Store Tool  
  - Role: Answers questions about terms, conditions, shipping policies using vector search.  
  - Config: Named "company" with description for policy-related queries.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: Vector store unavailability, no matching documents.

- **get_order**  
  - Type: WooCommerce Tool (order get)  
  - Role: Retrieves details for a single order by order ID.  
  - Config: Order ID dynamically set from AI input with strict email verification logic embedded in prompt.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: Order not found, API auth errors, invalid order ID.

- **get_orders**  
  - Type: WooCommerce Tool (order getAll)  
  - Role: Retrieves multiple orders filtered by search criteria.  
  - Config: Search and returnAll parameters dynamically set from AI input.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: Large result sets, API rate limits.

- **get_user**  
  - Type: WooCommerce Tool (customer getAll)  
  - Role: Retrieves customer profile(s) by email filter.  
  - Config: Email filter dynamically set from AI input.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: No user found, multiple users returned.

- **get_tracking**  
  - Type: Langchain Tool Workflow (sub-workflow)  
  - Role: Retrieves tracking number for a specific order by order number.  
  - Config: Calls sub-workflow "WooCommerce Get Tracking Number" with order_number input.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: Sub-workflow failures, missing tracking info.

- **human_assistence**  
  - Type: Telegram Tool  
  - Role: Escalates complex or unresolved queries to human operators via Telegram chat.  
  - Config: Sends AI-generated text to configured Telegram CHAT_ID.  
  - Inputs: From AI agent  
  - Outputs: None (side effect)  
  - Edge Cases: Telegram API errors, invalid chat ID.

- **GPT 4o-mini**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Language model used by the AI agent for generating responses.  
  - Config: Model set to "gpt-4o-mini".  
  - Inputs: From AI agent (as language model)  
  - Outputs: To AI agent  
  - Edge Cases: API rate limits, model unavailability.

---

#### 2.3 WooCommerce Data Retrieval

**Overview:**  
This block interacts directly with WooCommerce APIs to fetch order and customer data needed for verification and response.

**Nodes Involved:**  
- get_order  
- get_orders  
- get_user

**Node Details:**  
(Details covered above in 2.2 as these nodes are tools used by the AI agent.)

---

#### 2.4 Tracking Information Retrieval

**Overview:**  
Fetches shipment tracking details from WooCommerce using the YITH WooCommerce Order & Shipment Tracking plugin API, then extracts relevant metadata.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Get tracking (HTTP Request)  
- Set tracking code

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered by other workflows with JSON input (order_number).  
  - Config: Example JSON input with order_number field.  
  - Inputs: External trigger  
  - Outputs: To Get tracking node  
  - Edge Cases: Invalid input JSON.

- **Get tracking**  
  - Type: HTTP Request  
  - Role: Calls WooCommerce REST API endpoint to get order details including tracking metadata.  
  - Config: URL dynamically built with order_number; uses HTTP Basic Auth credentials for WooCommerce.  
  - Inputs: From Execute Workflow Trigger  
  - Outputs: To Set tracking code  
  - Edge Cases: API auth failures, invalid order number, network errors.

- **Set tracking code**  
  - Type: Set Node  
  - Role: Extracts tracking_code, carrier_url, and pick_up date from order meta_data array.  
  - Config: Uses expressions to find keys "ywot_tracking_code", "ywot_carrier_url", "ywot_pick_up_date" in meta_data.  
  - Inputs: From Get tracking  
  - Outputs: To AI agent via get_tracking tool (indirect)  
  - Edge Cases: Missing meta_data keys, malformed response.

---

#### 2.5 Policy & FAQ Handling via Vector Store

**Overview:**  
This block manages the vectorized knowledge base containing company policies and FAQs, using Qdrant vector store and Google Drive documents for data ingestion.

**Nodes Involved:**  
- Create collection  
- Refresh collection  
- Get folder  
- Download Files  
- Default Data Loader  
- Token Splitter  
- Qdrant Vector Store1  
- Embeddings OpenAI1  
- Qdrant Vector Store  
- Embeddings OpenAI  
- ToS

**Node Details:**

- **Create collection**  
  - Type: HTTP Request  
  - Role: Creates a new Qdrant collection for storing vectorized documents.  
  - Config: POST to QDRANTURL/collections/COLLECTION with empty filter, uses HTTP header auth.  
  - Inputs: Manual trigger or setup workflow  
  - Outputs: To Refresh collection  
  - Edge Cases: Collection already exists, auth errors.

- **Refresh collection**  
  - Type: HTTP Request  
  - Role: Deletes all points in the Qdrant collection to refresh data.  
  - Config: POST to QDRANTURL/collections/COLLECTION/points/delete with empty filter.  
  - Inputs: From Create collection or manual trigger  
  - Outputs: To Get folder  
  - Edge Cases: API errors, collection not found.

- **Get folder**  
  - Type: Google Drive  
  - Role: Lists files in a specified Google Drive folder ("test-whatsapp").  
  - Config: Uses Google Drive OAuth2 credentials, folder ID set to "test-whatsapp".  
  - Inputs: From Refresh collection  
  - Outputs: To Download Files  
  - Edge Cases: Folder not found, permission denied.

- **Download Files**  
  - Type: Google Drive  
  - Role: Downloads each file from the folder, converting Google Docs to plain text.  
  - Config: File ID from Get folder, conversion to text/plain enabled.  
  - Inputs: From Get folder  
  - Outputs: To Qdrant Vector Store1  
  - Edge Cases: Download failures, unsupported file types.

- **Default Data Loader**  
  - Type: Langchain Document Default Data Loader  
  - Role: Loads downloaded binary documents for processing.  
  - Config: Data type set to binary.  
  - Inputs: From Token Splitter  
  - Outputs: To Qdrant Vector Store1  
  - Edge Cases: Data corruption, unsupported formats.

- **Token Splitter**  
  - Type: Langchain Text Splitter (Token Splitter)  
  - Role: Splits documents into chunks of 300 tokens with 30 tokens overlap for vectorization.  
  - Config: chunkSize=300, chunkOverlap=30  
  - Inputs: From Default Data Loader  
  - Outputs: To Default Data Loader (loop)  
  - Edge Cases: Improper splitting, tokenization errors.

- **Qdrant Vector Store1**  
  - Type: Langchain Vector Store Qdrant (Insert mode)  
  - Role: Inserts vectorized document chunks into Qdrant collection.  
  - Config: Collection name set to COLLECTION, uses Qdrant API credentials.  
  - Inputs: From Download Files and Default Data Loader  
  - Outputs: None  
  - Edge Cases: API errors, insertion failures.

- **Embeddings OpenAI1**  
  - Type: Langchain OpenAI Embeddings  
  - Role: Generates embeddings for documents before insertion into Qdrant.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From Default Data Loader  
  - Outputs: To Qdrant Vector Store1  
  - Edge Cases: API rate limits, embedding failures.

- **Qdrant Vector Store**  
  - Type: Langchain Vector Store Qdrant (Query mode)  
  - Role: Used by ToS tool to query the vector store for policy answers.  
  - Config: Collection name COLLECTION, Qdrant API credentials.  
  - Inputs: From Embeddings OpenAI  
  - Outputs: To ToS tool  
  - Edge Cases: Query failures, no results.

- **Embeddings OpenAI**  
  - Type: Langchain OpenAI Embeddings  
  - Role: Generates embeddings for user queries to match against vector store.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From ToS tool  
  - Outputs: To Qdrant Vector Store  
  - Edge Cases: API errors.

- **ToS**  
  - Type: Langchain Tool Vector Store  
  - Role: Answers customer questions about terms, shipping, and policies using vector search.  
  - Config: Named "company" with description for policy questions.  
  - Inputs: From AI agent  
  - Outputs: To AI agent  
  - Edge Cases: No matching documents, vector store downtime.

---

#### 2.6 Human Escalation via Telegram

**Overview:**  
Escalates unresolved or complex customer queries to human agents via Telegram chat.

**Nodes Involved:**  
- human_assistence  
- Sticky Note (Telegram CHAT_ID reminder)

**Node Details:**

- **human_assistence**  
  - Type: Telegram Tool  
  - Role: Sends escalation messages to a configured Telegram chat ID.  
  - Config: Text dynamically generated by AI, chatId set to CHAT_ID placeholder.  
  - Inputs: From AI agent  
  - Outputs: None (side effect)  
  - Edge Cases: Invalid chat ID, Telegram API errors.

- **Sticky Note**  
  - Content: Reminder to add Telegram CHAT_ID in human_assistence node.  
  - Position: Near human_assistence node.  
  - Purpose: Setup instruction.

---

#### 2.7 Setup & Maintenance Utilities

**Overview:**  
Nodes to create and refresh Qdrant collections, download and process documents from Google Drive, and embed them for vector search.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Create collection  
- Refresh collection  
- Get folder  
- Download Files  
- Token Splitter  
- Default Data Loader  
- Embeddings OpenAI1  
- Qdrant Vector Store1  
- Sticky Notes (for setup instructions)

**Node Details:**  
(Details covered in 2.5 and additional notes below.)

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual testing of collection creation and refresh.  
  - Inputs: Manual trigger  
  - Outputs: To Create collection and Refresh collection nodes  
  - Edge Cases: Manual errors.

- **Sticky Notes**  
  - Provide stepwise setup instructions for Qdrant URL, collection name, Telegram CHAT_ID, and WooCommerce tracking plugin URL.  
  - Contain useful links (e.g., YITH WooCommerce Order Tracking plugin).

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                          | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                  |
|----------------------------|----------------------------------------|----------------------------------------|-------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger (Webhook)       | Entry point for chat messages           | External webhook               | Post-Sales Agent              |                                                                                                              |
| Simple Memory              | Langchain Memory Buffer Window          | Maintains conversation context          | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| Post-Sales Agent           | Langchain Agent                         | Core AI agent managing conversation     | When chat message received, Simple Memory | Simple Memory, Tools (get_order, get_user, etc.) |                                                                                                              |
| Calculator                 | Langchain Calculator Tool               | Performs calculations                    | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| ToS                       | Langchain Vector Store Tool             | Answers policy & FAQ questions           | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| get_order                 | WooCommerce Tool (order get)            | Retrieves single order details           | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| get_orders                | WooCommerce Tool (order getAll)         | Retrieves multiple orders                 | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| get_user                  | WooCommerce Tool (customer getAll)      | Retrieves customer profile by email      | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| get_tracking              | Langchain Tool Workflow (sub-workflow) | Retrieves tracking info via sub-workflow | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| human_assistence          | Telegram Tool                          | Escalates queries to human via Telegram | Post-Sales Agent              | None                         | # STEP 3 - Add your Telegram CHAT_ID in the "human_assistance" tool                                          |
| GPT 4o-mini               | Langchain OpenAI Chat Model             | Language model for AI agent              | Post-Sales Agent              | Post-Sales Agent              |                                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger           | Allows external workflow trigger         | External trigger              | Get tracking                 |                                                                                                              |
| Get tracking              | HTTP Request                           | Fetches WooCommerce order tracking data | When Executed by Another Workflow | Set tracking code            | # STEP 4 - The tracking code request uses "YITH WooCommerce Order & Shipment Tracking" plugin. See link in note. |
| Set tracking code         | Set Node                              | Extracts tracking metadata from response | Get tracking                 | (Indirect to AI agent)       |                                                                                                              |
| Create collection         | HTTP Request                         | Creates Qdrant collection                 | When clicking ‘Test workflow’ | Refresh collection           | # STEP 1 - Create Qdrant Collection. Change QDRANTURL and COLLECTION                                         |
| Refresh collection        | HTTP Request                         | Deletes all points in Qdrant collection   | Create collection            | Get folder                  | # STEP 1 - Create Qdrant Collection. Change QDRANTURL and COLLECTION                                         |
| Get folder                | Google Drive                         | Lists files in Google Drive folder        | Refresh collection           | Download Files              |                                                                                                              |
| Download Files            | Google Drive                         | Downloads and converts files to text      | Get folder                  | Qdrant Vector Store1        |                                                                                                              |
| Default Data Loader       | Langchain Document Default Data Loader | Loads downloaded documents for processing | Token Splitter              | Qdrant Vector Store1        |                                                                                                              |
| Token Splitter            | Langchain Text Splitter (Token Splitter) | Splits documents into chunks for vectorization | Default Data Loader         | Default Data Loader         |                                                                                                              |
| Qdrant Vector Store1      | Langchain Vector Store Qdrant (Insert) | Inserts document vectors into Qdrant      | Download Files, Default Data Loader | None                      |                                                                                                              |
| Embeddings OpenAI1        | Langchain OpenAI Embeddings           | Generates embeddings for documents         | Default Data Loader          | Qdrant Vector Store1        |                                                                                                              |
| Qdrant Vector Store       | Langchain Vector Store Qdrant (Query) | Queries Qdrant for policy answers          | Embeddings OpenAI            | ToS                        |                                                                                                              |
| Embeddings OpenAI         | Langchain OpenAI Embeddings           | Generates embeddings for queries            | ToS                         | Qdrant Vector Store         |                                                                                                              |
| When clicking ‘Test workflow’ | Manual Trigger                      | Manual start for collection setup          | Manual                      | Create collection, Refresh collection |                                                                                                              |
| Sticky Note3              | Sticky Note                          | Instruction for Qdrant collection setup    | None                       | None                       | # STEP 1 - Create Qdrant Collection. Change QDRANTURL and COLLECTION                                         |
| Sticky Note5              | Sticky Note                          | Instruction for document vectorization setup | None                       | None                       | # STEP 2 - Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION             |
| Sticky Note               | Sticky Note                          | Reminder to add Telegram CHAT_ID            | None                       | None                       | # STEP 3 - Add your Telegram CHAT_ID in the "human_assistance" tool                                          |
| Sticky Note1              | Sticky Note                          | Instruction for WooCommerce tracking plugin | None                       | None                       | # STEP 4 - Tracking code request uses YITH WooCommerce Order & Shipment Tracking plugin. Download link included |
| Sticky Note2              | Sticky Note                          | Workflow description and overview           | None                       | None                       | Workflow overview and purpose description                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create webhook trigger node**  
   - Type: Langchain Chat Trigger  
   - Configure as public webhook with response mode set to "responseNode".  
   - This node receives incoming customer chat messages.

2. **Add conversation memory node**  
   - Type: Langchain Memory Buffer Window  
   - Connect input from AI agent output and output back to AI agent input for context.

3. **Create AI agent node**  
   - Type: Langchain Agent  
   - Configure system message with detailed role, verification rules, tool usage instructions, and communication style as per workflow description.  
   - Connect input from webhook and memory nodes.  
   - Add tools: get_order, get_orders, get_user, get_tracking, ToS, human_assistence, Calculator.  
   - Connect outputs to memory and tools accordingly.

4. **Add WooCommerce API nodes**  
   - **get_order**: WooCommerce Tool node, operation "get" on "order" resource.  
     - Set orderId parameter dynamically from AI input (order number).  
     - Use WooCommerce API credentials.  
   - **get_orders**: WooCommerce Tool node, operation "getAll" on "order" resource.  
     - Configure search and returnAll parameters dynamically.  
     - Use WooCommerce API credentials.  
   - **get_user**: WooCommerce Tool node, operation "getAll" on "customer" resource.  
     - Filter by email dynamically from AI input.  
     - Use WooCommerce API credentials.

5. **Add tracking retrieval sub-workflow node**  
   - Type: Langchain Tool Workflow  
   - Configure to call sub-workflow "WooCommerce Get Tracking Number" with input schema containing order_number string.  
   - Connect to AI agent as a tool.

6. **Add Telegram escalation node**  
   - Type: Telegram Tool  
   - Configure chatId with your Telegram CHAT_ID.  
   - Set text parameter dynamically from AI input.  
   - Use Telegram API credentials.

7. **Add OpenAI GPT-4o-mini model node**  
   - Type: Langchain OpenAI Chat Model  
   - Set model to "gpt-4o-mini".  
   - Use OpenAI API credentials.  
   - Connect as language model for AI agent.

8. **Create sub-workflow for tracking info retrieval**  
   - Trigger: Execute Workflow Trigger node with JSON input containing order_number.  
   - HTTP Request node to WooCommerce REST API endpoint `/wp-json/wc/v3/orders/{{order_number}}` using HTTP Basic Auth credentials.  
   - Set Node to extract tracking_code, carrier_url, and pick_up date from meta_data array.  
   - Output tracking info back to calling workflow.

9. **Set up Qdrant vector store for policy documents**  
   - HTTP Request node to create Qdrant collection at `https://QDRANTURL/collections/COLLECTION` with POST and empty filter.  
   - HTTP Request node to refresh collection by deleting all points.  
   - Google Drive node to list files in folder "test-whatsapp" (or your folder).  
   - Google Drive node to download files, converting Google Docs to plain text.  
   - Langchain Document Default Data Loader node to load downloaded files.  
   - Langchain Text Splitter (Token Splitter) node to split documents into 300-token chunks with 30-token overlap.  
   - Langchain OpenAI Embeddings node to generate embeddings for document chunks.  
   - Langchain Vector Store Qdrant node (insert mode) to insert embeddings into Qdrant collection.  
   - Connect nodes in sequence: Create collection → Refresh collection → Get folder → Download Files → Token Splitter → Default Data Loader → Embeddings OpenAI1 → Qdrant Vector Store1.

10. **Set up vector store query path for AI agent**  
    - Langchain OpenAI Embeddings node to embed user queries.  
    - Langchain Vector Store Qdrant node (query mode) to query Qdrant collection.  
    - Langchain Tool Vector Store node (ToS) to answer policy questions using vector store results.  
    - Connect AI agent to ToS tool.

11. **Add manual trigger node for testing collection setup**  
    - Manual Trigger node connected to Create collection and Refresh collection nodes.

12. **Add sticky notes for setup instructions**  
    - Add sticky notes near relevant nodes with instructions to:  
      - Replace QDRANTURL and COLLECTION placeholders.  
      - Add Telegram CHAT_ID in human_assistence node.  
      - Install and configure YITH WooCommerce Order & Shipment Tracking plugin and update HTTP Request URL accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow designed for WooCommerce post-sales support automation with AI and human escalation.                           | Workflow description and purpose.                                                                 |
| Replace `QDRANTURL` and `COLLECTION` placeholders with your Qdrant API URL and collection name before running.           | Setup instructions in Sticky Note3 and Sticky Note5.                                              |
| Add your Telegram `CHAT_ID` in the `human_assistence` node to enable escalation.                                         | Sticky Note near human_assistence node.                                                           |
| Use YITH WooCommerce Order & Shipment Tracking plugin for tracking code retrieval. Free plugin available at WordPress.org | [https://wordpress.org/plugins/yith-woocommerce-order-tracking/](https://wordpress.org/plugins/yith-woocommerce-order-tracking/) |
| Contact info for customization and support: [info@n3w.it](mailto:info@n3w.it), LinkedIn: https://www.linkedin.com/in/davideboizza/ | Provided in workflow description.                                                                 |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the WooCommerce AI Post-Sales Chatbot workflow. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and automation agents to work effectively with the workflow.