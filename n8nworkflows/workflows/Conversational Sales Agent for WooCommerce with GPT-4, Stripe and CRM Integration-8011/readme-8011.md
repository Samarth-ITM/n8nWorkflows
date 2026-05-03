Conversational Sales Agent for WooCommerce with GPT-4, Stripe and CRM Integration

https://n8nworkflows.xyz/workflows/conversational-sales-agent-for-woocommerce-with-gpt-4--stripe-and-crm-integration-8011


# Conversational Sales Agent for WooCommerce with GPT-4, Stripe and CRM Integration

### 1. Workflow Overview

This workflow implements a **Conversational Sales Agent for WooCommerce** powered by GPT-4 and integrated with Stripe and CRM systems. It automates customer interactions by understanding intent, searching products, answering FAQs, generating payment links, capturing leads, and escalating complex queries to humans. The workflow combines AI-driven conversation handling with real-time e-commerce and CRM operations, enabling a 24/7 intelligent sales assistant.

The workflow is logically divided into two main blocks:

- **1.1 Conversational Sales Agent Runtime**  
  Handles incoming chat messages, extracts buyer intent and key fields, routes requests via a Sales AI Agent using GPT-4 with multiple integrated tools (WooCommerce, FAQ vector search, payment, CRM, escalation), and manages session memory.

- **1.2 Knowledge Base Update and Vector Store Maintenance**  
  A manual-triggered flow that reloads sales and policy documents from Google Drive, processes them into embeddings, and updates a Qdrant vector store to keep the FAQ knowledge base current and accurate for retrieval-augmented generation (RAG).

---

### 2. Block-by-Block Analysis

#### 2.1 Conversational Sales Agent Runtime

- **Overview:**  
  This block receives chat messages, extracts buyer intents and details using GPT-4, maintains session context, and routes queries to appropriate tools (product search, FAQ retrieval, payment link generation, CRM lead creation, or human escalation). It uses a layered agent with GPT-4 models and vector store assisted FAQ answering.

- **Nodes Involved:**  
  - When chat message received  
  - Edit Fields1  
  - Sales Info Extractor1  
  - OpenAI Chat Model (Extractor)1  
  - Window Buffer Memory1  
  - Sales AI Agent1  
  - OpenAI Chat Model (Agent)1  
  - Qdrant Vector Store (runtime)1  
  - RAG_FAQ1  
  - PRODUCT_SEARCH_WOO1  
  - INVENTORY_DETAIL_WOO1  
  - PAYMENT_LINK1  
  - CRM_LEAD1  
  - HUMAN_ESCALATION1  
  - Google Gemini Chat Model1

