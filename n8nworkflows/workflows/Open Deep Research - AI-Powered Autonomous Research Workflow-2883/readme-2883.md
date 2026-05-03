Open Deep Research - AI-Powered Autonomous Research Workflow

https://n8nworkflows.xyz/workflows/open-deep-research---ai-powered-autonomous-research-workflow-2883


# Open Deep Research - AI-Powered Autonomous Research Workflow

### 1. Workflow Overview

The **Open Deep Research - AI-Powered Autonomous Research Workflow** automates comprehensive research by combining AI-driven query generation, web search, content extraction, evaluation, iterative refinement, and final report compilation. It is designed to assist researchers, analysts, journalists, developers, and students in efficiently gathering and synthesizing high-quality information on any given topic.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception:** Captures the research topic submitted by the user via chat.
- **1.2 AI Query Generation:** Uses a language model to generate up to four refined search queries based on the user input.
- **1.3 Search Execution and Result Processing:** Executes Google searches via SerpAPI for each query, formats the results, and batches them for processing.
- **1.4 Webpage Content Extraction:** Uses Jina AI to scrape and summarize webpage content from URLs obtained in search results.
- **1.5 AI-Powered Content Evaluation:** Extracts relevant context from the scraped content using an AI agent with memory buffering.
- **1.6 Iterative Refinement (Implicit):** The workflow structure supports iterative query refinement if content quality is insufficient (though explicit iteration control is managed by AI agents and memory buffers).
- **1.7 Final Report Generation:** Compiles a structured markdown report summarizing findings, including citations.
- **1.8 Auxiliary Tools and Setup:** Includes Wikipedia information fetching and sticky notes for setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception

- **Overview:**  
  Captures the initial research topic from the user through a chat interface, triggering the workflow.

- **Nodes Involved:**  
  - Chat Message Trigger

- **Node Details:**  
  - **Chat Message Trigger**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point; listens for user chat input containing the research topic.  
    - Configuration: Default options; webhook enabled for receiving chat messages.  
    - Inputs: None (trigger node)  
    - Outputs: Passes user input as `chatInput` JSON field to next node.  
    - Edge Cases: Missing or empty user input may cause downstream nodes to generate empty queries or errors.  
    - Version: 1.1

---

#### 2.2 AI Query Generation

- **Overview:**  
  Generates up to four precise and distinct search queries from the user's original query to guide the research process.

- **Nodes Involved:**  
  - Generate Search Queries using LLM  
  - LLM Response Provider (OpenRouter)  
  - Parse and Chunk JSON Data

- **Node Details:**  
  - **Generate Search Queries using LLM**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Uses a language model chain to produce refined search queries.  
    - Configuration:  
      - Input text: User query from `Chat Message Trigger` (`chatInput`).  
      - Prompt: Expert research assistant instructed to return a JSON list of up to four queries.  
      - Output: Raw JSON string with queries.  
    - Inputs: User query string  
    - Outputs: Raw JSON string of queries to be parsed.  
    - Edge Cases: Model may return malformed JSON or fewer than four queries.  
    - Version: 1.5  
    - Sub-workflow: Uses LLM Response Provider as language model backend.

  - **LLM Response Provider (OpenRouter)**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: Provides AI model responses using OpenRouter API (Google Gemini 2.0 Flash).  
    - Configuration:  
      - Model: `google/gemini-2.0-flash-001`  
      - Credentials: OpenRouter API key configured securely.  
    - Inputs: Prompts from chain LLM nodes.  
    - Outputs: AI-generated text responses.  
    - Edge Cases: API rate limits, authentication errors, or timeouts.  
    - Version: 1

  - **Parse and Chunk JSON Data**  
    - Type: `n8n-nodes-base.code` (JavaScript)  
    - Role: Parses the JSON string of queries and splits them into up to four chunks for batching.  
    - Configuration:  
      - Removes markdown code block formatting if present.  
      - Parses JSON array and divides into chunks.  
      - Returns array of chunked queries for parallel processing.  
    - Inputs: Raw JSON string from LLM output.  
    - Outputs: Array of JSON objects each containing a chunk of queries.  
    - Edge Cases: JSON parse errors, non-array JSON, empty input.  
    - Version: 2

