Perplexity-Style Iterative Research with Gemini and Google Search

https://n8nworkflows.xyz/workflows/perplexity-style-iterative-research-with-gemini-and-google-search-4758


# Perplexity-Style Iterative Research with Gemini and Google Search

---

### 1. Workflow Overview

This workflow is designed to perform **iterative, Perplexity-style AI-assisted web research** using Google Gemini 2.0/2.5 language models combined with Google Search API. It automates the process of generating search queries, conducting web searches, synthesizing results, reflecting on knowledge gaps, and iterating the research until sufficient information is gathered or a maximum loop count is reached. The final output is a well-structured, citation-rich answer to the user’s original question.

The workflow logically divides into the following blocks:

- **1.1 Input Reception & Configuration:** Receives user queries, sets initial configuration variables.
- **1.2 Query Generation:** Uses Gemini 2.0 Flash to generate initial diverse web search queries.
- **1.3 Web Research Execution:** Iteratively executes web searches using Gemini-powered Google Search, processes and aggregates results.
- **1.4 Reflection & Follow-up Query Generation:** Analyzes gathered summaries, identifies knowledge gaps, and generates follow-up queries for further research loops.
- **1.5 Loop Control & State Management:** Manages loop counters, stores intermediate results and sources in Redis for state persistence.
- **1.6 Finalization:** Once research is sufficient or loops maxed out, synthesizes and formats the final answer with proper citations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration

- **Overview:**  
  Receives the user’s chat input via a webhook trigger, sets up constants and context variables for the entire workflow execution.

- **Nodes Involved:**  
  - When chat message received  
  - Configs  
  - Sticky Note1, Sticky Note5 (documentation notes)

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger (Webhook)  
    - Role: Entry point; receives user chat input and session ID  
    - Config: Public webhook with initial message describing the workflow  
    - Outputs user input as `chatInput` and sessionId for conversation context  
    - Potential failure: Webhook connectivity, malformed input  
  - **Configs**  
    - Type: Set node  
    - Role: Defines workflow constants:  
      - number_of_initial_queries = 3  
      - max_research_loops = 3  
      - current_date (formatted current date)  
      - conversation_id (composed from workflow and sessionId)  
    - Used to parameterize queries and loop controls  
    - Potential failure: Date formatting issues, missing session ID  
  - **Sticky Notes**  
    - Provide human-readable documentation on configs and project context.

#### 2.2 Query Generation

- **Overview:**  
  Generates a set of relevant, diverse search queries based on the user’s question, limiting the number of queries and ensuring topical breadth.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - generate_query (LangChain Chain LLM)  
  - set search_query  
  - Sticky Note (generate_query explanation)

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini LLM Node  
    - Role: Provides conversational AI interface to Gemini 2.0 Flash model for query generation  
    - Config: Uses model `models/gemini-2.0-flash` with Google Palm API credentials  
    - Outputs raw model response for query generation  
    - Potential failure: API auth issues, model timeout, quota exceeded  
  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Parses Gemini output JSON structure with schema requiring `query` (array of strings) and `rationale` (string)  
    - Ensures structured, validated output from Gemini for queries  
    - Potential failure: Parsing errors if output deviates from schema  
  - **generate_query**  
    - Type: LangChain Chain LLM  
    - Role: Defines prompt instructing Gemini to generate up to 3 diverse, specific search queries based on user input  
    - Uses expressions to limit query count and include current date context from Configs  
    - Output is a JSON with rationale and query list  
  - **set search_query**  
    - Type: Set node  
    - Role: Stores parsed query list into workflow data for downstream use  
  - **Sticky Note**  
    - Explains the role of `generate_query` and use of Gemini 2.0 Flash for query generation.

#### 2.3 Web Research Execution

- **Overview:**  
  Executes each search query individually using Google Gemini Search API, processes results to extract text, citations, and sources, and aggregates all search results.

