Scrape and ingest web content into Supabase pgvector with Firecrawl

https://n8nworkflows.xyz/workflows/scrape-and-ingest-web-content-into-supabase-pgvector-with-firecrawl-13911


# Scrape and ingest web content into Supabase pgvector with Firecrawl

# 1. Workflow Overview

This workflow has two distinct entry points built around a shared Supabase pgvector knowledge base:

1. **Website ingestion path**: accepts a company URL by webhook, validates and normalizes it, checks whether that URL was already ingested, scrapes the website with Firecrawl, creates embeddings, and stores the resulting chunks in Supabase pgvector.
2. **Question-answering path**: accepts chat input, uses an AI agent with memory and an OpenRouter chat model, retrieves relevant content from Supabase using vector similarity plus Cohere reranking, and answers questions grounded in the ingested website content.

The workflow is designed for use cases such as:
- Building a lightweight company knowledge base from a public website
- Enriching lead data with scraped website content
- Powering a retrieval-augmented chat assistant over ingested web pages
- Avoiding duplicate ingestion of the same site

## 1.1 Ingestion Entry and URL Preparation

This block receives a POST request containing a URL, validates the payload, normalizes the domain, and prepares a canonical HTTPS URL for downstream use.

## 1.2 Duplicate Detection and Routing

This block checks Supabase for an already-ingested document matching the normalized URL and routes execution either to an early duplicate response or to scraping.

## 1.3 Website Scraping and Vector Storage

This block scrapes the target website via Firecrawl, loads the scraped content as documents, generates OpenAI embeddings, and inserts the documents plus vectors into the Supabase `documents` table.

## 1.4 Chat Entry and RAG Agent

This block receives chat messages, runs an AI agent with memory and an OpenRouter LLM, and equips the agent with a retrieval tool backed by Supabase pgvector.

## 1.5 Vector Retrieval and Reranking

This block generates embeddings for incoming user queries, retrieves candidate documents from Supabase, optionally filters by URL, reranks results with Cohere, and exposes them as a tool to the agent.

---

# 2. Block-by-Block Analysis

## 2.1 Ingestion Entry and URL Preparation

### Overview
This block is the ingestion entry point. It expects a POST request containing a `url` field in the body, validates the value, strips protocol and path to derive a normalized domain, and returns a canonical `https://domain` URL for later scraping and metadata tagging.

### Nodes Involved
- Receive company URL
- Validate and normalize URL
- Return URL validation error

### Node Details

#### 1) Receive company URL
- **Type and technical role:** `n8n-nodes-base.webhook`; HTTP entry point for ingestion.
- **Configuration choices:**
  - Method: `POST`
  - Response mode: handled by a separate Respond to Webhook node
  - Path: auto-generated webhook path
- **Key expressions or variables used:**
  - Downstream code reads `body.url`
- **Input and output connections:**
  - No input; starts the ingestion branch
  - Output -> `Validate and normalize URL`
- **Version-specific requirements:**
  - Type version `2.1`
  - Uses response-node mode, so the branch must end in a Respond to Webhook node
- **Edge cases or potential failure types:**
  - Missing request body
  - Wrong HTTP method
  - No downstream response path if execution is interrupted
- **Sub-workflow reference:** None

#### 2) Validate and normalize URL
- **Type and technical role:** `n8n-nodes-base.code`; validates payload and standardizes the URL.
- **Configuration choices:**
  - JavaScript code reads the first input item
  - `onError: continueErrorOutput`, which is important because invalid URLs are intentionally routed to a dedicated error response path
- **Key expressions or variables used:**
  - `const body = $input.first().json.body`
  - `body?.url?.trim()`
  - Regex validation for the domain
  - Returns:
    - success: `{ status: 200, domain, url: https://domain }`
    - missing URL: `{ status: 422, message: "Missing 'url' field in request body." }`
  - Throws an error for invalid domain format
