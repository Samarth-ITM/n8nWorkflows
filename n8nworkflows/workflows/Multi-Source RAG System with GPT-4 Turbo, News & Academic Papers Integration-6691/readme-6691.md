Multi-Source RAG System with GPT-4 Turbo, News & Academic Papers Integration

https://n8nworkflows.xyz/workflows/multi-source-rag-system-with-gpt-4-turbo--news---academic-papers-integration-6691


# Multi-Source RAG System with GPT-4 Turbo, News & Academic Papers Integration

### 1. Workflow Overview

This workflow implements a **Multi-Source Retrieval-Augmented Generation (RAG) system** leveraging GPT-4 Turbo to provide intelligent, context-aware content generation from diverse knowledge bases. It is designed for enterprise-grade research and content production scenarios, integrating multiple data sources such as web search engines, academic databases, news APIs, and company internal documents.

**Target Use Cases:**  
- Enterprise knowledge search and summarization  
- Multi-format content generation (Markdown, HTML, JSON, PDF, PPT)  
- Dynamic, language-specific AI responses with style customization  
- Real-time query validation, priority queueing, and user notifications  

**Logical Blocks:**  
- **1.1 Input Reception & Validation:** Captures user queries via a web form, validates and preprocesses inputs.  
- **1.2 Search Routing:** Directs queries to relevant external/internal knowledge sources based on user-selected scope.  
- **1.3 Multi-Source Data Aggregation:** Collects and normalizes search results from all sources into a unified data structure.  
- **1.4 Context Construction for LLM:** Builds a detailed context with query and source metadata tailored for GPT-4 Turbo.  
- **1.5 AI Processing:** Sends the prepared context to GPT-4 Turbo for source-grounded generation with style and language preferences.  
- **1.6 Response Enhancement & Formatting:** Post-processes AI output, adds metadata, and formats the response for delivery.  
- **1.7 Output Delivery:** Responds to the webhook request with the final formatted output and metadata.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  Captures user input through a dynamic form, validates query length, cleans and enriches the input for downstream processing.

- **Nodes Involved:**  
  - 🚀 Advanced RAG Form (formTrigger)  
  - 🔍 Query Preprocessor (code)  
  - Query Preprocessor Info (stickyNote)

- **Node Details:**  

  - **🚀 Advanced RAG Form**  
    - *Type:* formTrigger  
    - *Role:* Entry point capturing query, scope, style, language, output format, user email, and priority.  
    - *Configuration:* Form fields include textarea for query, select dropdowns for search_scope, response_style, language, output_format, priority_level, and an email field for notifications.  
    - *Key expressions:* Captures raw JSON data of user inputs.  
    - *Connections:* Outputs to Query Preprocessor.  
    - *Edge cases:* Missing or too short queries trigger errors downstream.  
    - *Sticky note:* Explains form structure, fields, and features.

  - **🔍 Query Preprocessor**  
    - *Type:* code node  
    - *Role:* Validates query length (minimum 3 characters), normalizes whitespace, extracts keywords, assigns default values for optional fields, and generates a unique session ID and timestamp.  
    - *Key expressions:*  
      - Checks query length and throws error if invalid.  
      - Generates `search_keywords` by filtering words >3 characters.  
      - Maps search scopes to search strategies (e.g., google, scholar, news_api).  
      - Adds processing metadata with timestamps.  
    - *Inputs:* Receives raw form JSON.  
    - *Outputs:* Structured JSON object with enriched query metadata.  
    - *Edge cases:* Query too short, missing fields defaulted, unexpected search_scope fallback to Google.  
    - *Sticky note:* Describes validation and preprocessing logic.

#### 2.2 Search Routing

- **Overview:**  
  Routes the processed query to appropriate search nodes based on the selected search scope.

- **Nodes Involved:**  
  - 🔄 Search Router (if node)  
  - 🌐 Enhanced Web Search (webSearch node)  
  - 🎓 Academic Papers Search (httpRequest to Crossref API)  
  - 📰 News Search API (httpRequest to NewsAPI)  
  - 🏢 Internal Knowledge Search (googleDriveSearch node)  
  - Search Router Info (stickyNote)

