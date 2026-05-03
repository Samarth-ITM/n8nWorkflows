Draft and manage academic research papers with GPT-4 and Pinecone

https://n8nworkflows.xyz/workflows/draft-and-manage-academic-research-papers-with-gpt-4-and-pinecone-12730


# Draft and manage academic research papers with GPT-4 and Pinecone

## 1. Workflow Overview

**Purpose:**  
This workflow automates continuous academic research monitoring, synthesizes multi-source research intelligence with GPT-4, drafts publication-ready manuscripts, validates references and originality, adapts formatting for target venues, and logs outcomes/learning signals into Google Sheets.

**Primary use cases:**
- Automated literature and competitive intelligence (papers, patents, datasets, blogs)
- Theme clustering, novelty/gap detection, and methodology consistency checks
- Manuscript drafting with citation integrity support
- Pre-submission validation (references + plagiarism)
- Publication ops: submission tracking, reviewer feedback logging, and learning optimization

### 1.1 Scheduled Input & Configuration
Runs on a daily schedule and defines environment variables (API endpoints, keywords, validation services).

### 1.2 Multi-Source Retrieval & Consolidation
Fetches academic papers, patents, datasets, and blog feeds; merges them into one stream.

### 1.3 Ingestion, Chunking, Embedding & Vector Storage (In-Memory)
Loads merged content as documents, splits into chunks, embeds with OpenAI embeddings, and inserts into an in-memory vector store.

### 1.4 Research Intelligence (Agent + Memory + Structured Output)
A research analysis agent (GPT-4) summarizes, clusters, extracts citation signals, and finds gaps; output is parsed into a defined JSON schema and logged to Google Sheets.

### 1.5 Manuscript Drafting with Citation Retrieval Tooling
A drafting agent (GPT-4) produces a structured manuscript, using the vector store as a citation lookup tool.

### 1.6 Validation Gate (References + Plagiarism) and Rework Loop
Validates references and runs a plagiarism check; an IF gate routes either to formatting/adaptation or back to drafting for remediation.

### 1.7 Formatting, Submission Tracking, Feedback Logging, and Learning Loop
Adapts manuscript format for venues, tracks deadlines, logs reviewer feedback, generates learning insights, and stores insights.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Input & Configuration

**Overview:**  
Triggers the workflow daily and centralizes configurable endpoints/keywords used by subsequent HTTP nodes.

**Nodes involved:**
- Schedule Research Monitoring
- Workflow Configuration

#### Node: Schedule Research Monitoring
- **Type / Role:** Schedule Trigger; workflow entry point.
- **Key configuration:**
- **Inputs/Outputs:** No inputs. Outputs to **Workflow Configuration**.
- **Edge cases / failures:**
  - Misconfigured schedule/timezone expectations (instance timezone affects run time).
  - Workflow inactive (`active: false`) means it won’t run until enabled.

#### Node: Workflow Configuration
- **Type / Role:** Set; defines variables used throughout workflow.
- **Key configuration (interpreted):**
  - Sets string placeholders for:
    - `arxivApiUrl`, `patentApiUrl`, `datasetApiUrl`, `blogRssUrl`
    - `searchKeywords`
    - `referenceValidationApiUrl`, `plagiarismCheckApiUrl`
  - “Include other fields” enabled (passes through any existing fields).
- **Inputs/Outputs:** Input from Schedule Trigger; outputs to all four fetch nodes.
- **Edge cases / failures:**
  - Placeholders must be replaced with real URLs/keywords; otherwise HTTP requests will fail (invalid URL / DNS / 4xx).
  - If APIs require auth headers/keys, they are not configured here (must be added in HTTP nodes).

---

### Block 2 — Multi-Source Retrieval & Consolidation

**Overview:**  
Calls external sources (papers/patents/datasets/blog RSS) using shared keywords, then merges results into one combined dataset.

**Nodes involved:**
- Fetch Academic Papers
- Fetch Patents
- Fetch Datasets
- Fetch Technical Blogs
- Combine All Sources

#### Node: Fetch Academic Papers
- **Type / Role:** HTTP Request; queries ArXiv-like endpoint.
- **Key configuration:**
  - URL from `Workflow Configuration.arxivApiUrl`
  - Query params:
    - `search_query` = `searchKeywords`
    - `max_results` = `50`
- **Inputs/Outputs:** Input from Workflow Configuration; output to Combine All Sources (input index 0).
- **Edge cases / failures:**
  - ArXiv commonly returns Atom/XML; if not converted to JSON upstream, downstream LLM receives raw XML/string.
  - Rate limits, 429 responses, network timeouts.

#### Node: Fetch Patents
- **Type / Role:** HTTP Request; patent search API.
- **Key configuration:**
  - URL from `patentApiUrl`
  - Query param `query` = `searchKeywords`
- **Connections:** Output to Combine All Sources (input index 1).
- **Edge cases:**
  - API authentication and pagination not implemented.
  - Response shape variability (array vs object) can break later assumptions.

#### Node: Fetch Datasets
- **Type / Role:** HTTP Request; dataset search API.
- **Key configuration:**
  - URL from `datasetApiUrl`
  - Query param `search` = `searchKeywords`