- **Nodes Involved:**  
  - Split Out  
  - number_of_ran_queries (Redis INCR counter)  
  - attach index as id (Code)  
  - GeminiSearch (HTTP Request to Google Gemini Search API)  
  - web_search (Code node for processing search results and citations)  
  - merge web_search_result (Aggregate)  
  - history_web_research_result (Redis push)  
  - history_sources_gathered (Redis push)  
  - web_search step record (Code)  
  - push web_search step record (Redis push)  
  - Sticky Note2 (documentation on web_research)

- **Node Details:**  
  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the array of search queries into individual items to run searches one by one  
    - Outputs each query as `search_query`  
  - **number_of_ran_queries**  
    - Type: Redis node (INCR)  
    - Role: Tracks how many queries have been executed in this research loop  
    - Redis key based on conversation_id + `web_search_idx`  
    - Potential failure: Redis connectivity or increment errors  
  - **attach index as id**  
    - Type: Code (JavaScript)  
    - Role: Attaches an index/id to each query item for tracking in later citation resolution  
  - **GeminiSearch**  
    - Type: HTTP Request  
    - Role: Sends raw POST request to Google Gemini 2.0 Flash Search API with the current query, requesting text summary with citations  
    - Uses Google Palm API credentials  
    - Configured with instructions to search latest credible info, consolidate key findings, and track sources  
    - Potential failure: API errors, rate limits, malformed requests  
  - **web_search**  
    - Type: Code (JavaScript)  
    - Role: Parses GeminiSearch response, extracts text parts, grounding metadata, resolves URLs to short IDs, and inserts citation markers into text  
    - Deduplicates sources by short URL  
    - Returns:  
      - `sources_gathered`: deduplicated source metadata  
      - `web_research_result`: annotated text summary array  
    - Potential failure: Unexpected API response format, missing grounding metadata, JS errors in parsing  
  - **merge web_search_result**  
    - Type: Aggregate node  
    - Role: Combines lists of `sources_gathered` and `web_research_result` from multiple queries into unified lists  
  - **history_web_research_result & history_sources_gathered**  
    - Type: Redis nodes (push to list)  
    - Role: Store incremental web research results and sources persistently keyed by conversation_id  
    - Potential failure: Redis write errors  
  - **web_search step record**  
    - Type: Code  
    - Role: Creates a summary step text reporting number and labels of sources gathered for logging or final output  
  - **push web_search step record**  
    - Type: Redis push  
    - Role: Stores the step record into Redis for later compilation  
  - **Sticky Note2**  
    - Explains the web_research function and rationale for using raw HTTP requests rather than builtin nodes for Gemini search due to schema limitations.

#### 2.4 Reflection & Follow-up Query Generation

- **Overview:**  
  Analyzes the aggregated research summaries to identify knowledge gaps and generate follow-up search queries if necessary, enabling iterative research loops.

- **Nodes Involved:**  
  - history_web_research_result1 (Redis get)  
  - Build reflection request body (Code)  
  - reflection (HTTP Request to Gemini 2.5 Flash Preview)  
  - reflection_output_parse (Code)  
  - research_loop_count (Redis INCR)  
  - retrieve value (Code)  
  - If finish (If node)  
  - push reflection step (Redis push)  
  - Sticky Note3 (documentation on reflection)

- **Node Details:**  
  - **history_web_research_result1**  
    - Redis get node  
    - Retrieves all accumulated web research summaries for current conversation  
  - **Build reflection request body**  
    - Code node to build a structured prompt for Gemini 2.5 Flash Preview model  
    - Instructions: Analyze summaries, identify sufficiency, knowledge gaps, and generate follow-up queries if needed  
    - Output includes a structured JSON schema for response with keys: `is_sufficient`, `knowledge_gap`, `follow_up_queries`, `research_loop_count`, `number_of_ran_queries`  
  - **reflection**  
    - HTTP Request node to Gemini 2.5 flash preview model endpoint  
    - Sends reflection prompt and expects structured JSON output following schema  
    - Uses Google Palm API credentials  
    - Potential failure: API errors, malformed response, timeout  
  - **reflection_output_parse**  
    - Code node to parse Gemini JSON response, stripping markdown code fences, and converting text to JSON object  
    - Potential failure: JSON parse errors if response is malformed  
  - **research_loop_count**  
    - Redis INCR node to track how many research loops (reflection + search) have been performed  
  - **retrieve value**  
    - Code node extracts `research_loop_count` and attaches reflection output for conditional logic  
  - **If finish**  
    - Conditional node to decide whether to stop or continue research:  
      - Stop if `is_sufficient` is true OR `research_loop_count` >= max_research_loops from Configs  
      - Continues otherwise  
  - **push reflection step**  
    - Redis push to append a reflection step summary to the list for logging  
  - **Sticky Note3**  
    - Explains the reflection block purpose and why raw HTTP requests are used here (due to structured output schema limitations in builtin nodes).

