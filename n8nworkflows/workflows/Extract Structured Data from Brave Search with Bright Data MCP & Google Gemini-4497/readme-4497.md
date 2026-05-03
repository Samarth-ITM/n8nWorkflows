Extract Structured Data from Brave Search with Bright Data MCP & Google Gemini

https://n8nworkflows.xyz/workflows/extract-structured-data-from-brave-search-with-bright-data-mcp---google-gemini-4497


# Extract Structured Data from Brave Search with Bright Data MCP & Google Gemini

### 1. Workflow Overview

This n8n workflow automates the process of extracting structured search result data from Brave Search, leveraging Bright Data's MCP (Managed Client Proxy) service for web scraping and Google Gemini (PaLM) large language model for advanced AI processing. It supports multiple search types: images, videos, news, and all-content search. The structured data is parsed, saved locally, optionally sent to a webhook, and appended to a Google Sheets document.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Search Criteria Setup:** Manual trigger initiates the flow; search type and query parameters are set.
- **1.2 Search URL Preparation and Request Routing:** Using a switch node, the workflow routes to different search URL setups depending on the search type.
- **1.3 Web Scraping via Bright Data MCP Client:** Executes the scraping of Brave Search results HTML through Bright Data MCP.
- **1.4 Search Result Extraction and AI Processing:** Extracts raw text from search results, then uses Google Gemini LLM to extract structured data.
- **1.5 Structured Data Parsing:** Parses the AI output into a predefined JSON schema.
- **1.6 Data Output and Distribution:** Saves structured data as a local JSON file, sends a webhook notification with the data, and appends results to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Search Criteria Setup

- **Overview:** This block receives the manual trigger to start the workflow and sets the initial search parameters including search type and query string.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`  
  - `Set the search criteria's`
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution of the workflow.  
    - Configuration: Default manual trigger without parameters.  
    - Input/Output: No input; output triggers `Set the search criteria's`.  
    - Edge cases: None (manual trigger).  
    - Version: 1

  - **Set the search criteria's**  
    - Type: Set  
    - Role: Defines key input variables: `search_type` (default "news"), `search_query` (default "bright+data&source=web&lang=en-in"), and a JSON structured search query for advanced use.  
    - Configuration: Static assignment of values to variables used downstream.  
    - Expressions: N/A, static values set for demonstration.  
    - Input: From manual trigger; output to `Switch`.  
    - Edge cases: User must update `search_type` and queries to match search intent; invalid or empty queries may cause no results.  
    - Version: 3.4

---

#### 2.2 Search URL Preparation and Request Routing

- **Overview:** Routes workflow to the appropriate search URL base on `search_type`, and sets the base URL for the search accordingly.
- **Nodes Involved:**  
  - `Switch`  
  - `Set the input for image search`  
  - `Set the input for video search`  
  - `Set the input fields for news search`  
  - `Set the input fields for All search`
- **Node Details:**

  - **Switch**  
    - Type: Switch  
    - Role: Routes data flow based on `search_type` variable: "images", "videos", "news", or "all".  
    - Configuration: Four string equals conditions matching the search_type field.  
    - Input: From `Set the search criteria's`; outputs to one of four Set nodes.  
    - Edge cases: If `search_type` is not one of the four expected strings, no branch is taken; workflow may halt.  
    - Version: 3.2

  - **Set the input for image search**  
    - Type: Set  
    - Role: Assigns `search_base_url` to Brave's image search URL.  
    - Configuration: Sets `search_base_url` to `"https://search.brave.com/images"`.  
    - Input: From `Switch` if search_type = "images"; output to MCP client node for images.  
    - Edge cases: None expected; URL is static.  
    - Version: 3.4

  - **Set the input for video search**  
    - Type: Set  
    - Role: Assigns `search_base_url` to Brave's video search URL.  
    - Configuration: Sets `search_base_url` to `"https://search.brave.com/video"`.  
    - Input: From `Switch` if search_type = "videos"; output to MCP client node for videos.  
    - Edge cases: None expected.  
    - Version: 3.4

  - **Set the input fields for news search**  
    - Type: Set  
    - Role: Assigns `search_base_url` to Brave's news search URL.  
    - Configuration: Sets `search_base_url` to `"https://search.brave.com/news"`.  
    - Input: From `Switch` if search_type = "news"; output to MCP client node for news.  
    - Edge cases: None expected.  
    - Version: 3.4

  - **Set the input fields for All search**  
    - Type: Set  
    - Role: Assigns `search_base_url` to Brave's general search URL.  
    - Configuration: Sets `search_base_url` to `"https://search.brave.com/search"`.  
    - Input: From `Switch` if search_type = "all"; output to MCP client node for all search.  
    - Edge cases: None expected.  
    - Version: 3.4

