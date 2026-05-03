Web Scraping & Screenshot Automation with GPT 4.1 mini and Firecrawl

https://n8nworkflows.xyz/workflows/web-scraping---screenshot-automation-with-gpt-4-1-mini-and-firecrawl-6343


# Web Scraping & Screenshot Automation with GPT 4.1 mini and Firecrawl

### 1. Workflow Overview

This workflow automates web scraping and screenshot capture using GPT 4.1 mini combined with the Firecrawl search API. It is designed to accept natural language search queries, translate them into structured Firecrawl queries with advanced search operators, execute those queries against the Firecrawl API, and return detailed search results including markdown content and full-page screenshots.

The workflow is logically organized into the following functional blocks:

- **1.1 Input Reception:** Captures incoming chat messages with search requests.
- **1.2 AI Query Processing:** Uses GPT 4.1 mini and an agent to convert user input into structured Firecrawl queries and determine the result limit.
- **1.3 Firecrawl Search Execution:** Performs the actual search via Firecrawl API, returning detailed results including screenshots.
- **1.4 Sample Query Demonstrations:** Contains preset example queries showcasing specific Firecrawl search operators and use cases.
- **1.5 Notation and Documentation:** Sticky notes provide contextual explanations for each query type to aid understanding.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages that trigger the workflow, serving as the entry point.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Webhook listener for chat messages initiating the workflow.  
    - Configuration: Default options, linked to a unique webhook ID for external triggering.  
    - Inputs: External chat messages.  
    - Outputs: Passes received messages downstream to the AI Query Processing block.  
    - Edge Cases: Possible webhook downtime, malformed or empty messages, or unauthorized access if webhook security is not enforced.

#### 2.2 AI Query Processing

- **Overview:**  
  Converts user natural language queries into structured Firecrawl queries and determines parameters like result limits using GPT 4.1 mini and a specialized LangChain agent.

- **Nodes Involved:**  
  - `GPT 4.1 mini`  
  - `Search Agent`

- **Node Details:**  
  - **GPT 4.1 mini**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: Acts as the language model backend running GPT 4.1 mini for NLP tasks.  
    - Configuration: Default options, no custom prompt here, forwards chat context to the Agent.  
    - Inputs: Chat messages from the trigger node.  
    - Outputs: Processed NLP results forwarded to the `Search Agent`.  
    - Edge Cases: Possible API rate limits, timeouts, or malformed responses. Requires valid OpenAI credentials.  
  - **Search Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Uses a custom system message to convert natural language into Firecrawl-compatible search queries and parameters.  
    - Configuration:  
      - System message explicitly defines query construction rules (e.g., `site:`, `inurl:`, exclusion with `-`, etc.).  
      - Default limit is 5 if user does not specify.  
      - Outputs a structured query and limit for passing to Firecrawl.  
    - Inputs: Output from GPT 4.1 mini node.  
    - Outputs: Sends a constructed query string and limit to the Firecrawl Search tool node.  
    - Edge Cases: Expression errors if the system message or outputs are malformed, ambiguity in user queries, or failure to parse parameters correctly.

#### 2.3 Firecrawl Search Execution

- **Overview:**  
  Executes the structured search query using Firecrawl’s API, fetching search results with markdown content and full-page screenshots.

- **Nodes Involved:**  
  - `Firecrawl Search`

- **Node Details:**  
  - **Firecrawl Search**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Performs HTTP POST requests to Firecrawl’s `/v1/search` endpoint.  
    - Configuration:  
      - URL: `https://api.firecrawl.dev/v1/search`  
      - Method: POST  
      - Body: JSON constructed dynamically using user query and limit extracted from AI agent outputs.  
      - `scrapeOptions`: Set to request markdown and full-page screenshots (`["markdown", "screenshot@fullPage"]`).  
    - Inputs: Query string and limit from `Search Agent` node via expressions.  
    - Outputs: Full search results including title, URL, markdown summaries, and screenshots.  
    - Edge Cases: HTTP errors (timeouts, 4xx/5xx), invalid API keys, malformed JSON, or empty results from Firecrawl.

