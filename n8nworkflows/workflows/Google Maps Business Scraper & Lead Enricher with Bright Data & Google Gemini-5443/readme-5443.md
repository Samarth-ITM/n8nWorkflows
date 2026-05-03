Google Maps Business Scraper & Lead Enricher with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/google-maps-business-scraper---lead-enricher-with-bright-data---google-gemini-5443


# Google Maps Business Scraper & Lead Enricher with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the scraping and enrichment of business lead data from Google Maps and Yelp using Bright Data’s proxy service and Google Gemini Large Language Model (LLM). It is designed for lead generation and data enrichment scenarios, particularly targeting businesses such as dentists in Texas.

The workflow is logically divided into three main blocks:

- **1.1 Input Initialization and Data Acquisition:**  
  Receives manual trigger input, sets search parameters, and performs a web request through Bright Data to scrape Google Maps search results.

- **1.2 Google Maps Data Extraction and Structuring:**  
  Uses Google Gemini LLM to parse raw Google Maps data into structured JSON, then processes and stores this structured data, including triggering webhook notifications and updating a Google Sheet.

- **1.3 Yelp Data Enrichment:**  
  For each lead obtained from Google Maps, performs a Yelp search via MCP Client, extracts Yelp URLs and descriptions, scrapes Yelp content, and saves results to disk for further enrichment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Data Acquisition

- **Overview:**  
  This block starts the workflow manually, sets search parameters such as the Google Maps search URL, query, pagination, and Bright Data zone, then makes an authenticated HTTP POST request to Bright Data’s API to scrape Google Maps search results.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set input fields  
  - Perform Bright Data Web Request

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - Type: Manual Trigger  
     - Role: Initiates the workflow manually for testing or ad hoc runs  
     - Configuration: No parameters set  
     - Inputs: None  
     - Outputs: To "Set input fields"  
     - Edge cases: Trigger not activated, no inputs provided

  2. **Set input fields**  
     - Type: Set  
     - Role: Defines static input parameters for the scraping such as:  
       - `url`: Base Google Maps search URL  
       - `webhook_notification_url`: URL to send notifications to  
       - `search`: Search query string (e.g., "dentists+in+texas/?q=dentists+in+texas")  
       - `zone`: Bright Data proxy zone  
       - `start`: Pagination start index  
       - `num`: Number of results to fetch  
     - Inputs: From manual trigger  
     - Outputs: To "Perform Bright Data Web Request"  
     - Edge cases: Incorrect or missing parameter values could cause scraping failure

  3. **Perform Bright Data Web Request**  
     - Type: HTTP Request  
     - Role: Sends POST request to Bright Data API with authentication to scrape Google Maps results  
     - Configuration:  
       - URL: https://api.brightdata.com/request  
       - Method: POST  
       - Body parameters include `zone`, `url` combined with `search`, and response format as `raw`  
       - Auth: Header-based authentication with Bright Data credentials  
     - Inputs: From "Set input fields"  
     - Outputs: Raw scraped data to "Google Maps Data Extractor"  
     - Edge cases: Authentication failures, network timeouts, API rate limits, malformed inputs

---

#### 2.2 Google Maps Data Extraction and Structuring

- **Overview:**  
  This block processes raw Google Maps HTML with Google Gemini LLM to extract structured business data as JSON. It then writes this data to disk, updates Google Sheets, and triggers webhook notifications.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Google Maps Data Extractor  
  - Code (JSON output extraction)  
  - Create a binary data for Structured Data Extract  
  - Write the structured content to disk  
  - Initiate a Webhook Notification for Structured Data  
  - Update Google Sheets for Structured Data  
  - Loop Over Items

