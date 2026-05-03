Convert Natural Language to Web Searches with GPT-4.1 mini and Log in Google Sheets

https://n8nworkflows.xyz/workflows/convert-natural-language-to-web-searches-with-gpt-4-1-mini-and-log-in-google-sheets-8982


# Convert Natural Language to Web Searches with GPT-4.1 mini and Log in Google Sheets

### 1. Workflow Overview

This workflow automates the process of converting natural language search requests into advanced, structured web searches using GPT-4.1 mini and the Firecrawl search API, then logs the results into Google Sheets and responds to the original requester.

**Target Use Cases:**  
- Users or systems submitting natural language queries for web search.  
- Automated generation of precise search queries leveraging AI for better search results.  
- Aggregation of search results across multiple targeted queries (site-specific, URL inclusion/exclusion, YouTube filtered).  
- Logging search results for audit, analysis, or downstream processing.  
- Returning structured results to the request originator via webhook.

**Logical Blocks:**  
- **1.1 Entry & Orchestration:** Receives incoming webhook requests containing a natural language query, triggers the search agent.  
- **1.2 AI Processing & Query Construction:** Uses GPT-4.1 mini via OpenRouter to interpret input and generate a structured Firecrawl search query.  
- **1.3 Firecrawl Search Execution:** Runs multiple parallel HTTP requests using the Firecrawl API with various query modifiers for broad search coverage.  
- **1.4 Results Persistence & Response:** Appends search results to a Google Sheets document and returns the results to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry & Orchestration

**Overview:**  
Receives natural language search requests via a webhook and initializes the query generation through the Search Agent.

**Nodes Involved:**  
- Webhook  
- Search Agent  
- Sticky Note (description)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Role: Entry point accepting POST requests with search queries.  
  - Configuration: Path set uniquely to "0d916768-023f-4d0e-8b76-4cbd5ffa07ee" with default options.  
  - Input: External HTTP request with natural language query.  
  - Output: Triggers next node with request data.  
  - Edge Cases: Malformed requests, missing parameters, large payloads. No authentication configured, so publicly accessible unless protected externally.

- **Search Agent**  
  - Type: Langchain Agent node  
  - Role: Converts natural language into structured Firecrawl search queries and parameters.  
  - Configuration:  
    - System message instructs transformation rules for queries, default limit 5 if unspecified.  
    - Uses Firecrawl Search tool to execute the constructed query.  
  - Input: Incoming webhook data (natural language query).  
  - Output: Structured search query dispatched to Firecrawl Search.  
  - Edge Cases: AI model misinterpretation, unexpected input formats, limit parameter missing or invalid.  
  - Version-specific: Langchain agent v2; requires OpenRouter API credentials.

- **Sticky Note (Step 1)**  
  - Content describes the entry and orchestration phase for human operators.

---

#### 2.2 AI Processing & Query Construction

**Overview:**  
Uses GPT 4.1 mini model (via OpenRouter) to support the Search Agent in interpreting and converting user input into a query string for Firecrawl.

**Nodes Involved:**  
- GPT 4.1 mini  
- Firecrawl Search (HTTP Request Tool)  
- Sticky Note (description)

**Node Details:**

- **GPT 4.1 mini**  
  - Type: Langchain AI Chat model node (OpenRouter API)  
  - Role: Provides the language model intelligence backing the Search Agent’s query construction.  
  - Configuration: Linked to OpenRouter API credentials. No additional options specified.  
  - Input: Receives prompt from Search Agent.  
  - Output: Generates structured query and parameters.  
  - Edge Cases: API rate limits, auth failures, timeouts, incomplete responses.

- **Firecrawl Search**  
  - Type: HTTP Request Tool (custom tool node)  
  - Role: Calls Firecrawl API with generated query and limit; requests markdown and screenshot results.  
  - Configuration:  
    - URL: https://api.firecrawl.dev/v1/search  
    - Method: POST  
    - JSON body dynamically constructed using expressions from AI output:  
      - `query` from AI variable `"searchQuery"`  
      - `limit` from AI variable `"limit"` or default  
      - `scrapeOptions` requesting markdown and full page screenshots  
  - Input: Structured query from Search Agent.  
  - Output: Search results with rich formats.  
  - Edge Cases: API errors, malformed queries, network timeouts.

- **Sticky Note (Step 2)**  
  - Explains the AI and tooling integration and default behaviors.

