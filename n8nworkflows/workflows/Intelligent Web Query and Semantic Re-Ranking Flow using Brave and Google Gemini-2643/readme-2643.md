Intelligent Web Query and Semantic Re-Ranking Flow using Brave and Google Gemini

https://n8nworkflows.xyz/workflows/intelligent-web-query-and-semantic-re-ranking-flow-using-brave-and-google-gemini-2643


# Intelligent Web Query and Semantic Re-Ranking Flow using Brave and Google Gemini

### 1. Workflow Overview

This workflow implements an **Intelligent Web Query and Semantic Re-Ranking System** designed to automate detailed and precise web searches, semantically refine queries, rerank search results based on relevance, and deliver structured, actionable output via a webhook. It is optimized for real-time data retrieval and market research, leveraging AI models (Google Gemini primarily) and the free Brave Search API to balance cost-efficiency and high-quality results.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Capturing the user’s research question via a Webhook node.
- **1.2 Semantic Query Generation:** Using AI (Google Gemini) to transform the user’s raw query into an expert-level, context-aware search query.
- **1.3 Web Search Execution:** Submitting the refined query to the Brave Search API to fetch raw search results.
- **1.4 Result Aggregation and Preparation:** Combining raw search results into a text format suitable for AI processing.
- **1.5 Semantic Re-Ranking:** Applying AI (Google Gemini) to rank search results by relevance and extract key information.
- **1.6 Structured Output Generation:** Parsing the AI re-ranking output into a consistent JSON structure.
- **1.7 Real-Time Reporting:** Delivering the structured top 10 ranked results back to the requester via a webhook response node.

Supporting nodes provide date/time context and parsing enhancements, with sticky notes offering setup instructions and customization guidance.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block captures the user’s input query via a webhook, enabling integration with external systems and triggering the workflow.

**Nodes Involved:**  
- Webhook  
- Date & Time

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point capturing incoming HTTP requests with a JSON payload containing the user's "Research Question".  
  - Configuration: Path set to a unique identifier; response mode set to "responseNode" for responding after processing.  
  - Inputs: External HTTP request with JSON body (expects `query["Research Question"]`).  
  - Outputs: JSON object with user query passed downstream.  
  - Edge cases: Missing or malformed input can cause failure or unexpected behavior; ensure proper client request formatting.

- **Date & Time**  
  - Type: Date/Time node  
  - Role: Provides current date/time context for downstream reasoning, aiding time-sensitive query refinement and content freshness assessment.  
  - Configuration: Default (current execution time).  
  - Inputs: From Webhook, triggered downstream.  
  - Outputs: Current date/time in ISO format under key `currentDate`.  
  - Edge cases: Generally robust, but server time drift may affect accuracy.

---

#### 1.2 Semantic Query Generation

**Overview:**  
Transforms the raw user query into a refined, context-aware search query optimized for web search engines, using multi-step AI reasoning chains.

**Nodes Involved:**  
- Semantic Search -Query Maker  
- Auto-fixing Output Parser  
- Structured Output Parser1

**Node Details:**

- **Semantic Search -Query Maker**  
  - Type: LangChain AI LLM Chain Node (Google Gemini)  
  - Role: Performs multi-chain reasoning to break down the user question, explore context, and generate a concise, effective search query.  
  - Configuration: Custom prompt guiding three chains of thought (keyword extraction, context exploration, query refinement), with examples of optimized queries to avoid zero-result long-tail searches.  
  - Inputs: User query JSON and current date/time from upstream nodes.  
  - Outputs: JSON with `reasoning_summary` and `final_search_query`.  
  - Edge cases: API errors, prompt misinterpretation, overly generic or too specific queries causing poor downstream results.

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser (Auto-fixing)  
  - Role: Parses and auto-corrects the AI output to maintain structural consistency and handle minor formatting errors.  
  - Inputs: Raw AI output from Semantic Search -Query Maker.  
  - Outputs: Cleaned JSON for subsequent parsing.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and enforces JSON schema with `reasoning_summary` and `final_search_query` keys for consistent downstream data handling.  
  - Inputs: Output from the Auto-fixing Output Parser.  
  - Outputs: Strictly validated JSON object.

---

#### 1.3 Web Search Execution

**Overview:**  
Executes the refined search query against the Brave Search API to retrieve web search results.