#### 2.5 Loop Control & State Management

- **Overview:**  
  Manages iteration counters and stores intermediate results in Redis to maintain state across asynchronous executions and multiple loops.

- **Nodes Involved:**  
  - research_loop_count (Redis INCR)  
  - number_of_ran_queries (Redis INCR)  
  - history_web_research_result (Redis push)  
  - history_sources_gathered (Redis push)  
  - push web_search step record (Redis push)  
  - push reflection step (Redis push)  
  - get:history_sources_gathered (Redis get)  
  - get:steps (Redis get)  
  - Sticky Notes (documentation on Redis use)

- **Node Details:**  
  - Redis nodes are used extensively to maintain:  
    - Number of queries run in current loop  
    - Number of research loops completed  
    - Accumulated web research results  
    - Accumulated source metadata  
    - Step-wise logs for research and reflection steps  
  - This external state management is critical because n8n does not natively support complex global state across runs  
  - Potential failure: Redis connectivity or data corruption

#### 2.6 Finalization

- **Overview:**  
  After finishing iterative research, synthesizes all gathered information and sources into a final well-cited answer for the user.

- **Nodes Involved:**  
  - finalize_answer (LangChain Chain LLM)  
  - Google Gemini Chat Model2  
  - get:history_sources_gathered (Redis get)  
  - get:steps (Redis get)  
  - Merge (Merge node combining sources and steps)  
  - format answer (Code node)  
  - Sticky Note4 (documentation on finalization)

- **Node Details:**  
  - **finalize_answer**  
    - LangChain Chain LLM node  
    - Uses Gemini 2.0 Flash to generate the final answer with citations based on all accumulated summaries and the user’s original question  
    - Prompt instructs to include citations properly and not mention internal workflow details  
  - **Google Gemini Chat Model2**  
    - Gemini 2.0 Flash model interface for `finalize_answer` node  
  - **get:history_sources_gathered & get:steps**  
    - Redis get nodes to retrieve all stored sources and step logs for inclusion in final output  
  - **Merge**  
    - Combines the outputs of sources and steps into one data stream for formatting  
  - **format answer**  
    - Code node that:  
      - Parses sources from JSON strings  
      - Replaces short URLs in answer text with original URLs for clarity  
      - Prepends step logs and formats the final markdown answer text  
  - **Sticky Note4**  
    - Explains that this block prepares the final output by deduplicating and formatting sources and combining them with summaries into a citation-rich research report.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                        | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                   |
