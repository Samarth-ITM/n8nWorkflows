Build your ow Youtube MCP server

https://n8nworkflows.xyz/workflows/build-your-ow-youtube-mcp-server-3637


# Build your ow Youtube MCP server

### 1. Workflow Overview

This workflow implements a custom MCP (Modular Chatbot Protocol) server for YouTube video research, enabling users to search YouTube videos and retrieve their transcripts for AI-driven analysis. It is designed for research purposes where video content is lengthy or complex, allowing AI agents to process and extract insights from video transcripts efficiently.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger & Input Reception:** Listens for incoming MCP client requests specifying operations such as YouTube search, transcript retrieval, or usage metrics.
- **1.2 Operation Routing:** Routes the request to the appropriate processing block based on the requested operation.
- **1.3 YouTube Search Processing:** Uses Apify’s YouTube scraper API to search videos by query and simplifies the results.
- **1.4 YouTube Transcript Retrieval:** Uses Apify’s YouTube scraper API to download video transcripts from provided URLs and simplifies the transcript data.
- **1.5 Usage Metrics Reporting:** Retrieves and simplifies monthly usage and spending metrics from the Apify account to monitor API usage.
- **1.6 Aggregation & Response:** Aggregates results from search or transcript retrieval before sending back to the MCP client.

The workflow leverages external scraping services via Apify.com to avoid YouTube API rate limits and simplify data extraction. It is designed to be integrated with MCP clients such as Claude Desktop or other AI agents.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger & Input Reception

- **Overview:**  
  This block listens for incoming MCP client requests and extracts input parameters such as operation type, search query, or video URLs.

- **Nodes Involved:**  
  - Apify Youtube MCP Server (MCP Trigger)  
  - When Executed by Another Workflow (Execute Workflow Trigger)

- **Node Details:**

  - **Apify Youtube MCP Server**  
    - Type: MCP Trigger (Langchain)  
    - Role: Entry point for MCP client requests via webhook.  
    - Configuration: Webhook path set to a unique ID; listens for MCP requests.  
    - Inputs: MCP client requests containing JSON with `operation`, `query`, and `urls`.  
    - Outputs: Passes data to the Execute Workflow Trigger node.  
    - Edge Cases: Requires authentication for production use to prevent unauthorized access.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Receives inputs from MCP Trigger and passes them downstream.  
    - Configuration: Defines expected inputs `operation`, `query`, and `urls`.  
    - Inputs: Triggered by MCP Server node.  
    - Outputs: Sends data to the Operation switch node.  
    - Edge Cases: Input validation failure if expected parameters are missing or malformed.

---

#### 2.2 Operation Routing

- **Overview:**  
  Routes the workflow execution based on the `operation` parameter to one of three processing paths: YouTube Search, YouTube Transcripts, or Usage Metrics.

- **Nodes Involved:**  
  - Operation (Switch)

- **Node Details:**

  - **Operation**  
    - Type: Switch  
    - Role: Routes workflow based on `operation` field in input JSON.  
    - Configuration:  
      - Routes `"youtube_search"` to YouTube Search path.  
      - Routes `"youtube_transcripts"` to YouTube Transcripts path.  
      - Routes `"usage_metrics"` to Usage Metrics path.  
    - Inputs: Receives JSON with `operation` from Execute Workflow Trigger.  
    - Outputs: Three outputs connected to respective processing nodes.  
    - Edge Cases: If `operation` is missing or unrecognized, no output path is triggered.

---

#### 2.3 YouTube Search Processing

- **Overview:**  
  Performs a YouTube search using Apify’s YouTube scraper API, simplifies the results, and aggregates them for response.

- **Nodes Involved:**  
  - Youtube Search (Tool Workflow)  
  - Apify Youtube Search (HTTP Request)  
  - Simplify Search Results (Set)  
  - Aggregate Search Results (Aggregate)