- **Node Details:**

  1. **When chat message received**  
     - Type: Langchain ChatTrigger (webhook)  
     - Role: Entry point for incoming chat messages.  
     - Config: Default options, webhookId `"SALES-CHAT-TRIGGER"`.  
     - Connections: Outputs to `Edit Fields1`.  
     - Failure modes: Webhook misconfiguration, malformed input.

  2. **Edit Fields1**  
     - Type: Set node  
     - Role: Extracts `sessionId` and `chatInput` from incoming JSON to prepare data for processing.  
     - Config: Assigns two string fields (`sessionId`, `chatInput`) from input JSON.  
     - Inputs: From chat trigger node.  
     - Outputs: To `Sales Info Extractor1`.  
     - Edge cases: Missing sessionId or chatInput could affect downstream context.

  3. **Sales Info Extractor1**  
     - Type: Langchain Information Extractor  
     - Role: Extracts buyer intent and key fields (keyword, sku, price range, name, email) from chat input using a custom system prompt.  
     - Config: System prompt guides extraction of intents (product, faq, payment, lead), fields are typed and structured manually via schema.  
     - Inputs: `chatInput` string from `Edit Fields1`.  
     - Outputs: To `Sales AI Agent1`.  
     - Edge cases: Extraction errors, hallucination avoided by prompt. Missing fields handled by agent routing.

  4. **OpenAI Chat Model (Extractor)1**  
     - Type: Langchain OpenAI Chat Model (GPT-4.1-mini)  
     - Role: Provides LLM power to the Information Extractor node.  
     - Config: GPT-4.1-mini model, linked via OpenAI API credential.  
     - Inputs: Feeds from `Sales Info Extractor1`.  
     - Outputs: Parsed structured extraction to `Sales Info Extractor1`.  
     - Edge cases: API throttling, auth failure.

  5. **Window Buffer Memory1**  
     - Type: Langchain Memory Buffer Window  
     - Role: Maintains conversational context per session with a 12-message sliding window, keyed by `sessionId`.  
     - Config: Session key from `Edit Fields1.sessionId`.  
     - Inputs: From `Sales AI Agent1` (context update).  
     - Outputs: Back to `Sales AI Agent1` as conversational memory.  
     - Edge cases: Session key missing, memory overflow.

  6. **Sales AI Agent1**  
     - Type: Langchain Agent  
     - Role: Core decision-making agent that routes requests based on extracted intent to appropriate tools: FAQ retrieval, product search, payment link creation, CRM lead, or human escalation.  
     - Config: System prompt defines role, routing logic, output formatting, and next action suggestions.  
     - Inputs: Info extractor output, memory context.  
     - Outputs: Tool invocations (RAG_FAQ1, PRODUCT_SEARCH_WOO1, PAYMENT_LINK1, CRM_LEAD1, HUMAN_ESCALATION1).  
     - Edge cases: Incorrect routing, API failures in downstream tools.

  7. **OpenAI Chat Model (Agent)1**  
     - Type: GPT-4.1-mini OpenAI model  
     - Role: Supplies LLM processing power to the Sales AI Agent.  
     - Config: Uses same OpenAI credentials as extractor.  
     - Inputs: Sales AI Agent prompts.  
     - Outputs: To `Sales AI Agent1`.  
     - Edge cases: API limits.

  8. **Qdrant Vector Store (runtime)1**  
     - Type: Langchain Qdrant Vector Store (runtime query)  
     - Role: Provides vector search over FAQ documents for RAG retrieval.  
     - Config: Collection named "sales_docs" on Qdrant instance.  
     - Inputs: From `Sales AI Agent1` via RAG_FAQ1.  
     - Outputs: To `RAG_FAQ1`.  
     - Edge cases: Connection errors, empty collections.

  9. **RAG_FAQ1**  
     - Type: Langchain Tool Vector Store  
     - Role: Answers FAQs and policy queries by retrieving relevant documents from Qdrant vector store.  
     - Config: Named "RAG_FAQ", description clarifies FAQ answering.  
     - Inputs: From `Sales AI Agent1`.  
     - Outputs: Back to `Sales AI Agent1`.  
     - Edge cases: No relevant docs found.

  10. **PRODUCT_SEARCH_WOO1**  
      - Type: WooCommerce Tool  
      - Role: Searches WooCommerce products based on extracted SKU, keywords, price range, and stock status.  
      - Config: Uses parameters from extractor fields; stock status filtered to "instock".  
      - Inputs: From `Sales AI Agent1`.  
      - Outputs: To `Sales AI Agent1`.  
      - Edge cases: API errors, empty search results.

  11. **INVENTORY_DETAIL_WOO1**  
      - Type: WooCommerce Tool  
      - Role: Retrieves detailed inventory info for a specific product SKU shortlisted by the agent.  
      - Config: Product ID parameter dynamically from AI output.  
      - Inputs: From `Sales AI Agent1`.  
      - Outputs: To `Sales AI Agent1`.  
      - Edge cases: Invalid product ID.

  12. **PAYMENT_LINK1**  
      - Type: Langchain Tool Workflow (Stripe payment link)  
      - Role: Invokes a sub-workflow to generate Stripe payment links for chosen items.  
      - Config: Sub-workflow ID referenced ("PAYLINK-AGENT-ID"), no input parameters defined (empty schema).  
      - Inputs: From `Sales AI Agent1`.  
      - Outputs: Back to `Sales AI Agent1`.  
      - Edge cases: Stripe API failures, payment link generation errors.

  13. **CRM_LEAD1**  
      - Type: Langchain Tool Workflow (CRM integration)  
      - Role: Invokes a sub-workflow to create or update leads in CRM platforms like HubSpot or Pipedrive.  
      - Config: Sub-workflow ID referenced ("CRM-AGENT-ID"), no explicit inputs defined.  
      - Inputs: From `Sales AI Agent1`.  
      - Outputs: Back to `Sales AI Agent1`.  
      - Edge cases: CRM API issues, data mapping errors.

  14. **HUMAN_ESCALATION1**  
      - Type: Telegram Tool  
      - Role: Sends escalation messages to human agents via Telegram chat when queries are blocked or unclear.  
      - Config: Sends AI-generated text to a Telegram chat ID `"SALES-ESCALATION-CHAT"`.  
      - Inputs: From `Sales AI Agent1`.  
      - Outputs: None further.  
      - Edge cases: Telegram API downtime, chat ID misconfiguration.

  15. **Google Gemini Chat Model1**  
      - Type: Langchain Google Gemini Chat Model  
      - Role: Supplies LLM power for the RAG_FAQ node to answer FAQ queries using Google’s PaLM API.  
      - Config: Uses Google Palm API credentials.  
      - Inputs: From `RAG_FAQ1`.  
      - Outputs: Back to `RAG_FAQ1`.  
      - Edge cases: API quota, latency.