- **Input and output connections:**
  - Input <- `Receive company URL`
  - Main success output -> `Check for duplicate in Supabase`
  - Error output -> `Return URL validation error`
- **Version-specific requirements:**
  - Type version `2`
  - Requires the Code node error output behavior supported in current n8n versions
- **Edge cases or potential failure types:**
  - `body.url` missing
  - Domains with unsupported formats
  - Internationalized domain names may fail this regex
  - URLs including query strings or nonstandard ports are stripped and may not validate as intended
  - The missing-url branch returns a normal item, not an error item; because of the current wiring, that item still flows into duplicate checking unless additional guarding is added
- **Sub-workflow reference:** None

#### 3) Return URL validation error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends the HTTP validation error response.
- **Configuration choices:**
  - Response code: `422`
  - Response key: expression based on `$json.error`
- **Key expressions or variables used:**
  - `={{ $json.error }}`
- **Input and output connections:**
  - Input <- error output of `Validate and normalize URL`
  - No output
- **Version-specific requirements:**
  - Type version `1.5`
- **Edge cases or potential failure types:**
  - If the Code node emits a normal item with `{status: 422, message: ...}` instead of throwing, this node will not be triggered
  - `$json.error` may differ depending on n8n error object shape/version
- **Sub-workflow reference:** None

---

## 2.2 Duplicate Detection and Routing

### Overview
This block checks whether the normalized URL already exists in the `documents` table and branches accordingly. If a match is found, the workflow returns a friendly duplicate notice instead of scraping again.

### Nodes Involved
- Check for duplicate in Supabase
- Skip if already ingested
- Return duplicate notice

### Node Details

#### 4) Check for duplicate in Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`; queries Supabase for matching rows.
- **Configuration choices:**
  - Operation: `get`
  - Table: `documents`
  - Filter condition on `metadata`
  - The filter value is a full JSON object expression containing the normalized URL in `url`
- **Key expressions or variables used:**
  - `{{ $json.url }}`
  - Metadata filter object shaped like:
    - `loc.lines.from/to`
    - `url`
    - `line`
    - `source`
    - `blobType`
- **Input and output connections:**
  - Input <- `Validate and normalize URL`
  - Output -> `Skip if already ingested`
- **Version-specific requirements:**
  - Type version `1`
  - Requires valid Supabase credentials and an existing `documents` table
- **Edge cases or potential failure types:**
  - Exact JSON-match filtering on `metadata` is brittle
  - If stored metadata differs slightly per chunk or loader version, duplicates may not be detected
  - Auth or table permission failures
  - If the `documents` schema differs from what the vector store writes, the filter may never match
- **Sub-workflow reference:** None

#### 5) Skip if already ingested
- **Type and technical role:** `n8n-nodes-base.if`; branches based on whether the duplicate check returned data.
- **Configuration choices:**
  - Condition uses `notEmpty` on `{{$input.all()[0].json}}`
  - True branch = duplicate found
  - False branch = proceed to scraping
- **Key expressions or variables used:**
  - `={{$input.all()[0].json}}`
- **Input and output connections:**
  - Input <- `Check for duplicate in Supabase`
  - True output -> `Return duplicate notice`
  - False output -> `Scrape company website with Firecrawl`
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - The duplicate check node has `alwaysOutputData: true`; depending on returned shape, this condition can behave differently than expected
  - If Supabase returns an empty placeholder object rather than no rows, the workflow may incorrectly mark as duplicate
- **Sub-workflow reference:** None

#### 6) Return duplicate notice
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; ends the ingestion request early if the URL is already present.
- **Configuration choices:**
  - Response code: `200`
  - JSON body: `{ "message": "Already in the database" }`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input <- true output of `Skip if already ingested`
  - No output
- **Version-specific requirements:**
  - Type version `1.5`
- **Edge cases or potential failure types:**
  - Returns success rather than conflict status; consumers must interpret the message, not the code