**Nodes Involved:**  
- Query

**Node Details:**

- **Query**  
  - Type: HTTP Request  
  - Role: Calls Brave Search API with refined query to fetch search results.  
  - Configuration:  
    - URL: `https://api.search.brave.com/res/v1/web/search`  
    - Query Parameter: `q` set dynamically to `final_search_query` from previous block.  
    - Headers include `Accept: application/json`, `Accept-Encoding: gzip`, and `X-Subscription-Token` set to the user's Brave API key (must replace placeholder).  
  - Inputs: Receives refined search query JSON.  
  - Outputs: JSON containing raw search results under `web.results`.  
  - Edge cases: API key missing/invalid, rate limiting, network timeouts, malformed queries causing no results.

---

#### 1.4 Result Aggregation and Preparation

**Overview:**  
Aggregates raw search results (titles, URLs, descriptions) into a formatted text blob suitable for input to the AI reranker.

**Nodes Involved:**  
- Query-1 Combined

**Node Details:**

- **Query-1 Combined**  
  - Type: Code (JavaScript)  
  - Role: Iterates through the Brave Search results array, concatenating title, URL, and description into a single string with clear labeling for each result.  
  - Configuration: Custom JS code looping over search results, handling missing fields gracefully with placeholders ("No Title", "No URL", "No Description").  
  - Inputs: Raw JSON from Query node with search results.  
  - Outputs: Single JSON object with aggregated text under key `aggregated_text`.  
  - Edge cases: Empty or malformed results array; ensures output is a readable string even if no results found.

---

#### 1.5 Semantic Re-Ranking

**Overview:**  
Re-ranks the aggregated search results semantically using AI (Google Gemini), extracting relevant information and generating a ranked list of top URLs.

**Nodes Involved:**  
- Semantic Search - Result Re-Ranker  
- Auto-fixing Output Parser6  
- Structured Output Parser2

**Node Details:**

- **Semantic Search - Result Re-Ranker**  
  - Type: LangChain AI LLM Chain Node (Google Gemini)  
  - Role:  
    - Analyzes the user’s original query and the AI-generated search query.  
    - Applies multi-step reasoning to rank the top 10 URLs by relevance, extract key info, and suggest query improvements if needed.  
    - Uses a detailed multi-step prompt outlining objectives, process, and output JSON schema.  
  - Configuration: Includes instructions to think aloud, consider today's date, and handle cases of no or insufficient results.  
  - Inputs:  
    - Aggregated text from Query-1 Combined (search results).  
    - Original user query and refined query for contextual evaluation.  
    - Current date/time to assess timeliness.  
  - Outputs: JSON with step-by-step chain of thought, top 10 ranked URLs (title, link, description), and extracted info.  
  - Edge cases:  
    - AI generation errors or hallucinations.  
    - Incomplete or insufficient search results affecting ranking quality.  
    - API latency or quota limits.

- **Auto-fixing Output Parser6**  
  - Type: LangChain Output Parser (Auto-fixing)  
  - Role: Ensures structured AI output is corrected for formatting errors before final parsing.  
  - Inputs: Raw AI output from re-ranker.  
  - Outputs: Corrected JSON.

- **Structured Output Parser2**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates final ranked results JSON against a schema requiring chain_of_thought and top 10 ranked URLs with title, link, and description fields, plus extracted info.  
  - Inputs: Output from Auto-fixing Output Parser6.  
  - Outputs: Clean, validated JSON used for final response.

---

#### 1.6 Real-Time Reporting

**Overview:**  
Sends the structured, ranked search results as a JSON response back to the original webhook caller.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Returns the final, structured top 10 ranked search results and extracted information as a JSON payload to the user or calling system.  
  - Configuration: Constructs nested JSON using expressions to pull data from the Semantic Search - Result Re-Ranker output node, providing each ranked URL’s title, link, and description, plus extracted information.  
  - Inputs: Final validated ranked results JSON.  
  - Outputs: HTTP JSON response to webhook invoker.  
  - Edge cases: If re-ranker output is missing fields, response may have null or empty values; ensure graceful handling.

---

#### Supporting Nodes and Notes

- **Sticky Notes**: Provide detailed setup instructions for Brave API key setup, webhook configuration, and model customization options. These notes guide users on required API keys, where to input them, and how to adapt nodes.

