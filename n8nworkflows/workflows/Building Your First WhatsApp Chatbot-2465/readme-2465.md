Building Your First WhatsApp Chatbot

https://n8nworkflows.xyz/workflows/building-your-first-whatsapp-chatbot-2465


# Building Your First WhatsApp Chatbot

### 1. Workflow Overview

This workflow builds a simple WhatsApp chatbot acting as a Sales Agent for Yamaha Powered Loudspeakers 2024. It leverages a product catalog vector store to provide factual answers to user queries via WhatsApp messages.

The workflow is logically divided into two main parts:

**1.1 Product Catalog Vector Store Creation**  
- Downloads a product brochure PDF via HTTP request  
- Extracts text content from the PDF  
- Splits the text into chunks  
- Converts text chunks into embeddings using OpenAI embeddings  
- Stores embeddings into an in-memory vector store acting as a searchable knowledgebase  

**1.2 WhatsApp AI Chatbot Interaction**  
- Listens for incoming WhatsApp messages (text only)  
- Filters out non-text messages with a polite reply  
- Uses an AI Sales Agent (powered by OpenAI GPT-4) that accesses the vector store tool and maintains conversation memory  
- Sends the AI-generated response back to the WhatsApp user  

This structure enables a conversational AI agent that uses a product brochure as its knowledge source to answer questions effectively.

---

### 2. Block-by-Block Analysis

#### 2.1 Product Catalog Vector Store Creation

**Overview:**  
This block downloads the Yamaha product brochure PDF, extracts its text, splits it into manageable chunks, converts these chunks into embeddings, and stores them in an in-memory vector store. This vector store serves as the knowledge base for the chatbot.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- get Product Brochure (HTTP Request)  
- Extract from File (Extract text from PDF)  
- Recursive Character Text Splitter (Splits extracted text)  
- Default Data Loader (Loads text chunks as documents)  
- Embeddings OpenAI1 (Generates embeddings for chunks)  
- Create Product Catalogue (Inserts embeddings into in-memory vector store)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the vector store creation manually (run once or to update)  
  - Connections: Triggers `get Product Brochure`  
  - Edge cases: User must run this explicitly; no automatic scheduling  

- **get Product Brochure**  
  - Type: HTTP Request  
  - Configuration: Fetches Yamaha Powered Loudspeakers 2024 brochure PDF from a public URL  
  - Outputs raw PDF binary content  
  - Connected to: `Extract from File`  
  - Edge cases: HTTP errors, URL changes, network issues  

- **Extract from File**  
  - Type: Extract from File (PDF operation)  
  - Configuration: Extracts text from the PDF file content provided by the previous node  
  - Outputs extracted text in JSON format  
  - Connected to: `Create Product Catalogue` (via `Default Data Loader` and splitter)  
  - Edge cases: Unsupported PDF format, extraction errors  

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Configuration: Splits the extracted text into chunks of size 2000 characters with no overlap  
  - Purpose: Enables manageable chunk size for embeddings and vector store  
  - Connected to: `Default Data Loader`  

- **Default Data Loader**  
  - Type: Document Loader  
  - Configuration: Loads split text chunks as documents for embedding generation  
  - Input: Expression referencing text from `Extract from File`  
  - Connected to: `Create Product Catalogue`  

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Configuration: Uses `text-embedding-3-small` model to create vector embeddings for documents  
  - Credentials: OpenAI API credentials required  
  - Connected to: `Create Product Catalogue`  

- **Create Product Catalogue**  
  - Type: Vector Store In-Memory (Insert mode)  
  - Configuration: Inserts embeddings into an in-memory vector store with key `whatsapp-75`  
  - Clears existing vector store to refresh data on each run  
  - Output: Populated vector store for querying  
  - Edge cases: Memory limitations, embedding generation failures  

---

#### 2.2 WhatsApp AI Chatbot Interaction

**Overview:**  
This block listens for incoming WhatsApp messages, filters out unsupported message types, invokes an AI Sales Agent that queries the vector store with conversation memory to generate responses, and replies to the user via WhatsApp.