#### 2.2 Knowledge Base Update and Vector Store Maintenance

- **Overview:**  
  This block updates the FAQ knowledge base by downloading sales and policy documents from Google Drive, splitting text into manageable chunks, generating embeddings, and inserting them into the Qdrant vector store. Triggered manually to refresh the AI’s document knowledge for accurate FAQ handling.

- **Nodes Involved:**  
  - When clicking 'Execute workflow' (manual trigger)  
  - Qdrant Wipe1  
  - Google Drive: List1  
  - Google Drive: Download1  
  - Default Data Loader1  
  - Token Splitter1  
  - Embeddings OpenAI (build)1  
  - Qdrant Vector Store (insert)1

- **Node Details:**

  1. **When clicking 'Execute workflow'**  
     - Type: Manual Trigger  
     - Role: Entry point to start the knowledge base refresh process.  
     - Config: No parameters.  
     - Outputs: To `Qdrant Wipe1`.  
     - Edge cases: Manual initiation required.

  2. **Qdrant Wipe1**  
     - Type: HTTP Request  
     - Role: Clears all points from the Qdrant collection "sales_docs" to wipe old vectors before reindexing.  
     - Config: POST request to Qdrant API delete endpoint, sends empty filter to delete all points.  
     - Authentication: HTTP Header Auth with custom headers.  
     - Inputs: From manual trigger.  
     - Outputs: To `Google Drive: List1`.  
     - Edge cases: API errors, network issues, permission denied.

  3. **Google Drive: List1**  
     - Type: Google Drive node (list files)  
     - Role: Lists all files in a configured Google Drive folder containing sales and policy documents.  
     - Config: Drive "My Drive", folder ID parameterized as `"DRIVE_FOLDER_ID"`.  
     - Inputs: From `Qdrant Wipe1`.  
     - Outputs: To `Google Drive: Download1`.  
     - Edge cases: Folder not found, permission errors.

  4. **Google Drive: Download1**  
     - Type: Google Drive node (download file)  
     - Role: Downloads each file found in the previous step, converts Google Docs to plain text.  
     - Config: File ID from list node, conversion enabled for Docs → text/plain.  
     - Inputs: From `Google Drive: List1`.  
     - Outputs: To `Qdrant Vector Store (insert)1` via intermediary nodes.  
     - Edge cases: Download failures, unsupported file types.

  5. **Default Data Loader1**  
     - Type: Langchain Document Default Data Loader  
     - Role: Loads downloaded binary data for further processing.  
     - Config: Data type set to binary.  
     - Inputs: From `Google Drive: Download1`.  
     - Outputs: To `Token Splitter1`.  
     - Edge cases: Data corruption.

  6. **Token Splitter1**  
     - Type: Langchain Text Splitter (Token Splitter)  
     - Role: Splits text into chunks of 300 tokens with 30-token overlap for embedding.  
     - Config: Chunk size 300 tokens, overlap 30 tokens.  
     - Inputs: From `Default Data Loader1`.  
     - Outputs: To `Embeddings OpenAI (build)1`.  
     - Edge cases: Text too short, splitting errors.

  7. **Embeddings OpenAI (build)1**  
     - Type: Langchain OpenAI Embeddings  
     - Role: Generates vector embeddings for each text chunk using OpenAI API.  
     - Config: Uses same OpenAI credentials as chat models.  
     - Inputs: From `Token Splitter1`.  
     - Outputs: To `Qdrant Vector Store (insert)1`.  
     - Edge cases: API quota limits, failures.

  8. **Qdrant Vector Store (insert)1**  
     - Type: Langchain Qdrant Vector Store (insert mode)  
     - Role: Inserts new embeddings into the "sales_docs" Qdrant collection, updating the vector database.  
     - Config: Insert mode, collection "sales_docs".  
     - Inputs: From `Embeddings OpenAI (build)1` and `Google Drive: Download1` (documents).  
     - Outputs: None further.  
     - Edge cases: Insert failures, connection issues.