- **Node Details:**

  1. **Google Gemini Chat Model**  
     - Type: AI Language Model (Google Gemini)  
     - Role: Processes raw HTML or text input to generate structured data extraction prompt completion  
     - Config: Uses "models/gemini-2.0-flash-exp" model  
     - Inputs: Raw data from Bright Data HTTP request  
     - Outputs: Prompt response to "Structured Output Parser"  
     - Credentials: Google PaLM API  
     - Edge cases: API quota limits, malformed data input, latency

  2. **Structured Output Parser**  
     - Type: Structured Output Parser (LLM Output)  
     - Role: Parses LLM’s JSON string output into native JSON objects with defined schema example  
     - Inputs: From Google Gemini Chat Model  
     - Outputs: Parsed JSON to "Google Maps Data Extractor"  
     - Edge cases: Parsing errors if output is malformed or unexpected

  3. **Google Maps Data Extractor**  
     - Type: Chain LLM Node (n8n Langchain)  
     - Role: Coordinates prompt and parser to extract structured business leads from Google Maps data  
     - Config: Includes prompt to extract JSON, with retry enabled  
     - Inputs: Parsed data from Structured Output Parser  
     - Outputs: Processed JSON data to downstream nodes  
     - Edge cases: Retry on failure mitigates transient issues

  4. **Code**  
     - Type: Code (JavaScript)  
     - Role: Extracts the `output` property from the JSON result for further processing  
     - Inputs: From Google Maps Data Extractor  
     - Outputs: Cleaned JSON output to "Update Google Sheets for Structured Data" and "Loop Over Items"  
     - Edge cases: Unexpected data structure could cause runtime errors

  5. **Create a binary data for Structured Data Extract**  
     - Type: Function  
     - Role: Converts the JSON result into base64-encoded binary data for file writing  
     - Inputs: From "Google Maps Data Extractor"  
     - Outputs: Binary data to "Write the structured content to disk"  
     - Edge cases: Buffer encoding issues on large payloads

  6. **Write the structured content to disk**  
     - Type: Read/Write File  
     - Role: Saves the structured JSON data to a file on disk at path `d:\GoogleMaps_Response.json`  
     - Inputs: Binary data  
     - Outputs: None  
     - Edge cases: Filesystem permission errors, disk space issues

  7. **Initiate a Webhook Notification for Structured Data**  
     - Type: HTTP Request  
     - Role: Sends POST request to the configured webhook URL containing the structured JSON data as body parameter  
     - Inputs: From "Google Maps Data Extractor" (JSON output serialized)  
     - Outputs: None  
     - Edge cases: Webhook URL unreachable, invalid URL, request timeout

  8. **Update Google Sheets for Structured Data**  
     - Type: Google Sheets  
     - Role: Appends or updates rows in a Google Sheet with the structured business lead data  
     - Inputs: From "Code" node (clean JSON output)  
     - Configuration: Uses Google Sheets OAuth2 credentials, auto-maps columns such as name, rating, address, category, appointment link, etc., matching by `name`  
     - Edge cases: Credential expiration, sheet access rights, API quota exceeded

  9. **Loop Over Items**  
     - Type: Split in Batches  
     - Role: Processes each lead item individually for downstream Yelp enrichment  
     - Inputs: From "Update Google Sheets for Structured Data"  
     - Outputs: To "Wait" node for pacing requests

---

#### 2.3 Yelp Data Enrichment

- **Overview:**  
  For each individual lead from Google Maps, searches Yelp for matching business entries, extracts Yelp URLs and descriptions, scrapes Yelp pages, and saves enriched Yelp data to disk.

- **Nodes Involved:**  
  - Wait  
  - MCP Search Client  
  - Search Data Extractor  
  - Google Gemini Chat Model for Google Search  
  - Structured Output Parser for Google Search  
  - MCP Client for Web Scraping  
  - Create a binary data for Structured Data Extract for Yelp  
  - Write the Yelp content to disk