- **Node Details:**  

  - **🔄 Search Router**  
    - *Type:* if node  
    - *Role:* Checks if search_scope equals "company_internal" to route accordingly.  
    - *Configuration:* Simple string equality on `search_scope`.  
    - *Connections:*  
      - True branch → 🏢 Internal Knowledge Search  
      - False branch → Parallel outputs to 🎓 Academic Papers Search and 📰 News Search API (per connections)  
    - *Edge cases:* Other scopes fallback to web search or combined sources (multi-source not explicitly shown).  
    - *Sticky note:* Details routing logic per search scope.

  - **🌐 Enhanced Web Search**  
    - *Type:* webSearch (built-in)  
    - *Role:* Conducts web searches using multiple engines (implicit).  
    - *Configuration:* Empty in JSON; likely default.  
    - *Connections:* Not connected in provided JSON but indicated in routing logic.  
    - *Edge cases:* API limits, timeouts.

  - **🎓 Academic Papers Search**  
    - *Type:* httpRequest  
    - *Role:* Queries Crossref API with cleaned query, retrieving top 10 relevant papers sorted by relevance.  
    - *Configuration:* 30s timeout, query parameters dynamically from cleaned query.  
    - *Edge cases:* API failures, empty results, rate limits.

  - **📰 News Search API**  
    - *Type:* httpRequest  
    - *Role:* Queries NewsAPI for relevant news articles, adjusting language parameter based on user selection.  
    - *Configuration:* Header authentication, 30s timeout, query parameters for relevancy, paging, and language.  
    - *Edge cases:* Authentication failures, no articles, API throttling.

  - **🏢 Internal Knowledge Search**  
    - *Type:* googleDriveSearch  
    - *Role:* Searches Google Drive for internal documents when scope is company_internal.  
    - *Configuration:* Not detailed; requires Google Drive OAuth2 credentials.  
    - *Edge cases:* Permission errors, empty results.

#### 2.3 Multi-Source Data Aggregation

- **Overview:**  
  Consolidates results from all search sources into a normalized, enriched data structure for LLM consumption.

- **Nodes Involved:**  
  - 📊 Data Aggregator (code)  

- **Node Details:**  

  - **📊 Data Aggregator**  
    - *Type:* code node  
    - *Role:* Iterates over all incoming search results, extracting relevant fields, assigning type labels (web_content, academic_paper, news_article, internal_document), and computing relevance scores and quality tags.  
    - *Key expressions:*  
      - Normalizes titles, contents, URLs, authors, timestamps.  
      - Calculates total results and average quality score.  
      - Sorts and limits sources to top 15 by relevance.  
      - Adds aggregation timestamps and processing duration.  
    - *Input:* Multiple inputs from all search nodes.  
    - *Output:* Single aggregated JSON object.  
    - *Edge cases:* Missing fields, empty arrays, inconsistent source data.

#### 2.4 Context Construction for LLM

- **Overview:**  
  Builds a comprehensive prompt context for GPT-4 Turbo including query details, source excerpts, metadata, and customized AI instructions.

- **Nodes Involved:**  
  - 🧠 Context Builder (code)  
  - Context Builder Info (stickyNote)

- **Node Details:**  

  - **🧠 Context Builder**  
    - *Type:* code node  
    - *Role:* Concatenates query information, source summaries (with truncation to 1500 chars), and metadata into a formatted string.  
    - *Key expressions:*  
      - Maps response_style to specific LLM instructions (comprehensive, concise, technical, executive_summary, bullet_points, detailed_analysis).  
      - Maps language codes to language instructions for AI.  
      - Creates system and user messages array compliant with OpenAI chat completion format.  
    - *Input:* Aggregated data from previous node.  
    - *Output:* JSON object with `messages` array ready for GPT call and additional context metadata.  
    - *Edge cases:* Empty sources, unsupported languages/styles fallback to defaults.  
    - *Sticky note:* Explains context building and AI prompt design.

#### 2.5 AI Processing

- **Overview:**  
  Sends the prepared context to GPT-4 Turbo to generate a source-grounded, style-adapted response.

- **Nodes Involved:**  
  - 🤖 Advanced LLM Processor (openAi node)  
  - LLM Processor Info (stickyNote)

- **Node Details:**  

  - **🤖 Advanced LLM Processor**  
    - *Type:* openAi node  
    - *Role:* Calls the GPT-4 Turbo (gpt-4-turbo-preview) model with defined parameters: max tokens 2048, temperature 0.3, topP 0.9.  
    - *Key expressions:* Uses messages supplied by Context Builder node.  
    - *Edge cases:* API rate limits, token overflows, network errors.  
    - *Sticky note:* Details model configuration and processing expectations.

#### 2.6 Response Enhancement & Formatting

- **Overview:**  
  Processes the raw AI response, enriches with metadata, quality indicators, and formats output per user preference.

- **Nodes Involved:**  
  - ✨ Response Enhancer (code)  
  - Response Enhancer Info (stickyNote)

