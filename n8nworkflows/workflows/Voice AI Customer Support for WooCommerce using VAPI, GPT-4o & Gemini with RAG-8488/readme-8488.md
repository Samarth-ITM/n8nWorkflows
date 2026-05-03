Voice AI Customer Support for WooCommerce using VAPI, GPT-4o & Gemini with RAG

https://n8nworkflows.xyz/workflows/voice-ai-customer-support-for-woocommerce-using-vapi--gpt-4o---gemini-with-rag-8488


# Voice AI Customer Support for WooCommerce using VAPI, GPT-4o & Gemini with RAG

### 1. Workflow Overview

This workflow implements a **Voice AI Customer Support system for WooCommerce** by integrating **VAPI**, **GPT-4o-mini**, **Google Gemini**, and a **Retrieval-Augmented Generation (RAG) system** using vector search (Qdrant) and OpenAI embeddings. It is designed to handle **post-sales customer inquiries**, such as order tracking, order details, and general information retrieval from a knowledge base.

The workflow is structured into two main logical blocks:

- **1.1 Post-Sales AI Agent Block:**  
  Handles customer requests related to WooCommerce orders, customer verification, order tracking, and customer profile information. It uses GPT-4o-mini as the language model and WooCommerce API integrations to fetch data. It also includes verification steps to ensure privacy and security.

- **1.2 RAG (Retrieval-Augmented Generation) Block:**  
  Implements a knowledge retrieval system that queries a vector database (Qdrant) to find relevant contextual information and answers customer questions using Google Gemini. This block is used for general information queries that require retrieval from a knowledge base.

These two blocks are accessible via separate **VAPI webhooks**, allowing external voice applications (e.g., Twilio phone numbers) to interact with the system. The workflow returns structured responses to VAPI, which can be converted to voice outputs.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Post-Sales AI Agent Block

**Overview:**  
This block manages customer support for order-related queries. It verifies customer identity by matching email and order number, retrieves order details, order lists, customer profiles, and tracking information. It uses GPT-4o-mini as the main language model and employs various tools to fetch data from WooCommerce and external APIs.

**Nodes Involved:**  
- Webhook (Webhook)  
- Post-Sales Agent (Langchain Agent)  
- get_order (WooCommerce API)  
- get_orders (WooCommerce API)  
- get_user (WooCommerce API)  
- get_tracking (Sub-Workflow tool)  
- Calculator (Langchain Calculator tool)  
- GPT 4o-mini (OpenAI LLM)  
- Set response (Set node)  
- Send response to VAPI (Respond to Webhook)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Get tracking (HTTP Request)  
- Set tracking code (Set node)  
- Sticky Notes (explanations and instructions)

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (Entry point for post-sales requests)  
  - *Config:* Listens for POST requests at path `f265a558-2787-4f8c-96a1-7b1068e45d3c`  
  - *Role:* Receives JSON payloads from VAPI containing customer email and order number  
  - *Edge Cases:* Invalid or missing payloads may cause failure or empty responses  

- **Post-Sales Agent**  
  - *Type:* Langchain Agent (AI agent for customer support)  
  - *Config:* Uses a detailed system prompt defining role, communication guidelines, tool use, and verification rules  
  - *Tools Used:* `get_order`, `get_orders`, `get_user`, `get_tracking`, `Calculator`  
  - *Language Model:* GPT 4o-mini (connected via `ai_languageModel`)  
  - *Input:* Customer email and order number from webhook  
  - *Output:* Structured textual response for the customer  
  - *Edge Cases:* Email/order mismatch halts data disclosure; agent must handle unknown or ambiguous requests gracefully  

- **get_order**  
  - *Type:* WooCommerce API tool (get single order)  
  - *Config:* Retrieves order details by order ID from AI input `Order_ID`  
  - *Credentials:* WooCommerce API (magnanigioielli.com)  
  - *Edge Cases:* Order not found, invalid order ID, API errors, or authentication issues  