- **Connections:** Output to Combine All Sources (input index 2).
- **Edge cases:** Similar auth/pagination/shape issues.

#### Node: Fetch Technical Blogs
- **Type / Role:** HTTP Request; fetches RSS feed content.
- **Key configuration:**
  - URL from `blogRssUrl`
  - No parsing node included; response may be XML.
- **Connections:** Output to Combine All Sources (input index 3).
- **Edge cases:** RSS often needs XML parsing for structured downstream usage.

#### Node: Combine All Sources
- **Type / Role:** Merge; combines four inputs by position.
- **Key configuration:**
  - Mode: **Combine**
  - CombineBy: **Position** (`combineByPosition`)
  - Expects **4** inputs.
- **Connections:** Outputs to Rate Limit Delay.
- **Edge cases / failures:**
  - If any upstream fetch returns zero items or errors, positional combine may misalign data or reduce output unexpectedly.
  - Combine-by-position assumes each input produces same-length item lists; otherwise, results can be partial or mismatched.

---

### Block 3 — Ingestion, Chunking, Embedding & Vector Storage (In-Memory)

**Overview:**  
Introduces a delay (rudimentary rate limiting), loads content into documents, splits text, generates embeddings, and inserts into an in-memory vector store.

**Nodes involved:**
- Rate Limit Delay
- Document Loader
- Text Splitter
- OpenAI Embeddings
- Knowledge Base Vector Store

#### Node: Rate Limit Delay
- **Type / Role:** Wait; introduces a pause before vector insertion/analysis.
- **Key configuration:** Wait amount = `2` (seconds by default in n8n Wait).
- **Connections:** Output to Knowledge Base Vector Store (main).
- **Edge cases:**
  - Does not truly protect against upstream API rate limits (those already happened); only throttles subsequent operations.
  - For large runs, a fixed 2-second delay may be insufficient to manage OpenAI embedding limits.

#### Node: Document Loader
- **Type / Role:** LangChain Document Loader (default); converts incoming data to documents.
- **Key configuration:**
  - Text splitting mode set to **custom**
  - Uses **Text Splitter** via `ai_textSplitter` connection.
- **Connections:**
  - Receives text splitter tool from Text Splitter (`ai_textSplitter`)
  - Outputs documents to Knowledge Base Vector Store (`ai_document`)
- **Edge cases:**
  - If upstream JSON is complex (nested objects), loader may stringify to `[object Object]`-like representations unless configured/flattened.
  - No explicit mapping of “content field” is shown; behavior depends on node defaults.

#### Node: Text Splitter
- **Type / Role:** Recursive Character Text Splitter; chunks text for embeddings.
- **Key configuration:** Options default (chunk size/overlap not specified).
- **Connections:** Provides splitter to Document Loader (`ai_textSplitter`).
- **Edge cases:**
  - Default chunk sizing may be suboptimal for academic PDFs/long abstracts.
  - If text is already short or malformed XML, chunking may add noise.

#### Node: OpenAI Embeddings
- **Type / Role:** OpenAI embeddings provider for vectorization.
- **Key configuration:** Uses OpenAI credentials “OpenAi account”. Model not specified (node default applies).
- **Connections:** Provides embeddings to Knowledge Base Vector Store (`ai_embedding`).
- **Edge cases / failures:**
  - Missing/invalid OpenAI key; quota exhaustion; 429s.
  - Large payloads can exceed token limits or embedding request limits.

#### Node: Knowledge Base Vector Store
- **Type / Role:** Vector store (in-memory) in **insert** mode; stores embedded chunks.
- **Key configuration:**
  - Mode: **insert**
  - `memoryKey` = `vector_store_key` (shared handle used later for retrieval-as-tool)
- **Connections:**
  - Main input from Rate Limit Delay
  - Receives documents from Document Loader (`ai_document`)
  - Receives embeddings from OpenAI Embeddings (`ai_embedding`)
  - Outputs to Research Analysis Agent (main)
- **Edge cases:**
  - In-memory store is ephemeral per execution; it does not persist across runs. This conflicts with the sticky notes’ mention of Pinecone (persistent vector DB).
  - If main input drives “insert” behavior unexpectedly (depending on node implementation), may require consistent item structure.

---

### Block 4 — Research Intelligence (Agent + Memory + Structured Output + Logging)

**Overview:**  
Runs a research analysis agent (GPT-4) with short-term memory and a strict structured output schema, then saves analysis results into a Google Sheets “Research Analysis” table.

**Nodes involved:**
- Research Analysis Agent
- OpenAI GPT-4 Model
- Research Context Memory
- Analysis Output Parser
- Save to Research Database

#### Node: Research Context Memory
- **Type / Role:** Buffer Window Memory; provides conversational context to the agent.
- **Key configuration:** Defaults (window size not specified).
- **Connections:** Supplies memory to Research Analysis Agent (`ai_memory`).
- **Edge cases:**
  - Without explicit memory window size, context depth depends on defaults.
  - Memory is per execution unless persisted; may not provide cross-day continuity.

