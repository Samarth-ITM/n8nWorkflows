Generate Twitter Content in Personal Style with OpenAI & Supabase RAG

https://n8nworkflows.xyz/workflows/generate-twitter-content-in-personal-style-with-openai---supabase-rag-10672


# Generate Twitter Content in Personal Style with OpenAI & Supabase RAG

### 1. Workflow Overview

This workflow, titled **"Generate Twitter Content in Personal Style with OpenAI & Supabase RAG,"** is designed to build a personalized content generation engine that learns from a user's past Twitter posts or notes. Its main use case is to enable creators to ingest their content into a knowledge base (KB), and then generate new Twitter posts, quote tweets, replies, and image prompts that align with their personal style and topics of interest.

The workflow consists of two main logical blocks:

- **1.1 Ingestion Block:** Accepts user-submitted text (past posts or ideas), normalizes and enriches it with metadata, splits it into manageable chunks, generates embeddings via OpenAI, and stores them in a Supabase vector store. This block constructs a searchable knowledge base reflecting the user's style and topics.

- **1.2 Generation Block:** Accepts user parameters (topic, number of top matches, and optional hint), uses those to query the Supabase KB for relevant content snippets, invokes an OpenAI chat model with a custom prompt to generate personalized Twitter content (post, quote, reply, image prompt), formats the output into rich HTML, and presents the results in a user-friendly form.

Supplementary sticky notes provide user guidance, troubleshooting tips, and a high-level conceptual overview.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Ingestion Block (Build Knowledge Base)

**Overview:**  
This block allows users to submit past posts or notes via a form, normalizes and adds metadata, splits the content into chunks, generates embeddings using OpenAI, and stores the data in Supabase for later retrieval.

**Nodes Involved:**  
- Form: Add to KB  
- Normalize (ingest)  
- Embeddings OpenAI (ingest)  
- Document Loader (+metadata)  
- Text Splitter  
- VectorStore (Supabase)  
- End Page (ingest)  

**Node Details:**

- **Form: Add to KB**  
  - Type: Form Trigger (Webhook-based form input)  
  - Role: Entry point for ingestion; collects user input for content and optional topic tag.  
  - Configuration: Two fields — "content" (textarea, required) and "topic" (optional). Form titled "Add to KB" with description guiding to paste past posts or notes and optionally tag topic/style.  
  - Inputs: External user form submission  
  - Outputs: To Normalize (ingest)  
  - Edge cases: Missing content (form validation enforces required), malformed inputs  

- **Normalize (ingest)**  
  - Type: Set node  
  - Role: Transforms form fields into a structured JSON object, ensuring fields: content, topic (default empty string), and style (set to string 'true').  
  - Key expression: `jsonOutput` set to `{ content: $json.fields.content, topic: ($json.fields.topic || ''), style: 'true' }`  
  - Inputs: From Form: Add to KB  
  - Outputs: To VectorStore (Supabase) (and embedding-related nodes)  
  - Edge cases: Missing topic handled by defaulting to empty string  

- **Embeddings OpenAI (ingest)**  
  - Type: LangChain OpenAI Embedding node  
  - Role: Generates vector embeddings from the content text for semantic search.  
  - Configuration: Uses OpenAI credentials ("OpenAi account 2"), no additional options specified.  
  - Inputs: From Normalize (ingest)  
  - Outputs: To VectorStore (Supabase) and KB (Supabase VectorStore) (embedding outputs)  
  - Edge cases: API quota limits, network issues, invalid API key, content too large (chunking handled downstream)  

- **Document Loader (+metadata)**  
  - Type: LangChain Document Loader with metadata  
  - Role: Wraps the content text into a document object, attaching metadata fields for source, style, and topic.  
  - Configuration: Metadata includes fixed "source" = "user_ingest", "style" = "true", and "topic" derived dynamically.  
  - Key expression for metadata: `topic` is assigned `{{$json.topic || ''}}`  
  - Inputs: From Text Splitter (chunked content)  
  - Outputs: To VectorStore (Supabase)  
  - Edge cases: Empty content or metadata fields  

