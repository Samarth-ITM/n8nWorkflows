[New York Times] Article Search API MCP Server

https://n8nworkflows.xyz/workflows/-new-york-times--article-search-api-mcp-server-5649


# [New York Times] Article Search API MCP Server

### 1. Workflow Overview

This workflow, titled **[New York Times] Article Search API MCP Server**, transforms the New York Times Article Search API into an MCP (Multi-Channel Platform) server compatible with AI agents. It enables AI-driven queries to the New York Times archive, covering articles from 1851 to the present, returning detailed metadata such as headlines, abstracts, multimedia links, and article content.

**Target Use Cases:**  
- AI agents querying historical or recent articles from the New York Times database.  
- Integration of article search capabilities into AI chatbots or automation systems with dynamic parameter input.  
- Returning structured API responses directly to AI agents without manual handling.

**Logical Blocks:**

- **1.1 Setup & Documentation**: Provides setup instructions and workflow overview to guide users on configuration and usage.  
- **1.2 MCP Server Endpoint**: Implements the MCP trigger node acting as the server endpoint for AI agent requests.  
- **1.3 Article Search Operation**: Executes the actual API call to the New York Times Article Search API with dynamic parameters populated from AI input.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation

**Overview:**  
This block contains sticky notes that provide essential setup instructions, usage notes, customization tips, and a detailed overview of the workflow’s purpose and structure. It serves as quick-reference documentation embedded within the workflow.

**Nodes Involved:**  
- Setup Instructions (sticky note)  
- Workflow Overview (sticky note)  
- Sticky Note ("Stories" label)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Provides comprehensive setup guidance including importing the workflow, API key credential configuration, activation steps, MCP URL retrieval, AI agent connection, usage notes, customization options, and support contacts.  
  - Key Content Highlights:
    - API key configured as query parameter named `api-key`
    - MCP trigger webhook URL used for AI agent integration
    - Parameters auto-populated via `$fromAI()` expressions
    - Suggestion to add custom error handling, logging, or data transformation nodes  
  - Input/Output: None (informational only)  
  - Edge Cases: None  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Summarizes the workflow's functionality, explaining the Article Search API capabilities, the MCP trigger role, HTTP request handling, and available operations (one endpoint: Search Articles).  
  - Input/Output: None  
  - Edge Cases: None  

- **Sticky Note ("Stories")**  
  - Type: Sticky Note  
  - Role: Simple label to visually group or identify the Article Search operation block.  
  - Input/Output: None  
  - Edge Cases: None  

---

#### 1.2 MCP Server Endpoint

**Overview:**  
This block sets up the MCP trigger node that acts as the server endpoint to receive and process requests from AI agents. It listens on a specific path and initiates the workflow execution in response to incoming requests.

**Nodes Involved:**  
- Article Search MCP Server (MCP Trigger)

**Node Details:**

- **Article Search MCP Server**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Entry point for AI agent requests via an MCP-compatible webhook.  
  - Configuration:
    - Webhook path: `article-search-mcp`  
    - This serves as the listening endpoint URL for AI agents to send requests.  
  - Input: External HTTP requests from AI agents.  
  - Output: Triggers downstream nodes for processing the request.  
  - Version-specific: Requires n8n version supporting MCP Trigger node (LangChain integration).  
  - Edge Cases:
    - Network or webhook unavailability issues  
    - Malformed or unauthorized requests may require additional validation nodes if implemented  
  - Sub-workflow: None  

---

#### 1.3 Article Search Operation

**Overview:**  
This block performs the actual querying of the New York Times Article Search API using dynamic parameters populated from AI input via `$fromAI()` expressions. It returns the full native API response to the requesting AI agent.

**Nodes Involved:**  
- Search Articles (HTTP Request Tool)

**Node Details:**