- **Node Details:**

  - **Youtube Search**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates the YouTube search logic as a reusable tool.  
    - Configuration: Passes `query` and `operation` to the sub-workflow.  
    - Inputs: Receives `query` and `operation` from Operation node.  
    - Outputs: Sends data to Apify Youtube Search node.  
    - Edge Cases: Requires valid `query` string; empty or invalid queries may return no results.

  - **Apify Youtube Search**  
    - Type: HTTP Request  
    - Role: Calls Apify API to perform YouTube search.  
    - Configuration:  
      - POST to Apify’s YouTube scraper run-sync endpoint.  
      - JSON body includes `searchQueries` array with the query, `maxResults` set to 5.  
      - Authentication via HTTP Header with Apify personal token credential.  
    - Inputs: Receives search query from Youtube Search node.  
    - Outputs: Returns raw search results JSON.  
    - Edge Cases: API rate limits, network errors, invalid token, or malformed query may cause failure.

  - **Simplify Search Results**  
    - Type: Set  
    - Role: Extracts and simplifies key fields from raw search results for easier consumption.  
    - Configuration: Maps fields such as `channelName`, `title`, `url`, truncated `description`, `viewCount`, and `likes`.  
    - Inputs: Raw JSON from Apify Youtube Search.  
    - Outputs: Simplified JSON objects.  
    - Edge Cases: Missing fields in API response may cause undefined values.

  - **Aggregate Search Results**  
    - Type: Aggregate  
    - Role: Aggregates all simplified search result items into a single response array.  
    - Configuration: Aggregates all item data into a field named `response`.  
    - Inputs: Simplified search results.  
    - Outputs: Aggregated array for MCP response.  
    - Edge Cases: Empty input results in empty aggregation.

---

#### 2.4 YouTube Transcript Retrieval

- **Overview:**  
  Downloads transcripts for one or more YouTube video URLs using Apify’s scraper, simplifies the transcript data, and aggregates it.

- **Nodes Involved:**  
  - Youtube Transcripts (Tool Workflow)  
  - Apify Youtube Transcripts (HTTP Request)  
  - Simplify Transcript Results (Set)  
  - Aggregate Transcript Results (Aggregate)

- **Node Details:**

  - **Youtube Transcripts**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates transcript retrieval logic as a reusable tool.  
    - Configuration: Passes `urls` and `operation` to the sub-workflow.  
    - Inputs: Receives `urls` (comma-separated string) and `operation` from Operation node.  
    - Outputs: Sends data to Apify Youtube Transcripts node.  
    - Edge Cases: Requires valid URLs; malformed or empty URLs cause no transcript retrieval.

  - **Apify Youtube Transcripts**  
    - Type: HTTP Request  
    - Role: Calls Apify API to download video transcripts.  
    - Configuration:  
      - POST to Apify’s YouTube scraper run-sync endpoint.  
      - JSON body includes `startUrls` array constructed from splitting `urls` string.  
      - Requests subtitles in plaintext English format.  
      - Authentication via HTTP Header with Apify personal token credential.  
      - Retries twice on failure with 5-second wait between tries.  
    - Inputs: Receives URLs from Youtube Transcripts node.  
    - Outputs: Raw transcript JSON data.  
    - Edge Cases: Network failures, invalid URLs, or missing subtitles may cause empty or failed responses.

  - **Simplify Transcript Results**  
    - Type: Set  
    - Role: Extracts key transcript fields such as `title`, `url`, and plaintext `transcript`.  
    - Configuration: Maps first subtitle plaintext from API response.  
    - Inputs: Raw JSON from Apify Youtube Transcripts.  
    - Outputs: Simplified transcript JSON.  
    - Edge Cases: Missing subtitles or empty transcript fields.

  - **Aggregate Transcript Results**  
    - Type: Aggregate  
    - Role: Aggregates all simplified transcript items into a single response array.  
    - Configuration: Aggregates all item data into a field named `response`.  
    - Inputs: Simplified transcript results.  
    - Outputs: Aggregated array for MCP response.  
    - Edge Cases: Empty input results in empty aggregation.

---

#### 2.5 Usage Metrics Reporting

- **Overview:**  
  Retrieves monthly usage and spending metrics from the Apify account and simplifies the data for monitoring.

