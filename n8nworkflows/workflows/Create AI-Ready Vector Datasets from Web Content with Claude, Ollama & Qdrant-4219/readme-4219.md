Create AI-Ready Vector Datasets from Web Content with Claude, Ollama & Qdrant

https://n8nworkflows.xyz/workflows/create-ai-ready-vector-datasets-from-web-content-with-claude--ollama---qdrant-4219


# Create AI-Ready Vector Datasets from Web Content with Claude, Ollama & Qdrant

---

### 1. Workflow Overview

This n8n workflow automates the process of creating AI-ready vector datasets from web content by integrating web scraping, AI extraction and formatting, embedding generation, vector database storage, and notification dispatch. It is designed for use cases that require reliable extraction of structured data from web pages, transformation into semantic vector embeddings, and persistent storage for AI-powered semantic search or downstream AI applications.

The workflow is logically divided into the following blocks:

- **1.1 Vector Database Preparation**: Ensures the Qdrant vector database collection exists or creates it if missing.
- **1.2 Input Configuration and Web Content Scraping**: Sets target URLs and webhook parameters, then scrapes web content using Scrapeless Web Unlocker.
- **1.3 AI Processing and Data Extraction**: Sends scraped HTML to Claude AI for structured data extraction, validates and enhances results with an AI agent.
- **1.4 Data Formatting and Embeddings Generation**: Parses and formats Claude AI output, generates embeddings using Ollama.
- **1.5 Vector Storage in Qdrant**: Stores the vectorized data in Qdrant with metadata.
- **1.6 Notification Dispatch**: Sends success or error notifications through configured webhooks (Discord, Slack, Teams, Telegram).
- **1.7 Optional Data Export**: Sends formatted extraction data to specified webhook endpoints as files or messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Vector Database Preparation

- **Overview:**  
  This block verifies if the target Qdrant collection exists; if not, it creates the collection to store vector data.