- **Sub-workflow reference:** None

---

## 2.3 Website Scraping and Vector Storage

### Overview
This block performs the actual ingestion. Firecrawl scrapes the normalized URL, the content is converted into document items, OpenAI generates embeddings, and the vector store node inserts documents into Supabase pgvector.

### Nodes Involved
- Scrape company website with Firecrawl
- Generate OpenAI embeddings
- Load scraped content
- Store embeddings in Supabase
- Return ingestion result

### Node Details

#### 7) Scrape company website with Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawl`; scrapes and converts website content.
- **Configuration choices:**
  - Operation: `scrape`
  - URL comes from normalized URL node
  - Scrape formats configured with a single default format entry, which in practice means default Firecrawl output formatting
- **Key expressions or variables used:**
  - `={{ $('Validate and normalize URL').item.json.url }}`
- **Input and output connections:**
  - Input <- false output of `Skip if already ingested`
  - Main output -> `Store embeddings in Supabase`
- **Version-specific requirements:**
  - Community node: `@mendable/n8n-nodes-firecrawl`
  - Requires Firecrawl credentials and installed package support in the n8n instance
- **Edge cases or potential failure types:**
  - Firecrawl auth issues
  - Target site blocking scraping
  - Timeout on slow websites
  - Empty or malformed scrape result
  - Single-page scrape may miss important subpages if recursive crawl is not configured
- **Sub-workflow reference:** None

#### 8) Generate OpenAI embeddings
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; embedding model used during insertion.
- **Configuration choices:**
  - Default options
  - Connected as the embedding provider for the vector store node
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No main input; connected via `ai_embedding`
  - Output -> `Store embeddings in Supabase` as embedding model
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires OpenAI embeddings-capable API access
- **Edge cases or potential failure types:**
  - OpenAI auth/rate limits
  - Model defaults may change over time if not pinned explicitly
  - Long content may increase token use and cost
- **Sub-workflow reference:** None

#### 9) Load scraped content
- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`; converts input content into LangChain-compatible document objects.
- **Configuration choices:**
  - Adds metadata field `url`
  - Uses the normalized canonical URL as metadata
- **Key expressions or variables used:**
  - `={{ $('Validate and normalize URL').item.json.url }}`
- **Input and output connections:**
  - No visible main input in the JSON connections
  - Output -> `Store embeddings in Supabase` as `ai_document`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - In this exported workflow, the node is **not connected by main input** from the Firecrawl node
  - As configured, vector store insertion expects documents from this loader; without a proper main/content feed into the loader, ingestion may fail or insert nothing
  - Metadata shape may not match the duplicate-check filter exactly
- **Sub-workflow reference:** None

#### 10) Store embeddings in Supabase
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreSupabase`; writes documents and embeddings into pgvector-backed storage.
- **Configuration choices:**
  - Mode: `insert`
  - Table name: `documents`
  - Receives:
    - main input from Firecrawl
    - document input from the loader
    - embedding input from OpenAI embeddings
- **Key expressions or variables used:**
  - Table selected via resource locator as `documents`
- **Input and output connections:**
  - Main input <- `Scrape company website with Firecrawl`
  - `ai_document` <- `Load scraped content`
  - `ai_embedding` <- `Generate OpenAI embeddings`
  - Main output -> `Return ingestion result`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Supabase pgvector-compatible schema
- **Edge cases or potential failure types:**
  - If the `documents` table was not created with the expected schema, insertion will fail
  - Missing or mismatched document input from loader
  - Supabase row-level security or API key restrictions
  - Large scraped content may create many chunks and increase latency/cost
- **Sub-workflow reference:** None

#### 11) Return ingestion result
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns the ingestion outcome to the original HTTP caller.
- **Configuration choices:**
  - Response code: `200`
  - JSON body dynamically reports inserted item count
  - `executeOnce: true`
- **Key expressions or variables used:**
  - `Added {{$input.all().length}} items to Supabase`