- **Node Details:**

  1. **Wait**  
     - Type: Wait  
     - Role: Implements pacing/wait time between batch processing to avoid rate limits or overload  
     - Inputs: From "Loop Over Items"  
     - Outputs: To "MCP Search Client"  
     - Edge cases: Unconfigured or zero wait may cause API throttling

  2. **MCP Search Client**  
     - Type: MCP Client (Community Node)  
     - Role: Executes a search engine query to find Yelp page URLs for the business name  
     - Config: Tool name "search_engine", query is `{{ $json.name }} in Yelp`  
     - Credentials: MCP Client API  
     - Inputs: From "Wait"  
     - Outputs: Search results JSON to "Search Data Extractor"  
     - Edge cases: MCP service unavailability, query format issues

  3. **Search Data Extractor**  
     - Type: Chain LLM Node  
     - Role: Extracts Yelp URLs and descriptions from MCP search results using Google Gemini LLM  
     - Config: Prompt instructs to extract Yelp URL and description, output structured as JSON  
     - Inputs: From "MCP Search Client" via LLM node "Google Gemini Chat Model for Google Search" and "Structured Output Parser for Google Search"  
     - Outputs: Yelp URL and description JSON for scraping  
     - Edge cases: LLM parsing errors, incomplete data extraction

  4. **Google Gemini Chat Model for Google Search**  
     - Type: AI Language Model  
     - Role: Runs the LLM prompt for extracting Yelp info from search engine results  
     - Inputs: From "Search Data Extractor"  
     - Outputs: To "Structured Output Parser for Google Search"  
     - Edge cases: API rate limits, malformed inputs

  5. **Structured Output Parser for Google Search**  
     - Type: Structured Output Parser  
     - Role: Parses LLM output into JSON with properties `url` and `desc`  
     - Inputs: From "Google Gemini Chat Model for Google Search"  
     - Outputs: To "Search Data Extractor" node for further processing  
     - Edge cases: Parsing errors if output format inconsistent

  6. **MCP Client for Web Scraping**  
     - Type: MCP Client  
     - Role: Scrapes the Yelp page content as markdown given the extracted Yelp URL  
     - Config: Tool name "scrape_as_markdown", parameter: `url` from extracted Yelp data  
     - Inputs: From "Search Data Extractor"  
     - Outputs: Raw Yelp page content JSON to "Create a binary data for Structured Data Extract for Yelp"  
     - Edge cases: Scraping blocked by CAPTCHA or firewall, URL invalid

  7. **Create a binary data for Structured Data Extract for Yelp**  
     - Type: Function  
     - Role: Converts Yelp JSON result to base64-encoded binary for file writing  
     - Inputs: From "MCP Client for Web Scraping"  
     - Outputs: To "Write the Yelp content to disk"  
     - Edge cases: Encoding issues on large or malformed data

  8. **Write the Yelp content to disk**  
     - Type: Read/Write File  
     - Role: Writes Yelp JSON content to disk with timestamped filename in `d:\` drive  
     - Inputs: Binary data  
     - Outputs: Loops back to "Loop Over Items" to continue processing  
     - Edge cases: File system errors, timestamp collisions

---

### 3. Summary Table

| Node Name                                   | Node Type                             | Functional Role                              | Input Node(s)                   | Output Node(s)                             | Sticky Note                                                                                             |
|---------------------------------------------|-------------------------------------|---------------------------------------------|--------------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’               | Manual Trigger                      | Initiates workflow manually                  | None                           | Set input fields                           |                                                                                                       |
| Set input fields                            | Set                                 | Defines input parameters for scraping        | When clicking ‘Test workflow’  | Perform Bright Data Web Request             | **Note:** Set filtering criteria, Bright Data zone, webhook notification URL                           |
| Perform Bright Data Web Request             | HTTP Request                       | Performs authenticated scraping request      | Set input fields               | Google Maps Data Extractor                  |                                                                                                       |
| Google Gemini Chat Model                    | AI Language Model (Google Gemini)  | Processes raw data for structured extraction | Perform Bright Data Web Request | Structured Output Parser                    | **LLM Usages:** Google Gemini LLM is used for structured data extraction                               |
| Structured Output Parser                    | Output Parser                      | Parses LLM JSON output                        | Google Gemini Chat Model       | Google Maps Data Extractor                  |                                                                                                       |
| Google Maps Data Extractor                  | Chain LLM                          | Coordinates data extraction and parsing      | Structured Output Parser       | Code, Create a binary data..., Initiate Webhook | **Note:** Handles data extraction and transformation                                                  |
| Code                                        | Code                              | Extracts JSON output property                 | Google Maps Data Extractor     | Update Google Sheets for Structured Data, Loop Over Items |                                                                                                       |
| Create a binary data for Structured Data Extract | Function                         | Encodes JSON to base64 binary for file write | Google Maps Data Extractor     | Write the structured content to disk        |                                                                                                       |
| Write the structured content to disk        | Read/Write File                   | Saves structured JSON to disk                  | Create a binary data for Structured Data Extract | None                                         |                                                                                                       |
| Initiate a Webhook Notification for Structured Data | HTTP Request                   | Sends webhook notification with structured data | Google Maps Data Extractor     | None                                         |                                                                                                       |
| Update Google Sheets for Structured Data    | Google Sheets                    | Appends or updates Google Sheet with data     | Code                         | Loop Over Items                             |                                                                                                       |
| Loop Over Items                             | Split In Batches                 | Processes each lead individually              | Update Google Sheets for Structured Data, Write the Yelp content to disk | Wait (first batch), None (second batch)    |                                                                                                       |
| Wait                                        | Wait                             | Adds delay between item processing             | Loop Over Items               | MCP Search Client                          |                                                                                                       |
| MCP Search Client                           | MCP Client (Community Node)      | Searches Yelp for each lead                    | Wait                         | Search Data Extractor                      |                                                                                                       |
| Google Gemini Chat Model for Google Search  | AI Language Model                | Extracts Yelp URL and description from search | Search Data Extractor         | Structured Output Parser for Google Search |                                                                                                       |
| Structured Output Parser for Google Search | Output Parser                   | Parses LLM output for Yelp search data        | Google Gemini Chat Model for Google Search | Search Data Extractor                      |                                                                                                       |
| Search Data Extractor                       | Chain LLM                       | Coordinates Yelp search data extraction        | MCP Search Client             | MCP Client for Web Scraping                | **Data Enrichment with Yelp data extraction**                                                         |
| MCP Client for Web Scraping                 | MCP Client                     | Scrapes Yelp page content                       | Search Data Extractor         | Create a binary data for Structured Data Extract for Yelp |                                                                                                       |
| Create a binary data for Structured Data Extract for Yelp | Function                    | Encodes Yelp JSON to base64 binary             | MCP Client for Web Scraping   | Write the Yelp content to disk              |                                                                                                       |
| Write the Yelp content to disk              | Read/Write File               | Saves Yelp JSON content to disk with timestamp | Create a binary data for Structured Data Extract for Yelp | Loop Over Items                             |                                                                                                       |
| Sticky Note1                                | Sticky Note                   | Notes about Google Maps data extraction        | None                         | None                                         | Deals with Google Maps data extraction using Bright Data and Google Gemini LLM                         |
| Sticky Note2                                | Sticky Note                   | Disclaimer about node usage                     | None                         | None                                         | Template only available on n8n self-hosted due to MCP Client community node usage                      |
| Sticky Note3                                | Sticky Note                   | Data enrichment with Yelp data extraction      | None                         | None                                         |                                                                                                       |
| Sticky Note4                                | Sticky Note                   | LLM usage explanation                           | None                         | None                                         | Google Gemini LLM is used for structured data extraction                                              |
| Sticky Note5                                | Sticky Note                   | Branding logo display                            | None                         | None                                         | ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |
| Sticky Note6                                | Sticky Note                   | Input fields configuration instructions         | None                         | None                                         | Step 1: Set input fields with URL, webhook, search, zone, start, num                                  |
| Sticky Note7                                | Sticky Note                   | Credential setup instructions                    | None                         | None                                         | Step 2: Set credentials for Bright Data and Google Gemini                                            |
| Sticky Note8                                | Sticky Note                   | Output nodes configuration instructions          | None                         | None                                         | Step 3: Configure output nodes for disk saving and Google Sheets                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed

2. **Create Set Node for Input Fields**  
   - Name: `Set input fields`  
   - Assign the following fields (type: string):  
     - `url`: `https://www.google.com/maps/search/`  
     - `webhook_notification_url`: `https://webhook.site/c9118da2-1c54-460f-a83a-e5131b7098db`  
     - `search`: `dentists+in+texas/?q=dentists+in+texas`  
     - `zone`: `serp_api1`  
     - `start`: `0`  
     - `num`: `20`  
   - Connect manual trigger output to this node