- **Search Articles**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Executes an HTTP GET request to the New York Times Article Search API endpoint to fetch articles based on AI-provided search parameters.  
  - Configuration:
    - URL: `http://api.nytimes.com/svc/search/v2/articlesearch.json`  
    - Authentication: API Key passed as query parameter (`api-key`) configured via n8n credentials (generic queryAuth)  
    - Query Parameters:  
      - `q`: Search query term (string, optional)  
      - `fq`: Filtered search query using Lucene syntax (string, optional)  
      - `begin_date`: Start date filter in `YYYYMMDD` format (string, optional)  
      - `end_date`: End date filter in `YYYYMMDD` format (string, optional)  
      - `sort`: Result sorting, default relevance or `pub_date` (string, optional)  
      - `fl`: Comma-delimited fields to return (string, optional)  
      - `hl`: Highlighting enabled (boolean, optional, default false)  
      - `page`: Pagination index, each page corresponds to 10 results (number, default 0)  
      - `facet_field`: Comma-delimited list of facets to include (string, optional)  
      - `facet_filter`: Apply filters to facets (boolean, optional, default false)  
    - Each parameter uses `$fromAI()` expressions with descriptive hints for AI dynamic filling.  
  - Input: Triggered by MCP trigger node (Article Search MCP Server)  
  - Output: API response JSON returned directly to AI agent via MCP framework  
  - Version-specific: Requires HTTP Request Tool node version 4.2+ for tool integration and expression support  
  - Edge Cases:
    - API key invalid or missing causing authentication errors  
    - Rate limiting or API downtime leading to request failures  
    - Invalid or malformed query parameters causing API errors  
    - Timeout or network connectivity issues  
  - Sub-workflow: None  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                   | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                                                                                                                                     |
|-------------------------|--------------------------------|---------------------------------|-------------------------|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions      | Sticky Note                    | Setup and usage documentation    | None                    | None                   | ### ⚙️ Setup Instructions 1. **Import Workflow**: Load this workflow into your n8n instance 2. **Configure Authentication**: Set up apiKey credentials - Type: API Key in query - Key name: api-key 3. **Activate Workflow**: Enable the workflow to start the MCP server 4. **Get MCP URL**: Copy the webhook URL from the MCP trigger 5. **Connect AI Agent**: Use the MCP URL in your AI agent configuration … [discord](https://discord.me/cfomodz) and [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) |
| Workflow Overview      | Sticky Note                    | Workflow description and summary| None                    | None                   | ## 🛠️ Article Search MCP Server ✅ 1 operations About The Article Search API lets you query New York Times articles from 1851 to present, returning metadata including headlines, abstracts, multimedia links, and article content. |
| Sticky Note ("Stories") | Sticky Note                    | Visual label for grouping        | None                    | None                   | ## Stories                                                                                                                                                                                                                      |
| Article Search MCP Server| MCP Trigger                   | Entry point for AI requests      | None                    | Search Articles         |                                                                                                                                                                                                                                |
| Search Articles         | HTTP Request Tool             | Executes Article Search API call | Article Search MCP Server| None                   | Article Search Parameters include q, fq, begin_date, end_date, sort, fl, hl, page, facet_field, facet_filter dynamically populated by AI inputs via $fromAI().                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: "Setup Instructions"**  
   - Type: Sticky Note  
   - Content: Copy all setup instructions provided, including API key setup, workflow activation, MCP URL retrieval, AI agent connection, customization tips, and support links.  
   - Position on canvas: (-1380, -240)  

2. **Create Sticky Note: "Workflow Overview"**  
   - Type: Sticky Note  
   - Content: Paste the provided workflow overview describing the Article Search API, MCP trigger role, HTTP request usage, AI parameter auto-population, and available operations.  
   - Size: Width 420, Height 920  
   - Position: (-1120, -240)  

3. **Create Sticky Note: "Stories"**  
   - Type: Sticky Note  
   - Content: "## Stories"  
   - Color: Yellow (color index 2)  
   - Size: Width 300, Height 200  
   - Position: (-660, -100)  