- **Input and output connections:**
  - Input <- `Store embeddings in Supabase`
  - No output
- **Version-specific requirements:**
  - Type version `1.5`
- **Edge cases or potential failure types:**
  - Reported count depends on how many output items the vector store emits, which may not exactly equal inserted chunks in all node versions
- **Sub-workflow reference:** None

---

## 2.4 Chat Entry and RAG Agent

### Overview
This block is the conversational interface over the stored knowledge base. It starts from a chat trigger, invokes an AI agent, attaches a chat model and memory, and allows the agent to call a Supabase retrieval tool.

### Nodes Involved
- Receive chat message
- Answer query from enriched leads
- OpenRouter LLM
- Chat memory

### Node Details

#### 12) Receive chat message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`; chat-based workflow trigger.
- **Configuration choices:**
  - Uses default options
  - Has its own webhook-style identifier internally
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input; starts the chat branch
  - Output -> `Answer query from enriched leads`
- **Version-specific requirements:**
  - Type version `1.4`
  - Requires an n8n environment supporting chat trigger UI/runtime
- **Edge cases or potential failure types:**
  - Session handling varies by deployment
  - Chat UI/API integration must be configured externally if used outside editor test mode
- **Sub-workflow reference:** None

#### 13) Answer query from enriched leads
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestration layer that reasons over user input and calls tools.
- **Configuration choices:**
  - Uses default options
  - Receives:
    - main input from chat trigger
    - LLM from OpenRouter node
    - memory from buffer window
    - retrieval tool from Supabase vector store tool node
- **Key expressions or variables used:** None directly in the node config shown
- **Input and output connections:**
  - Main input <- `Receive chat message`
  - `ai_languageModel` <- `OpenRouter LLM`
  - `ai_memory` <- `Chat memory`
  - `ai_tool` <- `Retrieve documents from Supabase`
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases or potential failure types:**
  - Tool-use behavior depends heavily on model support and prompt defaults
  - Hallucination risk if retrieval is weak or empty
  - If the retriever tool errors, the agent may fail or answer without context
- **Sub-workflow reference:** None

#### 14) OpenRouter LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; chat model provider for the agent.
- **Configuration choices:**
  - Model: `anthropic/claude-sonnet-4.6`
  - Default options
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output -> `Answer query from enriched leads` as language model
- **Version-specific requirements:**
  - Type version `1`
  - Requires OpenRouter credentials and model availability on the account
- **Edge cases or potential failure types:**
  - Model naming/availability can change
  - OpenRouter quota, auth, or provider routing issues
  - Some tool-calling behaviors may vary across models
- **Sub-workflow reference:** None

#### 15) Chat memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; stores recent chat context for continuity.
- **Configuration choices:**
  - Default settings; no custom window size configured in export
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output -> `Answer query from enriched leads` as memory
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Memory persistence behavior depends on session handling
  - Default window may be too small or too large depending on use case
- **Sub-workflow reference:** None

---

## 2.5 Vector Retrieval and Reranking

### Overview
This block turns the Supabase knowledge base into an AI tool. It creates embeddings for user questions, retrieves similar documents from `documents`, allows optional filtering by URL, reranks results with Cohere, and provides the result set to the agent.

### Nodes Involved
- Generate OpenAI embeddings1
- Rerank results with Cohere
- Retrieve documents from Supabase

### Node Details

#### 16) Generate OpenAI embeddings1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; embedding model for similarity search during retrieval.
- **Configuration choices:**
  - Default options
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output -> `Retrieve documents from Supabase` as embedding model
- **Version-specific requirements:**
  - Type version `1.2`
- **Edge cases or potential failure types:**
  - Same as ingestion embedding node: auth, quota, model defaults, cost
- **Sub-workflow reference:** None