#### Node: OpenAI GPT-4 Model
- **Type / Role:** Chat LLM provider for the analysis agent.
- **Key configuration:** `gpt-4.1-mini`
- **Connections:** Supplies LLM to Research Analysis Agent (`ai_languageModel`).
- **Failures:** Credential issues, model availability, quota.

#### Node: Analysis Output Parser
- **Type / Role:** Structured output parser with a manual JSON schema.
- **Key configuration:** Requires the agent to output JSON with fields:
  - `summary` (string)
  - `themes` (string[])
  - `clusters` (string[])
  - `citations` (string[])
  - `novelty_score` (number 0–100)
  - `research_gaps` (string[])
  - `hypothesis_gaps` (string[])
  - `methodology` (string)
- **Connections:** Attached to Research Analysis Agent as `ai_outputParser`.
- **Edge cases:**
  - If the LLM outputs invalid JSON or misses fields, parsing fails and halts the agent.
  - “number” parsing issues if the model emits `"85"` as a string.

#### Node: Research Analysis Agent
- **Type / Role:** LangChain Agent; orchestrates analysis instructions and returns structured JSON.
- **Key configuration:**
  - Uses the entire incoming `$json` as text input.
  - System message defines tasks (summarization, themes/clusters, citation graph analysis, novelty/gaps, methodology).
  - Output parser enabled.
- **Connections:**
  - Main input from Knowledge Base Vector Store
  - Outputs:
    - To Save to Research Database
    - To Manuscript Drafting Agent (to begin drafting)
  - Receives:
    - LLM from OpenAI GPT-4 Model
    - Memory from Research Context Memory
    - Output parser from Analysis Output Parser
- **Edge cases:**
  - Input payload may be too large/unstructured; consider pre-formatting sources into concise fields before agent.
  - Citation graph analysis is requested but not backed by an explicit citation database; results may be heuristic.

#### Node: Save to Research Database
- **Type / Role:** Google Sheets; appends or updates analysis records.
- **Key configuration:**
  - Operation: **Append or Update**
  - Sheet: `Research Analysis`
  - Document ID: placeholder (must be set)
  - Mapping:
    - `summary`, `themes`, `novelty_score`, `research_gaps`
  - Matching column: `summary` (used as upsert key)
- **Connections:** Input from Research Analysis Agent.
- **Edge cases / failures:**
  - Using `summary` as a unique key is fragile (duplicates likely). Consider a stable ID (date + query hash).
  - Arrays (`themes`, `research_gaps`) may be stored as comma-joined strings depending on Sheets node behavior.
  - OAuth token expiry / missing permissions.

---

### Block 5 — Manuscript Drafting with Citation Retrieval Tooling

**Overview:**  
Drafts a structured academic manuscript from the analysis and enables the agent to retrieve citations from the vector store as a tool.

**Nodes involved:**
- Manuscript Drafting Agent
- OpenAI GPT-4 Drafting Model
- Manuscript Output Parser
- Citation Reference Store
- Citation Embeddings

#### Node: OpenAI GPT-4 Drafting Model
- **Type / Role:** Chat LLM provider for drafting.
- **Key configuration:** `gpt-4.1-mini`
- **Connections:** Supplies LLM to Manuscript Drafting Agent.
- **Failures:** Same as other OpenAI nodes.

#### Node: Citation Embeddings
- **Type / Role:** OpenAI embeddings provider for retrieval tool vector store.
- **Key configuration:** Uses OpenAI credentials; model default.
- **Connections:** Feeds embeddings into Citation Reference Store (`ai_embedding`).
- **Edge cases:**
  - Redundant embeddings node if using same store already embedded; here it’s required by the retrieve-as-tool node wiring.

#### Node: Citation Reference Store
- **Type / Role:** Vector store (in-memory) in **retrieve-as-tool** mode; exposes semantic search as an agent tool.
- **Key configuration:**
  - Mode: **retrieve-as-tool**
  - `memoryKey` = `vector_store_key` (must match the insert store)
  - Tool description: “Search and retrieve citation references from the knowledge base”
- **Connections:** Provided as `ai_tool` to Manuscript Drafting Agent.
- **Edge cases:**
  - Because the store is in-memory, it only contains documents inserted earlier in the same run.
  - If insert didn’t run properly, tool returns empty results, harming citation integrity.

#### Node: Manuscript Output Parser
- **Type / Role:** Structured output parser defining manuscript schema.
- **Expected fields:**
  - `title`, `abstract`, `introduction`, `methods`, `results`, `discussion`, `conclusion` (strings)
  - `references` (string[])
- **Connections:** Attached to Manuscript Drafting Agent (`ai_outputParser`).
- **Edge cases:** LLM may output sections but omit valid JSON structure, causing parser failure.

#### Node: Manuscript Drafting Agent
- **Type / Role:** LangChain Agent; generates a publication-ready manuscript.
- **Key configuration:**
  - Input text: full `$json` from prior stage (analysis and/or remediation loop)
  - System message requires:
    - Standard academic sections
    - Proper citation integrity using citation tool
    - Structured return format