4. **Create MCP Trigger Node: "Article Search MCP Server"**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `article-search-mcp`  
   - Position: (-620, -240)  
   - This node will serve as the webhook endpoint for AI agent requests.  

5. **Create HTTP Request Tool Node: "Search Articles"**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - Parameters:  
     - URL: `http://api.nytimes.com/svc/search/v2/articlesearch.json`  
     - Authentication: Generic Credentials configured as an API Key in query parameter named `api-key`  
     - Send Query: True (Enable query parameters)  
     - Query Parameters (each using expressions to get AI inputs):  
       - `q`: `={{ $fromAI('q', 'Search query term. Search is performed on the article body, headline and byline.', 'string') }}`  
       - `fq`: `={{ $fromAI('fq', 'Filtered search query using standard Lucene syntax.', 'string') }}`  
       - `begin_date`: `={{ $fromAI('begin_date', 'Format: YYYYMMDD Restricts responses to results with publication dates of the date specified or later.', 'string') }}`  
       - `end_date`: `={{ $fromAI('end_date', 'Format: YYYYMMDD Restricts responses to results with publication dates of the date specified or earlier.', 'string') }}`  
       - `sort`: `={{ $fromAI('sort', 'By default, search results are sorted by relevance. Use sort=pub_date to sort by publication date.', 'string') }}`  
       - `fl`: `={{ $fromAI('fl', 'Comma-delimited list of fields to return.', 'string') }}`  
       - `hl`: `={{ $fromAI('hl', 'Enable highlighting in results.', 'boolean', false) }}`  
       - `page`: `={{ $fromAI('page', 'Pagination index for results.', 'number', 0) }}`  
       - `facet_field`: `={{ $fromAI('facet_field', 'Comma-delimited list of facets.', 'string') }}`  
       - `facet_filter`: `={{ $fromAI('facet_filter', 'Apply filters to facets.', 'boolean', false) }}`  
     - Tool Description: Include full parameter descriptions for clarity (optional but recommended).  
   - Position: (-520, -60)  

6. **Connect Nodes:**  
   - Connect output of "Article Search MCP Server" (MCP trigger) to input of "Search Articles" (HTTP Request Tool).  

7. **Credential Setup:**  
   - Configure a generic API Key credential in n8n with:  
     - Credential Type: API Key in query  
     - Key Name: `api-key`  
     - Enter your New York Times API key value.  

8. **Workflow Activation:**  
   - Save and activate the workflow.  
   - Retrieve the webhook URL from the MCP Trigger node (will be of form `https://<your-n8n-domain>/webhook/article-search-mcp`).  

9. **AI Agent Integration:**  
   - Provide the MCP URL to your AI agent configuration.  
   - AI agent queries will dynamically populate parameters using `$fromAI()` expressions embedded in the HTTP Request node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| For integration help or custom automation guidance, contact the maintainer on Discord: https://discord.me/cfomodz                                                                                                                                               | Support contact for workflow integration                                                                                      |
| Detailed documentation on MCP nodes and LangChain integration can be found at n8n docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                                                             | Official n8n documentation for MCP trigger nodes                                                                             |
| The New York Times Article Search API documentation explains filtering, facets, highlighting, and valid query parameters: https://developer.nytimes.com/docs/articlesearch-product/1/overview                                                                      | API reference for deeper understanding of query parameters and response structure                                              |
| This workflow assumes you have valid New York Times API Key; obtain it from https://developer.nytimes.com/                                                                                                                 | API key registration source                                                                                                   |
| Timezone configured as America/New_York to align with New York Times API request context and data relevance                                                                                                               | Workflow setting detail                                                                                                       |

---

**Disclaimer:**  
The provided text and workflow are created solely using n8n automation tools. All data handled is legal and public. The workflow complies strictly with content policies and does not contain illegal or offensive material.