#### 17) Rerank results with Cohere
- **Type and technical role:** `@n8n/n8n-nodes-langchain.rerankerCohere`; reranks retrieved chunks for better relevance.
- **Configuration choices:**
  - Default configuration
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output -> `Retrieve documents from Supabase` as reranker
- **Version-specific requirements:**
  - Type version `1`
  - Requires Cohere API credentials and reranking support
- **Edge cases or potential failure types:**
  - Cohere auth or quota limits
  - Additional latency on each retrieval
  - If reranking fails, tool execution may fail depending on node behavior
- **Sub-workflow reference:** None

#### 18) Retrieve documents from Supabase
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreSupabase`; exposes vector search as an AI-callable retrieval tool.
- **Configuration choices:**
  - Mode: `retrieve-as-tool`
  - Table: `documents`
  - `useReranker: true`
  - Tool description: `Retrieve data for the AI Agent.`
  - Metadata field `url` can be provided by the AI via `$fromAI(...)` to narrow retrieval
- **Key expressions or variables used:**
  - `={{ $fromAI('url', 'to filter by URL add the specific URL here.', "string") }}`
- **Input and output connections:**
  - `ai_embedding` <- `Generate OpenAI embeddings1`
  - `ai_reranker` <- `Rerank results with Cohere`
  - `ai_tool` -> `Answer query from enriched leads`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires the same Supabase vector store schema used for ingestion
- **Edge cases or potential failure types:**
  - If AI supplies an invalid or nonexistent URL filter, retrieval may return no documents
  - Metadata filter effectiveness depends on how metadata was stored during ingestion
  - Empty database causes context-free answers unless explicitly guarded
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive company URL | n8n-nodes-base.webhook | Ingestion HTTP trigger |  | Validate and normalize URL | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Validate and normalize URL | n8n-nodes-base.code | Validate request body and canonicalize URL | Receive company URL | Check for duplicate in Supabase; Return URL validation error | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Check for duplicate in Supabase | n8n-nodes-base.supabase | Look for existing ingested URL | Validate and normalize URL | Skip if already ingested | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Skip if already ingested | n8n-nodes-base.if | Branch on duplicate presence | Check for duplicate in Supabase | Return duplicate notice; Scrape company website with Firecrawl | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Return duplicate notice | n8n-nodes-base.respondToWebhook | Return early success response for duplicate URL | Skip if already ingested |  | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Scrape company website with Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawl | Scrape and convert target website content | Skip if already ingested | Store embeddings in Supabase | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Generate OpenAI embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for ingestion |  | Store embeddings in Supabase |  |
| Load scraped content | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Convert scraped result into documents with metadata |  | Store embeddings in Supabase |  |
| Store embeddings in Supabase | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Insert documents and vectors into pgvector table | Scrape company website with Firecrawl; Generate OpenAI embeddings; Load scraped content | Return ingestion result |  |
| Return ingestion result | n8n-nodes-base.respondToWebhook | Return ingestion completion message | Store embeddings in Supabase |  |  |
| Return URL validation error | n8n-nodes-base.respondToWebhook | Return 422 validation error | Validate and normalize URL |  | ### How it works<br>1. A webhook receives a URL via POST request<br>2. The URL is validated, normalized, and checked for duplicates in Supabase<br>3. Firecrawl scrapes the page and converts it to clean markdown<br>4. OpenAI generates vector embeddings from the scraped content<br>5. The content and embeddings are stored in Supabase pgvector<br>6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the README to create the `documents` table with pgvector enabled<br>2. Add your Firecrawl API key<br>3. Add your OpenAI API key (for embeddings)<br>4. Add your OpenRouter API key (for the chat agent)<br>5. Add your Cohere API key (for reranking)<br>6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook |
| Receive chat message | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point for querying ingested knowledge |  | Answer query from enriched leads | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Answer query from enriched leads | @n8n/n8n-nodes-langchain.agent | RAG agent answering questions with tool access | Receive chat message; OpenRouter LLM; Chat memory; Retrieve documents from Supabase |  | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| OpenRouter LLM | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chat model used by the agent |  | Answer query from enriched leads | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Chat memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversational memory for the agent |  | Answer query from enriched leads | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Generate OpenAI embeddings1 | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for retrieval queries |  | Retrieve documents from Supabase | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Rerank results with Cohere | @n8n/n8n-nodes-langchain.rerankerCohere | Improve retrieval relevance ordering |  | Retrieve documents from Supabase | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Retrieve documents from Supabase | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Retrieval tool over pgvector documents | Generate OpenAI embeddings1; Rerank results with Cohere | Answer query from enriched leads | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence for n8n.

## Prerequisites
1. Create a **Supabase** project.
2. Prepare a `documents` table compatible with **pgvector** and the n8n Supabase Vector Store node.
3. Obtain API credentials for:
   - **Firecrawl**
   - **OpenAI**
   - **OpenRouter**
   - **Cohere**
4. Ensure your n8n instance supports:
   - LangChain nodes
   - Chat Trigger
   - Supabase Vector Store
   - Firecrawl community node

## A. Build the ingestion branch

### 1. Add the ingestion webhook
1. Create a **Webhook** node.
2. Name it **Receive company URL**.
3. Set:
   - **HTTP Method**: `POST`
   - **Response Mode**: `Using Respond to Webhook Node`
4. Keep the generated path or set a custom path.

### 2. Add URL validation and normalization
1. Create a **Code** node.
2. Name it **Validate and normalize URL**.
3. Connect **Receive company URL** -> **Validate and normalize URL**.
4. Enable error continuation to the error output.
5. Paste logic equivalent to:
   - Read `body.url`
   - Trim it
   - Remove protocol and path
   - Validate domain with a regex
   - On success, output:
     - `status: 200`
     - `domain`
     - `url: https://domain`
   - On invalid URL, throw an error