- **Node Details:**  

  - **✨ Response Enhancer**  
    - *Type:* code node  
    - *Role:* Extracts AI text, computes response length, token usage, integrates source counts by type, and assesses response quality indicators such as citation presence and completeness.  
    - *Key expressions:*  
      - Formats response into Markdown, HTML (with CSS styling), or JSON structured output depending on user choice.  
      - Adds generation timestamps, session IDs.  
    - *Input:* AI response JSON and context metadata.  
    - *Output:* Enhanced, formatted response ready for delivery.  
    - *Edge cases:* Missing AI response, unsupported formats fallback to markdown.

#### 2.7 Output Delivery

- **Overview:**  
  Returns the final formatted response to the user via webhook.

- **Nodes Involved:**  
  - 📤 Webhook Response (respondToWebhook)  
  - Webhook Response Info (stickyNote)

- **Node Details:**  

  - **📤 Webhook Response**  
    - *Type:* respondToWebhook  
    - *Role:* Sends entire processed data back to the client who submitted the initial form.  
    - *Configuration:* Responds with all incoming data for full transparency.  
    - *Edge cases:* Client disconnects, webhook timeouts.  
    - *Sticky note:* Explains response delivery and integration options.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                  |
|-------------------------|-----------------------|-------------------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| 🚀 Advanced RAG Form     | formTrigger           | Capture user input via dynamic form             | -                            | 🔍 Query Preprocessor          | Describes form fields, advanced input features, multi-format and priority support.                            |
| 🔍 Query Preprocessor    | code                  | Validate and preprocess query                    | 🚀 Advanced RAG Form          | 🔄 Search Router               | Explains validation, keyword extraction, search strategy mapping, session id generation.                      |
| 🔄 Search Router         | if                    | Route queries to appropriate search sources     | 🔍 Query Preprocessor         | 🎓 Academic Papers Search, 📰 News Search API, 🏢 Internal Knowledge Search | Details routing logic by scope (web, academic, news, internal, multi-source).                                  |
| 🌐 Enhanced Web Search   | webSearch             | Conduct web search (not connected in JSON)      | -                            | -                             | Mentioned in routing logic but no active connection in JSON.                                                  |
| 🎓 Academic Papers Search| httpRequest           | Query academic papers API (Crossref)             | 🔄 Search Router              | 📊 Data Aggregator             | Queries Crossref with cleaned query, handles sorting and filtering.                                           |
| 📰 News Search API       | httpRequest           | Query news articles API (NewsAPI)                 | 🔄 Search Router              | 📊 Data Aggregator             | Supports language parameter and relevancy sorting, uses header authentication.                                |
| 🏢 Internal Knowledge Search | googleDriveSearch  | Search internal company documents                 | 🔄 Search Router              | 📊 Data Aggregator             | Uses Google Drive search for internal docs, OAuth2 required.                                                 |
| 📊 Data Aggregator       | code                  | Aggregate, normalize, and score all source data | 🎓 Academic Papers Search, 📰 News Search API, 🏢 Internal Knowledge Search | 🧠 Context Builder           | Combines all results, computes quality score, limits top 15 sources.                                         |
| 🧠 Context Builder       | code                  | Build prompt context for GPT-4 Turbo             | 📊 Data Aggregator            | 🤖 Advanced LLM Processor      | Creates system/user messages, includes style and language instructions, source citations.                     |
| 🤖 Advanced LLM Processor| openAi                 | Generate AI response with GPT-4 Turbo            | 🧠 Context Builder            | ✨ Response Enhancer           | Configured with max tokens, temperature, topP for balanced output.                                           |
| ✨ Response Enhancer     | code                  | Post-process and format AI output                 | 🤖 Advanced LLM Processor     | 📤 Webhook Response            | Adds metadata, formats output as Markdown/HTML/JSON, calculates quality indicators.                          |
| 📤 Webhook Response      | respondToWebhook       | Deliver response back to user interface           | ✨ Response Enhancer          | -                             | Delivers complete formatted response and metadata via webhook.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: formTrigger  
   - Name: "🚀 Advanced RAG Form"  
   - Configure form fields:  
     - textarea (label: query, required)  
     - select (search_scope, required) with options: web_general, academic_papers, news_articles, technical_docs, company_internal, multi_source  
     - select (response_style): comprehensive, concise, technical, executive_summary, bullet_points, detailed_analysis  
     - select (language): italian, english, spanish, french, german  
     - select (output_format): markdown, html, pdf, json_structured, ppt  
     - email (user_email)  
     - select (priority_level): low, medium, high  
   - Set form title and description as per overview.

2. **Add Code Node for Query Preprocessing:**  
   - Name: "🔍 Query Preprocessor"  
   - Paste JavaScript that:  
     - Validates query length (≥3 chars)  
     - Cleans whitespace  
     - Extracts keywords (words >3 chars, max 8)  
     - Defaults missing fields  
     - Generates session_id and timestamp  
     - Defines search strategies per scope  
   - Connect output of formTrigger to this node.

