Website Content Chatbot with Pinecone, Airtable & OpenAI for RAG Applications

https://n8nworkflows.xyz/workflows/website-content-chatbot-with-pinecone--airtable---openai-for-rag-applications-7966


# Website Content Chatbot with Pinecone, Airtable & OpenAI for RAG Applications

---

## 1. Workflow Overview

This workflow, titled **"Website Content Chatbot with Pinecone, Airtable & OpenAI for RAG Applications"**, is designed to build a chatbot that leverages website content and Airtable data to power Retrieval-Augmented Generation (RAG) applications. Its core purpose is to automatically extract, process, and index website content, and combine it with live billing data from Airtable to provide accurate, context-aware answers to customer queries via a chat interface.

The workflow is structured into the following logical blocks:

- **1.1 Content Extraction and Processing**: Fetch website content, extract and normalize HTML body, convert to Markdown, split text into chunks for embedding.
- **1.2 Embedding Generation and Vector Storage**: Generate OpenAI embeddings for text chunks and store them in Pinecone vector database under a project namespace.
- **1.3 Chat Interface and Query Handling**: Triggered by chat messages, this block includes chat agents that query Pinecone and Airtable, use memory buffering for multi-turn conversations, and route queries to appropriate AI models/tools.
- **1.4 Billing Data Integration**: Uses Airtable as a dynamic billing/payment data source accessed via a specialized billing tool.
- **1.5 AI Model and Tool Management**: Manages different AI models and tools for cost-effective and efficient query handling, including OpenAI GPT-4o-mini and OpenRouter’s o4-mini models.

---

## 2. Block-by-Block Analysis

### 2.1 Content Extraction and Processing

**Overview:**  
This block fetches website content via HTTP, extracts the HTML `<body>`, converts it to Markdown, normalizes the text to remove noise and duplication, and splits it into chunks for embedding.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request  
- Exctract HTML Body  
- Markdown  
- Normalize Text  
- Character Text Splitter  
- Default Data Loader1

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually for testing or data update.  
  - *Input:* None  
  - *Output:* Triggers HTTP Request node.  
  - *Failures:* None typical, but manual execution required.

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Downloads webpage content from a specified URL.  
  - *Configuration:* URL set to `https://content.eirevo.ie/billing-change-faqs` (replaceable).  
  - *Output:* Full HTML response under `.data`.  
  - *Failures:* Network errors, HTTP errors (404, 500).  
  - *Notes:* Sticky Note4 reminds to replace URL for your own content source.

- **Exctract HTML Body**  
  - *Type:* Set Node (Data extraction via regex)  
  - *Role:* Extracts the `<body>` content from HTML using regex match.  
  - *Configuration:* Uses expression `={{ $json?.data.match(/<body[^>]*>([\s\S]*?)<\/body>/i)[1] }}` to isolate body.  
  - *Input:* HTTP Request output with full HTML.  
  - *Output:* `.data` contains the extracted HTML body.  
  - *Failures:* If regex fails (malformed HTML), will error or return undefined.  
  - *Edge:* Should handle missing or invalid body tags gracefully.

- **Markdown**  
  - *Type:* Markdown Node  
  - *Role:* Converts extracted HTML body to clean Markdown format.  
  - *Input:* `.data` from previous node (HTML body).  
  - *Output:* Markdown text.  
  - *Failures:* Errors if HTML is malformed or empty.

- **Normalize Text**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Cleans Markdown text by removing noise such as "Page X" markers, multiple newlines, and trims whitespace.  
  - *Key Code:* Removes page markers, replaces multiple newlines with space, trims result.  
  - *Input:* Markdown output (`.data`).  
  - *Output:* `.normalizedText` normalized string.  
  - *Failures:* Expression errors if input missing or malformed.  
  - *Sticky Note6:* Explains rationale for text normalization to improve retrieval quality.

- **Character Text Splitter**  
  - *Type:* LangChain Character Text Splitter  
  - *Role:* Splits normalized text into chunks (~500 characters with 50 character overlap), using `######` as separator.  
  - *Configuration:* `chunkSize=500`, `chunkOverlap=50`, `separator="######"`.  
  - *Input:* Normalized text.  
  - *Output:* Array of text chunks for embedding.  
  - *Sticky Note1* advises adjusting chunk size and checking Markdown output.

- **Default Data Loader1**  
  - *Type:* LangChain Default Data Loader  
  - *Role:* Loads chunks for downstream embedding generation, set to custom text splitting mode (linked to previous splitter).  
  - *Input:* Output from Text Splitter.  
  - *Output:* Documents ready for embedding.  
  - *Failures:* Misconfiguration of splitting mode may cause empty or overlapping data.