- **Text Splitter**  
  - Type: Recursive Character Text Splitter (LangChain)  
  - Role: Splits large content into chunks with overlap to improve embedding quality and retrieval.  
  - Configuration: chunk size 1200 characters, chunk overlap 150 characters  
  - Inputs: From Document Loader (+metadata)  
  - Outputs: To Embeddings OpenAI (ingest)  
  - Edge cases: Very short content may produce one chunk only  

- **VectorStore (Supabase)**  
  - Type: LangChain Supabase Vector Store node  
  - Role: Inserts embedded chunks into Supabase table `documents` for later retrieval.  
  - Configuration: Mode "insert", uses query name "match_documents" for embeddings insertion, authenticated with Supabase API credentials.  
  - Inputs: From Embeddings OpenAI, Document Loader  
  - Outputs: To End Page (ingest)  
  - Edge cases: Supabase connection errors, RLS (Row Level Security) misconfiguration, insertion failures  

- **End Page (ingest)**  
  - Type: Form node (Completion type)  
  - Role: Presents a user confirmation page after ingestion with completion title and message derived from the topic and submitted content.  
  - Inputs: From VectorStore (Supabase)  
  - Outputs: None (end of ingestion flow)  
  - Edge cases: Rendering errors if topic or content missing  

---

#### 2.2 Generation Block (Generate Posts from KB)

**Overview:**  
This block receives user input via a form specifying a topic, number of top results (topK), and an optional hint. It builds retrieval parameters, queries the Supabase KB to retrieve relevant documents, invokes an OpenAI chat model with a custom prompt to generate content aligned with the user's style, formats the output into HTML, and displays the generated Twitter content on a completion page.

**Nodes Involved:**  
- Form: Generate  
- Build Params  
- Generator Agent  
- KB (Supabase VectorStore)  
- OpenAI Chat Model  
- Edit Fields (format HTML)  
- End Page (generate)  

**Node Details:**

- **Form: Generate**  
  - Type: Form Trigger  
  - Role: Entry point for content generation; collects user input for topic, number of top matches (topK), and an optional hint to guide generation.  
  - Configuration: Fields include "topic" (string), "topK" (number, default 5), and "hint" (optional string).  
  - Inputs: External user form submission  
  - Outputs: To Build Params  
  - Edge cases: Missing or malformed inputs, topK defaulted to 5 if empty  

- **Build Params**  
  - Type: Set node  
  - Role: Normalizes and structures the generation parameters into a JSON object for downstream nodes, enforcing defaults and filters.  
  - Key Expression:  
    ```js
    {
      topic: $json.fields.topic || '',
      style: 'true',
      topK: Number($json.fields.topK || 5),
      hint: $json.fields.hint || '',
      filters: { topic: ($json.fields.topic || ''), style: true }
    }
    ```
  - Inputs: From Form: Generate  
  - Outputs: To Generator Agent  
  - Edge cases: Invalid topK values coerced to Number (NaN would propagate issues downstream)  

- **Generator Agent**  
  - Type: LangChain Agent node  
  - Role: Orchestrates the AI content generation by querying the KB tool and generating structured output with JSON keys: post, quote, reply, image_prompt.  
  - Configuration:  
    - Custom prompt instructing to fetch up to `topK` relevant snippets using filters (topic, style), then output JSON only with specified keys.  
    - System message: "Be concise, concrete, and aligned to prior high-engagement style."  
    - Prompt type: define  
  - Inputs: From Build Params  
  - Outputs: To Edit Fields (format HTML)  
  - Edge cases: Prompt failures, JSON parse errors, API rate limits  

- **KB (Supabase VectorStore)**  
  - Type: LangChain Supabase Vector Store (Retrieve as Tool)  
  - Role: Tool used by Generator Agent to search the Supabase KB for relevant documents matching the topic and style filters.  
  - Configuration:  
    - Mode: retrieve-as-tool  
    - Query name: "match_documents"  
    - Table: "documents"  
    - Tool description explains filter usage (e.g. { "topic": "creator_mindset", "style": "true" })  
    - Uses Supabase API credentials  
  - Inputs: As AI tool available to Generator Agent  
  - Outputs: To Generator Agent (retrieval results)  
  - Edge cases: Supabase connectivity, empty or irrelevant results, filter misconfiguration  