- **get_orders**  
  - *Type:* WooCommerce API tool (get all orders for a customer)  
  - *Config:* Search parameter set from AI input `Search`; can return all or limited results based on AI input `Return_All`  
  - *Credentials:* WooCommerce API  
  - *Edge Cases:* Empty search results or API failures  

- **get_user**  
  - *Type:* WooCommerce API tool (get customer by email)  
  - *Config:* Filters by email from AI input `Email`  
  - *Credentials:* WooCommerce API  
  - *Edge Cases:* Customer not found, multiple matches, or API errors  

- **get_tracking**  
  - *Type:* Sub-Workflow tool  
  - *Config:* Calls internal workflow `get_tracking` by passing `order_number` parameter  
  - *Role:* Retrieves tracking code information for an order  
  - *Edge Cases:* Tracking info missing, order number invalid, sub-workflow failures  

- **Calculator**  
  - *Type:* Langchain Tool Calculator  
  - *Config:* Available for numerical or calculation tasks within AI agent  
  - *Role:* Assists agent with math-related queries  
  - *Edge Cases:* Calculation errors or invalid inputs  

- **GPT 4o-mini**  
  - *Type:* OpenAI Chat LLM node  
  - *Config:* Model `gpt-4o-mini` used as language model for the Post-Sales agent  
  - *Credentials:* OpenAI API  
  - *Edge Cases:* API rate limits, timeouts, or failed completions  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Config:* Accepts JSON input with `order_number` to trigger tracking info retrieval  
  - *Role:* Allows external workflows to call the tracking retrieval process  

- **Get tracking**  
  - *Type:* HTTP Request  
  - *Config:* Calls WooCommerce REST API endpoint `/orders/{order_number}` with Basic Auth credentials to retrieve order metadata  
  - *Credentials:* HTTP Basic Auth (WooCommerce shop)  
  - *Edge Cases:* Incorrect URL, authentication failure, network errors  

- **Set tracking code**  
  - *Type:* Set node  
  - *Config:* Extracts tracking code, carrier URL, and pick-up date from order metadata fields (`ywot_tracking_code`, `ywot_carrier_url`, `ywot_pick_up_date`)  
  - *Role:* Prepares structured tracking info for response  

- **Set response**  
  - *Type:* Set node  
  - *Config:* Assigns the AI agent's output text to the field `message` for webhook response  

- **Send response to VAPI**  
  - *Type:* Respond to Webhook  
  - *Config:* Returns the prepared response message back to VAPI for voice synthesis  

- **Sticky Notes**  
  - Provide important contextual instructions, e.g.:  
    - Use of YITH WooCommerce Order & Shipment Tracking plugin for tracking codes  
    - Setup instructions for VAPI API Requests and Wizard  
    - High-level explanation of the entire workflow and integration  

---

#### 2.2 RAG (Retrieval-Augmented Generation) Block

**Overview:**  
This block handles general information queries by retrieving relevant data from a vector database (Qdrant) using OpenAI embeddings and answering questions using Google Gemini chat model. It is designed for knowledge-base style Q&A beyond direct order details.

**Nodes Involved:**  
- rag (Webhook)  
- Question and Answer Chain (Langchain Retrieval QA Chain)  
- Vector Store Retriever (Retriever node)  
- Qdrant Vector Store1 (Vector store node)  
- Embeddings OpenAI (Embeddings generation)  
- Google Gemini Chat Model (Google PaLM model)  
- Send RAG response to VAPI (Respond to Webhook)  
- Sticky Note2 (RAG system reference)

**Node Details:**

- **rag**  
  - *Type:* Webhook  
  - *Config:* POST webhook at path `5ca93cd6-7f7b-4c91-acff-c324e594cca7` receiving JSON with a `search` field  
  - *Role:* Entry point for RAG queries from VAPI  

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings generation  
  - *Role:* Converts input text into vector embeddings for similarity search  