|----------------------------|------------------------------------|-------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger              | User query reception (entry point)  | —                                | Configs                          |                                                                                              |
| Configs                    | Set                                | Workflow constants setup             | When chat message received        | generate_query                   | See Sticky Note1: explains configs and variables                                              |
| Sticky Note1               | Sticky Note                        | Documentation - Configs              | —                                | —                                | Explains configuration variables                                                             |
| Sticky Note5               | Sticky Note                        | Documentation - Project context     | —                                | —                                | Notes about reproduction of gemini-fullstack-langgraph-quickstart                            |
| Google Gemini Chat Model   | LangChain Google Gemini LLM        | Generate initial search queries     | Configs                         | generate_query                   | See Sticky Note: generate_query explains query generation with Gemini 2.0 Flash              |
| Structured Output Parser   | LangChain Output Parser             | Parse Gemini output JSON             | Google Gemini Chat Model          | generate_query                   |                                                                                              |
| generate_query             | LangChain Chain LLM                 | Generate search queries prompt       | Google Gemini Chat Model, Parser  | set search_query                 |                                                                                              |
| set search_query           | Set                                | Store generated queries              | generate_query                   | Split Out                       | Sticky note: store search_query                                                              |
| Split Out                  | Split Out                          | Split query array into single queries| set search_query                 | number_of_ran_queries            | Sticky Note2: explains web research block using Gemini and HTTP raw request                  |
| number_of_ran_queries      | Redis (INCR)                      | Track number of queries run          | Split Out                      | attach index as id               |                                                                                              |
| attach index as id         | Code                               | Attach index/id to queries           | number_of_ran_queries            | GeminiSearch                    |                                                                                              |
| GeminiSearch               | HTTP Request                      | Query Google Gemini Search API       | attach index as id               | web_search                     |                                                                                              |
| web_search                 | Code                               | Parse search results and citations   | GeminiSearch                   | merge web_search_result          |                                                                                              |
| merge web_search_result    | Aggregate                         | Aggregate all search results         | web_search                    | history_web_research_result, history_sources_gathered |                                                                                              |
| history_web_research_result| Redis (push)                     | Persist research summaries           | merge web_search_result         | history_web_research_result1     |                                                                                              |
| history_sources_gathered   | Redis (push)                     | Persist gathered source metadata     | merge web_search_result         | —                                |                                                                                              |
| web_search step record     | Code                               | Create step summary for web search   | merge web_search_result         | push web_search step record      |                                                                                              |
| push web_search step record| Redis (push)                     | Store web search step summary        | web_search step record          | —                                |                                                                                              |
| history_web_research_result1| Redis (get)                     | Retrieve accumulated summaries       | history_web_research_result     | Build reflection request body    |                                                                                              |
| Build reflection request body| Code                           | Build prompt for reflection model    | history_web_research_result1    | reflection                     | Sticky Note3: explains reflection and rationale for raw HTTP request                        |
| reflection                 | HTTP Request                      | Gemini 2.5 Flash reflection query    | Build reflection request body   | reflection_output_parse          |                                                                                              |
| reflection_output_parse    | Code                               | Parse Gemini reflection output JSON  | reflection                     | research_loop_count, retrieve value |                                                                                              |
| research_loop_count        | Redis (INCR)                     | Track research loop count             | reflection_output_parse         | retrieve value                  |                                                                                              |
| retrieve value             | Code                               | Extract loop count and reflection output | research_loop_count           | If finish, push reflection step  |                                                                                              |
| If finish                 | If                                | Check if research is sufficient or max loops reached | retrieve value           | finalize_answer (if finished), set search_query (if continue) |                                                                                              |
| push reflection step       | Redis (push)                     | Store reflection step summary        | retrieve value                 | —                                |                                                                                              |
| set search_query           | Set                                | Set follow-up queries to continue loop| If finish (continue branch)    | Split Out                       |                                                                                              |
| finalize_answer            | LangChain Chain LLM                 | Generate final answer with citations | If finish (finish branch)       | get:history_sources_gathered, get:steps | Sticky Note4: explains final answer preparation with citations                              |
| Google Gemini Chat Model2  | LangChain Google Gemini LLM        | Underlying model for finalize_answer | finalize_answer                | —                                |                                                                                              |
| get:history_sources_gathered| Redis (get)                    | Retrieve all sources for formatting  | finalize_answer                | Merge                          |                                                                                              |
| get:steps                  | Redis (get)                    | Retrieve step logs for formatting     | finalize_answer                | Merge                          |                                                                                              |
| Merge                     | Merge                             | Combine sources and steps             | get:history_sources_gathered, get:steps | format answer                 |                                                                                              |
| format answer             | Code                               | Final formatting of answer text and source links | Merge                         | —                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a webhook trigger node "When chat message received"**  
   - Type: LangChain Chat Trigger  
   - Configure public webhook with initial message: "# gemini-fullstack-langgraph-quickstart\nPowered by Google Gemini and n8n"  
   - Outputs user input as `chatInput` and sessionId  