- **Connections:**
  - Main input from Research Analysis Agent and (conditionally) from Check Validation Results “false” branch (rework loop)
  - Output to Validate References API
  - Receives:
    - LLM from OpenAI GPT-4 Drafting Model
    - Tool from Citation Reference Store
    - Output parser from Manuscript Output Parser
- **Edge cases:**
  - If routed back here after failed validation, the workflow does not pass explicit remediation instructions (e.g., “reduce plagiarism, fix refs”), so the agent may repeat the same output unless the incoming JSON includes failure details.

---

### Block 6 — Validation Gate (References + Plagiarism) and Rework Loop

**Overview:**  
Validates the manuscript references via an external API and checks plagiarism via another API. An IF node gates whether the workflow proceeds to formatting and tracking or loops back to drafting.

**Nodes involved:**
- Validate References API
- Plagiarism Check API
- Check Validation Results

#### Node: Validate References API
- **Type / Role:** HTTP Request; external reference validation service.
- **Key configuration:**
  - URL from `referenceValidationApiUrl`
  - Method: POST
  - JSON body: `{ references: $json.references }`
- **Connections:** Input from Manuscript Drafting Agent; output to Plagiarism Check API.
- **Edge cases:**
  - Assumes `$json.references` exists and is an array.
  - Service response is expected to include `referencesValid` (boolean). If schema differs, IF node fails.

#### Node: Plagiarism Check API
- **Type / Role:** HTTP Request; external plagiarism scoring service.
- **Key configuration:**
  - URL from `plagiarismCheckApiUrl`
  - Method: POST
  - JSON body concatenates all manuscript sections into `text`
- **Connections:** Input from Validate References API; output to Check Validation Results.
- **Edge cases:**
  - Expects manuscript sections (`abstract`, `introduction`, etc.) still available in the item at this stage; but the immediate upstream node is Validate References API which typically replaces/overwrites JSON unless “Keep response/Include input” is configured (not shown). This is a common n8n pitfall:
    - If Validate References API outputs only `{referencesValid: ...}`, then plagiarism text concatenation will fail (undefined fields).
  - Response expected to include `plagiarismScore` (number). If missing, IF condition errors.

#### Node: Check Validation Results
- **Type / Role:** IF; gate on validation thresholds.
- **Key configuration:**
  - Condition 1: `$('Plagiarism Check API').item.json.plagiarismScore < 10`
  - Condition 2: `$('Validate References API').item.json.referencesValid == true`
  - Both must be true (AND).
- **Connections:**
  - **True** branch → Format Adaptation Agent
  - **False** branch → Manuscript Drafting Agent (rework loop)
- **Edge cases:**
  - `.item.json` usage assumes at least one item exists from each node and item indexing aligns.
  - If either API returns multiple items or different item counts, cross-node `.item` references may point to mismatched rows.

---

### Block 7 — Formatting, Submission Tracking, Feedback Logging, Learning Optimization

**Overview:**  
Formats the manuscript for target venues, logs submission metadata, tracks deadlines, logs reviewer feedback, then produces learning insights that are stored for future optimization.

**Nodes involved:**
- Format Adaptation Agent
- OpenAI Format Model
- Format Output Parser
- Track Submission Deadlines
- Log Reviewer Feedback
- Learning Optimization Agent
- OpenAI Learning Model
- Learning Output Parser
- Store Learning Insights

#### Node: OpenAI Format Model
- **Type / Role:** Chat LLM for formatting agent.
- **Key configuration:** `gpt-4.1-mini`
- **Connections:** Supplies LLM to Format Adaptation Agent.

#### Node: Format Output Parser
- **Type / Role:** Structured output parser (example-based).
- **Expected output example fields:**
  - `formatted_manuscript` (string)
  - `format_style` (string)
  - `word_count` (number)
- **Connections:** Attached to Format Adaptation Agent as output parser.
- **Edge cases:** Example schema is less strict than a full JSON schema; the agent may still produce inconsistent structures.

#### Node: Format Adaptation Agent
- **Type / Role:** LangChain Agent; adapts manuscript to style guidelines (IEEE/ACM/APA/Nature).
- **Key configuration:** System message instructs format transformation and citation style adaptation.
- **Connections:**
  - Input from Check Validation Results (true path)
  - Output to Track Submission Deadlines
  - Receives LLM from OpenAI Format Model and parser from Format Output Parser
- **Edge cases:**
  - No explicit “target venue” input is shown; unless present in incoming JSON, the agent must guess the style.
  - Formatting requirements often depend on templates; pure text formatting may be insufficient for real submissions.

#### Node: Track Submission Deadlines
- **Type / Role:** Google Sheets; tracks title/status/deadline/target venue.
- **Key configuration:**
  - Operation: Append or Update
  - Sheet: `Submission Deadlines`
  - Document ID placeholder
  - Upsert key: `title`
  - Requires fields: `title`, `status`, `deadline`, `target_venue` in incoming JSON
- **Connections:** Input from Format Adaptation Agent; output to Log Reviewer Feedback.
- **Edge cases:**
  - The formatting agent output schema does not include `status/deadline/target_venue`; unless provided elsewhere, these fields may be undefined.
  - Using `title` as unique key can collide across versions.