- **Qdrant Vector Store1**  
  - *Type:* Vector store connection to Qdrant  
  - *Config:* Uses collection `negozio-emporio-verde`  
  - *Credentials:* Qdrant API (Hetzner)  
  - *Role:* Stores and retrieves document vectors for similarity search  

- **Vector Store Retriever**  
  - *Type:* Retriever node  
  - *Config:* Retrieves top 5 similar documents based on embeddings query  
  - *Role:* Fetches relevant context to answer the user query  

- **Question and Answer Chain**  
  - *Type:* Langchain Retrieval QA Chain  
  - *Config:* Uses a system prompt that instructs the assistant to answer based on retrieved context, with explicit instructions not to fabricate answers  
  - *Language Model:* Google Gemini Chat Model  
  - *Input:* Text query from webhook `search` field  
  - *Output:* Contextual answer based on vector search results  

- **Google Gemini Chat Model**  
  - *Type:* Google PaLM language model  
  - *Config:* Model `models/gemini-1.5-flash`  
  - *Credentials:* Google Palm API (Eure)  
  - *Edge Cases:* API limits or failures  

- **Send RAG response to VAPI**  
  - *Type:* Respond to Webhook  
  - *Config:* Sends generated answer back to VAPI for voice response  

- **Sticky Note2**  
  - Provides reference to a detailed workflow for building a self-updating RAG system with OpenAI, Google Gemini, Qdrant, and Google Drive:  
    https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/  

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                                | Input Node(s)                   | Output Node(s)             | Sticky Note                                                                                                             |
|------------------------------|-----------------------------------------|------------------------------------------------|---------------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Webhook                      | Webhook                                 | Entry point for post-sales requests            | -                               | Post-Sales Agent            | ## Post-sale Agent: Get a WooCommerce user's orders and shipping tracking                                                |
| Post-Sales Agent             | Langchain Agent                         | AI agent handling order queries and verification | Webhook                         | Set response                |                                                                                                                         |
| get_order                   | WooCommerce API Tool                    | Fetch single order details                      | Post-Sales Agent (ai_tool)       | Post-Sales Agent (ai_tool)  |                                                                                                                         |
| get_orders                  | WooCommerce API Tool                    | Fetch multiple orders for a customer            | Post-Sales Agent (ai_tool)       | Post-Sales Agent (ai_tool)  |                                                                                                                         |
| get_user                    | WooCommerce API Tool                    | Fetch customer profile by email                  | Post-Sales Agent (ai_tool)       | Post-Sales Agent (ai_tool)  |                                                                                                                         |
| get_tracking                | Sub-Workflow Tool                      | Retrieve tracking info for an order              | Post-Sales Agent (ai_tool)       | Post-Sales Agent (ai_tool)  |                                                                                                                         |
| Calculator                  | Langchain Tool Calculator              | Perform calculations if needed                    | Post-Sales Agent (ai_tool)       | Post-Sales Agent (ai_tool)  |                                                                                                                         |
| GPT 4o-mini                 | OpenAI LLM                            | Language model for post-sales AI agent           | Post-Sales Agent (ai_languageModel) | Post-Sales Agent (ai_languageModel) |                                                                                                                         |
| Set response                | Set Node                              | Prepare AI response message for webhook          | Post-Sales Agent (main)          | Send response to VAPI       |                                                                                                                         |
| Send response to VAPI       | Respond to Webhook                    | Return response to VAPI for voice synthesis      | Set response (main)              | -                          |                                                                                                                         |
| When Executed by Another Workflow | Execute Workflow Trigger              | Allow external workflows to trigger tracking retrieval | -                               | Get tracking                |                                                                                                                         |
| Get tracking                | HTTP Request                         | WooCommerce API call to get order tracking metadata | When Executed by Another Workflow | Set tracking code           | The tracking code request uses the "YITH WooCommerce Order & Shipment Tracking" plugin. Free version: https://wordpress.org/plugins/yith-woocommerce-order-tracking/ |
| Set tracking code           | Set Node                              | Extract tracking code, carrier URL, pick-up date | Get tracking (main)              | -                          |                                                                                                                         |
| rag                        | Webhook                                 | Entry point for RAG knowledge-base queries       | -                               | Question and Answer Chain   | ## General info: To build the RAG system please view https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/ |
| Embeddings OpenAI           | OpenAI Embeddings                      | Generate embeddings for vector search             | -                               | Qdrant Vector Store1        |                                                                                                                         |
| Qdrant Vector Store1        | Vector Store Qdrant                   | Store and retrieve document vectors               | Embeddings OpenAI (ai_embedding) | Vector Store Retriever      |                                                                                                                         |
| Vector Store Retriever      | Retriever Node                       | Retrieve top K similar documents                   | Qdrant Vector Store1 (ai_vectorStore) | Question and Answer Chain   |                                                                                                                         |
| Question and Answer Chain   | Langchain Retrieval QA Chain          | Retrieve relevant context and answer questions    | rag (main), Vector Store Retriever (ai_retriever), Google Gemini Chat Model (ai_languageModel) | Send RAG response to VAPI   |                                                                                                                         |
| Google Gemini Chat Model    | Google PaLM LLM                      | Language model for RAG question answering          | Question and Answer Chain (ai_languageModel) | Question and Answer Chain (ai_languageModel) |                                                                                                                         |
| Send RAG response to VAPI   | Respond to Webhook                    | Send RAG answer back to VAPI for voice output      | Question and Answer Chain (main) | -                          |                                                                                                                         |
| Sticky Note1                | Sticky Note                           | Explains use of YITH WooCommerce tracking plugin and setup instructions | -                               | -                          | The tracking code request is made through the popular WooCommerce tracking plugin: "YITH WooCommerce Order & Shipment Tracking". Free version: https://wordpress.org/plugins/yith-woocommerce-order-tracking/ |
| Sticky Note2                | Sticky Note                           | RAG system reference link and description          | -                               | -                          | General info - Build the RAG system: https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/ |
| Sticky Note3                | Sticky Note                           | Step-by-step instructions for VAPI API Request and Wizard setup | -                               | -                          | Instructions for creating API Request tools and wizard in VAPI, and linking Twilio phone numbers                        |
| Sticky Note4                | Sticky Note                           | High-level summary of workflow purpose and integrations | -                               | -                          | Build AI Voice Assistant with VAPI & Twilio for WooCommerce Post-Sales & RAG integration                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook (Post-Sales Entry):**  
   - Node type: Webhook  
   - HTTP Method: POST  
   - Path: `f265a558-2787-4f8c-96a1-7b1068e45d3c`  
   - Purpose: Receive customer post-sales requests with JSON payload containing `email` and `n_order` (order number).