2. **Create a Set node "Configs"**  
   - Assign variables:  
     - `number_of_initial_queries` = 3 (number)  
     - `max_research_loops` = 3 (number)  
     - `current_date` = `={{$now.format('yyyy-LL-dd')}}` (string)  
     - `conversation_id` = `=conversation:gemini-fullstack-langgraph-quickstart:{{ $json.sessionId }}:` (string)  
   - Connect "When chat message received" → "Configs"

3. **Create LangChain Google Gemini Chat Model node "Google Gemini Chat Model"**  
   - Model: `models/gemini-2.0-flash`  
   - Credentials: Google Palm API (Gemini 2.0 Flash)  
   - Connect "Configs" → "Google Gemini Chat Model" via ai_languageModel connection  

4. **Create LangChain Structured Output Parser node "Structured Output Parser"**  
   - Schema: JSON schema requiring keys "query" (string array) and "rationale" (string)  
   - Connect "Google Gemini Chat Model" → "Structured Output Parser" via ai_outputParser  

5. **Create LangChain Chain LLM node "generate_query"**  
   - Prompt: Detailed instructions to generate up to 3 diverse, specific search queries based on user input, including current date context and example output format as JSON  
   - Use expressions to access:  
     - `$('Configs').item.json.number_of_initial_queries`  
     - `$('Configs').item.json.current_date`  
     - `$('When chat message received').item.json.chatInput` as user question context  
   - Enable structured output parser (link to previous node)  
   - Connect "Structured Output Parser" → "generate_query"

6. **Create Set node "set search_query"**  
   - Pass through all fields, assign `search_query` = `={{ $json.output.query }}` from generate_query output  
   - Connect "generate_query" → "set search_query"

7. **Create Split Out node "Split Out"**  
   - Field to split out: `search_query`  
   - Destination field name: `search_query`  
   - Connect "set search_query" → "Split Out"

8. **Create Redis INCR node "number_of_ran_queries"**  
   - Key: `={{ $('Configs').item.json.conversation_id }}web_search_idx`  
   - Connect "Split Out" → "number_of_ran_queries"

9. **Create Code node "attach index as id"**  
   - JavaScript: Extracts the incremented query index and attaches it as `id` along with the current `search_query`  
   - Connect "number_of_ran_queries" → "attach index as id"

10. **Create HTTP Request node "GeminiSearch"**  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
    - Body JSON with:  
      - Role: user  
      - Text prompt instructing to conduct Google searches for current query, emphasizing current date context and source tracking  
      - Tools: Google Search enabled  
      - Generation config: temperature 0, responseMimeType text/plain  
    - Authentication: Google Palm API credential for Gemini 2.0  
    - Connect "attach index as id" → "GeminiSearch"

11. **Create Code node "web_search"**  
    - Parses GeminiSearch response:  
      - Joins text parts  
      - Resolves grounding metadata URLs to shortened URLs with unique IDs  
      - Extracts citations and inserts citation markers in text  
      - Deduplicates sources by short URL  
      - Returns `sources_gathered` and `web_research_result`  
    - Connect "GeminiSearch" → "web_search"

12. **Create Aggregate node "merge web_search_result"**  
    - Merge lists for fields: `sources_gathered`, `web_research_result`  
    - Connect "web_search" → "merge web_search_result"

13. **Create Redis push nodes:**  
    - "history_web_research_result": push `web_research_result` to list keyed by `{{conversation_id}}history_web_research_result`  
    - "history_sources_gathered": push `sources_gathered` (stringified) to list keyed by `{{conversation_id}}history_sources_gathered`  
    - Connect "merge web_search_result" → both Redis nodes

14. **Create Code node "web_search step record"**  
    - Generates a markdown step summary of gathered sources count and labels  
    - Connect "merge web_search_result" → "web_search step record"

15. **Create Redis push "push web_search step record"**  
    - Push step summary to Redis list `{{conversation_id}}steps`  
    - Connect "web_search step record" → "push web_search step record"