**Nodes Involved:**  
- WhatsApp Trigger  
- Handle Message Types (Switch)  
- Reply To User1 (Reply to unsupported message types)  
- AI Sales Agent (Langchain Agent)  
- OpenAI Chat Model  
- Window Buffer Memory  
- Vector Store Tool  
- Product Catalogue (Vector Store In-Memory)  
- Reply To User (Send AI response)  

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp Trigger  
  - Configuration: Listens to incoming WhatsApp messages (messages update event)  
  - Credentials: WhatsApp OAuth account required  
  - Output: Incoming WhatsApp message JSON  
  - Edge cases: Webhook misconfiguration, WhatsApp API limits, message type variations  
  - Connected to: `Handle Message Types`  

- **Handle Message Types**  
  - Type: Switch  
  - Configuration: Checks if incoming message type equals `"text"`  
  - Outputs:  
    - `Supported` for text messages (routes to AI Sales Agent)  
    - `Not Supported` for other message types (routes to polite reply)  
  - Edge cases: Unexpected or new message types, expression evaluation failures  

- **Reply To User1**  
  - Type: WhatsApp (Send message)  
  - Configuration: Sends a fixed text reply: "I'm unable to process non-text messages. Please send only text messages. Thanks!"  
  - Credentials: WhatsApp API credentials  
  - Input: From `Handle Message Types` Not Supported output  
  - Edge cases: WhatsApp sending failures, invalid phone numbers  

- **AI Sales Agent**  
  - Type: Langchain Agent  
  - Configuration:  
    - Input text: User's WhatsApp message text  
    - System message instructs assistant to help users navigate Yamaha product catalog 2024, answer factually, and redirect sales inquiries appropriately  
    - Uses:  
      - `OpenAI Chat Model` (GPT-4o-2024-08-06) as LLM  
      - `Window Buffer Memory` keyed by user phone for conversation context  
      - `Vector Store Tool` connected to `Product Catalogue` vector store for knowledge access  
  - Credentials: OpenAI API required  
  - Edge cases: API timeouts, memory key collisions, vector store lookup failures  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Configuration: GPT-4o model, no special options  
  - Credentials: OpenAI API  
  - Connected as languageModel to `AI Sales Agent` and `Vector Store Tool`  

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer Window  
  - Configuration: Uses session key format `whatsapp-75-<userPhone>` to maintain conversation state per user  
  - Connected as memory to `AI Sales Agent`  

- **Vector Store Tool**  
  - Type: Langchain Vector Store Tool  
  - Configuration: Named `query_product_brochure` with description about usage in 2024  
  - Connected as tool to `AI Sales Agent`  
  - Input from `Product Catalogue` vector store  

- **Product Catalogue**  
  - Type: Vector Store In-Memory  
  - Configuration: Uses memory key `whatsapp-75` matching the creation block  
  - Connected as vector store source for `Vector Store Tool`  

- **Reply To User**  
  - Type: WhatsApp (Send message)  
  - Configuration: Sends AI agent's response text back to the user  
  - Uses dynamic expression to get output from `AI Sales Agent`  
  - Credentials: WhatsApp API  
  - Connected to: Output of `AI Sales Agent`  

---

### 3. Summary Table