3. **Create HTTP Request Node for Bright Data**  
   - Name: `Perform Bright Data Web Request`  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Body: JSON with parameters:  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}/{{ $json.search }}`  
     - `format`: `"raw"`  
   - Headers: `Content-Type: application/json`  
   - Authentication: HTTP Header Auth using Bright Data credentials (set in credentials manager)  
   - Connect output of `Set input fields` to this node

4. **Add Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Select model: `models/gemini-2.0-flash-exp`  
   - Credentials: Google PaLM API (configured in credentials manager)  
   - Connect `Perform Bright Data Web Request` output to this node’s input

5. **Add Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Paste example JSON schema for Google Maps business data (see schema in workflow)  
   - Connect output of `Google Gemini Chat Model` to this node

6. **Add Chain LLM Node for Google Maps Data Extraction**  
   - Name: `Google Maps Data Extractor`  
   - Prompt: `"Extract Google Maps content \n\n{{ $json.data }}\n\nMake sure to return the data in JSON"`  
   - Enable output parser  
   - Enable retry on failure  
   - Connect output of `Structured Output Parser` to this node

7. **Add Code Node**  
   - Name: `Code`  
   - JavaScript code: `return $input.first().json.output`  
   - Connect output of `Google Maps Data Extractor` to this node

8. **Add Google Sheets Node**  
   - Name: `Update Google Sheets for Structured Data`  
   - Operation: Append or Update Rows  
   - Document ID and Sheet Name: Set your Google Sheet ID and sheet tab (e.g., gid=0)  
   - Map columns: name, rating, address, reviews, category, details_url, related_terms, appointment_link using expressions from the `Code` node output (e.g., `={{ $json.output[0].name }}`)  
   - Credentials: Google Sheets OAuth2  
   - Connect output of `Code` node to this node

9. **Add Split In Batches Node**  
   - Name: `Loop Over Items`  
   - Connect output of `Update Google Sheets for Structured Data` to this node

10. **Add Wait Node**  
    - Name: `Wait`  
    - Default wait time (optional)  
    - Connect output of `Loop Over Items` to this node

11. **Add MCP Search Client Node**  
    - Name: `MCP Search Client`  
    - Tool Name: `search_engine`  
    - Operation: `executeTool`  
    - Tool Parameters: `{"query": "{{ $json.name }} in Yelp"}`  
    - Credentials: MCP Client API  
    - Connect output of `Wait` node to this node

12. **Add Chain LLM Node for Yelp Search Data Extraction**  
    - Name: `Search Data Extractor`  
    - Prompt: `"Extract the Yelp URL and Desc\n\n{{ $json.result.content[0].text }}\n\nMake sure to return the data in JSON"`  
    - Enable output parser with JSON schema:  
      ```json
      {
        "type": "object",
        "properties": {
          "url": {"type": "string"},
          "desc": {"type": "string"}
        }
      }
      ```  
    - Connect output of `MCP Search Client` to this node

13. **Add Google Gemini Chat Model for Google Search**  
    - Name: `Google Gemini Chat Model for Google Search`  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google PaLM API  
    - Connect output of `Search Data Extractor` to this node

14. **Add Structured Output Parser for Google Search**  
    - Name: `Structured Output Parser for Google Search`  
    - Use manual schema from above  
    - Connect output of `Google Gemini Chat Model for Google Search` to this node

15. **Add MCP Client for Web Scraping Node**  
    - Name: `MCP Client for Web Scraping`  
    - Tool Name: `scrape_as_markdown`  
    - Operation: `executeTool`  
    - Tool Parameters: `{"url": "{{ $json.output.url }}"}`  
    - Credentials: MCP Client API  
    - Connect output of `Structured Output Parser for Google Search` to this node

16. **Add Function Node for Binary Data Creation (Yelp)**  
    - Name: `Create a binary data for Structured Data Extract for Yelp`  
    - JavaScript code:  
      ```js
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect output of `MCP Client for Web Scraping` to this node