---

#### 2.3 Search Execution and Result Processing

- **Overview:**  
  Executes Google searches for each query chunk using SerpAPI, formats the organic search results, and prepares data for content extraction.

- **Nodes Involved:**  
  - Split Data for SerpAPI Batching  
  - Perform SerpAPI Search Request  
  - Format SerpAPI Organic Results

- **Node Details:**  
  - **Split Data for SerpAPI Batching**  
    - Type: `n8n-nodes-base.splitInBatches`  
    - Role: Processes each chunk of queries sequentially or in batches to avoid API overload.  
    - Configuration: Default batching options.  
    - Inputs: Chunked queries from previous block.  
    - Outputs: Individual query chunks for HTTP requests.  
    - Edge Cases: Large batch sizes may cause API rate limits.  
    - Version: 3

  - **Perform SerpAPI Search Request**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Calls SerpAPI Google Search API with each query chunk.  
    - Configuration:  
      - URL: `https://serpapi.com/search`  
      - Query Parameters:  
        - `q`: Search query chunk  
        - `api_key`: SerpAPI API key (secured credential)  
        - `engine`: `google`  
    - Inputs: Query chunk from batch splitter.  
    - Outputs: Raw JSON search results.  
    - Edge Cases: API key invalid, rate limits, network errors, empty results.  
    - Version: 4.2

  - **Format SerpAPI Organic Results**  
    - Type: `n8n-nodes-base.code` (JavaScript)  
    - Role: Extracts and formats organic search results from SerpAPI response for downstream processing.  
    - Configuration:  
      - Extracts `title`, `url`, and `source` from each organic result.  
      - Handles missing fields with default placeholders.  
      - Returns array of formatted results.  
    - Inputs: Raw SerpAPI JSON response.  
    - Outputs: Array of formatted search result objects.  
    - Edge Cases: No organic results found, malformed response.  
    - Version: 2

---

#### 2.4 Webpage Content Extraction

- **Overview:**  
  Extracts and summarizes webpage content from URLs obtained in search results using Jina AI API.

- **Nodes Involved:**  
  - Split Data for Jina AI Batching  
  - Perform Jina AI Analysis Request

- **Node Details:**  
  - **Split Data for Jina AI Batching**  
    - Type: `n8n-nodes-base.splitInBatches`  
    - Role: Batches formatted search results for sequential processing by Jina AI.  
    - Configuration: Default batching options.  
    - Inputs: Formatted search results.  
    - Outputs: Individual search result items for HTTP requests.  
    - Edge Cases: Large batch sizes may cause API rate limits.  
    - Version: 3

  - **Perform Jina AI Analysis Request**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Sends HTTP requests to Jina AI API to extract and summarize webpage content.  
    - Configuration:  
      - URL: Constructed dynamically as `https://r.jina.ai/{{ $json.url }}` where `url` is from search results.  
      - Authentication: HTTP header with Jina AI API key (secured credential).  
    - Inputs: Single search result item with URL.  
    - Outputs: Extracted webpage content and summary.  
    - Edge Cases: Invalid URLs, API authentication errors, network timeouts.  
    - Version: 4.2

---

#### 2.5 AI-Powered Content Evaluation

- **Overview:**  
  Uses an AI agent with memory buffering to extract relevant context from the webpage content, evaluating its relevance and credibility.

- **Nodes Involved:**  
  - Extract Relevant Context via LLM  
  - LLM Memory Buffer (Input Context)

- **Node Details:**  
  - **Extract Relevant Context via LLM**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: AI agent extracts relevant information from webpage content based on user queries.  
    - Configuration:  
      - Input text includes user queries and webpage content.  
      - System message instructs extraction of relevant context only, no commentary.  
    - Inputs: Webpage content from Jina AI, user queries from chunked data.  
    - Outputs: Extracted relevant context as plain text.  
    - Edge Cases: AI may miss relevant info or hallucinate content; input formatting errors.  
    - Version: 1.7  
    - Memory: Connected to **LLM Memory Buffer (Input Context)** for session context retention.

  - **LLM Memory Buffer (Input Context)**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window memory of input context to support iterative refinement.  
    - Configuration:  
      - Session key: `my_test_session` (custom key)  
      - Context window length: 20 messages  
    - Inputs: AI agent input context.  
    - Outputs: Context-aware input for AI agent.  
    - Edge Cases: Memory overflow or session key conflicts.  
    - Version: 1.3