#### Node: Log Reviewer Feedback
- **Type / Role:** Google Sheets; logs editorial decision and reviewer comments.
- **Key configuration:**
  - Sheet: `Reviewer Feedback`
  - Document ID placeholder
  - Upsert key: `manuscript_id`
  - Expected fields: `manuscript_id`, `reviewer_comments`, `decision`, `feedback_date`
- **Connections:** Input from Track Submission Deadlines; output to Learning Optimization Agent.
- **Edge cases:**
  - This workflow does not show an inbound step where real reviewer feedback is collected; as-is, this node will likely write empty values unless upstream provides them.
  - Consider splitting “post-submission feedback ingestion” into a separate workflow/webhook.

#### Node: OpenAI Learning Model
- **Type / Role:** Chat LLM powering learning agent.
- **Key configuration:** `gpt-4.1-mini`
- **Connections:** Supplies LLM to Learning Optimization Agent.

#### Node: Learning Output Parser
- **Type / Role:** Structured output parser (example-based).
- **Expected fields:**
  - `success_patterns` (string[])
  - `rejection_reasons` (string[])
  - `optimization_recommendations` (string[])
  - `trending_topics` (string[])
  - `reviewer_preferences` (string)
- **Connections:** Attached to Learning Optimization Agent.

#### Node: Learning Optimization Agent
- **Type / Role:** LangChain Agent; derives optimization insights from outcomes/feedback.
- **Key configuration:** System message focuses on acceptance/rejection pattern mining and recommendations.
- **Connections:**
  - Input from Log Reviewer Feedback
  - Output to Store Learning Insights
  - Receives LLM from OpenAI Learning Model and parser from Learning Output Parser
- **Edge cases:**
  - If feedback data is missing/empty, insights will be speculative.

#### Node: Store Learning Insights
- **Type / Role:** Google Sheets; stores learning signals.
- **Key configuration:**
  - Sheet: `Learning Insights`
  - Document ID placeholder
  - Upsert key: `success_patterns` (fragile because it’s an array)
  - Writes: `trending_topics`, `success_patterns`, `optimization_recommendations`
- **Edge cases:**
  - Arrays as matching columns are unreliable; consider `run_id` or timestamp as the key.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Research Monitoring | scheduleTrigger | Daily scheduler entry point | — | Workflow Configuration |  |