| Node Name                    | Node Type                                    | Functional Role                                | Input Node(s)                | Output Node(s)               | Sticky Note                                                  |
|------------------------------|----------------------------------------------|------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                              | Starts vector store creation manually          |                             | get Product Brochure          | You only have to run this part once! Run to populate vector. |
| get Product Brochure          | HTTP Request                                | Downloads product brochure PDF                  | When clicking ‘Test workflow’| Extract from File             | See HTTP Request tool docs; imports product brochure PDF.    |
| Extract from File             | Extract from File (PDF)                      | Extracts text from PDF                           | get Product Brochure          | Create Product Catalogue via Default Data Loader | Extracts text for vector store creation.                      |
| Recursive Character Text Splitter | Text Splitter                           | Splits extracted text into chunks               |                             | Default Data Loader          |                                                              |
| Default Data Loader           | Document Loader                             | Loads text chunks as documents                   | Recursive Character Text Splitter | Create Product Catalogue   |                                                              |
| Embeddings OpenAI1            | OpenAI Embeddings                           | Creates embeddings for document chunks           | Default Data Loader           | Create Product Catalogue     | Uses OpenAI embeddings model for vector store.               |
| Create Product Catalogue      | Vector Store In-Memory                      | Inserts embeddings into in-memory vector store   | Embeddings OpenAI1, Default Data Loader, Recursive Character Text Splitter, Extract from File |                             | Uses memory key whatsapp-75; clears previous data.           |
| WhatsApp Trigger             | WhatsApp Trigger                            | Receives incoming WhatsApp messages              |                             | Handle Message Types         | Filters incoming WhatsApp messages.                           |
| Handle Message Types          | Switch                                     | Routes based on message type (text or not)       | WhatsApp Trigger             | AI Sales Agent, Reply To User1 | Filters text messages; replies to unsupported types.         |
| Reply To User1               | WhatsApp (Send message)                     | Replies to unsupported (non-text) WhatsApp messages | Handle Message Types (Not Supported) |                             | Replies with polite message for non-text inputs.             |
| AI Sales Agent               | Langchain Agent                            | AI chatbot agent; queries vector store & memory | Handle Message Types (Supported) | Reply To User             | Uses vector store tool and conversation memory for replies.  |
| OpenAI Chat Model            | Langchain OpenAI Chat Model                 | Provides GPT-4o model for AI agent and tool      |                             | AI Sales Agent, Vector Store Tool | GPT-4o model for chat completions.                            |
| Window Buffer Memory         | Langchain Memory Buffer Window              | Maintains per-user session conversation memory   |                             | AI Sales Agent              | Session key format whatsapp-75-<userPhone>.                   |
| Vector Store Tool            | Langchain Vector Store Tool                  | Queries product brochure vector store             | Product Catalogue            | AI Sales Agent              | Named tool for querying vector store knowledge base.          |
| Product Catalogue            | Vector Store In-Memory                      | In-memory vector store holding product brochure   | Create Product Catalogue     | Vector Store Tool           | Keyed by whatsapp-75; linked to vector store tool.            |
| Reply To User                | WhatsApp (Send message)                     | Sends AI agent's response back to WhatsApp user   | AI Sales Agent               |                             | Sends chatbot reply text to user.                             |
| Sticky Note                  | Sticky Note                                 | Provides information and instructions             |                             |                             | See notes on downloading brochure, vector store, WhatsApp, AI agent, etc. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in n8n manually:

**Part 1: Product Catalog Vector Store Creation**

1. Create a **Manual Trigger** node named `When clicking ‘Test workflow’`.

2. Add an **HTTP Request** node named `get Product Brochure`:  
   - Method: GET  
   - URL: `https://usa.yamaha.com/files/download/brochure/1/1474881/Yamaha-Powered-Loudspeakers-brochure-2024-en-web.pdf`  
   - No special authentication needed (public URL)  
   - Connect `When clicking ‘Test workflow’` → `get Product Brochure`

3. Add an **Extract from File** node named `Extract from File`:  
   - Operation: PDF  
   - Input: Data from `get Product Brochure` node  
   - Connect `get Product Brochure` → `Extract from File`

4. Add a **Recursive Character Text Splitter** node named `Recursive Character Text Splitter`:  
   - Chunk Size: 2000 characters  
   - Chunk Overlap: 0  
   - Connect this node as a text splitter input (used later in data loader)

5. Add a **Default Data Loader** node named `Default Data Loader`:  
   - JSON Data: Expression pointing to `Extract from File` extracted text  
   - JSON Mode: Expression Data  
   - Connect `Recursive Character Text Splitter` → `Default Data Loader`

6. Add an **Embeddings OpenAI** node named `Embeddings OpenAI1`:  
   - Model: `text-embedding-3-small`  
   - Set OpenAI API credentials  
   - Connect `Default Data Loader` → `Embeddings OpenAI1`

7. Add a **Vector Store In-Memory** node named `Create Product Catalogue`:  
   - Mode: Insert  
   - Memory Key: `whatsapp-75`  
   - Clear Store: true (to refresh data on each run)  
   - Connect `Embeddings OpenAI1` → `Create Product Catalogue`

**Part 2: WhatsApp AI Chatbot Interaction**

8. Add a **WhatsApp Trigger** node named `WhatsApp Trigger`:  
   - Configure webhook for WhatsApp messages (updates: messages)  
   - Set WhatsApp OAuth credentials  
   - Connect to next logic node