---

#### 2.6 Final Report Generation

- **Overview:**  
  Compiles a comprehensive, structured markdown report from the extracted contexts and original user query.

- **Nodes Involved:**  
  - Generate Comprehensive Research Report  
  - LLM Memory Buffer (Report Context)  
  - Fetch Wikipedia Information (optional auxiliary)

- **Node Details:**  
  - **Generate Comprehensive Research Report**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: AI agent synthesizes extracted contexts into a detailed research report with headings and citations.  
    - Configuration:  
      - Input text: Merged extracted contexts.  
      - System message: Instructions to generate a markdown report with clear structure and no unnecessary commentary.  
    - Inputs: Extracted contexts from previous block.  
    - Outputs: Markdown formatted research report.  
    - Edge Cases: AI hallucination, incomplete synthesis, formatting errors.  
    - Version: 1.7  
    - Memory: Connected to **LLM Memory Buffer (Report Context)** for session continuity.

  - **LLM Memory Buffer (Report Context)**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains memory of report generation context for iterative improvements or follow-ups.  
    - Configuration: Same as input context buffer.  
    - Edge Cases: Same as input context buffer.  
    - Version: 1.3

  - **Fetch Wikipedia Information**  
    - Type: `@n8n/n8n-nodes-langchain.toolWikipedia`  
    - Role: Optional tool to fetch additional background information from Wikipedia to enrich the report.  
    - Configuration: Default.  
    - Inputs: Can be triggered by AI agents as needed.  
    - Outputs: Wikipedia content for report inclusion.  
    - Edge Cases: Wikipedia API downtime or missing articles.  
    - Version: 1

---

#### 2.7 Setup and Documentation Notes

- **Nodes Involved:**  
  - Sticky Note: SerpAPI Setup  
  - Sticky Note: Jina AI Setup  
  - Sticky Note: OpenRouter API Setup

- **Node Details:**  
  - **Sticky Notes**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Provide setup instructions and API key management guidance for SerpAPI, Jina AI, and OpenRouter integrations.  
    - Content includes links to API key management portals and security best practices.  
    - Positioned near relevant nodes for user clarity.  
    - No inputs or outputs.  
    - Version: 1

---

### 3. Summary Table