- **Nodes Involved:**  
  - When clicking 'Test workflow' (Manual Trigger)  
  - Check Collection Exists (HTTP Request)  
  - Collection Exists Check (If)  
  - Create Qdrant Collection (HTTP Request)  
  - Set Fields - URL and Webhook URL (Set)  

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - Type: Manual Trigger  
    - Role: Entry point to initiate the workflow manually.  
    - Configuration: No parameters; triggers downstream nodes on manual execution.  
    - Connections: Outputs to "Check Collection Exists".  
    - Edge cases: None.

  - **Check Collection Exists**  
    - Type: HTTP Request  
    - Role: Checks if the Qdrant collection "hacker-news" exists by querying the local Qdrant server.  
    - Configuration: GET request to `http://localhost:6333/collections/hacker-news` with JSON content-type header.  
    - Output: JSON indicating collection status (e.g., "ok" or "not_found").  
    - Connections: Outputs to "Collection Exists Check".  
    - Edge cases: Server unreachable, timeout, 404 (collection not found).

  - **Collection Exists Check**  
    - Type: If  
    - Role: Conditional branching based on the collection existence check.  
    - Configuration: Checks if the "status" field in the HTTP response equals "ok".  
    - True branch: Proceeds to "Set Fields - URL and Webhook URL".  
    - False branch: Proceeds to "Create Qdrant Collection".  
    - Edge cases: Missing or malformed response JSON.

  - **Create Qdrant Collection**  
    - Type: HTTP Request  
    - Role: Creates the "hacker-news" collection in Qdrant with cosine similarity and vector size configuration.  
    - Configuration: PUT request to `http://localhost:6333/collections/hacker-news` with empty JSON body (collection schema is inferred during storage).  
    - Connections: After creation, proceeds to "Set Fields - URL and Webhook URL".  
    - Edge cases: Creation failure, connection errors.

  - **Set Fields - URL and Webhook URL**  
    - Type: Set  
    - Role: Sets parameters including the target URL(s) for scraping, Discord webhook URLs, and Scrapeless API parameters.  
    - Configuration: User-configured URL (e.g., https://news.ycombinator.com/) and webhook URLs; acts as configuration input for scraping requests.  
    - Connections: Outputs to "Scrapeless Web Request".  
    - Edge cases: Missing or invalid URL inputs.

---

#### 2.2 Input Configuration and Web Content Scraping

- **Overview:**  
  This block performs web scraping using Scrapeless Web Unlocker API to fetch fully rendered HTML content from the target URL.

- **Nodes Involved:**  
  - Set Fields - URL and Webhook URL (Set)  
  - Scrapeless Web Request (HTTP Request)  
  - AI Data Checker (Code)  
  - Claude Data extractor (Code)  

- **Node Details:**

  - **Scrapeless Web Request**  
    - Type: HTTP Request  
    - Role: Calls Scrapeless Web Unlocker API to scrape and render the target web page.  
    - Configuration: POST request to `https://api.scrapeless.com/api/v1/unlocker/request` with JSON body including URL, HTTP method, JS rendering enabled, blocking images/fonts/scripts for efficiency, and a small wait time. Header includes `x-api-token` for Scrapeless authentication.  
    - Output: JSON with rendered HTML content.  
    - Connections: Sends output to both "AI Data Checker" and "Claude Data extractor" concurrently.  
    - Edge cases: API key invalid, rate limiting, network issues, malformed response.

  - **AI Data Checker**  
    - Type: Code  
    - Role: Sends scraped HTML to Claude AI to extract main articles and titles into a simplified structured JSON format for quick checks.  
    - Configuration: Calls Claude API with prompt to extract search results with title and articles list. Uses API key for Claude authentication.  
    - Output: Raw Claude API response JSON.  
    - Connections: Outputs to "Expot data webhook" for optional export.  
    - Edge cases: Claude API failure, prompt misunderstanding, response parsing errors.

  - **Claude Data extractor**  
    - Type: Code  
    - Role: Performs detailed extraction of important page content from HTML using Claude AI with instructions for structured JSON output (including metadata, content, structured data, entities).  
    - Configuration: Calls Claude API with a complex prompt instructing extraction and formatting using model `claude-3-7-sonnet-20250219`. API key required.  
    - Output: Claude raw extraction response.  
    - Connections: Outputs to "Claude AI Agent".  
    - Edge cases: API call failure, malformed JSON in response, rate limits.

---

#### 2.3 AI Processing and Data Validation

- **Overview:**  
  This block validates, corrects, and enriches the raw extraction from Claude AI using a secondary AI agent for higher data quality and consistency.

- **Nodes Involved:**  
  - Claude Data extractor (Code)  
  - Claude AI Agent (Code)  
  - Format Claude Output (Code)  

- **Node Details:**

  - **Claude AI Agent**  
    - Type: Code  
    - Role: AI agent that analyzes raw Claude extraction results, fixes JSON errors, fills missing metadata, standardizes format, and quality-checks content before output.  
    - Configuration: Calls Claude API with a validation and enhancement prompt, asking for corrected JSON output. Uses model `claude-3-7-sonnet-20250219`.  
    - Input: Raw JSON from Claude Data extractor.  
    - Output: Enhanced JSON string embedded in `content[0].text` field.  
    - Connections: Outputs to "Format Claude Output".  
    - Edge cases: API failure, JSON parse errors, no JSON found in AI response.

  - **Format Claude Output**  
    - Type: Code  
    - Role: Parses and structures the enhanced JSON from the AI agent, adds technical metadata, prepares data for embedding and vector storage.  
    - Configuration: Extracts JSON from code block or raw text, creates synthetic IDs if missing, flags vector readiness, concatenates searchable content fields.  
    - Input: Enhanced AI JSON response.  
    - Output: Well-structured JSON with extraction metadata and vector_ready flag.  
    - Connections: Outputs to "Ollama Embeddings".  
    - Edge cases: JSON parse errors, missing expected fields, empty content.

---

#### 2.4 Data Formatting and Embeddings Generation

- **Overview:**  
  This block generates semantic vector embeddings of the cleaned and structured content using Ollama's embedding service.

- **Nodes Involved:**  
  - Format Claude Output (Code)  
  - Ollama Embeddings (Code)  

- **Node Details:**

  - **Ollama Embeddings**  
    - Type: Code  
    - Role: Creates a 384-dimensional vector embedding from content text using Ollama's `all-minilm` model for semantic search.  
    - Configuration: Extracts text from multiple possible fields (main_text, summary, searchable_content, metadata) and truncates to 2000 characters. Calls local Ollama embedding API at `http://127.0.0.1:11434/api/embeddings`.  
    - Input: Formatted Claude output JSON.  
    - Output: Adds `vector` array and `vector_info` metadata to JSON.  
    - Connections: Outputs to "Qdrant Vector store".  
    - Edge cases: Ollama service unavailable, malformed responses, empty or insufficient content for embedding.

---

#### 2.5 Vector Storage in Qdrant

- **Overview:**  
  This block stores the generated vector embeddings along with metadata into the Qdrant vector database.

- **Nodes Involved:**  
  - Ollama Embeddings (Code)  
  - Qdrant Vector store (Code)  

- **Node Details:**

  - **Qdrant Vector store**  
    - Type: Code  
    - Role: Stores the embedding vector with metadata in the Qdrant collection "hacker-news". Generates a numeric ID for Qdrant compatibility. Handles collection creation retry if missing.  
    - Configuration: Sends PUT request to `http://127.0.0.1:6333/collections/hacker-news/points` with vector and payload metadata (title, URL, author, language, tags, content snippet). Uses cosine distance.  
    - Input: JSON with vector and metadata from Ollama Embeddings.  
    - Output: Confirmation of storage success or error details.  
    - Connections: Outputs to "Webhook for structured AI agent response".  
    - Edge cases: No vector found, Qdrant service down, collection missing (triggers auto-creation), network errors.

---

#### 2.6 Notification Dispatch

- **Overview:**  
  Sends notifications about the success or failure of vector storage to configured webhook endpoints (Discord, Slack, Teams, Telegram, or custom).

- **Nodes Involved:**  
  - Qdrant Vector store (Code)  
  - Webhook for structured AI agent response (Code)  

- **Node Details:**

  - **Webhook for structured AI agent response**  
    - Type: Code  
    - Role: Parses Qdrant storage results, formats messages for multiple platforms, and sends notifications via webhook URLs for monitoring and alerting.  
    - Configuration: Prepares platform-specific JSON payloads with status, item details, errors if any, and timestamps. Supports Discord embeds, Slack blocks, Teams message cards, Telegram documents, and generic JSON.  
    - Input: Qdrant storage response JSON.  
    - Output: Summary of notification results.  
    - Edge cases: Missing webhook URLs (skips sending), webhook HTTP errors, JSON formatting issues, message size limits.

---

#### 2.7 Optional Data Export

- **Overview:**  
  Exports formatted AI extraction data to various webhook endpoints as text files or messages for further use or archival.

- **Nodes Involved:**  
  - AI Data Checker (Code)  
  - Expot data webhook (Code)  

- **Node Details:**

  - **Expot data webhook**  
    - Type: Code  
    - Role: Parses AI output, prepares a text file containing structured extracted data and raw debug data, and sends it to configured webhooks (Discord, Slack, Linear, Teams, Telegram).  
    - Configuration: Supports file uploads or JSON messages depending on platform. Skips platforms where webhook URL is empty.  
    - Input: Response from "AI Data Checker".  
    - Output: Summary of webhook sending results.  
    - Edge cases: Parsing failures, webhook errors, file size limits, unsupported platforms.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                   | Input Node(s)                    | Output Node(s)                             | Sticky Note                                                                                                                                |
|----------------------------------|---------------------|-------------------------------------------------|---------------------------------|-------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'     | Manual Trigger      | Entry point to start workflow                    | -                               | Check Collection Exists                    |                                                                                                                                           |
| Check Collection Exists           | HTTP Request        | Verify if Qdrant collection exists               | When clicking 'Test workflow'    | Collection Exists Check                    |                                                                                                                                           |
| Collection Exists Check           | If                  | Branch logic: collection exists or create        | Check Collection Exists          | Set Fields - URL and Webhook URL, Create Qdrant Collection |                                                                                                                                           |
| Create Qdrant Collection          | HTTP Request        | Create Qdrant collection if missing               | Collection Exists Check          | Set Fields - URL and Webhook URL           |                                                                                                                                           |
| Set Fields - URL and Webhook URL  | Set                 | Set target URL and webhook parameters             | Collection Exists Check, Create Qdrant Collection | Scrapeless Web Request                 | Configure URL, webhook Discord, and Scrapeless parameters                                                                                  |
| Scrapeless Web Request            | HTTP Request        | Scrape and render target web page                 | Set Fields - URL and Webhook URL | AI Data Checker, Claude Data extractor     | Scrapeless Web Unlocker for web scraping. Note about Scrapeless configuration and API key.                                                |
| AI Data Checker                  | Code                | Quick AI extraction for simplified structured data | Scrapeless Web Request           | Expot data webhook                         |                                                                                                                                           |
| Expot data webhook                | Code                | Export formatted extracted data to webhooks      | AI Data Checker                 | -                                         |                                                                                                                                           |
| Claude Data extractor            | Code                | Detailed AI extraction and structured JSON output | Scrapeless Web Request           | Claude AI Agent                            | AI Data Formatter using Claude 3.7 Sonnet                                                                                                |
| Claude AI Agent                 | Code                | Validate, correct, and enrich extracted data      | Claude Data extractor           | Format Claude Output                       | Data Extraction/Formatting with Claude AI Agent, enhanced validation and correction                                                        |
| Format Claude Output             | Code                | Parse and structure AI JSON response for next steps | Claude AI Agent                | Ollama Embeddings                         |                                                                                                                                           |
| Ollama Embeddings               | Code                | Generate semantic vector embeddings from text     | Format Claude Output            | Qdrant Vector store                       | Vector Database Persistence using Ollama embeddings and Qdrant                                                                           |
| Qdrant Vector store             | Code                | Store vectors and metadata into Qdrant collection | Ollama Embeddings              | Webhook for structured AI agent response  |                                                                                                                                           |
| Webhook for structured AI agent response | Code          | Send notifications to Discord, Slack, Teams, Telegram | Qdrant Vector store           | -                                         | Webhook Discord Handler: Sends formatted responses to Discord, Slack, etc., handles structured and AI responses, JSON formatting         |
| Sticky Note                     | Sticky Note          | Workflow note about overall architecture          | -                              | -                                         | Using Qdrant (Docker) for vector storage. Scrapeless Web Unlocker for web scraping. Uses Claude 3.7 Sonnet. Discord webhook integration. |
| Sticky Note1                    | Sticky Note          | AI Data Formatter note                             | -                              | -                                         | Using Claude 3.7 Sonnet                                                                                                                    |
| Sticky Note2                    | Sticky Note          | Vector Database Persistence notes                  | -                              | -                                         | Using Ollama Embeddings + Qdrant, automatic collection creation, cosine similarity, structured payload storage, numeric IDs, IPv4        |
| Sticky Note3                    | Sticky Note          | Webhook Discord Handler notes                      | -                              | -                                         | Sends formatted responses to Discord, Slack, handles both structured and AI responses, JSON formatted messages                            |
| Sticky Note4                    | Sticky Note          | Data Extraction/Formatting with Claude AI Agent   | -                              | -                                         | Extracts HTML content, formats as structured JSON, direct Claude API calls with proper headers, uses claude-3-7-sonnet-20250219 model     |
| Scrapeless Config Info          | Sticky Note          | Scrapeless Config Info                             | -                              | -                                         | Configure your web scraping parameters at https://app.scrapeless.com/exemple/products/unlocker. Fully customizable settings.              |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create the Manual Trigger Node**  
- Add a "Manual Trigger" node named "When clicking 'Test workflow'".  
- No parameters needed.

**Step 2: Check if Qdrant Collection Exists**  
- Add an "HTTP Request" node named "Check Collection Exists".  
- Set method: GET  
- URL: `http://localhost:6333/collections/hacker-news`  
- Headers: Content-Type: application/json  
- Connect from "When clicking 'Test workflow'".

**Step 3: Add Conditional Branch for Collection Existence**  
- Add an "If" node named "Collection Exists Check".  
- Condition: Expression checking if `{{$node["Check Collection Exists"].json.result ? $node["Check Collection Exists"].json.status : 'not_found'}}` equals "ok".  
- True branch leads to "Set Fields - URL and Webhook URL".  
- False branch leads to "Create Qdrant Collection".

**Step 4: Create Qdrant Collection if Missing**  
- Add an "HTTP Request" node named "Create Qdrant Collection".  
- Method: PUT  
- URL: `http://localhost:6333/collections/hacker-news`  
- Headers: Content-Type: application/json  
- Body: Empty JSON object `{}`  
- Connect from false branch of "Collection Exists Check".  
- Connect output to "Set Fields - URL and Webhook URL".

**Step 5: Set URL and Webhook Parameters**  
- Add a "Set" node named "Set Fields - URL and Webhook URL".  
- Define parameters such as:  
  - Target URL (e.g., https://news.ycombinator.com/)  
  - Discord webhook URL (optional)  
  - Scrapeless API key (for header injection in next node)  
- Connect from both "Collection Exists Check" true branch and "Create Qdrant Collection".

**Step 6: Scrapeless Web Unlocker Request**  
- Add an "HTTP Request" node named "Scrapeless Web Request".  
- Method: POST  
- URL: `https://api.scrapeless.com/api/v1/unlocker/request`  
- Headers: `x-api-token` with Scrapeless API key from credentials or Set node.  
- Body: JSON specifying the target URL, method GET, JS render enabled, blocking images/fonts/scripts, short wait, etc.  
- Connect from "Set Fields - URL and Webhook URL".

**Step 7: AI Data Checker (Optional Quick Extraction)**  
- Add a "Code" node named "AI Data Checker".  
- Script: Send scraped HTML to Claude API with prompt to extract search results and articles.  
- Configure Claude API call with x-api-key header.  
- Connect from "Scrapeless Web Request".  

**Step 8: Export Data Webhook (Optional Data Export)**  
- Add a "Code" node named "Expot data webhook".  
- Script: Format and send extracted AI data as text files or JSON messages to configured webhooks (Discord, Slack, Telegram, etc.).  
- Connect from "AI Data Checker".

**Step 9: Claude Data Extractor (Detailed Extraction)**  
- Add a "Code" node named "Claude Data extractor".  
- Script: Send full HTML to Claude with detailed extraction prompt requesting structured JSON output (metadata, content, entities).  
- Configure Claude API call with x-api-key header.  
- Connect from "Scrapeless Web Request".

**Step 10: Claude AI Agent (Validation and Enhancement)**  
- Add a "Code" node named "Claude AI Agent".  
- Script: Validate, fix, enrich, and standardize raw extraction using another Claude API call with an enhancement prompt.  
- Connect from "Claude Data extractor".

**Step 11: Format Claude Output**  
- Add a "Code" node named "Format Claude Output".  
- Script: Parse AI agent JSON output, add synthetic IDs, technical metadata, concatenate searchable content, evaluate vector readiness.  
- Connect from "Claude AI Agent".

**Step 12: Ollama Embeddings**  
- Add a "Code" node named "Ollama Embeddings".  
- Script: Generate semantic vector of text content using local Ollama embedding API with model "all-minilm".  
- Connect from "Format Claude Output".

**Step 13: Qdrant Vector Store**  
- Add a "Code" node named "Qdrant Vector store".  
- Script: Store vector and metadata into Qdrant collection "hacker-news", retry collection creation if missing.  
- Connect from "Ollama Embeddings".

**Step 14: Webhook Notification**  
- Add a "Code" node named "Webhook for structured AI agent response".  
- Script: Parse Qdrant storage results, construct platform-specific messages (Discord, Slack, Teams, Telegram), send via configured webhook URLs.  
- Connect from "Qdrant Vector store".

**Credentials Setup:**  
- Claude API key with x-api-key header for all Claude nodes.  
- Scrapeless API token for Scrapeless Web Request.  
- Local Ollama embedding API assumed running on `127.0.0.1:11434`.  
- Qdrant vector database running locally on port 6333.  
- Webhook URLs configured as environment variables or in Set node.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Using Qdrant (Docker) for vector storage.                                                       | Vector database backend for storing embeddings.                                                         |
| Scrapeless Web Unlocker for web scraping.                                                       | https://app.scrapeless.com/exemple/products/unlocker — Fully customizable scraping parameters.          |
| Workflow uses Claude 3.7 Sonnet AI model for extraction and validation.                          | Claude API v3.7 with x-api-key authentication.                                                          |
| Discord webhook integration for notifications.                                                  | Webhook URLs configurable in Set node for alerts.                                                       |
| Ollama embeddings with all-minilm model produce 384-dimensional vectors for semantic search.   | Local Ollama embedding API assumed at http://127.0.0.1:11434/api/embeddings.                             |
| Uses cosine similarity distance metric in Qdrant.                                               | Ensures semantic relevance in vector search.                                                            |
| Structured payload includes metadata such as title, URL, author, language, tags, and summary.   | Enables rich filtering and contextual search in Qdrant.                                                 |
| Numeric IDs generated randomly for Qdrant compatibility.                                        | Qdrant requires integer IDs for points.                                                                 |
| Webhook notifications support Discord, Slack, Microsoft Teams, Telegram, and custom endpoints.  | Messages formatted per platform capabilities (embeds, blocks, cards).                                   |
| AI agent validates and enriches extraction to reduce parsing errors and improve data quality.   | Prompts enforce consistent JSON schema.                                                                 |

---

This completes the comprehensive analysis and documentation of the "Create AI-Ready Vector Datasets from Web Content with Claude, Ollama & Qdrant" n8n workflow. The provided information supports understanding, reproduction, and extension of the workflow.