9. Add a **Switch** node named `Handle Message Types`:  
   - Condition: Check if `{{$json["messages"][0]["type"]}}` equals `text`  
   - Outputs:  
     - Supported (text messages)  
     - Not Supported (all others)  
   - Connect `WhatsApp Trigger` → `Handle Message Types`

10. Add a **WhatsApp** node named `Reply To User1`:  
    - Operation: Send  
    - Text Body: `"I'm unable to process non-text messages. Please send only text messages. Thanks!"`  
    - Use WhatsApp API credentials  
    - Connect `Handle Message Types` Not Supported output → `Reply To User1`

11. Add **OpenAI Chat Model** node named `OpenAI Chat Model`:  
    - Model: `gpt-4o-2024-08-06`  
    - Set OpenAI API credentials  

12. Add **Window Buffer Memory** node named `Window Buffer Memory`:  
    - Session Key: Expression `whatsapp-75-{{$json["messages"][0]["from"]}}`  
    - Session Id Type: Custom Key  

13. Add **Vector Store In-Memory** node named `Product Catalogue`:  
    - Memory Key: `whatsapp-75` (same as created earlier)  

14. Add **Vector Store Tool** node named `Vector Store Tool`:  
    - Name: `query_product_brochure`  
    - Description: `Call this tool to query the product brochure. Valid for the year 2024.`  
    - Connect `Product Catalogue` → `Vector Store Tool`  

15. Add **Langchain Agent** node named `AI Sales Agent`:  
    - Text Input: Expression `{{$json["messages"][0]["text"]["body"]}}`  
    - System Message:  
      ```
      You are an assistant working for a company who sells Yamaha Powered Loudspeakers and helping the user navigate the product catalog for the year 2024. Your goal is not to facilitate a sale but if the user enquires, direct them to the appropriate website, url or contact information.

      Do your best to answer any questions factually. If you don't know the answer or unable to obtain the information from the datastore, then tell the user so.
      ```  
    - Prompt Type: Define  
    - Connect:  
      - Language Model: `OpenAI Chat Model`  
      - Memory: `Window Buffer Memory`  
      - Tool: `Vector Store Tool`  
    - Connect `Handle Message Types` Supported output → `AI Sales Agent`

16. Add a **WhatsApp** node named `Reply To User`:  
    - Operation: Send  
    - Text Body: Expression `{{$json["output"]}}` (output of AI Sales Agent)  
    - Use WhatsApp API credentials  
    - Connect `AI Sales Agent` → `Reply To User`

**Final connections summary:**  
- `When clicking ‘Test workflow’` → `get Product Brochure` → `Extract from File` → `Recursive Character Text Splitter` → `Default Data Loader` → `Embeddings OpenAI1` → `Create Product Catalogue`  
- `WhatsApp Trigger` → `Handle Message Types`  
  - Supported → `AI Sales Agent` → `Reply To User`  
  - Not Supported → `Reply To User1`

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This n8n template builds a simple WhatsApp chatbot acting as a Sales Agent, backed by a product catalog vector store.             | Workflow description                                                                              |
| [Read more about the HTTP Request Tool](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)          | Product brochure download node                                                                   |
| [Read more about the In-Memory Vector Store](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory/) | Vector store explanation                                                                          |
| [Learn more about the WhatsApp Trigger](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.whatsapptrigger/)   | WhatsApp Trigger documentation                                                                   |
| [Learn more about using AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)   | AI Sales Agent node documentation                                                                |
| [Learn more about the WhatsApp Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.whatsapp/)                 | WhatsApp send message node documentation                                                         |
| Want to handle all WhatsApp message types? Check out the creator’s other WhatsApp template: https://n8n.io/creators/jimleuk/         | Sticky note on expanding message type handling                                                   |
| Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/) for help and community | Support resources                                                                                |
| For production use, upgrade vector store to persistent options like [Qdrant](https://qdrant.tech) or [Pinecone](https://pinecone.io) | Recommendations for scaling vector store                                                         |
| WhatsApp Business Account and OpenAI API credentials are required to operate this workflow.                                        | Workflow prerequisites                                                                           |

---

This reference document fully describes the workflow structure, node configurations, and reproduction instructions to allow advanced users and AI agents to understand, modify, or replicate the WhatsApp AI Sales Agent chatbot powered by a product catalog vector store.