| Workflow Configuration | set | Central config variables for endpoints/keywords | Schedule Research Monitoring | Fetch Academic Papers; Fetch Patents; Fetch Datasets; Fetch Technical Blogs |  |
| Fetch Academic Papers | httpRequest | Retrieve papers from ArXiv-like API | Workflow Configuration | Combine All Sources | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Fetch Patents | httpRequest | Retrieve patent results | Workflow Configuration | Combine All Sources | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Fetch Datasets | httpRequest | Retrieve dataset results | Workflow Configuration | Combine All Sources | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Fetch Technical Blogs | httpRequest | Retrieve RSS/blog content | Workflow Configuration | Combine All Sources | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Combine All Sources | merge | Merge 4 sources into one stream | Fetch Academic Papers; Fetch Patents; Fetch Datasets; Fetch Technical Blogs | Rate Limit Delay | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Rate Limit Delay | wait | Slow down downstream processing | Combine All Sources | Knowledge Base Vector Store | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| Knowledge Base Vector Store | vectorStoreInMemory | Insert embedded chunks into in-memory vector store | Rate Limit Delay; Document Loader; OpenAI Embeddings | Research Analysis Agent | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| Document Loader | documentDefaultDataLoader | Convert items into documents for embedding | (implicit from flow); Text Splitter | Knowledge Base Vector Store | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| Text Splitter | textSplitterRecursiveCharacterTextSplitter | Chunk text for embeddings | — | Document Loader | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| OpenAI Embeddings | embeddingsOpenAi | Generate embeddings for documents | — | Knowledge Base Vector Store | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| Research Analysis Agent | agent | Summarize/cluster/gap analysis and produce structured JSON | Knowledge Base Vector Store | Save to Research Database; Manuscript Drafting Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| OpenAI GPT-4 Model | lmChatOpenAi | LLM backend for research analysis | — | Research Analysis Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Research Context Memory | memoryBufferWindow | Short-term memory for analysis agent | — | Research Analysis Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Analysis Output Parser | outputParserStructured | Enforce structured JSON for analysis | — | Research Analysis Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Save to Research Database | googleSheets | Persist analysis results | Research Analysis Agent | — |  |
| Manuscript Drafting Agent | agent | Draft structured manuscript and citations | Research Analysis Agent; Check Validation Results (false) | Validate References API | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| OpenAI GPT-4 Drafting Model | lmChatOpenAi | LLM backend for drafting | — | Manuscript Drafting Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Manuscript Output Parser | outputParserStructured | Enforce structured JSON for manuscript | — | Manuscript Drafting Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Citation Reference Store | vectorStoreInMemory | Expose vector search as a tool for citations | Citation Embeddings | Manuscript Drafting Agent | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Citation Embeddings | embeddingsOpenAi | Embeddings backend for citation retrieval tool | — | Citation Reference Store | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |
| Validate References API | httpRequest | External validation of references | Manuscript Drafting Agent | Plagiarism Check API |  |
| Plagiarism Check API | httpRequest | External plagiarism scoring | Validate References API | Check Validation Results |  |
| Check Validation Results | if | Gate pass/fail and loop back if needed | Plagiarism Check API | Format Adaptation Agent (true); Manuscript Drafting Agent (false) |  |
| Format Adaptation Agent | agent | Adapt manuscript to venue formatting | Check Validation Results (true) | Track Submission Deadlines |  |
| OpenAI Format Model | lmChatOpenAi | LLM backend for formatting | — | Format Adaptation Agent |  |
| Format Output Parser | outputParserStructured | Structured output for formatting results | — | Format Adaptation Agent |  |
| Track Submission Deadlines | googleSheets | Track submission status and deadlines | Format Adaptation Agent | Log Reviewer Feedback |  |
| Log Reviewer Feedback | googleSheets | Store reviewer feedback and decisions | Track Submission Deadlines | Learning Optimization Agent |  |
| Learning Optimization Agent | agent | Derive patterns and recommendations | Log Reviewer Feedback | Store Learning Insights |  |
| OpenAI Learning Model | lmChatOpenAi | LLM backend for learning agent | — | Learning Optimization Agent |  |
| Learning Output Parser | outputParserStructured | Structured output for learning insights | — | Learning Optimization Agent |  |
| Store Learning Insights | googleSheets | Persist learning insights | Learning Optimization Agent | — |  |
| Sticky Note | stickyNote | Comment block | — | — | ## Document Ingestion  / Extracts text from PDFs and academic papers using multiple extraction methods for reliability. / ## Why: / Ensures comprehensive text capture from diverse document formats, preventing data loss from complex academic formatting. |
| Sticky Note1 | stickyNote | Comment block | — | — | ## Prerequisites / Active accounts and API keys for Pinecone, OpenAI  / ## Use Cases / Literature review automation with semantic paper discovery.  / ## Customization / Modify AI model selection logic for domain-specific optimization. / ## Benefits / Reduces research processing time by 60% through automated routing. |
| Sticky Note2 | stickyNote | Comment block | — | — | ## Setup Steps / 1. Configure Pinecone credentials / 2. Add OpenAI API key for GPT-4 access and embeddings / 3. Set up Anthropic Claude API credentials for advanced reasoning / 4. Configure NVIDIA NIM API key for specialized academic models / 5. Connect Google Sheets for query logging and result tracking / 6. Set Gmail OAuth credentials for automated result delivery / 7. Configure webhook URL for query submission endpoint |
| Sticky Note3 | stickyNote | Comment block | — | — | ## How It Works / This workflow automates academic research processing by routing queries through specialized AI models while maintaining contextual memory... (full note content in section 5) |
| Sticky Note4 | stickyNote | Comment block | — | — | ## Vector Storage  / Embeds processed content into Pinecone vector database with metadata tagging. / ## Why: / Enables semantic search across research corpus, allowing contextually relevant retrieval rather than keyword matching. |
| Sticky Note5 | stickyNote | Comment block | — | — | ## AI Routing  / Analyzes query complexity and routes to OpenAI GPT-4, Claude, or NVIDIA models based on task requirements. / ## Why:  / Optimizes cost and performance by matching specialized models to specific research tasks. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `AI-Powered Academic Research Intelligence and Publication Management System`

2. **Add Trigger**
   - Add node: **Schedule Trigger**
   - Name: `Schedule Research Monitoring`
   - Configure: run daily at **06:00** (instance timezone)

3. **Add configuration node**
   - Add node: **Set**
   - Name: `Workflow Configuration`
   - Add fields (strings):
     - `arxivApiUrl`
     - `patentApiUrl`
     - `datasetApiUrl`
     - `blogRssUrl`
     - `searchKeywords`
     - `referenceValidationApiUrl`
     - `plagiarismCheckApiUrl`
   - Set real values for all placeholders (URLs + keywords)
   - Connect: `Schedule Research Monitoring` → `Workflow Configuration`

4. **Add four retrieval HTTP nodes**
   1) **HTTP Request** named `Fetch Academic Papers`
   - URL: `{{$('Workflow Configuration').first().json.arxivApiUrl}}`
   - Query parameters:
     - `search_query` = `{{$('Workflow Configuration').first().json.searchKeywords}}`
     - `max_results` = `50`

   2) **HTTP Request** named `Fetch Patents`
   - URL: `{{$('Workflow Configuration').first().json.patentApiUrl}}`
   - Query parameter: `query` = `{{$('Workflow Configuration').first().json.searchKeywords}}`

   3) **HTTP Request** named `Fetch Datasets`
   - URL: `{{$('Workflow Configuration').first().json.datasetApiUrl}}`
   - Query parameter: `search` = `{{$('Workflow Configuration').first().json.searchKeywords}}`

   4) **HTTP Request** named `Fetch Technical Blogs`
   - URL: `{{$('Workflow Configuration').first().json.blogRssUrl}}`

   - Connect `Workflow Configuration` → each of the four HTTP nodes.