- **OpenAI Chat Model**  
  - Type: LangChain Chat Model with OpenAI (GPT-4.1-mini)  
  - Role: Provides the language generation engine for the Generator Agent.  
  - Configuration: Model set to "gpt-4.1-mini" (a specialized lightweight GPT-4 variant).  
  - Inputs: Connected internally as language model to Generator Agent  
  - Outputs: To Generator Agent  
  - Edge cases: API quota limits, model unavailability, network timeouts  

- **Edit Fields (format HTML)**  
  - Type: Set node  
  - Role: Parses the JSON output returned by the Generator Agent, formats it into HTML with labeled sections for post, quote, reply, and image prompt.  
  - Key Expression:  
    Parses `output` field (string or object), extracts keys, and builds an HTML string with headings and paragraphs.  
  - Inputs: From Generator Agent  
  - Outputs: To End Page (generate)  
  - Edge cases: Malformed JSON output, missing keys  

- **End Page (generate)**  
  - Type: Form node (Completion type)  
  - Role: Displays the formatted generated Twitter content in a completion page with a title and message containing the formatted HTML.  
  - Inputs: From Edit Fields (format HTML)  
  - Outputs: None (end of generation flow)  
  - Edge cases: Rendering errors if input missing or malformed  

---

### 3. Summary Table

| Node Name                    | Node Type                                  | Functional Role                           | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                                         |
|------------------------------|--------------------------------------------|-----------------------------------------|-----------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Form: Add to KB              | Form Trigger                              | Receive user content for KB ingestion   | External form submission     | Normalize (ingest)             | Use **Add to KB** to add posts and notes. Ingest (build KB) block description.                                                     |
| Normalize (ingest)           | Set                                       | Normalize and enrich ingestion data     | Form: Add to KB             | VectorStore (Supabase)         |                                                                                                                                    |
| Embeddings OpenAI (ingest)   | LangChain Embeddings OpenAI                | Generate vector embeddings               | Normalize (ingest)          | VectorStore (Supabase), KB (Supabase VectorStore) |                                                                                                                                    |
| Document Loader (+metadata)  | LangChain Document Loader with metadata   | Wrap content with metadata               | Text Splitter               | VectorStore (Supabase)         |                                                                                                                                    |
| Text Splitter               | LangChain Recursive Character Text Splitter | Chunk large text content                 | Document Loader (+metadata) | Embeddings OpenAI (ingest)     |                                                                                                                                    |
| VectorStore (Supabase)       | LangChain VectorStore Supabase             | Store embeddings in Supabase             | Embeddings OpenAI, Document Loader | End Page (ingest)          | No inserts → check Supabase creds / RLS. Troubleshooting sticky note.                                                               |
| End Page (ingest)            | Form (Completion)                          | Show ingestion confirmation page        | VectorStore (Supabase)      | None                          |                                                                                                                                    |
| Form: Generate              | Form Trigger                              | Receive generation parameters            | External form submission     | Build Params                  | Use **Generate Posts** to create content using KB. Generate block description.                                                     |
| Build Params                | Set                                       | Build and normalize generation params   | Form: Generate              | Generator Agent               |                                                                                                                                    |
| Generator Agent             | LangChain Agent                            | Generate content using KB and OpenAI    | Build Params                | Edit Fields (format HTML)      |                                                                                                                                    |
| KB (Supabase VectorStore)   | LangChain VectorStore Supabase (Retrieve) | Retrieve relevant documents from KB     | Used as tool by Generator Agent | Generator Agent             |                                                                                                                                    |
| OpenAI Chat Model           | LangChain Chat Model (OpenAI GPT-4.1-mini) | Language model for generation            | Internal to Generator Agent | Generator Agent               |                                                                                                                                    |
| Edit Fields (format HTML)   | Set                                       | Parse AI output and format as HTML      | Generator Agent             | End Page (generate)            |                                                                                                                                    |
| End Page (generate)          | Form (Completion)                          | Display generated Twitter content       | Edit Fields (format HTML)   | None                          |                                                                                                                                    |
| Sticky: Overview (yellow)1  | Sticky Note                               | Workflow overview and setup instructions | None                       | None                          | ## Self-Learning X Content Engine (Creator RAG Booster) How it works and setup steps explained.                                     |
| Sticky: Step                | Sticky Note                               | Ingest block summary                     | None                       | None                          | ## Ingest (build KB) Normalize → embed → store in Supabase. Use Add to KB.                                                         |
| Sticky: Step 3              | Sticky Note                               | Generate block summary                   | None                       | None                          | ## Generate (use KB) Set topic + topK + hint, agent searches KB, outputs post/quote/reply/image_prompt.                             |
| Sticky: Troubleshooting1    | Sticky Note                               | Troubleshooting common issues            | None                       | None                          | Blank page → check form Completion type. No inserts → Supabase creds/RLS. Generic tone → add more samples. Odd matches → tune params.|

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Credentials**  
- Configure OpenAI API credentials (e.g., "OpenAi account 2").  
- Configure Supabase API credentials (e.g., "Supabase account") with access to a table named `documents` for vector storage.