---

#### 2.3 Web Scraping via Bright Data MCP Client

- **Overview:** Executes the scraping of Brave Search results HTML by invoking Bright Data MCP Client configured with the appropriate Brave Search URL and query.
- **Nodes Involved:**  
  - `Bright Data MCP Client for Brave Image Search`  
  - `Bright Data MCP Client for Brave Video Search`  
  - `Bright Data MCP Client for Brave News Search`  
  - `Bright Data MCP Client for Brave All Search`
- **Node Details:**

  - **Bright Data MCP Client nodes** (4 nodes, one per search type)  
    - Type: MCP Client (community node)  
    - Role: Scrapes the HTML content of Brave Search result pages using Bright Data's proxy infrastructure.  
    - Configuration: Each node runs the `"scrape_as_html"` tool with the URL constructed as:  
      `{{$json.search_base_url}}?q={{encodeURI($('Set the search criteria\'s').item.json.search_query)}}`  
    - Credentials: Requires valid MCP Client API credentials (configured).  
    - Input: Receives `search_base_url` from respective Set node; uses `search_query` for query.  
    - Output: Returns scraping results with raw HTML content.  
    - Edge cases: Potential failures include MCP service unavailability, network timeout, invalid credentials, or rate limiting.  
    - Version: 1  
    - Notes: Requires self-hosted n8n due to MCP client community node usage.

---

#### 2.4 Search Result Extraction and AI Processing

- **Overview:** Extracts text content from the scraped HTML results, then uses Google Gemini LLM to extract structured data from the raw search result content.
- **Nodes Involved:**  
  - `Set the Search Response`  
  - `Google Gemini Chat Model`  
  - `Structured Data Extractor`
- **Node Details:**

  - **Set the Search Response**  
    - Type: Set  
    - Role: Extracts the main text content from the scrape response and assigns it to `search_response`.  
    - Configuration: Sets `search_response` to `{{$json.result.content[0].text}}` — the first text element from scrape result.  
    - Input: From any of the Bright Data MCP Client nodes.  
    - Output: To `Structured Data Extractor`.  
    - Edge cases: Scrape response might be missing or empty, which would cause downstream AI processing to fail.  
    - Version: 3.4

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini LLM node  
    - Role: Provides advanced AI processing capabilities for generating structured data from text.  
    - Configuration: Model name set to `"models/gemini-2.0-flash-exp"`. No additional options.  
    - Credentials: Uses Google PaLM API credentials.  
    - Input: AI language model input connection from `Structured Data Extractor`.  
    - Output: AI model output to `Structured Data Extractor`.  
    - Edge cases: API quota limits, network errors, or invalid credentials may cause failures.  
    - Version: 1

  - **Structured Data Extractor**  
    - Type: Langchain Chain LLM node  
    - Role: Handles the prompt to extract structured data from the `search_response` text using AI.  
    - Configuration:  
      - Prompt type: `define`  
      - Text input: `Extract structured data from the search result.\n\n{{ $json.search_response }}`  
      - Output parser enabled with a structured output parser downstream.  
    - Input: Receives text from `Set the Search Response` and AI model from `Google Gemini Chat Model`.  
    - Output: To Google Sheets, webhook, and binary file creation.  
    - Edge cases: AI may produce incomplete or malformed output; retry enabled on failure.  
    - Version: 1.6

---

#### 2.5 Structured Data Parsing

- **Overview:** Parses AI-generated output into a structured JSON schema to ensure consistent data fields and formats.
- **Nodes Involved:**  
  - `Structured Output Parser`