16. **Create Redis get node "history_web_research_result1"**  
    - Get all web research results from `{{conversation_id}}history_web_research_result` list  
    - Connect "history_web_research_result" → "history_web_research_result1"

17. **Create Code node "Build reflection request body"**  
    - Builds a prompt for Gemini 2.5 model to analyze summaries and generate follow-up queries if needed  
    - Output JSON with keys: `is_sufficient`, `knowledge_gap`, `follow_up_queries`, plus loop counts  
    - Connect "history_web_research_result1" → "Build reflection request body"

18. **Create HTTP Request node "reflection"**  
    - POST to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent`  
    - Body: JSON from previous node  
    - Authentication: Google Palm API credential for Gemini 2.5  
    - Connect "Build reflection request body" → "reflection"

19. **Create Code node "reflection_output_parse"**  
    - Parses Gemini response JSON, strips markdown fences, returns JSON object  
    - Connect "reflection" → "reflection_output_parse"

20. **Create Redis INCR node "research_loop_count"**  
    - Increment research loop count key: `{{conversation_id}}research_loop_count`  
    - Connect "reflection_output_parse" → "research_loop_count"

21. **Create Code node "retrieve value"**  
    - Extracts `research_loop_count` and reflection output for conditional logic  
    - Connect "research_loop_count" → "retrieve value"

22. **Create If node "If finish"**  
    - Conditions:  
      - `is_sufficient` == true OR  
      - `research_loop_count` >= `max_research_loops` from Configs  
    - Connect "retrieve value" → If finish

23. **Create Redis push node "push reflection step"**  
    - Pushes reflection summary (sufficiency and loop status) to steps list  
    - Connect "retrieve value" → "push reflection step"

24. **Connect If finish:**  
    - If true (finish branch) → "finalize_answer"  
    - If false (continue branch) → "set search_query" to set follow-up queries and loop back to "Split Out"

25. **Create LangChain Chain LLM node "finalize_answer"**  
    - Prompt Gemini 2.0 Flash to produce final answer with citations based on all summaries and user question  
    - Connect "If finish" (true branch) → "finalize_answer"  

26. **Create Google Gemini Chat Model node "Google Gemini Chat Model2"**  
    - Model: Gemini 2.0 Flash  
    - Credentials: Google Palm API  
    - Connect "finalize_answer" → "Google Gemini Chat Model2" via ai_languageModel

27. **Create Redis get nodes:**  
    - "get:history_sources_gathered" retrieves all stored sources  
    - "get:steps" retrieves all stored step logs  
    - Connect "finalize_answer" → both Redis get nodes

28. **Create Merge node "Merge"**  
    - Combines outputs of above two Redis get nodes  
    - Connect both Redis get nodes → "Merge"

29. **Create Code node "format answer"**  
    - Parses sources, replaces short URLs with original URLs in final answer text  
    - Prepends research steps as markdown  
    - Connect "Merge" → "format answer"

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Reproduction of [`gemini-fullstack-langgraph-quickstart`](https://github.com/google-gemini/gemini-fullstack-langgraph-quickstart) in n8n | Sticky Note5 describes the project and use of external Redis for global state management                     |
| Use of raw HTTP Requests for Gemini search and reflection due to limitations of builtin LangChain nodes | Sticky Note2 and Sticky Note3 explain why raw HTTP is preferred over builtin nodes for structured outputs    |
| Workflow tagged as PROD and uses external Redis to maintain global variables and loop counters          | Tags and Redis nodes throughout the workflow                                                                |
| The workflow uses Google Gemini models 2.0 Flash and 2.5 Flash Preview for different stages              | Gemini 2.0 Flash for query generation and final answer; Gemini 2.5 Flash for reflection and follow-up queries|
| Proper error handling is limited, so consider adding try/catch or fallback nodes for API and Redis errors| Potential failure points include API auth failures, response format errors, Redis connectivity issues        |

---

**Disclaimer:** The provided content is solely derived from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All data handled is legal and public.

---