#### 2.4 Sample Query Demonstrations

- **Overview:**  
  Contains preset HTTP Request nodes with sample Firecrawl search queries demonstrating specific operator usages, accompanied by explanatory sticky notes.

- **Nodes Involved:**  
  - `Site`  
  - `In URL`  
  - `Exclusion`  
  - `Pro`  
  - `Sticky Note1`  
  - `Sticky Note2`  
  - `Sticky Note3`  
  - `Sticky Note4`

- **Node Details:**  
  - **Site**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Executes a Firecrawl search limiting results to a specific site (`site:www.geeky-gadgets.com`).  
    - Configuration:  
      - Query: `"nate herk site:www.geeky-gadgets.com"`  
      - Limit: 5  
    - Edge Cases: Site-specific results might be limited or unavailable.  
  - **In URL**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Searches for pages with a word in the URL (`inurl:skool`).  
    - Configuration:  
      - Query: `"nate herk inurl:skool"`  
      - Limit: 5  
  - **Exclusion**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Searches for results excluding URLs with the word `skool` (`-inurl:skool`).  
    - Configuration:  
      - Query: `"nate herk -inurl:skool"`  
      - Limit: 6  
  - **Pro**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Combines multiple operators for advanced search: site-limited to YouTube, excludes shorts, includes `intitle:automation`.  
    - Configuration:  
      - Query: `"Nate Herk site:youtube.com -shorts intitle:automation"`  
      - Limit: 5  
  - **Sticky Notes**  
    - Provide labeled explanations for each query type:  
      - Sticky Note1: "Within a Specific Website" (related to `Site`)  
      - Sticky Note2: "Word Appears in URL" (related to `In URL`)  
      - Sticky Note3: "Exclude Terms" (related to `Exclusion`)  
      - Sticky Note4: "Pro Tip" (related to `Pro`)  
    - Positioned near their respective query nodes for easy reference.

#### 2.5 Notation and Documentation

- **Overview:**  
  A large sticky note summarizing the Firecrawl agent’s role and guidance.

- **Nodes Involved:**  
  - `Sticky Note8`

- **Node Details:**  
  - **Sticky Note8**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Provides a high-level label "Firecrawl Agent" describing the core AI processing block.  
    - Positioned near the LangChain agent and GPT nodes for clarity.

---

### 3. Summary Table

| Node Name                | Node Type                               | Functional Role                            | Input Node(s)              | Output Node(s)             | Sticky Note                                                      |
|--------------------------|---------------------------------------|-------------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point webhook for chat messages     | (external)                 | Search Agent               |                                                                 |
| GPT 4.1 mini             | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Language model backend (GPT 4.1 mini)    | When chat message received | Search Agent               |                                                                 |
| Search Agent             | @n8n/n8n-nodes-langchain.agent        | Converts user text to Firecrawl query     | When chat message received, GPT 4.1 mini | Firecrawl Search           |                                                                 |
| Firecrawl Search         | n8n-nodes-base.httpRequestTool         | Executes Firecrawl API search              | Search Agent               | (workflow output)          |                                                                 |
| Site                     | n8n-nodes-base.httpRequest             | Example: Search within a specific site     | (none)                    | (none)                    | ## Within a Specific Website                                    |
| In URL                   | n8n-nodes-base.httpRequest             | Example: Search for word in URL            | (none)                    | (none)                    | ## Word Appears in URL                                          |
| Exclusion                | n8n-nodes-base.httpRequest             | Example: Search excluding terms            | (none)                    | (none)                    | ## Exclude Terms                                               |
| Pro                      | n8n-nodes-base.httpRequest             | Example: Advanced combined search          | (none)                    | (none)                    | ## Pro Tip                                                     |
| Sticky Note1             | n8n-nodes-base.stickyNote              | Explains `Site` query node                  | (none)                    | (none)                    | ## Within a Specific Website                                    |
| Sticky Note2             | n8n-nodes-base.stickyNote              | Explains `In URL` query node                | (none)                    | (none)                    | ## Word Appears in URL                                          |
| Sticky Note3             | n8n-nodes-base.stickyNote              | Explains `Exclusion` query node             | (none)                    | (none)                    | ## Exclude Terms                                               |
| Sticky Note4             | n8n-nodes-base.stickyNote              | Explains `Pro` query node                   | (none)                    | (none)                    | ## Pro Tip                                                     |
| Sticky Note8             | n8n-nodes-base.stickyNote              | Labels the Firecrawl Agent block            | (none)                    | (none)                    | ## Firecrawl Agent                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Add node: `When chat message received` (`@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Use default options.  
   - Note the webhook URL generated for external chat integration.