---

**Step 2: Create Ingestion Flow**

1. Add a **Form Trigger** node named `Form: Add to KB`  
   - Title: "Add to KB"  
   - Fields:  
     - `content` (textarea, required, placeholder: "📝 Paste your past post or idea here…")  
     - `topic` (string, optional, placeholder: "💬 e.g. creator_mindset, ai_automation, productivity")  
   - Description: "Paste your past posts or notes. Optionally tag topic and mark as style sample."  
   - This form triggers on HTTP webhook.

2. Add a **Set** node named `Normalize (ingest)` connected from `Form: Add to KB`  
   - Mode: Raw  
   - JSON Output:  
     ```js
     {
       content: $json.fields.content,
       topic: $json.fields.topic || '',
       style: 'true'
     }
     ```

3. Add a **LangChain Recursive Character Text Splitter** node named `Text Splitter`  
   - Chunk Size: 1200  
   - Chunk Overlap: 150  
   - Connect from `Normalize (ingest)`

4. Add a **LangChain Document Loader (+metadata)** node named `Document Loader (+metadata)`  
   - JSON Data: `{{$json.content}}`  
   - Metadata:  
     - `source`: "user_ingest" (static)  
     - `style`: "true" (static)  
     - `topic`: `{{$json.topic || ''}}` (expression)  
   - Connect from `Text Splitter`

5. Add a **LangChain Embeddings OpenAI** node named `Embeddings OpenAI (ingest)`  
   - Credentials: OpenAI API credentials  
   - Connect from `Document Loader (+metadata)`

6. Add a **LangChain VectorStore Supabase** node named `VectorStore (Supabase)`  
   - Mode: "insert"  
   - Table: "documents"  
   - Query Name: "match_documents" (used internally)  
   - Credentials: Supabase API  
   - Connect from `Embeddings OpenAI (ingest)`

7. Add a **Form** node named `End Page (ingest)`  
   - Operation: Completion  
   - Title: `{{$json.metadata.topic}}`  
   - Message: `{{$json.pageContent}}`  
   - Connect from `VectorStore (Supabase)`

---

**Step 3: Create Generation Flow**

1. Add a **Form Trigger** node named `Form: Generate`  
   - Title: "Generate Posts"  
   - Fields:  
     - `topic` (string)  
     - `topK` (number, placeholder: "Default 5")  
     - `hint` (string, optional, placeholder: "Optional hint (e.g., Consistency vs creativity)")  
   - Description: "Create post/quote/reply/image_prompt from your knowledge base."  
   - Trigger on webhook.

2. Add a **Set** node named `Build Params` connected from `Form: Generate`  
   - Mode: Raw  
   - JSON Output:  
     ```js
     {
       topic: $json.fields.topic || '',
       style: 'true',
       topK: Number($json.fields.topK || 5),
       hint: $json.fields.hint || '',
       filters: { topic: $json.fields.topic || '', style: true }
     }
     ```