5. **Merge the sources**
   - Add node: **Merge**
   - Name: `Combine All Sources`
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **4**
   - Connect each fetch node output to a different merge input (0..3).

6. **Add a wait node**
   - Add node: **Wait**
   - Name: `Rate Limit Delay`
   - Amount: `2` (seconds)
   - Connect: `Combine All Sources` → `Rate Limit Delay`

7. **Add LangChain ingestion components**
   1) Add node: **Recursive Character Text Splitter**
   - Name: `Text Splitter`
   - Leave defaults or set chunk size/overlap as needed

   2) Add node: **Default Data Loader**
   - Name: `Document Loader`
   - Set “Text splitting mode” to **Custom**
   - Connect: `Text Splitter` → `Document Loader` using the **AI Text Splitter** connection

   3) Add node: **OpenAI Embeddings**
   - Name: `OpenAI Embeddings`
   - Set OpenAI credentials (see step 10)
   - Keep defaults unless you need a specific embedding model

   4) Add node: **Vector Store (In-Memory)**
   - Name: `Knowledge Base Vector Store`
   - Mode: **Insert**
   - Memory key: `vector_store_key`
   - Connect:
     - `Rate Limit Delay` → `Knowledge Base Vector Store` (main)
     - `Document Loader` → `Knowledge Base Vector Store` (AI Document)
     - `OpenAI Embeddings` → `Knowledge Base Vector Store` (AI Embedding)

8. **Add Research Analysis Agent**
   1) Add node: **OpenAI Chat Model**
   - Name: `OpenAI GPT-4 Model`
   - Model: `gpt-4.1-mini`
   - OpenAI credentials set

   2) Add node: **Memory Buffer Window**
   - Name: `Research Context Memory`

   3) Add node: **Structured Output Parser**
   - Name: `Analysis Output Parser`
   - Schema type: **Manual**
   - Paste the schema with fields: `summary, themes, clusters, citations, novelty_score, research_gaps, hypothesis_gaps, methodology`

   4) Add node: **AI Agent**
   - Name: `Research Analysis Agent`
   - Prompt type: **Define**
   - Text: `{{$json}}`
   - System message: (use the one from the workflow) specifying summarization, clustering, gaps, etc.
   - Enable output parser
   - Connect:
     - `Knowledge Base Vector Store` → `Research Analysis Agent` (main)
     - `OpenAI GPT-4 Model` → `Research Analysis Agent` (AI Language Model)
     - `Research Context Memory` → `Research Analysis Agent` (AI Memory)
     - `Analysis Output Parser` → `Research Analysis Agent` (AI Output Parser)

9. **Log analysis to Google Sheets**
   - Add node: **Google Sheets**
   - Name: `Save to Research Database`
   - Operation: **Append or Update**
   - Document ID: your research DB spreadsheet ID
   - Sheet name: `Research Analysis`
   - Map columns: `summary, themes, novelty_score, research_gaps`
   - Matching column: `summary` (recommended to replace with a stable ID)
   - Connect: `Research Analysis Agent` → `Save to Research Database`

10. **Add Manuscript Drafting subchain (agent + citation tool)**
   1) Add node: **OpenAI Chat Model**
   - Name: `OpenAI GPT-4 Drafting Model`
   - Model: `gpt-4.1-mini`

   2) Add node: **Structured Output Parser**
   - Name: `Manuscript Output Parser`
   - Schema with: `title, abstract, introduction, methods, results, discussion, conclusion, references[]`

   3) Add node: **OpenAI Embeddings**
   - Name: `Citation Embeddings`

   4) Add node: **Vector Store (In-Memory)**
   - Name: `Citation Reference Store`
   - Mode: **Retrieve as tool**
   - Memory key: `vector_store_key`
   - Tool description: “Search and retrieve citation references from the knowledge base”
   - Connect: `Citation Embeddings` → `Citation Reference Store` (AI Embedding)

   5) Add node: **AI Agent**
   - Name: `Manuscript Drafting Agent`
   - Text: `{{$json}}`
   - System message: drafting instructions + “Use the Citation Reference Store tool…”
   - Enable output parser
   - Connect:
     - `Research Analysis Agent` → `Manuscript Drafting Agent` (main)
     - `OpenAI GPT-4 Drafting Model` → `Manuscript Drafting Agent` (AI Language Model)
     - `Citation Reference Store` → `Manuscript Drafting Agent` (AI Tool)
     - `Manuscript Output Parser` → `Manuscript Drafting Agent` (AI Output Parser)

11. **Add validation APIs**
   1) Add node: **HTTP Request**
   - Name: `Validate References API`
   - Method: **POST**
   - URL: `{{$('Workflow Configuration').first().json.referenceValidationApiUrl}}`
   - Body type: JSON
   - Body: `{ "references": {{$json.references}} }`
   - Connect: `Manuscript Drafting Agent` → `Validate References API`

   2) Add node: **HTTP Request**
   - Name: `Plagiarism Check API`
   - Method: **POST**
   - URL: `{{$('Workflow Configuration').first().json.plagiarismCheckApiUrl}}`
   - Body type: JSON
   - Body text expression concatenating sections:
     - `{{$json.abstract + ' ' + $json.introduction + ' ' + $json.methods + ' ' + $json.results + ' ' + $json.discussion + ' ' + $json.conclusion}}`
   - Connect: `Validate References API` → `Plagiarism Check API`

   **Important build note:** to make the plagiarism node work reliably, configure the HTTP nodes to *keep the original manuscript fields available* (e.g., by enabling “Include input data” / “Keep input data” depending on your n8n version) or store manuscript in a separate field before calling external APIs.