- **Node Details:**

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses AI text output into a JSON object following a manual schema definition.  
    - Configuration:  
      - Schema defines `search_results` (array of objects with title, url, description) and optional `related_queries` array of strings.  
      - Enforces required fields `title`, `url`, `description` for each search result.  
    - Input: AI output from `Structured Data Extractor`.  
    - Output: Parsed structured data back into `Structured Data Extractor` node input.  
    - Edge cases: Parsing errors if AI output does not conform to schema; may require prompt tuning.  
    - Version: 1.2

---

#### 2.6 Data Output and Distribution

- **Overview:** Handles output of structured data by saving to Google Sheets, writing to disk as JSON, and sending webhook notifications.
- **Nodes Involved:**  
  - `Google Sheets`  
  - `Create a binary data for Structured Data Extract`  
  - `Write the structured content to disk`  
  - `Initiate a Webhook Notification for the Structured Data`
- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends or updates a Google Sheet with the structured search results.  
    - Configuration:  
      - Document ID and sheet name set to a specific Google Sheets document.  
      - Output column "Output" contains serialized JSON string of `search_results`.  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Input: From `Structured Data Extractor`.  
    - Edge cases: OAuth token expiration, permissions errors, invalid document ID.  
    - Version: 4.5

  - **Create a binary data for Structured Data Extract**  
    - Type: Function  
    - Role: Converts structured JSON data into a base64-encoded binary buffer for file writing.  
    - Configuration: Custom JavaScript function that encodes JSON string.  
    - Input: From `Structured Data Extractor`.  
    - Output: Binary data for file write node.  
    - Edge cases: JSON serialization errors if data malformed.  
    - Version: 1

  - **Write the structured content to disk**  
    - Type: Read & Write File  
    - Role: Writes the binary JSON data to a local file with timestamped filename.  
    - Configuration:  
      - File path: `d:\Brave-Search-Result-<ISO timestamp>.json`  
      - Operation: Write  
    - Input: Binary data from previous node.  
    - Edge cases: File system permission errors, disk full errors, path invalid.  
    - Version: 1

  - **Initiate a Webhook Notification for the Structured Data**  
    - Type: HTTP Request  
    - Role: Sends the structured search results as JSON payload to a configured webhook URL for notifications or further downstream processing.  
    - Configuration:  
      - POST request with body parameter `summary` containing stringified JSON of search results.  
      - Default webhook URL: `https://webhook.site/24878284-919d-4d39-bff0-5f36bfae17b6` (placeholder).  
    - Input: From `Structured Data Extractor`.  
    - Edge cases: Webhook URL invalid or unreachable, network errors.  
    - Version: 4.2

---

### 3. Summary Table