---

#### 2.3 Firecrawl Search Execution (Targeted Queries)

**Overview:**  
Executes multiple parallel targeted Firecrawl searches using HTTP Request nodes with preset queries to broaden search coverage.

**Nodes Involved:**  
- Site  
- In URL  
- Exclusion  
- Pro  
- Sticky Note (description)

**Node Details:**

- **Site**  
  - Type: HTTP Request  
  - Role: Search Firecrawl for "nate herk site:www.geeky-gadgets.com" with limit 5.  
  - Configuration: POST to Firecrawl API with JSON body containing fixed query string.  
  - Input: Triggered by Search Agent.  
  - Output: Feeds into Exclusion node.  
  - Edge Cases: API errors, fixed query limits flexibility.

- **In URL**  
  - Type: HTTP Request  
  - Role: Search Firecrawl with "nate herk inurl:skool" query, limit 5.  
  - Input: Triggered by Site node.  
  - Output: Feeds into Pro node.  
  - Edge Cases: Same as above.

- **Exclusion**  
  - Type: HTTP Request  
  - Role: Searches with query "nate herk -inurl:skool", limit 6.  
  - Input: Triggered by Site node.  
  - Output: Feeds into Append row in sheet node.  
  - Edge Cases: Slightly different limit (6); consider consistency.

- **Pro**  
  - Type: HTTP Request  
  - Role: Searches with query "Nate Herk site:youtube.com -shorts intitle:automation", limit 5.  
  - Input: Triggered by In URL node.  
  - Output: Feeds into Append row in sheet node.  
  - Edge Cases: Fixed query limits; may need parameterization.

- **Sticky Note (Step 3)**  
  - Describes the rationale and query examples for parallel search execution.

---

#### 2.4 Results Persistence & Response

**Overview:**  
Appends search results to a Google Sheets document and returns the results via webhook response.

**Nodes Involved:**  
- Append row in sheet  
- Respond to Webhook  
- Sticky Note (description)

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets node  
  - Role: Persists search results into a specific Google Sheets tab.  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet name are set to a specific spreadsheet and tab (Founders Academy Waitlist / Tabellenblatt1).  
  - Credentials: Google Sheets OAuth2 configured.  
  - Input: Receives data from Exclusion and Pro nodes.  
  - Output: Passes control to Respond to Webhook node.  
  - Edge Cases: Google API quota limits, auth token expiration, data mapping errors.

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends the final search results back to the original webhook caller.  
  - Configuration: Defaults with no special options.  
  - Input: From Append row in sheet node.  
  - Output: HTTP response to caller.  
  - Edge Cases: Large payload sizes, timing out before response sent.

- **Sticky Note (Step 4)**  
  - Explains data persistence and response step, with tips on field mapping.

---

### 3. Summary Table

| Node Name          | Node Type                            | Functional Role                         | Input Node(s)          | Output Node(s)           | Sticky Note                                                   |
|--------------------|------------------------------------|---------------------------------------|------------------------|--------------------------|---------------------------------------------------------------|
| Webhook            | HTTP Webhook Trigger                | Entry point for natural language query| -                      | Search Agent             | ## STEP 1 · Entry & Orchestration: Receives natural language requests and triggers search agent |
| Search Agent       | Langchain Agent                    | Converts NL query to Firecrawl search | Webhook                | Site, In URL              | ## STEP 1 · Entry & Orchestration                              |
| GPT 4.1 mini       | Langchain AI Chat (OpenRouter)     | Provides LLM intelligence for agent   | Search Agent (ai_languageModel) | Search Agent (ai_languageModel) | ## STEP 2 · Agent & Tooling: AI supports query construction    |
| Firecrawl Search   | HTTP Request Tool                  | Executes Firecrawl search with query  | Search Agent (ai_tool)  | Search Agent             | ## STEP 2 · Agent & Tooling                                    |
| Site               | HTTP Request                      | Targeted search: site query            | Search Agent            | Exclusion                | ## STEP 3 · Targeted Queries (HTTP): Parallel Firecrawl searches |
| In URL             | HTTP Request                      | Targeted search: inurl query            | Site                   | Pro                      | ## STEP 3 · Targeted Queries (HTTP)                            |
| Exclusion          | HTTP Request                      | Targeted search: exclusion query       | Site                   | Append row in sheet       | ## STEP 3 · Targeted Queries (HTTP)                            |
| Pro                | HTTP Request                      | Targeted search: YouTube pro query     | In URL                 | Append row in sheet       | ## STEP 3 · Targeted Queries (HTTP)                            |
| Append row in sheet| Google Sheets                     | Persists search results                 | Exclusion, Pro          | Respond to Webhook        | ## STEP 4 · Persist & Respond: Logs results and responds      |
| Respond to Webhook | Respond to Webhook                | Sends results back to caller            | Append row in sheet     | -                        | ## STEP 4 · Persist & Respond                                  |
| Sticky Note        | Sticky Note                      | Descriptions for blocks                 | -                      | -                        | Multiple sticky notes describing each step                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: HTTP Webhook  
   - Path: `0d916768-023f-4d0e-8b76-4cbd5ffa07ee` (or custom)  
   - Purpose: Receive incoming natural language search requests.