| Node Name                      | Node Type                                | Functional Role                         | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                      |
|--------------------------------|-----------------------------------------|---------------------------------------|-------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Chat Message Trigger            | @n8n/n8n-nodes-langchain.chatTrigger   | User input reception                   | None                          | Generate Search Queries using LLM  |                                                                                                |
| Generate Search Queries using LLM | @n8n/n8n-nodes-langchain.chainLlm     | AI query generation                    | Chat Message Trigger           | Parse and Chunk JSON Data          |                                                                                                |
| LLM Response Provider (OpenRouter) | @n8n/n8n-nodes-langchain.lmChatOpenRouter | AI language model backend              | Generate Search Queries using LLM | Generate Search Queries using LLM, Extract Relevant Context via LLM, Generate Comprehensive Research Report | Sticky Note: OpenRouter API Setup - https://openrouter.ai/settings/keys                        |
| Parse and Chunk JSON Data       | n8n-nodes-base.code                     | Parse and chunk queries for batching  | Generate Search Queries using LLM | Split Data for SerpAPI Batching    |                                                                                                |
| Split Data for SerpAPI Batching | n8n-nodes-base.splitInBatches           | Batch processing of search queries    | Parse and Chunk JSON Data      | Perform SerpAPI Search Request, Format SerpAPI Organic Results | Sticky Note: SerpAPI Setup - https://serpapi.com/manage-api-key                               |
| Perform SerpAPI Search Request  | n8n-nodes-base.httpRequest               | Execute Google search via SerpAPI     | Split Data for SerpAPI Batching | Split Data for SerpAPI Batching    | Sticky Note: SerpAPI Setup - https://serpapi.com/manage-api-key                               |
| Format SerpAPI Organic Results  | n8n-nodes-base.code                     | Format search results for processing  | Perform SerpAPI Search Request | Split Data for Jina AI Batching    | Sticky Note: SerpAPI Setup - https://serpapi.com/manage-api-key                               |
| Split Data for Jina AI Batching | n8n-nodes-base.splitInBatches           | Batch processing of search results    | Format SerpAPI Organic Results | Extract Relevant Context via LLM, Perform Jina AI Analysis Request | Sticky Note: Jina AI Setup - https://jina.ai/api-dashboard/key-manager                        |
| Perform Jina AI Analysis Request | n8n-nodes-base.httpRequest               | Extract webpage content via Jina AI   | Split Data for Jina AI Batching | Split Data for Jina AI Batching    | Sticky Note: Jina AI Setup - https://jina.ai/api-dashboard/key-manager                        |
| Extract Relevant Context via LLM | @n8n/n8n-nodes-langchain.agent          | AI content evaluation and extraction  | Split Data for Jina AI Batching | Generate Comprehensive Research Report |                                                                                                |
| LLM Memory Buffer (Input Context) | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain input context memory          | None                          | Extract Relevant Context via LLM   |                                                                                                |
| Generate Comprehensive Research Report | @n8n/n8n-nodes-langchain.agent          | Final report generation                | Extract Relevant Context via LLM, Fetch Wikipedia Information | None                              |                                                                                                |
| LLM Memory Buffer (Report Context) | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain report context memory         | None                          | Generate Comprehensive Research Report |                                                                                                |
| Fetch Wikipedia Information     | @n8n/n8n-nodes-langchain.toolWikipedia  | Auxiliary info fetching from Wikipedia | None                          | Generate Comprehensive Research Report |                                                                                                |
| Sticky Note: SerpAPI Setup      | n8n-nodes-base.stickyNote                | Setup instructions for SerpAPI        | None                          | None                             | ## SerpAPI Setup Instructions 1. Obtain your API key from https://serpapi.com/manage-api-key. 2. Save your API key securely in n8n credentials (do not use plain text). |
| Sticky Note: Jina AI Setup      | n8n-nodes-base.stickyNote                | Setup instructions for Jina AI        | None                          | None                             | ## Jina AI Setup Instructions 1. Obtain your API key from https://jina.ai/api-dashboard/key-manager. 2. Configure your Jina AI credential in n8n to ensure secure API access. |
| Sticky Note: OpenRouter API Setup | n8n-nodes-base.stickyNote                | Setup instructions for OpenRouter API | None                          | None                             | ## OpenRouter API Setup Instructions 1. Obtain your API key from https://openrouter.ai/settings/keys. 2. Set up your OpenRouter credential in n8n for secure integration. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Message Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: Receive user research topic input.  
   - Configure webhook as needed.

2. **Create Generate Search Queries using LLM Node**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Connect input from Chat Message Trigger.  
   - Set parameter `text` to: `=User Query: {{ $('Chat Message Trigger').item.json.chatInput }}`  
   - Add system message prompt instructing to generate up to four distinct search queries as JSON list.  
   - Use LLM Response Provider (OpenRouter) as language model backend.