---

### 2.2 Embedding Generation and Vector Storage

**Overview:**  
Generates embeddings from text chunks using OpenAI embeddings and stores them in Pinecone vector database under a namespace for semantic search.

**Nodes Involved:**  
- Generate Embeddings1  
- Store in Pinecone1  
- Normalize Text (as input)  
- Embeddings OpenAI  
- Pinecone Store (Q&A)

**Node Details:**

- **Generate Embeddings1**  
  - *Type:* LangChain OpenAI Embeddings Node  
  - *Role:* Creates 1536-dimensional embeddings for each text chunk.  
  - *Credentials:* Uses OpenAI API credentials named `nextweb-openai`.  
  - *Input:* Text chunks from Default Data Loader1.  
  - *Output:* Embeddings array for each chunk.  
  - *Failures:* API key errors, rate limits, invalid inputs.

- **Store in Pinecone1**  
  - *Type:* LangChain Pinecone Vector Store  
  - *Role:* Inserts embeddings with metadata into Pinecone under namespace `"eirevo"`.  
  - *Credentials:* Pinecone API credentials `learnby-PineconeApi-account`.  
  - *Input:* Embeddings and documents.  
  - *Output:* Confirmation of insert success.  
  - *Failures:* API key errors, network issues, namespace mismatch.

- **Embeddings OpenAI**  
  - *Type:* Another embeddings node (used in Q&A path)  
  - *Role:* Generates query embeddings for chat queries.  
  - *Credentials:* Same OpenAI API.  
  - *Input:* User query text.  
  - *Output:* Query embeddings.  
  - *Failures:* Same as above.

- **Pinecone Store (Q&A)**  
  - *Type:* Pinecone Vector Store (Query)  
  - *Role:* Retrieves relevant documents from Pinecone based on query embeddings.  
  - *Credentials:* Same Pinecone API.  
  - *Namespace:* `"eirevo"` consistent with storage node.  
  - *Output:* Most relevant FAQ or knowledge base snippets for chat agent.  
  - *Failures:* Connectivity, empty results, API limits.

---

### 2.3 Chat Interface and Query Handling

**Overview:**  
Handles incoming chat messages, manages conversation memory, routes queries to AI chat models and tools (knowledge or billing), and produces responses.

**Nodes Involved:**  
- When chat message received  
- Chat Agent  
- Simple Memory  
- Knowledge Tool  
- Billing Tool  
- OpenAI Chat Model  
- OpenRouter Chat Model

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger (Webhook)  
  - *Role:* Starts workflow upon chat message received, public webhook available.  
  - *Initial Message:* "Hi how I can help you."  
  - *Failures:* Webhook connectivity, malformed input.

- **Chat Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Central chatbot logic coordinating tools and models.  
  - *System Message:* Defines assistant role as a billing system upgrade helper, lists tools available (knowledge and billing), response guidelines, formatting rules, and examples.  
  - *Input:* User chat message, memory buffer, tool outputs.  
  - *Output:* Chatbot response.  
  - *Failures:* Tool invocation errors, model errors, conversation state errors.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores last 10 conversational messages to maintain context in multi-turn dialogues.  
  - *Input/Output:* Connects to Chat Agent’s memory.  
  - *Failures:* Memory overflow or truncation not handled.

- **Knowledge Tool**  
  - *Type:* LangChain Tool Vector Store  
  - *Role:* Provides knowledge base answers by querying Pinecone Q&A vector store.  
  - *Input:* Query embeddings, user query.  
  - *Failures:* Empty or irrelevant results.

- **Billing Tool**  
  - *Type:* LangChain Agent Tool  
  - *Role:* Queries Airtable live data for billing/payment questions. Uses a prompt with Airtable Payments table schema.  
  - *Input:* User message text.  
  - *Failures:* Airtable API errors, missing data, filter formula issues.

- **OpenAI Chat Model**  
  - *Type:* LangChain LLM Chat OpenAI  
  - *Role:* Runs GPT-4o-mini model for complex queries, integrated with Chat Agent and Knowledge Tool.  
  - *Credentials:* OpenAI API.  
  - *Failures:* API limits, model errors.

- **OpenRouter Chat Model**  
  - *Type:* LangChain LLM Chat OpenRouter  
  - *Role:* Uses OpenRouter’s openai/o4-mini as a cheaper alternative for simpler queries, linked to Billing Tool.  
  - *Credentials:* OpenRouter API.  
  - *Failures:* Connectivity, authentication, model availability.