3. Add a **LangChain VectorStore Supabase** node named `KB (Supabase VectorStore)`  
   - Mode: "retrieve-as-tool"  
   - Table: "documents"  
   - Query Name: "match_documents"  
   - Tool Name: "kb_vectorstore"  
   - Tool Description: "KB search over Supabase `documents` (use filters like {\"topic\":\"creator_mindset\",\"style\":\"true\"})"  
   - Credentials: Supabase API

4. Add a **LangChain Chat Model (OpenAI)** node named `OpenAI Chat Model`  
   - Model: "gpt-4.1-mini"  
   - Credentials: OpenAI API

5. Add a **LangChain Agent** node named `Generator Agent`  
   - Text prompt:  
     ```
     You are a content engineer for X posts.
     Use the KB (tool) to fetch up to {{$json.topK || 5}} relevant snippets using {{$json.filters}}.
     Then output **JSON only** with keys exactly:
     {"post","quote","reply","image_prompt"}
     - post: one original tweet (<=230 chars, no hashtags/links)
     - quote: sharp quote-tweet for a given topic/link (<=200 chars)
     - reply: constructive reply (<=180 chars, end with a short question)
     - image_prompt: brief photoreal/graphic prompt
     Keep tone aligned to retrieved examples. Topic: {{$json.topic}}. Hint: {{$json.hint}}
     ```
   - System Message: "Be concise, concrete, and aligned to prior high-engagement style."  
   - Prompt Type: define  
   - Connect `Build Params` to input  
   - Add `KB (Supabase VectorStore)` as AI tool input  
   - Add `OpenAI Chat Model` as AI language model input

6. Add a **Set** node named `Edit Fields (format HTML)` connected from `Generator Agent`  
   - Mode: Raw  
   - JSON Output: JavaScript function that:  
     - Parses `output` JSON string/object from AI response  
     - Builds HTML with headings and paragraphs for keys: post, quote, reply, image_prompt  
     Example structure:  
     ```html
     <div style="font-family:sans-serif; line-height:1.5;">
       <h3>📝 Post</h3><p>{post}</p><hr/>
       <h3>💬 Quote</h3><p>{quote}</p><hr/>
       <h3>💭 Reply</h3><p>{reply}</p><hr/>
       <h3>🎨 Image Prompt</h3><p>{image_prompt}</p>
     </div>
     ```

7. Add a **Form** node named `End Page (generate)`  
   - Operation: Completion  
   - Title: `{{$json.completionTitle}}`  
   - Message: `{{$json.completionMessage}}`  
   - Connect from `Edit Fields (format HTML)`

---

**Step 4: Connect Nodes Appropriately**

- Ingestion flow:  
  `Form: Add to KB` → `Normalize (ingest)` → `Text Splitter` → `Document Loader (+metadata)` → `Embeddings OpenAI (ingest)` → `VectorStore (Supabase)` → `End Page (ingest)`

- Generation flow:  
  `Form: Generate` → `Build Params` → `Generator Agent` → `Edit Fields (format HTML)` → `End Page (generate)`  
  Also: `Generator Agent` uses `KB (Supabase VectorStore)` as a retrieval tool and `OpenAI Chat Model` for language generation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Self-Learning X Content Engine (Creator RAG Booster): Ingest your past posts to build a personal KB and generate aligned Twitter content.          | Sticky Note Overview (yellow) - workflow concept explanation                                         |
| Ingest block: Normalize text → add metadata → split → embed → store in Supabase. Use **Add to KB** form to submit samples.                         | Sticky Note Step                                                                                       |
| Generate block: Set topic + topK + optional hint. Agent searches Supabase and returns post, quote, reply, and image prompt.                         | Sticky Note Step 3                                                                                     |
| Troubleshooting tips: Blank page → check form is Completion type; No inserts → Supabase creds or RLS; Generic tone → add more samples; Odd matches → tune topic or topK. | Sticky Note Troubleshooting                                                                           |
| For embedding and retrieval, Supabase table `documents` is used with RLS enabled and a query named `match_documents`.                              | Implied in VectorStore (Supabase) node configuration; ensure Supabase setup aligns with this           |
| Use OpenAI GPT-4.1-mini model for efficient generation with good quality.                                                                            | OpenAI Chat Model node configuration                                                                   |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public._