---

### 3. Summary Table

| Node Name                      | Node Type                                   | Functional Role                                 | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                   |
|--------------------------------|---------------------------------------------|------------------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received      | Langchain ChatTrigger (webhook)             | Entry point for incoming chat messages          |                              | Edit Fields1                    | 🛒 **WooCommerce AI Sales Agent – Overview** This workflow turns n8n into an AI-powered sales assistant...    |
| Edit Fields1                   | Set                                         | Extract sessionId and chatInput                  | When chat message received     | Sales Info Extractor1           |                                                                                                              |
| Sales Info Extractor1          | Langchain Information Extractor              | Extract buyer intent and key fields              | Edit Fields1                  | Sales AI Agent1                |                                                                                                              |
| OpenAI Chat Model (Extractor)1 | Langchain OpenAI Chat Model                   | LLM for information extraction                    | Sales Info Extractor1         | Sales Info Extractor1           |                                                                                                              |
| Window Buffer Memory1          | Langchain Memory Buffer Window                | Maintains conversation context per session       | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| Sales AI Agent1               | Langchain Agent                               | Routes intents to tools and generates responses | Sales Info Extractor1, Window Buffer Memory1 | RAG_FAQ1, PRODUCT_SEARCH_WOO1, PAYMENT_LINK1, CRM_LEAD1, HUMAN_ESCALATION1 |                                                                                                              |
| OpenAI Chat Model (Agent)1    | Langchain OpenAI Chat Model                   | LLM for Sales AI Agent                            | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| Qdrant Vector Store (runtime)1 | Langchain Qdrant Vector Store                 | Vector search over FAQ documents                  | Sales AI Agent1               | RAG_FAQ1                      |                                                                                                              |
| RAG_FAQ1                     | Langchain Tool Vector Store                    | FAQ answering via vector store retrieval          | Qdrant Vector Store (runtime)1 | Sales AI Agent1                |                                                                                                              |
| PRODUCT_SEARCH_WOO1           | WooCommerce Tool                              | Searches WooCommerce products                      | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| INVENTORY_DETAIL_WOO1         | WooCommerce Tool                              | Retrieves inventory details for selected product  | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| PAYMENT_LINK1                 | Langchain Tool Workflow                        | Generates Stripe payment links                     | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| CRM_LEAD1                    | Langchain Tool Workflow                        | Creates/updates lead in CRM                         | Sales AI Agent1               | Sales AI Agent1                |                                                                                                              |
| HUMAN_ESCALATION1             | Telegram Tool                                 | Escalates unclear queries to human agents         | Sales AI Agent1               |                               |                                                                                                              |
| Google Gemini Chat Model1     | Langchain Google Gemini Chat Model             | LLM for RAG FAQ answering                          | RAG_FAQ1                     | RAG_FAQ1                      |                                                                                                              |
| When clicking 'Execute workflow' | Manual Trigger                              | Triggers knowledge base update flow               |                              | Qdrant Wipe1                  | 📂 **Knowledge Base Update Flow** Purpose: Keeps the AI’s FAQ knowledge accurate and up to date...             |
| Qdrant Wipe1                 | HTTP Request                                  | Deletes all old vectors in Qdrant collection       | When clicking 'Execute workflow' | Google Drive: List1            |                                                                                                              |
| Google Drive: List1          | Google Drive List                             | Lists sales documents in configured folder         | Qdrant Wipe1                 | Google Drive: Download1        |                                                                                                              |
| Google Drive: Download1      | Google Drive Download                         | Downloads and converts Google Docs to text         | Google Drive: List1          | Default Data Loader1           |                                                                                                              |
| Default Data Loader1         | Langchain Document Default Data Loader         | Loads binary data for text splitting               | Google Drive: Download1      | Token Splitter1               |                                                                                                              |
| Token Splitter1              | Langchain Text Splitter (Token Splitter)        | Splits text into chunks for embedding               | Default Data Loader1         | Embeddings OpenAI (build)1    |                                                                                                              |
| Embeddings OpenAI (build)1    | Langchain OpenAI Embeddings                   | Generates vector embeddings for text chunks         | Token Splitter1              | Qdrant Vector Store (insert)1 |                                                                                                              |
| Qdrant Vector Store (insert)1 | Langchain Qdrant Vector Store (insert mode)   | Inserts new embeddings into Qdrant collection        | Embeddings OpenAI (build)1, Google Drive: Download1 |                             |                                                                                                              |
| Sticky Note                  | Sticky Note                                   | Overview and summary note for the WooCommerce AI Sales Agent runtime |                              |                               | 🛒 **WooCommerce AI Sales Agent – Overview** This workflow turns n8n into an AI-powered sales assistant...    |
| Sticky Note1                 | Sticky Note                                   | Setup and customization instructions                |                              |                               | ⚙️ **Setup & Customization Notes** Before using this workflow: Connect credentials, adjust localization...   |
| Sticky Note2                 | Sticky Note                                   | Knowledge base update flow explanation               |                              |                               | 📂 **Knowledge Base Update Flow** Purpose: Keeps the AI’s FAQ knowledge accurate and up to date...             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a webhook trigger node** named `When chat message received` using the Langchain ChatTrigger node type, set webhook ID to `"SALES-CHAT-TRIGGER"`.