- **billing-airtable** (Airtable Search Node)  
  - *Type:* Airtable Tool  
  - *Role:* Searches Airtable Payments table with filter formula dynamically generated from user query context.  
  - *Credentials:* Airtable API token.  
  - *Failures:* API errors, formula syntax errors.  
  - *Sticky Note5* describes sample Airtable schema and record for reference.

---

### 2.4 Billing Data Integration

**Overview:**  
Provides live billing and payment data access by querying Airtable data, integrated as a tool within the chatbot agent.

**Nodes Involved:**  
- billing-airtable  
- Billing Tool  
- OpenRouter Chat Model

**Node Details:**

- **billing-airtable**  
  - *Type:* Airtable Tool Node  
  - *Role:* Search operation on Payments table, returns relevant payment records.  
  - *FilterByFormula:* Dynamically generated from AI prompt context.  
  - *Input:* Query from Billing Tool.  
  - *Output:* Matching payment records.  
  - *Failures:* No matches, API limits, formula errors.

- **Billing Tool**  
  - *Type:* LangChain Agent Tool  
  - *Role:* Uses Airtable data to answer billing-related queries; passes user prompt and system message defining Airtable schema.  
  - *Connected AI Model:* OpenRouter Chat Model (o4-mini).  
  - *Failures:* Airtable or model errors.

- **OpenRouter Chat Model**  
  - *See above in 2.3.*

---

### 2.5 AI Model and Tool Management

**Overview:**  
Manages multiple AI models and tools to optimize cost, performance, and scalability by routing queries to appropriate tools/models based on query type.

**Nodes Involved:**  
- Sticky Note2 (Documentation)  
- OpenAI Chat Model  
- OpenRouter Chat Model  
- Knowledge Tool  
- Billing Tool

**Details:**

- **Sticky Note2** explains the rationale: Using a premium OpenAI GPT-4o-mini for complex queries and cheaper OpenRouter o4-mini for simpler/billing queries.  
- The Chat Agent orchestrates these tools, using Pinecone for knowledge and Airtable for billing.  
- This hybrid approach balances cost and accuracy.

---

## 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                         | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                            |
|-----------------------------|-------------------------------------|---------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Starts workflow manually               | None                             | HTTP Request                      |                                                                                                                        |
| HTTP Request                | HTTP Request                        | Fetches website content                | When clicking ‘Execute workflow’ | Exctract HTML Body                | Replace the sample website URL in the **HTTP Request** node with your own domain or content source.                      |
| Exctract HTML Body          | Set Node                            | Extracts HTML body from full HTML      | HTTP Request                    | Markdown                         |                                                                                                                        |
| Markdown                   | Markdown Node                       | Converts HTML body to Markdown         | Exctract HTML Body              | Normalize Text                   |                                                                                                                        |
| Normalize Text             | Code Node                          | Cleans and normalizes Markdown text    | Markdown                       | Store in Pinecone1               | ## Normalized Content: Removes noise, duplication; improves retrieval quality. Update code as needed.                   |
| Character Text Splitter     | LangChain Text Splitter (Character) | Splits text into chunks for embedding  | Normalize Text                 | Default Data Loader1             | Adjust chunk size in the **Text Splitter** for website markdown output; `######` separator works well.                   |
| Default Data Loader1        | LangChain Document Loader           | Loads text chunks for embedding        | Character Text Splitter        | Generate Embeddings1             |                                                                                                                        |
| Generate Embeddings1        | LangChain OpenAI Embeddings         | Generates embeddings for text chunks   | Default Data Loader1           | Store in Pinecone1               |                                                                                                                        |
| Store in Pinecone1          | LangChain Pinecone Vector Store     | Inserts embeddings into Pinecone       | Normalize Text, Generate Embeddings1 |                                   |                                                                                                                        |
| When chat message received  | LangChain Chat Trigger (Webhook)    | Starts chat on incoming message        | None                         | Chat Agent                      |                                                                                                                        |
| Simple Memory              | LangChain Memory Buffer              | Maintains conversation context         | None                         | Chat Agent                      |                                                                                                                        |
| Chat Agent                 | LangChain Agent                     | Main chatbot logic and response        | When chat message received, Simple Memory, Billing Tool, Knowledge Tool, OpenAI Chat Model |                              |                                                                                                                        |
| Knowledge Tool             | LangChain Vector Store Tool          | Queries Pinecone knowledge base        | Pinecone Store (Q&A)           | Chat Agent                      |                                                                                                                        |
| Pinecone Store (Q&A)       | LangChain Pinecone Vector Store      | Retrieves knowledge base data           | Generate Embeddings1           | Knowledge Tool                  |                                                                                                                        |
| billing-airtable           | Airtable Tool                      | Searches Airtable Payments table       | Billing Tool                  | Billing Tool                   | Sample Airtable record schema with Payments table and columns listed.                                                  |
| Billing Tool               | LangChain Agent Tool                | Handles billing queries via Airtable   | billing-airtable, OpenRouter Chat Model | Chat Agent                      |                                                                                                                        |
| OpenAI Chat Model          | LangChain LLM Chat OpenAI           | Advanced chat model for complex queries | Chat Agent                   | Chat Agent, Knowledge Tool      |                                                                                                                        |
| OpenRouter Chat Model      | LangChain LLM Chat OpenRouter       | Cost-effective chat model for billing  | Billing Tool                  | Billing Tool                   |                                                                                                                        |
| Sticky Note3               | Sticky Note                       | Workflow overview and usage instructions | None                         | None                          | Explains entire workflow purpose, structure, usage, requirements, and helpful links.                                    |
| Sticky Note4               | Sticky Note                       | Guidance on replacing website URL      | None                         | None                          | Replace sample website URL with your own URL in HTTP Request node.                                                      |
| Sticky Note5               | Sticky Note                       | Airtable schema example                 | None                         | None                          | Sample Airtable Payments table columns detailed.                                                                        |
| Sticky Note6               | Sticky Note                       | Text normalization explanation          | None                         | None                          | Explains benefits of normalized content for retrieval and token efficiency.                                            |
| Sticky Note1               | Sticky Note                       | Text splitter chunk size advice         | None                         | None                          | Advises on adjusting chunk size and separator for best Markdown splitting.                                             |
| Sticky Note2               | Sticky Note                       | AI tools usage and model selection rationale | None                         | None                          | Explains cost/performance tradeoffs of using multiple AI models and tools.                                            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node** named "When clicking ‘Execute workflow’" to start the content extraction process manually.