2. **Create Langchain Agent (Post-Sales Agent):**  
   - Node type: Langchain Agent  
   - Input: Text with customer email and order number from webhook JSON body  
   - System prompt:  
     - Define role as post-sales support for WooCommerce clothing store  
     - Include strict verification rules requiring exact email/order match  
     - Define tools usage (get_order, get_orders, get_user, get_tracking, Calculator)  
     - Communication style: professional, clear, direct  
   - Language Model: Connect GPT 4o-mini node as LLM  

3. **Add WooCommerce API Nodes:**  
   - `get_order`: Get single order by Order_ID from AI input  
   - `get_orders`: Get all orders matching search criteria from AI input  
   - `get_user`: Get customer data by email from AI input  
   - Credentials: Setup WooCommerce API credentials (store URL, consumer key/secret)  

4. **Add get_tracking Sub-Workflow Tool:**  
   - Create separate workflow to retrieve tracking data (see steps below)  
   - Add `get_tracking` node as tool in Post-Sales Agent, passing `order_number` parameter  

5. **Create Calculator Tool Node:**  
   - Add Langchain Calculator tool for number calculations  

6. **Add GPT 4o-mini Node:**  
   - Node type: Langchain OpenAI Chat  
   - Model: `gpt-4o-mini` (ensure OpenAI API credentials configured)  