| Node Name                                      | Node Type                            | Functional Role                              | Input Node(s)                        | Output Node(s)                                                   | Sticky Note                                                                                      |
|------------------------------------------------|------------------------------------|----------------------------------------------|------------------------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                   | Manual Trigger                     | Workflow start trigger                       | -                                  | Set the search criteria's                                        |                                                                                                 |
| Set the search criteria's                       | Set                                | Defines search type and query parameters     | When clicking ‘Test workflow’       | Switch                                                          | Please set the search type with videos, images, news, all based on needs.                        |
| Switch                                          | Switch                            | Routes to appropriate search URL setup       | Set the search criteria's           | Set the input for image search; video search; news search; all search |                                                                                                 |
| Set the input for image search                   | Set                                | Sets Brave image search base URL              | Switch (images branch)              | Bright Data MCP Client for Brave Image Search                    |                                                                                                 |
| Set the input for video search                   | Set                                | Sets Brave video search base URL              | Switch (videos branch)              | Bright Data MCP Client for Brave Video Search                    |                                                                                                 |
| Set the input fields for news search             | Set                                | Sets Brave news search base URL               | Switch (news branch)                | Bright Data MCP Client for Brave News Search                     |                                                                                                 |
| Set the input fields for All search              | Set                                | Sets Brave general search base URL            | Switch (all branch)                 | Bright Data MCP Client for Brave All Search                      |                                                                                                 |
| Bright Data MCP Client for Brave Image Search    | MCP Client                      | Scrapes Brave image search results via MCP   | Set the input for image search      | Set the Search Response                                         | This template is only available on n8n self-hosted as it uses community MCP Client node.         |
| Bright Data MCP Client for Brave Video Search    | MCP Client                      | Scrapes Brave video search results via MCP   | Set the input for video search      | Set the Search Response                                         |                                                                                                 |
| Bright Data MCP Client for Brave News Search     | MCP Client                      | Scrapes Brave news search results via MCP    | Set the input fields for news search | Set the Search Response                                         |                                                                                                 |
| Bright Data MCP Client for Brave All Search      | MCP Client                      | Scrapes Brave general search results via MCP | Set the input fields for All search  | Set the Search Response                                         |                                                                                                 |
| Set the Search Response                           | Set                                | Extracts primary text from scrape result      | Bright Data MCP Client nodes        | Structured Data Extractor                                       |                                                                                                 |
| Google Gemini Chat Model                          | Google Gemini LLM (Langchain)      | AI model for structured data extraction       | Structured Data Extractor (ai input) | Structured Data Extractor (ai output)                         | Google Gemini LLM is used here for structured data extraction.                                  |
| Structured Data Extractor                         | Chain LLM (Langchain)              | Extracts structured data from search response | Set the Search Response; Google Gemini Chat Model; Structured Output Parser | Google Sheets, Create a binary data..., Initiate Webhook Notification |                                                                                                 |
| Structured Output Parser                          | Output Parser (Langchain)          | Parses AI output to structured JSON schema    | Structured Data Extractor (ai outputParser) | Structured Data Extractor (ai_outputParser)                   |                                                                                                 |
| Google Sheets                                    | Google Sheets                     | Appends structured data to Google Sheets      | Structured Data Extractor           | -                                                               |                                                                                                 |
| Create a binary data for Structured Data Extract | Function                         | Converts JSON structured data to binary for file | Structured Data Extractor         | Write the structured content to disk                            |                                                                                                 |
| Write the structured content to disk             | Read & Write File                 | Saves structured data to local JSON file      | Create a binary data for Structured Data Extract | -                                                               |                                                                                                 |
| Initiate a Webhook Notification for the Structured Data | HTTP Request                     | Sends structured data JSON to a webhook       | Structured Data Extractor           | -                                                               | Please update the webhook URL to your preferred endpoint.                                       |
| Sticky Note2                                     | Sticky Note                      | Disclaimer about community node usage         | -                                  | -                                                               | This template is only available on n8n self-hosted as it's making use of the community node MCP Client. |
| Sticky Note3                                     | Sticky Note                      | Reminder on search type setting and webhook   | -                                  | -                                                               | Deals with Brave Search by leveraging Bright Data MCP Client. Please set search_type and webhook accordingly. |
| Sticky Note4                                     | Sticky Note                      | LLM usage explanation                          | -                                  | -                                                               | Google Gemini LLM is being utilized for the structured data extraction handling.                |
| Sticky Note5                                     | Sticky Note                      | Bright Data logo display                        | -                                  | -                                                               | ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)       |
| Sticky Note                                      | Sticky Note                      | Structured Data and Output Handling heading    | -                                  | -                                                               |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Manual start of the workflow.

2. **Create a Set Node for Search Criteria**  
   - Name: `Set the search criteria's`  
   - Set variables:  
     - `search_type` (string) e.g. "news" (can be "images", "videos", "news", "all")  
     - `search_query` (string) e.g. "bright+data&source=web&lang=en-in"  
     - `json_search_query` (string) with JSON text for advanced search (optional)  
   - Connect output of Manual Trigger to this node.

3. **Create a Switch Node to Route by Search Type**  
   - Name: `Switch`  
   - Add rules for exact string matches on `search_type`: "images", "videos", "news", "all".  
   - Connect output of Set Search Criteria to this node.

4. **Create Four Set Nodes for Base URLs**  
   - `Set the input for image search`: set `search_base_url` = `https://search.brave.com/images`  
   - `Set the input for video search`: set `search_base_url` = `https://search.brave.com/video`  
   - `Set the input fields for news search`: set `search_base_url` = `https://search.brave.com/news`  
   - `Set the input fields for All search`: set `search_base_url` = `https://search.brave.com/search`  
   - Connect each output of Switch to corresponding Set node.