2. **Create GPT 4.1 mini Node**  
   - Type: Langchain Chat Model (OpenRouter)  
   - Credentials: Setup OpenRouter API credentials with a valid account.  
   - No additional parameters needed.

3. **Create Firecrawl Search Node**  
   - Type: HTTP Request Tool  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/search`  
   - Body (JSON, raw):  
     ```json
     {
       "query": "={{$fromAI(\"searchQuery\")}}",
       "limit": {{$fromAI("limit", "the number of search results requested", "number") || 5}},
       "scrapeOptions": {
         "formats": ["markdown", "screenshot@fullPage"]
       }
     }
     ```  
   - Purpose: Send constructed query and limit to Firecrawl API.

4. **Create Search Agent Node**  
   - Type: Langchain Agent  
   - System Message: Use the provided template with query construction rules and tool usage instructions (see block 2.1).  
   - Connect GPT 4.1 mini node as the language model.  
   - Connect Firecrawl Search node as the tool.  
   - Purpose: Convert natural language input into structured Firecrawl queries and execute them.

5. **Connect Webhook Node → Search Agent Node**

6. **Create four HTTP Request Nodes for Parallel Targeted Queries:**  
   - Node "Site": POST to Firecrawl API with body:  
     ```json
     {
       "query": "nate herk site:www.geeky-gadgets.com",
       "limit": 5
     }
     ```  
   - Node "In URL": POST to Firecrawl API with body:  
     ```json
     {
       "query": "nate herk inurl:skool",
       "limit": 5
     }
     ```  
   - Node "Exclusion": POST to Firecrawl API with body:  
     ```json
     {
       "query": "nate herk -inurl:skool",
       "limit": 6
     }
     ```  
   - Node "Pro": POST to Firecrawl API with body:  
     ```json
     {
       "query": "Nate Herk site:youtube.com -shorts intitle:automation",
       "limit": 5
     }
     ```  
   - Connect flow as: Search Agent → Site → In URL → Pro  
     Also connect Site → Exclusion (parallel branch).  

7. **Create Google Sheets Node**  
   - Operation: Append  
   - Document ID: `1EAA7t2lal9462X_CHmfZQ7LbAVsbrrkpGN1hpDI0p8o`  
   - Sheet Name: `gid=0` (Tabellenblatt1)  
   - Credentials: Setup Google Sheets OAuth2 credentials with proper access.  
   - Connect inputs from Exclusion and Pro nodes.

8. **Create Respond to Webhook Node**  
   - Connect input from Google Sheets node.  
   - No special configuration needed.  
   - Purpose: Return final results to the webhook caller.

9. **Connect Google Sheets Node → Respond to Webhook Node**

10. **Add Sticky Notes** (optional but recommended) to describe each major step as documented above.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow uses OpenRouter API for GPT 4.1 mini, requiring a valid API key and plan.                   | https://openrouter.ai                                                                                     |
| Firecrawl API is used for web search, supporting advanced query operators like site:, inurl:, intitle:.  | https://firecrawl.dev/api                                                                                 |
| Google Sheets integration requires OAuth2 credentials with edit permissions for the target spreadsheet.  | https://developers.google.com/sheets/api/guides/authorizing                                               |
| Sticky notes within the workflow provide detailed step descriptions for easier understanding.            | Included in workflow at positions near relevant nodes                                                    |
| Default search result limit is 5 if the request does not specify otherwise.                              | Important for consistent and predictable output size                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.