7. **Set Response Node:**  
   - Node type: Set  
   - Assign AI output text to field `message` for webhook response  

8. **Respond to Webhook Node:**  
   - Node type: Respond to Webhook  
   - Return the `message` field as JSON response to VAPI  

9. **Create get_tracking Sub-Workflow:**  
   - Start with Execute Workflow Trigger node accepting `order_number` JSON parameter  
   - Add HTTP Request node to WooCommerce REST API endpoint:  
     `https://YOUR_SHOP_URL/wp-json/wc/v3/orders/{{ $json.order_number }}`  
   - Authentication: HTTP Basic Auth with WooCommerce credentials  
   - Add Set node to extract tracking metadata fields:  
     - `tracking_code` from meta key `ywot_tracking_code`  
     - `carrier_url` from meta key `ywot_carrier_url`  
     - `pick_up` from meta key `ywot_pick_up_date`  
   - Return extracted tracking info as JSON  

10. **Create Webhook (RAG Entry):**  
    - Node type: Webhook  
    - HTTP Method: POST  
    - Path: `5ca93cd6-7f7b-4c91-acff-c324e594cca7`  
    - Purpose: Receive general knowledge queries with JSON containing `search` field  

11. **Add RAG Nodes:**  
    - Embeddings OpenAI: generate embeddings from query text  
    - Qdrant Vector Store1: connect to Qdrant collection `negozio-emporio-verde` with API credentials  
    - Vector Store Retriever: retrieve top 5 documents based on embeddings  
    - Google Gemini Chat Model Node:  
      - Model: `models/gemini-1.5-flash`  
      - Credentials: Google Palm API credentials  
    - Question and Answer Chain:  
      - Use Langchain Retrieval QA chain  
      - System prompt instructs to answer based only on retrieved context, no hallucination  
      - Connect Retriever and Google Gemini as inputs  
    - Respond to Webhook: send generated answer back to VAPI  

12. **Connect Nodes in Both Blocks:**  
    - Post-Sales Webhook → Post-Sales Agent → Set Response → Respond to Webhook  
    - RAG Webhook → Embeddings → Qdrant → Retriever → QA Chain → Respond to Webhook  

13. **Add Sticky Notes:**  
    - Document important setup instructions, plugin usage, and integration tips per original sticky notes  

14. **Credential Setup:**  
    - WooCommerce API: store URL, consumer key, consumer secret  
    - OpenAI API: for GPT 4o-mini and embeddings  
    - Google Palm API: for Google Gemini model  
    - HTTP Basic Auth: WooCommerce shop credentials for tracking HTTP Request  

15. **VAPI & Twilio Setup (outside n8n):**  
    - Create two API Request tools in VAPI for WooCommerce and RAG, linked to respective webhook URLs  
    - Create a wizard combining these tools with voice configuration and assign Twilio inbound phone number  

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| The tracking code request is made through the popular WooCommerce tracking plugin: "YITH WooCommerce Order & Shipment Tracking". The free version can be downloaded here: https://wordpress.org/plugins/yith-woocommerce-order-tracking/ | Sticky Note1                                                                                                                            |
| For building the RAG system, see this example workflow demonstrating a self-updating RAG with OpenAI, Google Gemini, Qdrant, and Google Drive: https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/ | Sticky Note2                                                                                                                            |
| Steps to create API Request tools in VAPI and set up wizard including voice and Twilio phone number association are detailed in Sticky Note3. | Sticky Note3                                                                                                                            |
| Workflow integrates a post-sales AI agent with RAG system connected to VAPI webhooks and Twilio phone numbers for voice AI customer support. | Sticky Note4                                                                                                                            |

---

**Disclaimer:** The provided text is derived solely from an automated n8n workflow, compliant with content policies and handling only legal and public data.