5. **Create Four MCP Client Nodes for Scraping**  
   - For each search type, create an MCP Client node:  
     - Tool: `scrape_as_html`  
     - Operation: `executeTool`  
     - Tool parameters:  
       ```json
       {
         "url": "{{$json.search_base_url}}?q={{encodeURI($('Set the search criteria\\'s').item.json.search_query)}}"
       }
       ```  
     - Credentials: Use valid Bright Data MCP Client API credentials.  
   - Connect each Set base URL node to its corresponding MCP Client node.

6. **Create a Set Node to Extract Search Response Text**  
   - Name: `Set the Search Response`  
   - Assign `search_response` = `{{$json.result.content[0].text}}` (extracts main textual content from scrape result)  
   - Connect outputs of all MCP Client nodes to this node (fan-in).

7. **Create a Langchain Chain LLM Node for Structured Extraction**  
   - Name: `Structured Data Extractor`  
   - Parameters:  
     - Prompt Type: `define`  
     - Text: `"Extract structured data from the search result.\n\n{{ $json.search_response }}"`  
     - Enable output parser.  
   - Connect output of Set the Search Response to this node.

8. **Create Langchain Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credentials: Google PaLM API credentials  
   - Connect as AI language model input to `Structured Data Extractor`.

9. **Create Langchain Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Schema (manual) matching the expected structure:  
     - `search_results`: array of objects with `title` (string), `url` (string, URI), `description` (string)  
     - Optional `related_queries`: array of strings  
   - Connect AI output parser output of `Structured Data Extractor` to this parser node, and parser output back to `Structured Data Extractor`.

10. **Create Google Sheets Node**  
    - Name: `Google Sheets`  
    - Operation: Append or update  
    - Document ID and Sheet name: provide target Google Sheet access  
    - Map column "Output" to `{{$json.output.search_results.toJsonString()}}`  
    - Credentials: Google Sheets OAuth2  
    - Connect output of `Structured Data Extractor` to this node.

11. **Create Function Node to Convert JSON to Binary**  
    - Name: `Create a binary data for Structured Data Extract`  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64'),
        },
      };
      return items;
      ```  
    - Connect output of `Structured Data Extractor` to this node.

12. **Create Read & Write File Node**  
    - Name: `Write the structured content to disk`  
    - Operation: Write  
    - File Name: `d:\Brave-Search-Result-{{new Date().toISOString().replaceAll(":",".")}}.json`  
    - Connect output of Function node to this node.

13. **Create HTTP Request Node to Send Webhook Notification**  
    - Name: `Initiate a Webhook Notification for the Structured Data`  
    - Method: POST  
    - URL: Your webhook URL (default example: `https://webhook.site/...`)  
    - Body parameter: `summary` = `{{$json.output.search_results.toJsonString()}}`  
    - Connect output of `Structured Data Extractor` to this node.

14. **Add Sticky Notes as Needed**  
    - For disclaimers, LLM usage, notes on MCP client community node requirement, webhook reminders, and branding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires n8n self-hosted because it uses the community MCP Client node for Bright Data scraping.                                                  | See Sticky Note2                                                                                   |
| Google Gemini LLM (PaLM API) is used for AI-based structured data extraction from search results.                                                              | See Sticky Note4                                                                                   |
| Users must set the `search_type` parameter correctly to one of "images", "videos", "news", or "all" based on the desired Brave Search category.                 | See Sticky Note3                                                                                   |
| Please update the webhook URL in the HTTP Request node to your actual webhook endpoint for notifications.                                                       | See Sticky Note3                                                                                   |
| Bright Data Logo used for branding in Sticky Note5                                                                                                              | ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |
| For Google Sheets integration, ensure OAuth2 credentials have write access to the target spreadsheet.                                                           | N8N Google Sheets OAuth2 docs                                                                      |
| MCP Client API credentials must be properly configured and active to avoid scraping failures due to authentication or network issues.                         | Bright Data MCP docs                                                                               |

---

**Disclaimer:**  
The provided documentation is based exclusively on an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.