6. Important implementation note:
   - If you want missing `url` to go to the error response, either throw an error for missing input or add an IF node before the duplicate check.
   - The exported workflow returns a normal item for missing `url`, which is weaker than ideal.

### 3. Add validation error response
1. Create a **Respond to Webhook** node.
2. Name it **Return URL validation error**.
3. Connect the **error output** of **Validate and normalize URL** to this node.
4. Set:
   - **Response Code**: `422`
   - Response body/key based on the error payload
5. Prefer returning a stable JSON object such as:
   - `{"message":"Invalid URL"}`
   rather than depending only on `$json.error`.

### 4. Add duplicate check in Supabase
1. Create a **Supabase** node.
2. Name it **Check for duplicate in Supabase**.
3. Connect the **main success output** of **Validate and normalize URL** to it.
4. Configure:
   - **Operation**: `Get`
   - **Table**: `documents`
5. Add a filter intended to identify documents already stored for the same URL.
6. In the exported workflow, the filter compares a full JSON `metadata` object containing the URL. Recreate that only if your stored metadata matches exactly.
7. Recommended safer approach:
   - Store URL in a dedicated searchable column, or
   - Filter by a simpler metadata property if supported by your schema.

### 5. Add the duplicate router
1. Create an **If** node.
2. Name it **Skip if already ingested**.
3. Connect **Check for duplicate in Supabase** -> **Skip if already ingested**.
4. Configure the condition to detect whether the query returned any rows.
5. In the exported workflow the condition checks that the first item’s JSON is not empty.

### 6. Add duplicate response
1. Create a **Respond to Webhook** node.
2. Name it **Return duplicate notice**.
3. Connect the **true** output of **Skip if already ingested** to it.
4. Set:
   - **Response Code**: `200`
   - **Response Body**:
     ```json
     { "message": "Already in the database" }
     ```