- **Alternate AI Models Nodes** (OpenAI Chat Model, Anthropic Chat Model) are present but not connected; these serve as templates for replacing Google Gemini with other LLM providers if desired.

- **Webhook Call Node** (HTTP Request) included as an example to send test queries to the webhook.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                            | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                          |
|-------------------------------|-----------------------------------------|--------------------------------------------|---------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook                                 | Entry point capturing user query           | External HTTP request            | Date & Time                        | "If you require to change this Node to Webhook Or any Other Item: ..."                             |
| Date & Time                   | Date/Time                               | Provides current date/time context          | Webhook                        | Semantic Search -Query Maker       |                                                                                                    |
| Semantic Search -Query Maker  | LangChain Chain LLM (Google Gemini)    | AI-powered semantic query generation       | Date & Time                    | Auto-fixing Output Parser          |                                                                                                    |
| Auto-fixing Output Parser     | LangChain Output Parser (Auto-fixing)  | Ensures clean JSON output from AI           | Semantic Search -Query Maker   | Structured Output Parser1           |                                                                                                    |
| Structured Output Parser1     | LangChain Structured Output Parser      | Validates semantic query JSON structure    | Auto-fixing Output Parser      | Query                            |                                                                                                    |
| Query                        | HTTP Request                            | Executes web search against Brave API      | Structured Output Parser1       | Query-1 Combined                   | "Step 1. Set Up a Free Brave Web Search Query API Key" with instructions and API key placement      |
| Query-1 Combined              | Code (JavaScript)                       | Aggregates search results into text blob   | Query                         | Semantic Search - Result Re-Ranker |                                                                                                    |
| Semantic Search - Result Re-Ranker | LangChain Chain LLM (Google Gemini) | AI-powered semantic reranking and info extraction | Query-1 Combined               | Respond to Webhook                | "Customized Models to Replace" note indicates possible alternative LLMs                              |
| Auto-fixing Output Parser6    | LangChain Output Parser (Auto-fixing)  | Cleans AI reranker output                    | Semantic Search - Result Re-Ranker | Structured Output Parser2         |                                                                                                    |
| Structured Output Parser2     | LangChain Structured Output Parser      | Validates reranker output JSON structure    | Auto-fixing Output Parser6     | Respond to Webhook                 |                                                                                                    |
| Respond to Webhook            | Respond to Webhook                      | Returns final structured results as JSON   | Structured Output Parser2       | None                            |                                                                                                    |
| Sticky Note4                 | Sticky Note                            | Instructions for Brave API Key setup        | None                         | None                            | Details Brave API key acquisition and placement instructions                                       |
| Sticky Note                  | Sticky Note                            | Guidance on changing input node types       | None                         | None                            | Explains how to replace Webhook or other input nodes                                               |
| Sticky Note5                 | Sticky Note                            | Note on alternative LLM model options       | None                         | None                            | Describes replacing Google Gemini with OpenAI or Anthropic Claude                                 |
| Sticky Note6                 | Sticky Note                            | Instructions to set up webhook call node    | None                         | None                            | Explains how to send queries to the workflow via HTTP Request node                                |
| Webhook Call                 | HTTP Request                           | Example node sending test queries            | None                         | None                            |                                                                                                    |
| Agent Model                  | LangChain LM Chat (Google Gemini)     | AI model for semantic processing (not connected) | None                         | Semantic Search - Result Re-Ranker, Semantic Search -Query Maker |                                                                                                    |
| Parser Model                 | LangChain LM Chat (Google Gemini)     | AI model for semantic processing (not connected) | None                         | Auto-fixing Output Parser6, Auto-fixing Output Parser |                                                                                                    |
| OpenAI Chat Model            | LangChain LM Chat (OpenAI GPT)         | Alternative AI model (not connected)         | None                         | None                            |                                                                                                    |
| Anthropic Chat Model         | LangChain LM Chat (Anthropic Claude)   | Alternative AI model (not connected)         | None                         | None                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Unique identifier (e.g., `962f1468-c80f-4c0c-8555-a0acf648ede4`)  
   - Response Mode: `responseNode`  
   - Purpose: Capture inbound research question as JSON with key `query["Research Question"]`.