12. **Add the IF gate**
   - Add node: **IF**
   - Name: `Check Validation Results`
   - Conditions (AND):
     - `{{$('Plagiarism Check API').item.json.plagiarismScore}}` **<** `10`
     - `{{$('Validate References API').item.json.referencesValid}}` **equals** `true`
   - Connect: `Plagiarism Check API` → `Check Validation Results`

13. **Add formatting adaptation**
   1) Add **OpenAI Chat Model** named `OpenAI Format Model` (gpt-4.1-mini)
   2) Add **Structured Output Parser** named `Format Output Parser` with example fields (`formatted_manuscript`, `format_style`, `word_count`)
   3) Add **AI Agent** named `Format Adaptation Agent` with the provided system message
   - Connect:
     - IF **true** → `Format Adaptation Agent`
     - `OpenAI Format Model` → `Format Adaptation Agent` (AI Language Model)
     - `Format Output Parser` → `Format Adaptation Agent` (AI Output Parser)

14. **Create the rework loop**
   - Connect IF **false** output back to `Manuscript Drafting Agent` (main)

15. **Add submission tracking sheet**
   - Add **Google Sheets** node named `Track Submission Deadlines`
   - Operation: Append or Update
   - Spreadsheet ID: your tracking sheet
   - Sheet: `Submission Deadlines`
   - Columns: `title, status, deadline, target_venue`
   - Match on `title`
   - Connect: `Format Adaptation Agent` → `Track Submission Deadlines`

16. **Add reviewer feedback logging**
   - Add **Google Sheets** node named `Log Reviewer Feedback`
   - Sheet: `Reviewer Feedback`
   - Columns: `manuscript_id, reviewer_comments, decision, feedback_date`
   - Match on `manuscript_id`
   - Connect: `Track Submission Deadlines` → `Log Reviewer Feedback`

17. **Add learning optimization**
   1) Add **OpenAI Chat Model** named `OpenAI Learning Model` (gpt-4.1-mini)
   2) Add **Structured Output Parser** named `Learning Output Parser` (example fields)
   3) Add **AI Agent** named `Learning Optimization Agent` with learning system message
   - Connect:
     - `Log Reviewer Feedback` → `Learning Optimization Agent`
     - `OpenAI Learning Model` → `Learning Optimization Agent` (AI Language Model)
     - `Learning Output Parser` → `Learning Optimization Agent` (AI Output Parser)

18. **Store learning insights**
   - Add **Google Sheets** node named `Store Learning Insights`
   - Sheet: `Learning Insights`
   - Columns: `success_patterns, optimization_recommendations, trending_topics`
   - Match on `success_patterns` (recommended to change to a stable key)
   - Connect: `Learning Optimization Agent` → `Store Learning Insights`

19. **Credentials to configure**
   - **OpenAI API** credential used by:
     - OpenAI GPT-4 Model, OpenAI GPT-4 Drafting Model, OpenAI Format Model, OpenAI Learning Model
     - OpenAI Embeddings, Citation Embeddings
   - **Google Sheets OAuth2** credential used by:
     - Save to Research Database, Track Submission Deadlines, Log Reviewer Feedback, Store Learning Insights
   - Any required credentials for external HTTP sources (ArXiv/patent/dataset/blog/validation/plagiarism) if they require headers/tokens.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Document Ingestion… multiple extraction methods…” | Sticky note: Document Ingestion rationale (workflow currently uses HTTP fetch + default loader; no dedicated PDF extraction nodes are present). |
| “Prerequisites: Active accounts and API keys for Pinecone, OpenAI…” | Sticky note: prerequisites/use cases/customization/benefits. Note: the implemented vector store is **in-memory**, not Pinecone. |
| “Setup Steps: Configure Pinecone… Anthropic Claude… NVIDIA NIM… Gmail… webhook…” | Sticky note: setup checklist. Note: this workflow JSON does **not** include Pinecone/Claude/NVIDIA/Gmail/Webhook nodes; it only includes OpenAI + Google Sheets + HTTP + in-memory vector store. |
| “How It Works… accepts research queries via webhook… Pinecone vector storage… routes to Claude/NVIDIA…” | Sticky note: conceptual description. Implementation diverges: actual entry is **Schedule Trigger**, and vector store is **in-memory**. |
| “Vector Storage… Embeds… into Pinecone…” | Sticky note: intent to use Pinecone. Implementation currently uses `vectorStoreInMemory`. |
| “AI Routing… routes to OpenAI GPT-4, Claude, or NVIDIA…” | Sticky note: routing concept. Implementation currently uses OpenAI only, with three separate GPT-4.1-mini-backed agents. |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.