2. **Add GPT 4.1 mini Node:**  
   - Add node: `GPT 4.1 mini` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`).  
   - Connect input from `When chat message received`.  
   - Configure OpenAI credentials with GPT-4.1 access.  
   - Default options suffice.

3. **Add Search Agent Node:**  
   - Add node: `Search Agent` (`@n8n/n8n-nodes-langchain.agent`).  
   - Connect inputs from both `When chat message received` (main) and `GPT 4.1 mini` (ai_languageModel).  
   - Paste the provided detailed system prompt into the agent’s system message to define query construction rules and tool usage instructions.  
   - Set default limit to 5 if not specified by user.

4. **Add Firecrawl Search Node:**  
   - Add node: `Firecrawl Search` (`n8n-nodes-base.httpRequestTool`).  
   - Connect input from `Search Agent` (ai_tool output).  
   - Configure as HTTP POST to `https://api.firecrawl.dev/v1/search`.  
   - Set body type as JSON with dynamic expressions:  
     - `"query": "{{$fromAI(\"searchQuery\")}}"`  
     - `"limit": {{$fromAI(\"limit\", \"the number of search results requested\", number)}}`  
     - `"scrapeOptions": {"formats": ["markdown", "screenshot@fullPage"]}`  
   - Ensure proper credential setup if Firecrawl API requires authentication.

5. **(Optional) Add Sample Query Nodes for Demonstrations:**  
   - Add HTTP Request nodes named `Site`, `In URL`, `Exclusion`, and `Pro` with static POST requests to Firecrawl API.  
   - Use the provided sample queries in their JSON bodies.  
   - No input connections needed; these nodes serve as examples.

6. **Add Sticky Notes for Documentation:**  
   - Add sticky notes near corresponding nodes with provided content:  
     - Sticky Note1 near `Site`: "Within a Specific Website"  
     - Sticky Note2 near `In URL`: "Word Appears in URL"  
     - Sticky Note3 near `Exclusion`: "Exclude Terms"  
     - Sticky Note4 near `Pro`: "Pro Tip"  
     - Sticky Note8 near the AI nodes: "Firecrawl Agent"

7. **Finalize Connections and Test:**  
   - Verify connections: `When chat message received` → `GPT 4.1 mini` → `Search Agent` → `Firecrawl Search`.  
   - Test webhook by sending chat messages with search instructions.  
   - Confirm results include markdown and screenshots as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                               |
|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| The Firecrawl Agent uses a detailed system prompt to reliably convert natural language into optimized Firecrawl search queries, including operators like site:, inurl:, intitle:, and exclusions. | Embedded in the `Search Agent` node system message. |
| Screenshot capture is requested via `"screenshot@fullPage"` format in Firecrawl's scrapeOptions to obtain full page images of results. | Firecrawl Search node configuration.           |
| Firecrawl API documentation and capabilities can be explored at https://api.firecrawl.dev for advanced usage and troubleshooting. | External API reference.                         |
| For GPT 4.1 mini usage, ensure OpenAI credentials with GPT-4 access are configured in n8n credentials settings.               | OpenAI API management.                          |

---

**Disclaimer:**  
The content described is extracted exclusively from an automated n8n workflow utilizing public APIs and services. All data processed respects applicable content policies and contains no illegal or protected material.