- **Nodes Involved:**  
  - Usage Report (Tool Workflow)  
  - Get Usage Metrics (HTTP Request)  
  - Get Usage Limits (HTTP Request)  
  - Simplify Usage Metrics (Set)

- **Node Details:**

  - **Usage Report**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates usage metrics retrieval logic as a reusable tool.  
    - Configuration: Passes fixed `operation` value for usage report.  
    - Inputs: Triggered by Operation node for `usage_metrics`.  
    - Outputs: Sends data to Get Usage Metrics node.  
    - Edge Cases: None specific; depends on API availability.

  - **Get Usage Metrics**  
    - Type: HTTP Request  
    - Role: Calls Apify API to retrieve current monthly usage data.  
    - Configuration:  
      - GET request to `/users/me/usage/monthly` endpoint.  
      - Authentication via HTTP Header with Apify personal token credential.  
    - Inputs: Triggered by Usage Report node.  
    - Outputs: Returns monthly usage JSON.  
    - Edge Cases: Authentication failure, API downtime.

  - **Get Usage Limits**  
    - Type: HTTP Request  
    - Role: Calls Apify API to retrieve usage limits for the user.  
    - Configuration:  
      - GET request to `/users/me/limits` endpoint.  
      - Authentication via HTTP Header with Apify personal token credential.  
    - Inputs: Triggered by Get Usage Metrics node.  
    - Outputs: Returns usage limits JSON.  
    - Edge Cases: Authentication failure, API downtime.

  - **Simplify Usage Metrics**  
    - Type: Set  
    - Role: Extracts and formats key usage metrics and limits for reporting.  
    - Configuration: Maps fields such as monthly usage cycle dates, current spending vs limit, and detailed service usage costs.  
    - Inputs: Combined data from Get Usage Metrics and Get Usage Limits nodes.  
    - Outputs: Simplified usage report JSON.  
    - Edge Cases: Missing or incomplete data fields.

---

#### 2.6 Aggregation & Response

- **Overview:**  
  Aggregates the processed data from search or transcript retrieval before sending it back to the MCP client.

- **Nodes Involved:**  
  - Aggregate Search Results  
  - Aggregate Transcript Results

- **Node Details:**

  - These nodes aggregate all individual items into a single array under the `response` field, preparing the data for MCP client consumption.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                                            |