### 7. Add the Firecrawl scraper
1. Create a **Firecrawl** node.
2. Name it **Scrape company website with Firecrawl**.
3. Connect the **false** output of **Skip if already ingested** to it.
4. Configure:
   - **Operation**: `scrape`
   - **URL**:
     `{{ $('Validate and normalize URL').item.json.url }}`
5. Add Firecrawl credentials.
6. Keep default scrape format unless you need specific output formats.

### 8. Add the embeddings model for ingestion
1. Create an **OpenAI Embeddings** node.
2. Name it **Generate OpenAI embeddings**.
3. Add your OpenAI credentials.
4. Leave default options unless you want to pin a specific embedding model.

### 9. Add the document loader
1. Create a **Default Data Loader** node from the LangChain document nodes.
2. Name it **Load scraped content**.
3. Configure metadata:
   - `url = {{ $('Validate and normalize URL').item.json.url }}`
4. Important:
   - Unlike the exported JSON, make sure this node actually receives the scraped content.
   - Connect **Scrape company website with Firecrawl** -> **Load scraped content** on the main input if your node version expects main data input before emitting documents.

### 10. Add the Supabase vector store insertion node
1. Create a **Supabase Vector Store** node.
2. Name it **Store embeddings in Supabase**.
3. Configure:
   - **Mode**: `insert`
   - **Table**: `documents`
4. Add Supabase credentials.
5. Connect:
   - Main input: **Scrape company website with Firecrawl** -> **Store embeddings in Supabase**
   - AI document input: **Load scraped content** -> **Store embeddings in Supabase**
   - AI embedding input: **Generate OpenAI embeddings** -> **Store embeddings in Supabase**

### 11. Add the ingestion success response
1. Create a **Respond to Webhook** node.
2. Name it **Return ingestion result**.
3. Connect **Store embeddings in Supabase** -> **Return ingestion result**.
4. Configure:
   - **Response Code**: `200`
   - **Respond With**: JSON
   - Body:
     `{"message":"Added {{$input.all().length}} items to Supabase"}`
5. Optionally enable **Execute Once**.

---

## B. Build the chat and retrieval branch

### 12. Add the chat trigger
1. Create a **Chat Trigger** node.
2. Name it **Receive chat message**.
3. Leave default options unless you need a specific session setup.

### 13. Add the AI agent
1. Create an **AI Agent** node.
2. Name it **Answer query from enriched leads**.
3. Connect **Receive chat message** -> **Answer query from enriched leads**.

### 14. Add the LLM
1. Create an **OpenRouter Chat Model** node.
2. Name it **OpenRouter LLM**.
3. Set model to:
   - `anthropic/claude-sonnet-4.6`
4. Add OpenRouter credentials.
5. Connect this node to the **AI language model** input of **Answer query from enriched leads**.

### 15. Add chat memory
1. Create a **Memory Buffer Window** node.
2. Name it **Chat memory**.
3. Connect it to the **AI memory** input of **Answer query from enriched leads**.
4. Optionally tune the window size if you want more or less conversational context.

### 16. Add retrieval embeddings
1. Create another **OpenAI Embeddings** node.
2. Name it **Generate OpenAI embeddings1**.
3. Add the same OpenAI credentials.
4. This node will embed user queries for retrieval.

### 17. Add reranking
1. Create a **Cohere Reranker** node.
2. Name it **Rerank results with Cohere**.
3. Add Cohere credentials.

### 18. Add the Supabase retrieval tool
1. Create another **Supabase Vector Store** node.
2. Name it **Retrieve documents from Supabase**.
3. Configure:
   - **Mode**: `retrieve as tool`
   - **Table**: `documents`
   - **Use Reranker**: enabled
   - **Tool Description**: `Retrieve data for the AI Agent.`
4. Add optional metadata filtering:
   - Metadata field `url`
   - Value from AI:
     `{{ $fromAI('url', 'to filter by URL add the specific URL here.', "string") }}`