2. **Add a Set node** `Edit Fields1` connected from the webhook node. Configure it to assign two string fields:  
   - `sessionId` from incoming JSON `sessionId`  
   - `chatInput` from incoming JSON `chatInput`

3. **Add Langchain Information Extractor node** `Sales Info Extractor1` connected from `Edit Fields1`.  
   - Input text: `{{$json.chatInput}}`  
   - Use system prompt to extract buyer intents and fields: product, faq, payment, lead; fields: keyword, sku, min_price, max_price, name, email.  
   - Set input schema manually with these fields.

4. **Add OpenAI Chat Model node** `OpenAI Chat Model (Extractor)1` connected as LLM for extractor.  
   - Model: GPT-4.1-mini  
   - Use OpenAI API credentials.

5. **Add Window Buffer Memory node** `Window Buffer Memory1` connected to maintain session history.  
   - Session key: `{{$node["Edit Fields1"].json["sessionId"]}}`  
   - Context window length: 12 messages.

6. **Add Langchain Agent node** `Sales AI Agent1` connected from `Sales Info Extractor1` and `Window Buffer Memory1`.  
   - Configure system prompt: Role as Sales AI Agent, routing logic (faq → RAG_FAQ, product → WooCommerce search, payment → payment link, lead → CRM lead, else → human escalation).  
   - Define output formatting and next actions.  
   - Link to OpenAI Chat Model (Agent) node for LLM support.

7. **Add OpenAI Chat Model node** `OpenAI Chat Model (Agent)1` connected as LLM for Sales AI Agent.  
   - Model: GPT-4.1-mini  
   - Use OpenAI API credentials.

8. **Add Qdrant Vector Store node** `Qdrant Vector Store (runtime)1` for querying vector data.  
   - Collection: `sales_docs`  
   - Use Qdrant API credentials.

9. **Add Langchain Tool Vector Store node** `RAG_FAQ1` connected from Qdrant runtime vector store.  
   - Name: `RAG_FAQ`  
   - Description: Answer FAQs from vector store.