2. **Add an HTTP Request Node** named "HTTP Request":  
   - Set method to GET.  
   - Set URL to your target website page (default: `https://content.eirevo.ie/billing-change-faqs`).  
   - Connect output of manual trigger to this node.

3. **Add a Set Node** named "Exctract HTML Body":  
   - Use an expression to extract content inside `<body>` tags from the HTTP response HTML:  
     `={{ $json?.data.match(/<body[^>]*>([\s\S]*?)<\/body>/i)[1] }}`  
   - Connect HTTP Request output to this node.

4. **Add a Markdown Node** named "Markdown":  
   - Convert the extracted HTML body to Markdown format.  
   - Connect Exctract HTML Body output to this node.

5. **Add a Code Node** named "Normalize Text":  
   - JavaScript code to remove unwanted markers ("Page X"), multiple newlines, and trim spaces:  
     ```js
     const input = $json.data || $json.text || "";
     let text = input.replace(/Page\s*\d+/gi, "");
     text = text.replace(/\n\n+/g, " ");
     text = text.replace(/\n+/g, " ");
     text = text.trim();
     return { json: { normalizedText: text } };
     ```  
   - Connect Markdown output to this node.

6. **Add a Character Text Splitter Node** named "Character Text Splitter":  
   - Configure chunk size: 500 characters.  
   - Chunk overlap: 50 characters.  
   - Separator: `######`.  
   - Connect Normalize Text output to this node.

7. **Add a Default Data Loader Node** named "Default Data Loader1":  
   - Set text splitting mode to "custom" (to use above splitter).  
   - Connect Character Text Splitter output to this node.

8. **Add a LangChain OpenAI Embeddings Node** named "Generate Embeddings1":  
   - Set embedding dimension: 1536.  
   - Use OpenAI API credentials.  
   - Connect Default Data Loader1 output to this node.

9. **Add a Pinecone Vector Store Node** named "Store in Pinecone1":  
   - Set mode to "insert".  
   - Set Pinecone namespace to your project namespace (e.g., `"eirevo"`).  
   - Choose your Pinecone index.  
   - Use Pinecone API credentials.  
   - Connect Normalize Text and Generate Embeddings1 outputs to this node.

10. **Create a LangChain Chat Trigger Node** named "When chat message received":  
    - Enable public webhook.  
    - Set initial bot message: "Hi how I can help you.".

11. **Add a LangChain Memory Buffer Window Node** named "Simple Memory":  
    - Set context window length to 10 messages.  
    - Connect to Chat Agent later.