17. **Add Read/Write File Node for Yelp Content**  
    - Name: `Write the Yelp content to disk`  
    - Operation: Write  
    - File Name: `d:\Yelp_Response_{{ new Date().toISOString().replace(/[:.]/g, '-') }}.json`  
    - Connect output of `Create a binary data for Structured Data Extract for Yelp` to this node

18. **Connect output of `Write the Yelp content to disk` back to `Loop Over Items`**  
    - This creates a cycle to process next batch item

19. **Add Function Node for Binary Data Creation (Google Maps)**  
    - Name: `Create a binary data for Structured Data Extract`  
    - JavaScript code same as Yelp version above  
    - Connect output of `Google Maps Data Extractor` to this node

20. **Add Read/Write File Node for Google Maps Content**  
    - Name: `Write the structured content to disk`  
    - Operation: Write  
    - File Name: `d:\GoogleMaps_Response.json`  
    - Connect output of `Create a binary data for Structured Data Extract` to this node

21. **Add HTTP Request Node for Webhook Notification**  
    - Name: `Initiate a Webhook Notification for Structured Data`  
    - Method: POST  
    - URL: `={{ $('Set input fields').item.json.webhook_notification_url }}`  
    - Body parameter: `{ "response": "={{ $json.output.toJsonString() }}" }`  
    - Connect output of `Google Maps Data Extractor` to this node

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it uses the community MCP Client node.                      | Sticky Note2                                                                                       |
| Google Gemini LLM is utilized for structured data extraction tasks in this workflow.                              | Sticky Note4                                                                                       |
| Ensure Bright Data and Google Gemini credentials are properly configured before running.                          | Sticky Note7                                                                                       |
| Input parameters must be properly set in the "Set input fields" node before execution.                            | Sticky Note6                                                                                       |
| Output data is saved both locally (disk) and in Google Sheets for easy access and processing.                     | Sticky Note8                                                                                       |
| Bright Data logo and branding image available at: ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Sticky Note5                                                                                       |
| The workflow relies on multiple API services (Bright Data, Google Gemini, MCP Client, Google Sheets) requiring valid credentials and API quotas. | General workflow prerequisite                                                                     |

---

**Disclaimer:**  
This document is generated exclusively from an n8n automated workflow. All data processed is legal and public. The workflow complies with content policies and contains no illegal or offensive materials.