5. Connect:
   - **Generate OpenAI embeddings1** -> AI embedding input
   - **Rerank results with Cohere** -> AI reranker input
   - **Retrieve documents from Supabase** -> AI tool input of **Answer query from enriched leads**

---

## C. Add credentials

### 19. Configure credentials
Create and assign these credentials:

1. **Supabase account**
   - Used by `Store embeddings in Supabase`
2. **Supabase account 2**
   - Used by `Check for duplicate in Supabase`
   - Used by `Retrieve documents from Supabase`
   - These can be the same credential if desired
3. **Firecrawl account**
   - Used by `Scrape company website with Firecrawl`
4. **OpenAI account**
   - Used by both embedding nodes
5. **OpenRouter account**
   - Used by `OpenRouter LLM`
6. **Cohere account**
   - Used by `Rerank results with Cohere`

---

## D. Add optional notes on canvas

### 20. Add the two sticky notes

1. Add a sticky note near the ingestion flow with this content:

   **How it works**
   1. A webhook receives a URL via POST request  
   2. The URL is validated, normalized, and checked for duplicates in Supabase  
   3. Firecrawl scrapes the page and converts it to clean markdown  
   4. OpenAI generates vector embeddings from the scraped content  
   5. The content and embeddings are stored in Supabase pgvector  
   6. A built-in RAG chat agent lets you query the knowledge base using natural language, with Cohere reranking for better retrieval

   **Setup**
   1. Create a Supabase project and run the SQL from the workflow README to create the `documents` table with pgvector enabled  
   2. Add your Firecrawl API key  
   3. Add your OpenAI API key (for embeddings)  
   4. Add your OpenRouter API key (for the chat agent)  
   5. Add your Cohere API key (for reranking)  
   6. Activate the workflow and send a POST request with `{"url": "https://example.com"}` to the webhook

2. Add a second sticky note near the chat branch:

   **Supabase setup**  
   Run the SQL migration from the workflow README to create the `documents` table with pgvector enabled.

---

## E. Test the workflow

### 21. Test ingestion
Send a POST request to the webhook such as:
```json
{
  "url": "https://example.com"
}
```

Expected outcomes:
- Invalid URL -> `422`
- Existing URL -> `200` with duplicate message
- New URL -> scraping + embedding + insertion + success message

### 22. Test chat retrieval
Use the chat trigger UI or connected chat interface and ask:
- “What does this company do?”
- “Summarize the homepage”
- “What are the services mentioned on example.com?”

---

## F. Important implementation cautions

### 23. Fix the likely structural issue in the exported workflow
The exported workflow shows **Load scraped content** connected only as an AI document source to the vector store, but not visibly fed by the Firecrawl main output. When recreating, make sure the loader actually receives the scraped payload, otherwise insertion may fail or produce no documents.

### 24. Improve duplicate detection
The current duplicate check relies on an exact `metadata` JSON match, which is fragile. Prefer one of these:
1. Add a dedicated `source_url` column in Supabase and check that directly.
2. Normalize metadata consistently and filter only by URL.
3. Maintain a separate ingestion registry table.

### 25. Improve validation handling
For production:
- Throw on missing `url` as well as malformed URL
- Return a stable JSON body for errors
- Consider normalizing `www.` handling explicitly
- Consider whether you want to accept subpaths rather than collapsing everything to the domain root

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Supabase project and run the SQL migration that creates the `documents` table with pgvector enabled. | Supabase preparation required before ingestion or retrieval will work |
| Firecrawl is used to scrape the target website and convert it into cleaner content suitable for embedding. | Firecrawl service setup |
| OpenAI is used only for embeddings in this workflow, not for the final chat model. | Embedding generation |
| OpenRouter provides the chat LLM for the agent, configured here with `anthropic/claude-sonnet-4.6`. | Chat answering |
| Cohere is used as a reranker to improve retrieval quality before the agent answers. | Retrieval quality improvement |
| Example ingestion payload: `{"url": "https://example.com"}` | Webhook request body |