3. **Add If Node for Search Routing:**  
   - Name: "🔄 Search Router"  
   - Condition: If `{{$json["search_scope"]}}` equals "company_internal"  
   - True branch connects to Internal Knowledge Search node  
   - False branch connects to Academic Papers and News Search nodes (parallel execution).

4. **Add Search Nodes:**  
   - **Internal Knowledge Search:**  
     - Type: googleDriveSearch  
     - Configure with Google Drive OAuth2 credentials.  
   - **Academic Papers Search:**  
     - Type: httpRequest  
     - URL: https://api.crossref.org/works  
     - Query parameters: query (from cleaned_query), rows=10, sort=relevance, order=desc  
     - Timeout: 30000 ms  
   - **News Search API:**  
     - Type: httpRequest  
     - URL: https://newsapi.org/v2/everything  
     - Header Authentication with NewsAPI key  
     - Query parameters: q (cleaned_query), sortBy=relevancy, pageSize=10, language based on user input (it/en)  
     - Timeout: 30000 ms  

5. **Add Code Node for Data Aggregation:**  
   - Name: "📊 Data Aggregator"  
   - Implement JavaScript that iterates over all inputs, normalizes data fields, calculates relevance and quality scores, sorts and limits top 15 results, adds metadata.

6. **Add Code Node for Context Builder:**  
   - Name: "🧠 Context Builder"  
   - Compose detailed prompt context including query info, truncated source content, metadata, and AI instructions for style and language.  
   - Output structured messages array for OpenAI API.

7. **Add OpenAI Node for LLM Processing:**  
   - Name: "🤖 Advanced LLM Processor"  
   - Model: gpt-4-turbo-preview  
   - Parameters: maxTokens=2048, temperature=0.3, topP=0.9  
   - Input: messages array from Context Builder

8. **Add Code Node for Response Enhancement:**  
   - Name: "✨ Response Enhancer"  
   - Extract AI response text, compute metadata (tokens, length, sources used), assess quality indicators.  
   - Format output based on user’s output_format (Markdown, HTML, JSON).  
   - Add generation timestamps and session info.

9. **Add Respond to Webhook Node:**  
   - Name: "📤 Webhook Response"  
   - Configure to respond with all incoming data from previous node.

10. **Connect Nodes in Execution Order:**  
    - Form Trigger → Query Preprocessor → Search Router  
    - Search Router branches → respective search nodes → Data Aggregator  
    - Data Aggregator → Context Builder → Advanced LLM Processor → Response Enhancer → Webhook Response

11. **Configure Credentials:**  
    - Google Drive OAuth2 for internal search  
    - NewsAPI API key for news search  
    - OpenAI credentials for GPT-4 Turbo node  

12. **Add Sticky Notes (Optional but recommended):**  
    - Add explanatory sticky notes near major blocks summarizing purpose and key configurations as per provided content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow is tagged for Revenue Optimization, Content Strategy, Trend Monitoring, Dynamic Pricing, and Simple RAG use cases, indicating broad enterprise applicability.                                                               | Workflow tags metadata                                                                                                                                        |
| The form supports multi-language input and multi-format output, enabling diverse user needs and integration scenarios.                                                                                                                    | Form trigger node configuration                                                                                                                               |
| The system enforces source citation in AI responses to maintain transparency and reliability.                                                                                                                                            | Context Builder instructions to LLM                                                                                                                          |
| Usage of GPT-4 Turbo balances creativity and accuracy with moderate temperature and topP settings.                                                                                                                                       | LLM Processor sticky note                                                                                                                                     |
| The workflow includes extensive logging and error handling via JavaScript code nodes with explicit checks and descriptive error messages.                                                                                                | Query Preprocessor and Data Aggregator code nodes                                                                                                            |
| Integration with company internal systems requires appropriate OAuth2 credential provisioning and permissions (Google Drive).                                                                                                           | Internal Knowledge Search node                                                                                                                                |
| NewsAPI requires an API key set under headerAuth authentication in n8n credentials.                                                                                                                                                       | News Search API node parameter                                                                                                                                |
| For further optimization, consider adding retry and rate-limit handling on HTTP nodes to improve robustness.                                                                                                                            | General best practice recommendation                                                                                                                          |
| The workflow is designed for real-time responsiveness with webhook response nodes, suitable for interactive user interfaces or API endpoints.                                                                                           | Webhook Response node documentation                                                                                                                          |
| Sticky notes provide valuable documentation directly within the workflow editor, beneficial for team collaboration and future maintenance.                                                                                              | Sticky notes linked to key nodes                                                                                                                              |

---

**Disclaimer:** The provided content is extracted solely from an n8n workflow automation. It adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is publicly available and legal.