|-----------------------------|--------------------------------|----------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Apify Youtube MCP Server     | MCP Trigger (Langchain)         | Entry point for MCP client requests    | —                            | When Executed by Another Workflow | See Sticky Note2 for detailed workflow description and usage instructions.                                                                             |
| When Executed by Another Workflow | Execute Workflow Trigger      | Receives MCP inputs and triggers flow  | Apify Youtube MCP Server      | Operation                     |                                                                                                                                                        |
| Operation                   | Switch                         | Routes based on operation parameter     | When Executed by Another Workflow | Youtube Search, Youtube Transcripts, Usage Report |                                                                                                                                                        |
| Youtube Search              | Tool Workflow (Langchain)      | Encapsulates YouTube search logic      | Operation                    | Apify Youtube Search          |                                                                                                                                                        |
| Apify Youtube Search        | HTTP Request                   | Calls Apify API for YouTube search     | Youtube Search               | Simplify Search Results       |                                                                                                                                                        |
| Simplify Search Results     | Set                           | Simplifies YouTube search results      | Apify Youtube Search         | Aggregate Search Results      |                                                                                                                                                        |
| Aggregate Search Results    | Aggregate                     | Aggregates search results               | Simplify Search Results      | Youtube Search (output)       |                                                                                                                                                        |
| Youtube Transcripts         | Tool Workflow (Langchain)      | Encapsulates transcript retrieval logic| Operation                    | Apify Youtube Transcripts     |                                                                                                                                                        |
| Apify Youtube Transcripts   | HTTP Request                   | Calls Apify API to download transcripts| Youtube Transcripts          | Simplify Transcript Results   |                                                                                                                                                        |
| Simplify Transcript Results | Set                           | Simplifies transcript data              | Apify Youtube Transcripts    | Aggregate Transcript Results  |                                                                                                                                                        |
| Aggregate Transcript Results| Aggregate                     | Aggregates transcript results           | Simplify Transcript Results  | Youtube Transcripts (output)  |                                                                                                                                                        |
| Usage Report               | Tool Workflow (Langchain)      | Encapsulates usage metrics retrieval   | Operation                    | Get Usage Metrics             |                                                                                                                                                        |
| Get Usage Metrics          | HTTP Request                   | Retrieves monthly usage data            | Usage Report                 | Get Usage Limits              |                                                                                                                                                        |
| Get Usage Limits           | HTTP Request                   | Retrieves usage limits                  | Get Usage Metrics            | Simplify Usage Metrics        |                                                                                                                                                        |
| Simplify Usage Metrics     | Set                           | Simplifies usage metrics data           | Get Usage Limits             | Usage Report (output)         |                                                                                                                                                        |
| Sticky Note                | Sticky Note                   | Instruction: MCP Server Trigger setup  | —                            | —                             | [Read more about the MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger)                          |
| Sticky Note1               | Sticky Note                   | Info on Apify.com for YouTube scraping | —                            | —                             | [Sign up for Apify.com using 20JIMLEUK for 20% discount](https://www.apify.com?fpr=414q6)                                                              |
| Sticky Note2               | Sticky Note                   | Detailed workflow description and usage| —                            | —                             | Full workflow explanation, usage instructions, and customization tips.                                                                                |
| Sticky Note3               | Sticky Note                   | Reminder to enable authentication      | —                            | —                             | Always Authenticate Your Server! Before going to production, enable authentication on your MCP server trigger.                                        |
| Sticky Note4               | Sticky Note                   | Apify branding banner                   | —                            | —                             | [Apify.com](https://www.apify.com?fpr=414q6)                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set `path` to a unique webhook ID (e.g., UUID).  
   - This node will listen for MCP client requests.

2. **Create Execute Workflow Trigger Node**  
   - Node Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Configure expected inputs: `operation` (string), `query` (string), `urls` (string).  
   - Connect MCP Server Trigger output to this node.

3. **Create Switch Node for Operation Routing**  
   - Node Type: `n8n-nodes-base.switch`  
   - Add three rules based on `operation` field:  
     - `"youtube_search"` → output 1  
     - `"youtube_transcripts"` → output 2  
     - `"usage_metrics"` → output 3  
   - Connect Execute Workflow Trigger output to this node.

4. **YouTube Search Sub-Workflow Setup**  
   - Create a Tool Workflow node named `Youtube Search`.  
   - Configure inputs: `operation`, `query`, `urls`.  
   - Set `operation` to `"youtube_search"`.  
   - Connect Switch output 1 to this node.

5. **Apify YouTube Search HTTP Request Node**  
   - Node Type: `n8n-nodes-base.httpRequest`  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/run-sync-get-dataset-items`  
   - Body (JSON):  
     ```json
     {
       "searchQueries": ["{{$json.query}}"],
       "maxResultStreams": 0,
       "maxResults": 5
     }
     ```  
   - Authentication: HTTP Header Auth with Apify personal token credential.  
   - Connect `Youtube Search` node output to this node.

6. **Simplify Search Results Node**  
   - Node Type: Set  
   - Map fields:  
     - `channelName` ← `$json.channelName`  
     - `title` ← `$json.title`  
     - `url` ← `$json.url`  
     - `description` ← first 2000 chars of `$json.text`  
     - `viewCount` ← `$json.viewCount`  
     - `likes` ← `$json.likes`  
   - Connect Apify YouTube Search output to this node.

7. **Aggregate Search Results Node**  
   - Node Type: Aggregate  
   - Aggregate all items into field `response`.  
   - Connect Simplify Search Results output to this node.

8. **YouTube Transcripts Sub-Workflow Setup**  
   - Create a Tool Workflow node named `Youtube Transcripts`.  
   - Configure inputs: `operation`, `query`, `urls`.  
   - Set `operation` to `"youtube_transcripts"`.  
   - Connect Switch output 2 to this node.

9. **Apify YouTube Transcripts HTTP Request Node**  
   - Node Type: `n8n-nodes-base.httpRequest`  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/run-sync-get-dataset-items`  
   - Body (JSON):  
     ```json
     {
       "downloadSubtitles": true,
       "startUrls": {{$json.urls.split(',').map(url => ({ "url": url, "method": "GET" }))}},
       "subtitlesFormat": "plaintext",
       "subtitlesLanguage": "en",
       "maxResults": 1
     }
     ```  
   - Authentication: HTTP Header Auth with Apify personal token credential.  
   - Retry on failure: 2 times with 5 seconds delay.  
   - Connect `Youtube Transcripts` node output to this node.

10. **Simplify Transcript Results Node**  
    - Node Type: Set  
    - Map fields:  
      - `title` ← `$json.title`  
      - `url` ← `$json.url`  
      - `transcript` ← first subtitle plaintext `$json.subtitles[0].plaintext`  
    - Connect Apify YouTube Transcripts output to this node.

11. **Aggregate Transcript Results Node**  
    - Node Type: Aggregate  
    - Aggregate all items into field `response`.  
    - Connect Simplify Transcript Results output to this node.

12. **Usage Metrics Sub-Workflow Setup**  
    - Create a Tool Workflow node named `Usage Report`.  
    - Configure inputs: `operation`, `query`, `urls`.  
    - Set `operation` to `"usage_metrics"`.  
    - Connect Switch output 3 to this node.

13. **Get Usage Metrics HTTP Request Node**  
    - Node Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.apify.com/v2/users/me/usage/monthly`  
    - Authentication: HTTP Header Auth with Apify personal token credential.  
    - Connect Usage Report output to this node.

14. **Get Usage Limits HTTP Request Node**  
    - Node Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.apify.com/v2/users/me/limits`  
    - Authentication: HTTP Header Auth with Apify personal token credential.  
    - Connect Get Usage Metrics output to this node.

15. **Simplify Usage Metrics Node**  
    - Node Type: Set  
    - Map fields such as:  
      - `monthlyUsageCycle_startAt` ← `$json.data.monthlyUsageCycle.startAt`  
      - `monthlyUsageCycle_endAt` ← `$json.data.monthlyUsageCycle.endAt`  
      - `monthlyUsageUsd` ← formatted string of current usage vs limit  
      - Detailed service usage costs from `$json.data.monthlyServiceUsage`  
    - Connect Get Usage Limits output to this node.

16. **Connect all Tool Workflow outputs back to MCP Server Trigger**  
    - Connect outputs of Aggregate Search Results, Aggregate Transcript Results, and Simplify Usage Metrics nodes to the MCP Server Trigger’s `ai_tool` input to send responses back to the client.

17. **Credential Setup**  
    - Create HTTP Header Auth credential in n8n with your Apify personal token.  
    - Assign this credential to all HTTP Request nodes calling Apify APIs.

18. **Final Checks**  
    - Add authentication to MCP Server Trigger node before production.  
    - Test with sample queries such as `"what is MCP?"`, `"How can I use MCP in n8n?"`, and `"How can I use Apify's official MCP server?"`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates building a YouTube MCP server for research using Apify.com scrapers to bypass YouTube API rate limits.                                                                                            | Workflow description and design rationale.                                                                  |
| MCP Server Trigger documentation and integration guide.                                                                                                                                                                       | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop |
| Apify.com recommended for YouTube scraping with a $5 free tier and 20% discount using code `20JIMLEUK`.                                                                                                                       | https://www.apify.com?fpr=414q6                                                                              |
| Always enable authentication on MCP Server Trigger before production to secure your server.                                                                                                                                  | Security best practice.                                                                                       |
| MCP clients such as Claude Desktop can connect to this MCP server for AI-driven YouTube research.                                                                                                                             | https://claude.ai/download                                                                                   |
| Consider adding more Apify actors or using Apify’s official MCP server for access to 4000+ tools.                                                                                                                             | Workflow customization tip.                                                                                   |

---

This comprehensive reference document enables advanced users and AI agents to fully understand, reproduce, and modify the YouTube MCP server workflow, anticipate potential errors, and integrate it effectively with MCP clients and Apify services.