10. **Add WooCommerce node** `PRODUCT_SEARCH_WOO1` for product search.  
    - Operation: `getAll`  
    - Parameters: sku, search keyword, minPrice, maxPrice, stockStatus "instock" from extractor fields.  
    - Use WooCommerce API credentials.

11. **Add WooCommerce node** `INVENTORY_DETAIL_WOO1` for product inventory details.  
    - Operation: `get`  
    - productId dynamically from AI output.

12. **Add Langchain Tool Workflow node** `PAYMENT_LINK1` for payment link generation.  
    - Link to sub-workflow by ID `"PAYLINK-AGENT-ID"`  
    - No inputs defined.

13. **Add Langchain Tool Workflow node** `CRM_LEAD1` for CRM lead creation/updating.  
    - Link to sub-workflow by ID `"CRM-AGENT-ID"`  
    - No inputs defined.

14. **Add Telegram Tool node** `HUMAN_ESCALATION1` for escalation messages.  
    - Chat text from AI output.  
    - Chat ID: `"SALES-ESCALATION-CHAT"`  
    - Use Telegram API credentials.

15. **Add Google Gemini Chat Model node** `Google Gemini Chat Model1` for RAG FAQ answering.  
    - Use Google PaLM API credentials.

16. **Manually trigger knowledge base update flow:**

    a) Create Manual Trigger node `When clicking 'Execute workflow'`.  
    b) Connect to HTTP Request node `Qdrant Wipe1` configured with POST to Qdrant delete endpoint to wipe collection vectors.  
    c) Connect to Google Drive List node `Google Drive: List1` configured to list files in specific Google Drive folder (folder ID parameterized).  
    d) Connect to Google Drive Download node `Google Drive: Download1` to download and convert Google Docs to plain text.  
    e) Connect to Langchain Document Default Data Loader node `Default Data Loader1` set for binary data.  
    f) Connect to Langchain Text Splitter node `Token Splitter1` with chunk size 300 tokens, overlap 30 tokens.  
    g) Connect to Langchain OpenAI Embeddings node `Embeddings OpenAI (build)1` for embedding generation.  
    h) Connect to Langchain Qdrant Vector Store node `Qdrant Vector Store (insert)1` in insert mode for storing embeddings.

17. **Configure all credentials:**  
    - OpenAI API for GPT-4 and embeddings  
    - WooCommerce API  
    - Qdrant API  
    - Google Drive OAuth2  
    - Stripe credentials (referenced in sub-workflow)  
    - CRM OAuth2 or API credentials (referenced in sub-workflow)  
    - Telegram API credentials  
    - Google PaLM API for Gemini model

18. **Set workflow parameters:**  
    - Google Drive folder ID for sales documents  
    - Webhook URL for chat trigger  
    - Telegram chat ID for escalation  
    - Sub-workflow IDs for payment link and CRM lead workflows  
    - Localization parameters if needed

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                    |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------|
| 🛒 **WooCommerce AI Sales Agent – Overview**: An AI sales assistant that handles intent detection, product search, payment, CRM lead capture, FAQ answering, and human escalation. | Sticky Note node near runtime block               |
| ⚙️ **Setup & Customization Notes**: Connect credentials (WooCommerce, Stripe, CRM, Google Drive, OpenAI, Qdrant, Telegram), adjust localization, update Google Drive folder ID, wipe vector DB when needed, customize agent style. | Sticky Note1 near the start of the workflow       |
| 📂 **Knowledge Base Update Flow**: Manual trigger clears old vectors, downloads sales docs from Google Drive, splits text, generates embeddings, and inserts them into Qdrant for updated FAQ retrieval. | Sticky Note2 near knowledge base update section   |
| Sub-workflow references: `PAYLINK-AGENT-ID` (Stripe payment link generation), `CRM-AGENT-ID` (CRM lead management) must be created separately and linked properly. | See PAYMENT_LINK1 and CRM_LEAD1 nodes              |
| Telegram escalation requires correct chat ID and bot credentials configured for human agents to receive alerts. | HUMAN_ESCALATION1 node                             |

---

This documentation enables understanding, modification, and complete rebuilding of the Conversational Sales Agent workflow integrating GPT-4, WooCommerce, Stripe, CRM, Google Drive, Qdrant, and Telegram within n8n.