3. **Create LLM Response Provider (OpenRouter) Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
   - Configure with OpenRouter API credentials (API key from https://openrouter.ai/settings/keys).  
   - Model: `google/gemini-2.0-flash-001`.

4. **Create Parse and Chunk JSON Data Node**  
   - Type: `n8n-nodes-base.code`  
   - Connect input from Generate Search Queries using LLM.  
   - Add JavaScript code to parse JSON string and split into up to four chunks.

5. **Create Split Data for SerpAPI Batching Node**  
   - Type: `n8n-nodes-base.splitInBatches`  
   - Connect input from Parse and Chunk JSON Data.

6. **Create Perform SerpAPI Search Request Node**  
   - Type: `n8n-nodes-base.httpRequest`  
   - Connect input from Split Data for SerpAPI Batching.  
   - Configure URL: `https://serpapi.com/search`  
   - Query parameters:  
     - `q`: `={{ $('Parse and Chunk JSON Data').item.json.chunk }}`  
     - `api_key`: Use SerpAPI credential key  
     - `engine`: `google`  
   - Set HTTP method to GET.

7. **Create Format SerpAPI Organic Results Node**  
   - Type: `n8n-nodes-base.code`  
   - Connect input from Perform SerpAPI Search Request.  
   - Add JavaScript code to extract and format organic results (title, url, source).

8. **Create Split Data for Jina AI Batching Node**  
   - Type: `n8n-nodes-base.splitInBatches`  
   - Connect input from Format SerpAPI Organic Results.

9. **Create Perform Jina AI Analysis Request Node**  
   - Type: `n8n-nodes-base.httpRequest`  
   - Connect input from Split Data for Jina AI Batching.  
   - Configure URL dynamically: `=https://r.jina.ai/{{ $json.url }}`  
   - Set authentication type to HTTP Header Auth with Jina AI API key credential (from https://jina.ai/api-dashboard/key-manager).  
   - HTTP method: GET or POST as per Jina AI API requirements.

10. **Create Extract Relevant Context via LLM Node**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Connect input from Split Data for Jina AI Batching (also from Perform Jina AI Analysis Request).  
    - Configure text input to include user queries and webpage content.  
    - Set system message to instruct extraction of relevant context only.  
    - Connect to LLM Memory Buffer (Input Context) node.

11. **Create LLM Memory Buffer (Input Context) Node**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Configure session key as `my_test_session`.  
    - Set context window length to 20.

12. **Create Generate Comprehensive Research Report Node**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Connect input from Extract Relevant Context via LLM.  
    - Configure system message to generate a structured markdown report with headings and citations.  
    - Connect to LLM Memory Buffer (Report Context) node.

13. **Create LLM Memory Buffer (Report Context) Node**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Configure session key as `my_test_session`.  
    - Set context window length to 20.

14. **Create Fetch Wikipedia Information Node (Optional)**  
    - Type: `@n8n/n8n-nodes-langchain.toolWikipedia`  
    - Connect output to Generate Comprehensive Research Report node as auxiliary input.

15. **Add Sticky Notes for Setup Instructions**  
    - Create three sticky note nodes near relevant nodes:  
      - SerpAPI Setup: Instructions and link to https://serpapi.com/manage-api-key  
      - Jina AI Setup: Instructions and link to https://jina.ai/api-dashboard/key-manager  
      - OpenRouter API Setup: Instructions and link to https://openrouter.ai/settings/keys

16. **Connect all nodes according to the described flow**  
    - Ensure batch splitters feed into respective HTTP requests and AI agents.  
    - Ensure memory buffers are connected to AI agent nodes for context retention.

17. **Configure Credentials**  
    - SerpAPI API key stored securely in n8n credentials.  
    - Jina AI API key stored securely in n8n credentials.  
    - OpenRouter API key stored securely in n8n credentials.

18. **Test the workflow**  
    - Trigger with a sample research topic.  
    - Verify query generation, search results, content extraction, and final report generation.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automates deep research with iterative AI-driven query refinement and content analysis. | Workflow purpose summary                         |
| Estimated setup time: 10-15 minutes                                                              | Setup time estimate                              |
| Required API keys: SerpAPI, Jina AI, OpenRouter                                                  | API key requirements                            |
| Recommended enhancements: Add Google Drive or Notion integration for report saving                | Suggested future improvements                    |
| Sticky notes included in workflow provide setup instructions for API integrations                 | User guidance within n8n                         |
| Ideal users: Researchers, journalists, developers, students                                      | Target audience                                  |
| Open source and free to use                                                                      | Licensing and availability                       |
| SerpAPI documentation: https://serpapi.com/                                                     | External API documentation                        |
| Jina AI API key management: https://jina.ai/api-dashboard/key-manager                            | External API documentation                        |
| OpenRouter API keys: https://openrouter.ai/settings/keys                                        | External API documentation                        |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and extend the Open Deep Research workflow with clear insight into each node’s role, configuration, and integration points.