2. **Add Date & Time Node**  
   - Type: Date/Time  
   - Default settings to output current date/time  
   - Connect Webhook → Date & Time

3. **Add Semantic Search -Query Maker Node**  
   - Type: LangChain Chain LLM Node with Google Gemini model  
   - Credentials: Google Gemini API credentials  
   - Prompt: Multi-chain reasoning prompt to generate expert search query from user input and date  
   - Connect Date & Time → Semantic Search -Query Maker

4. **Add Auto-fixing Output Parser Node**  
   - Type: LangChain Output Parser (Auto-fixing)  
   - Connect Semantic Search -Query Maker → Auto-fixing Output Parser

5. **Add Structured Output Parser1 Node**  
   - Type: LangChain Structured Output Parser  
   - JSON Schema: expects `reasoning_summary` and `final_search_query` keys  
   - Connect Auto-fixing Output Parser → Structured Output Parser1

6. **Add HTTP Request Node (Query)**  
   - Type: HTTP Request  
   - URL: `https://api.search.brave.com/res/v1/web/search`  
   - Method: GET  
   - Query Parameter: `q` set to `={{ $json["final_search_query"] }}` from Structured Output Parser1  
   - Headers:  
     - `Accept: application/json`  
     - `Accept-Encoding: gzip`  
     - `X-Subscription-Token`: Set to your Brave Search API key  
   - Connect Structured Output Parser1 → Query

7. **Add Code Node (Query-1 Combined)**  
   - Type: Function (JavaScript)  
   - Purpose: Loop over `web.results` array from Query, concatenate titles, URLs, and descriptions into a single string  
   - Connect Query → Query-1 Combined

8. **Add Semantic Search - Result Re-Ranker Node**  
   - Type: LangChain Chain LLM Node with Google Gemini model  
   - Credentials: Google Gemini API credentials  
   - Prompt: Multi-step ranking and extraction prompt using user query, refined query, current date, and aggregated search results text  
   - Connect Query-1 Combined → Semantic Search - Result Re-Ranker  
   - Also connect Webhook and Semantic Search -Query Maker nodes as AI language model inputs for context.

9. **Add Auto-fixing Output Parser6 Node**  
   - Type: LangChain Output Parser (Auto-fixing)  
   - Connect Semantic Search - Result Re-Ranker → Auto-fixing Output Parser6

10. **Add Structured Output Parser2 Node**  
    - Type: LangChain Structured Output Parser  
    - JSON Schema: expects chain_of_thought and 10 ranked URLs with title, link, description, plus extracted info  
    - Connect Auto-fixing Output Parser6 → Structured Output Parser2

11. **Add Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Response Body: JSON format mapping each top-ranked URL’s info from Semantic Search - Result Re-Ranker output  
    - Connect Structured Output Parser2 → Respond to Webhook

12. **Configure Credentials**  
    - Google Gemini API credentials for LangChain LLM Chain Nodes  
    - Brave Search API key in HTTP Request node header (`X-Subscription-Token`)

13. **Optional: Add Sticky Notes** for instructions on API setup, webhook configuration, and model replacements.

14. **Test the Workflow**  
    - Send POST request to webhook URL with payload:  
      `{ "query": { "Research Question": "Your search query here" } }`  
    - Verify JSON response contains ranked results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Step 1. Set Up a Free Brave Web Search Query API Key with instructions to create an account and obtain the free subscription token. | [Brave Search API](https://api.search.brave.com)                                                |
| Instructions to change the input node from Webhook to other types, ensuring downstream nodes align accordingly.                    | Sticky Note advising about Query 1 and Query 1 Ranker node inputs                                |
| How to set up the Webhook Call node to send queries to this workflow and receive results, including copying the production URL.  | Guidance for integrating this workflow with external HTTP request nodes                          |
| Customized Models to Replace: Notes on replacing Google Gemini nodes with OpenAI GPT4o or Anthropic Claude models if preferred.   | Useful for adapting workflow to alternative AI providers                                        |
| Examples of effective web search queries to avoid zero-result long-tail queries (e.g., "latest stock market mid-term outlook 2024"). | Embedded in Semantic Search -Query Maker prompt for AI guidance                                 |

---

This documentation enables full understanding, modification, and recreation of the Intelligent Web Query and Semantic Re-Ranking Flow in n8n, including integration details and potential failure points for operational robustness.