12. **Add an Airtable Tool Node** named "billing-airtable":  
    - Configure Airtable base and table (e.g., Payments table).  
    - Set operation to "search".  
    - Set filter formula dynamically from AI prompt context.  
    - Use Airtable API credentials.

13. **Add a LangChain Agent Tool Node** named "Billing Tool":  
    - Configure system message explaining Airtable Payments schema.  
    - Pass the user prompt text dynamically.  
    - Connect billing-airtable and OpenRouter Chat Model outputs here.

14. **Add an OpenRouter Chat Model Node** named "OpenRouter Chat Model":  
    - Model: `openai/o4-mini`.  
    - Use OpenRouter API credentials.

15. **Add a LangChain Pinecone Vector Store Node** named "Pinecone Store (Q&A)":  
    - Configure to query Pinecone index and namespace `"eirevo"`.  
    - Use Pinecone API credentials.

16. **Add a LangChain OpenAI Embeddings Node** named "Embeddings OpenAI":  
    - Same OpenAI API credentials.  
    - Used to generate query embeddings for chat queries.

17. **Add a LangChain Tool Vector Store Node** named "Knowledge Tool":  
    - Connected to Pinecone Store (Q&A) for knowledge retrieval.

18. **Add a LangChain LLM Chat OpenAI Node** named "OpenAI Chat Model":  
    - Model: `gpt-4o-mini`.  
    - Use OpenAI API credentials.

19. **Add a LangChain Agent Node** named "Chat Agent":  
    - Compose a detailed system prompt defining assistant role, tools (Knowledge and Billing), guidelines, and examples.  
    - Connect inputs from:  
      - When chat message received (trigger)  
      - Simple Memory (context)  
      - Billing Tool (for billing queries)  
      - Knowledge Tool (for knowledge queries)  
      - OpenAI Chat Model (language model)  
    - Outputs chatbot responses.

20. **Connect Node Flow:**  
    - Manual Trigger → HTTP Request → Exctract HTML Body → Markdown → Normalize Text → Character Text Splitter → Default Data Loader1 → Generate Embeddings1 → Store in Pinecone1.  
    - When chat message received → Chat Agent.  
    - Chat Agent connected to Simple Memory, Billing Tool, Knowledge Tool, OpenAI Chat Model.  
    - Billing Tool connected to billing-airtable and OpenRouter Chat Model.  
    - Knowledge Tool connected to Pinecone Store (Q&A) and Embeddings OpenAI.

21. **Verify Credentials:**  
    - OpenAI API for embeddings and chat (GPT-4o-mini).  
    - Pinecone API for vector storage and query.  
    - Airtable API token for billing data access.  
    - OpenRouter API for cheaper chat model.

22. **Adjust Parameters:**  
    - Replace website URL in HTTP Request with your own.  
    - Tune chunk size and separator in Character Text Splitter based on your Markdown output.  
    - Update Pinecone namespace and index as per your project.  
    - Customize Chat Agent system prompt to your brand and domain.  
    - Modify Airtable table and columns if your schema differs.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This n8n workflow shows how to extract website content, index it in Pinecone, and leverage Airtable to power a chat agent for customer Q&A. Use it to build knowledge bases, chatbots, or RAG workflows for FAQs and support documentation.                | Sticky Note3 in workflow                                          |
| Adjust chunk size in the Text Splitter for your website markdown output. The example uses Character Text Splitter with separator `######`. Always check the Markdown output to fine-tune splitting logic.                                                   | Sticky Note1                                                     |
| AI Tool usage combines different tools/models to balance cost and performance: Knowledge Tool (Pinecone), Billing Tool (Airtable), cheaper Chat Model (OpenRouter o4-mini), and premium Chat Model (OpenAI GPT-4o-mini) for complex queries.                 | Sticky Note2                                                     |
| Replace the sample website URL in the HTTP Request node with your own domain or content source. The example URL is https://content.eirevo.ie/billing-change-faqs.                                                                                         | Sticky Note4                                                     |
| Sample Airtable Payments table schema includes columns like Name, Payment Date, Amount Paid, Payment Method, Customer, Billing Account, Invoice Status, Payment Summary (AI), and Payment Risk Assessment (AI). Use this as reference for your Airtable base. | Sticky Note5                                                     |
| Normalized content removes noise and duplication, improves retrieval quality and consistent formatting, and prevents wasted tokens in embeddings. Customize normalization code as needed for your content.                                                | Sticky Note6                                                     |
| For help or community support, visit the n8n Forum: https://community.n8n.io/                                                                                                                                                                             | Sticky Note3                                                     |

---

**Disclaimer:** The provided content originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected elements. All data handled is legal and public.

---