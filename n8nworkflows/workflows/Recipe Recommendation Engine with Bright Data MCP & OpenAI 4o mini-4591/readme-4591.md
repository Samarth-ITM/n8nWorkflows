Recipe Recommendation Engine with Bright Data MCP & OpenAI 4o mini

https://n8nworkflows.xyz/workflows/recipe-recommendation-engine-with-bright-data-mcp---openai-4o-mini-4591


# Recipe Recommendation Engine with Bright Data MCP & OpenAI 4o mini

---

### 1. Workflow Overview

This workflow is a **Recipe Recommendation Engine** designed to scrape, extract, and structure recipe data from web sources using **Bright Data MCP** (a web scraping proxy and data extraction service) combined with **OpenAI GPT-4o mini** for natural language processing and structured data parsing.

The key use cases include:

- Extracting paginated recipe listings from a given URL (e.g., Tesco realfood search pages)
- Scraping individual recipe pages to extract detailed recipe information
- Structuring the extracted data into JSON format for further use or analysis
- Storing data locally and optionally sending notifications via webhooks

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization and Setup**  
  Trigger and initial URL configuration, listing available scraping tools.

- **1.2 Recipe Listing Scraping and Pagination Extraction**  
  Scraping the initial recipe search page, extracting pagination links with LLM assistance.

- **1.3 Iterative Scraping of Individual Recipe Pages**  
  Looping through each paginated page link, scraping recipe details, and structuring data.

- **1.4 Data Output and Notifications**  
  Formatting structured data, saving to disk, and sending webhook notifications.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Initialization and Setup

**Overview:**  
This block triggers the workflow manually and sets initial parameters like the recipe search URL and webhook URL. It also lists available Bright Data MCP tools for reference.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- List all tools for Bright Data (MCP Client)  
- Set the Recipe Extract URL (Set)  
- Sticky Note (for labeling and instructions)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start workflow execution manually.  
  - Configuration: No parameters set; trigger only.  
  - Connections: Outputs to "List all tools for Bright Data".  
  - Edge cases: None (manual).  

- **List all tools for Bright Data**  
  - Type: MCP Client (Bright Data)  
  - Role: Retrieves a list of all available scraping tools from Bright Data MCP service.  
  - Configuration: Uses configured MCP API credentials ("MCP Client (STDIO) account").  
  - Connections: Outputs to "Set the Recipe Extract URL".  
  - Edge cases: API authentication failure, network timeout.

- **Set the Recipe Extract URL**  
  - Type: Set  
  - Role: Defines key input parameters:  
    - `url`: The initial recipe search page (e.g., Tesco tomato pasta search)  
    - `webhook_url`: URL for webhook notifications  
    - `base_url`: Base URL for relative link construction  
  - Configuration: Hardcoded string values for these parameters.  
  - Connections: Outputs to "Bright Data MCP Client For Recipe Extract".  
  - Edge cases: Wrong URLs or unreachable sites.

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Labels this block as "Recipe Data Scraper" for clarity.

---

#### 1.2 Recipe Listing Scraping and Pagination Extraction

**Overview:**  
This block scrapes the initial search page to extract paginated data links using Bright Data MCP scraping and processes the raw content with OpenAI GPT-4o mini to identify pagination info.

**Nodes Involved:**  
- Bright Data MCP Client For Recipe Extract (MCP Client)  
- Paginated Data Extract (Chain LLM)  
- OpenAI Chat Model for Paginated Data Extract (OpenAI LLM)  
- Structured Output Parser for Paginated Data (Output Parser Structured)  
- Code to output the array of paginated info (Code)  
- Loop Over Items (Split In Batches)  
- Sticky Notes (Structured Data Extract and LLM usages)

**Node Details:**

- **Bright Data MCP Client For Recipe Extract**  
  - Type: MCP Client  
  - Role: Scrapes the specified `url` from the Set node, returning raw page content in Markdown.  
  - Configuration: Tool set to "scrape_as_html", with parameter `url` from input JSON.  
  - Connections: Outputs raw content to "Paginated Data Extract".  
  - Edge cases: Scraping failures, site blocking, or rate-limits.

- **Paginated Data Extract**  
  - Type: Chain LLM (LangChain)  
  - Role: Uses OpenAI GPT-4o mini to extract pagination data (page numbers and links) from the raw scraped text.  
  - Configuration:  
    - Prompt: "Extract the paginated data from the provided content..."  
    - Input content via expression: `{{ $json.result.content[0].text }}`  
    - Base URL appended to links for completeness.  
    - Retry enabled on failure.  
  - Connections: Outputs parsed pagination array to "Code to output the array of paginated info".  
  - Edge cases: Model output mismatch, parsing errors.

- **OpenAI Chat Model for Paginated Data Extract**  
  - Type: OpenAI LLM Chat Model  
  - Role: Provides the language model backend for "Paginated Data Extract".  
  - Configuration: Model set to "gpt-4o-mini".  
  - Connections: Linked internally by LangChain node.  
  - Edge cases: OpenAI API errors, rate limits.

- **Structured Output Parser for Paginated Data**  
  - Type: Output Parser Structured (LangChain)  
  - Role: Parses the LLM output into a structured JSON array with keys: `page_number` and `link`.  
  - Configuration: JSON schema example provided.  
  - Connections: Linked internally by LangChain node.  
  - Edge cases: Parsing mismatches, schema validation errors.

- **Code to output the array of paginated info**  
  - Type: Code  
  - Role: Extracts the LLM output array from the first item’s JSON property `output`.  
  - Configuration: JavaScript code: `return $input.first().json.output`  
  - Connections: Outputs to "Loop Over Items" node.  
  - Edge cases: Empty or malformed output.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Iterates over each pagination link for further processing.  
  - Configuration: Default batch size (1).  
  - Connections: Outputs to "Bright Data MCP Client For Recipe Extract Within The Loop" and "Wait" node (second output).  
  - Edge cases: Large number of pages causing long execution.

- **Sticky Notes**  
  - "Structured Data Extract" explains use of MCP and LLM for paginated data extraction and looping.  
  - "LLM Usages" notes the usage of OpenAI GPT-4o mini for structured data extraction.

---

#### 1.3 Iterative Scraping of Individual Recipe Pages

**Overview:**  
This block loops through each extracted pagination link, scrapes full recipe page content, processes it with OpenAI GPT-4o mini to extract structured recipe details in JSON format.

**Nodes Involved:**  
- Bright Data MCP Client For Recipe Extract Within The Loop (MCP Client)  
- Wait (Wait)  
- Structured Recipe Data Extract (Chain LLM)  
- OpenAI Chat Model for Structured Data Extract (OpenAI LLM)  
- Structured Output Parser for Recipe Data Extract (Output Parser Structured)  
- Sticky Notes (disclaimer, LLM usages, notes)

**Node Details:**

- **Bright Data MCP Client For Recipe Extract Within The Loop**  
  - Type: MCP Client  
  - Role: Scrapes each paginated recipe page URL individually to get detailed content.  
  - Configuration: Tool "scrape_as_html" with URL from the current loop item (`{{ $json.link }}`).  
  - Connections: Outputs to "Structured Recipe Data Extract".  
  - Edge cases: Page not found, captchas, rate limits.

- **Wait**  
  - Type: Wait  
  - Role: Delays execution by 10 seconds between requests to avoid rate limits or bans.  
  - Configuration: 10 seconds delay.  
  - Connections: Loops back to "Bright Data MCP Client For Recipe Extract Within The Loop".  
  - Edge cases: Workflow timeout if too many pages.

- **Structured Recipe Data Extract**  
  - Type: Chain LLM (LangChain)  
  - Role: Uses OpenAI GPT-4o mini to parse raw scraped page content into detailed structured recipe data, including title, URL, category, rating, serves, dietary options, and healthiness.  
  - Configuration: Prompt to "Extract search data. Here's the content: Output the data in JSON".  
  - Retry enabled for robustness.  
  - Connections: Outputs structured data to "Create a binary data" and "Webhook Notification for Data Extract Within the Loop".  
  - Edge cases: Model parsing inaccuracies, incomplete data.

- **OpenAI Chat Model for Structured Data Extract**  
  - Type: OpenAI LLM Chat Model  
  - Role: Language model backend for "Structured Recipe Data Extract".  
  - Configuration: Model "gpt-4o-mini".  
  - Connections: Used internally by Chain LLM.  
  - Edge cases: API limits.

- **Structured Output Parser for Recipe Data Extract**  
  - Type: Output Parser Structured (LangChain)  
  - Role: Parses LLM output to a strict JSON schema detailing the recipe data structure (title, rating, dietary options, etc.)  
  - Configuration: Manual JSON schema with required fields.  
  - Connections: Used internally by Chain LLM.  
  - Edge cases: Schema validation errors.

- **Sticky Notes**  
  - Disclaimer: Template only for self-hosted n8n due to community MCP node usage.  
  - Notes: Reminder to set input URLs and webhook notification URLs properly.  
  - Branding logo note for Bright Data.

---

#### 1.4 Data Output and Notifications

**Overview:**  
This block handles the storage of structured recipe data to disk and sends webhook notifications with the extracted data, enabling downstream consumption or alerts.

**Nodes Involved:**  
- Create a binary data (Function)  
- Write the structured content to disk (Read/Write File)  
- Webhook Notification for Data Extract Within the Loop (HTTP Request)

**Node Details:**

- **Create a binary data**  
  - Type: Function  
  - Role: Converts the JSON structured recipe data into base64 encoded binary format for file writing.  
  - Configuration: JavaScript creates a binary property `data` with base64-encoded JSON string.  
  - Connections: Outputs to "Write the structured content to disk".  
  - Edge cases: Encoding errors.

- **Write the structured content to disk**  
  - Type: Read/Write File  
  - Role: Saves the binary-encoded JSON recipe data to a local file with dynamic filename based on page number.  
  - Configuration:  
    - File path: `d:\Recipe-Structured-Data-{{ page_number }}.json`  
    - Operation: Write  
  - Connections: No further outputs.  
  - Edge cases: Disk write permissions, path availability.

- **Webhook Notification for Data Extract Within the Loop**  
  - Type: HTTP Request  
  - Role: Sends a POST request with multipart-form-data containing the recipe JSON to a configured webhook URL for notifications or integrations.  
  - Configuration:  
    - URL from "Set the Recipe Extract URL" node's `webhook_url`  
    - Body parameter `recipe_content` contains JSON stringified structured recipe data  
  - Connections: Outputs back to "Loop Over Items" to continue looping.  
  - Edge cases: Network errors, webhook downtime, payload size limits.

---

### 3. Summary Table

| Node Name                                   | Node Type                          | Functional Role                                        | Input Node(s)                             | Output Node(s)                                         | Sticky Note                                                                                              |
|---------------------------------------------|----------------------------------|-------------------------------------------------------|------------------------------------------|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                | Manual Trigger                   | Entry point trigger for workflow                       | None                                     | List all tools for Bright Data                          |                                                                                                         |
| List all tools for Bright Data               | MCP Client                      | Retrieve available Bright Data MCP tools               | When clicking ‘Test workflow’            | Set the Recipe Extract URL                              |                                                                                                         |
| Set the Recipe Extract URL                    | Set                             | Define initial URL, webhook URL, and base URL          | List all tools for Bright Data           | Bright Data MCP Client For Recipe Extract               |                                                                                                         |
| Bright Data MCP Client For Recipe Extract    | MCP Client                      | Scrape initial recipe search page                       | Set the Recipe Extract URL                | Paginated Data Extract                                  | Scrape a single webpage URL with advanced options for content extraction and get back results in MarkDown language. |
| Paginated Data Extract                        | Chain LLM                      | Extract pagination data from raw scraped content       | Bright Data MCP Client For Recipe Extract| Code to output the array of paginated info             |                                                                                                         |
| OpenAI Chat Model for Paginated Data Extract| OpenAI LLM Chat Model           | Language model backend for pagination extraction        | Internal to Paginated Data Extract        | Internal to Paginated Data Extract                       |                                                                                                         |
| Structured Output Parser for Paginated Data  | Output Parser Structured        | Parse LLM pagination output into structured JSON       | Internal to Paginated Data Extract        | Internal to Paginated Data Extract                       |                                                                                                         |
| Code to output the array of paginated info   | Code                           | Extracts pagination array from LLM output               | Paginated Data Extract                    | Loop Over Items                                        |                                                                                                         |
| Loop Over Items                               | Split In Batches               | Iterate over each paginated page link                   | Code to output the array of paginated info| Bright Data MCP Client For Recipe Extract Within The Loop, Wait | Structured Data Extract; LLM Usages                                                                     |
| Wait                                          | Wait                           | Delay between requests in loop                           | Loop Over Items (second output)            | Bright Data MCP Client For Recipe Extract Within The Loop|                                                                                                         |
| Bright Data MCP Client For Recipe Extract Within The Loop | MCP Client       | Scrape individual recipe page content                   | Loop Over Items                           | Structured Recipe Data Extract                          | Scrape a single webpage URL with advanced options for content extraction and get back results in MarkDown language. |
| Structured Recipe Data Extract                | Chain LLM                      | Extract structured recipe details from raw page content| Bright Data MCP Client For Recipe Extract Within The Loop | Create a binary data, Webhook Notification for Data Extract Within the Loop |                                                                                                         |
| OpenAI Chat Model for Structured Data Extract| OpenAI LLM Chat Model           | Language model backend for structured recipe extraction| Internal to Structured Recipe Data Extract | Internal to Structured Recipe Data Extract             |                                                                                                         |
| Structured Output Parser for Recipe Data Extract | Output Parser Structured     | Parse LLM output into detailed recipe JSON              | Internal to Structured Recipe Data Extract | Internal to Structured Recipe Data Extract             |                                                                                                         |
| Create a binary data                          | Function                      | Convert JSON data to base64 binary for file output     | Structured Recipe Data Extract            | Write the structured content to disk                   |                                                                                                         |
| Write the structured content to disk         | Read/Write File                | Save structured recipe data to local disk               | Create a binary data                      | None                                                   |                                                                                                         |
| Webhook Notification for Data Extract Within the Loop | HTTP Request               | Send structured recipe data to configured webhook       | Structured Recipe Data Extract            | Loop Over Items                                        |                                                                                                         |
| Sticky Note                                   | Sticky Note                   | Label: Recipe Data Scraper                               | None                                     | None                                                   |                                                                                                         |
| Sticky Note1                                  | Sticky Note                   | Label: Structured Data Extract and LLM usage            | None                                     | None                                                   |                                                                                                         |
| Sticky Note2                                  | Sticky Note                   | Disclaimer about self-hosted n8n and community MCP node | None                                     | None                                                   |                                                                                                         |
| Sticky Note3                                  | Sticky Note                   | Note about setting input URLs and webhook URLs           | None                                     | None                                                   |                                                                                                         |
| Sticky Note5                                  | Sticky Note                   | Logo for Bright Data                                     | None                                     | None                                                   | https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png                        |
| Sticky Note6                                  | Sticky Note                   | LLM Usage explanation                                   | None                                     | None                                                   |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node ("When clicking ‘Test workflow’")**  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add MCP Client Node ("List all tools for Bright Data")**  
   - Type: MCP Client (Bright Data)  
   - Credentials: Configure MCP Client with your "MCP Client (STDIO) account".  
   - No parameters needed (default lists all tools).  
   - Connect output of Manual Trigger to this node.

3. **Add Set Node ("Set the Recipe Extract URL")**  
   - Type: Set  
   - Add three string fields:  
     - `url`: e.g., "https://realfood.tesco.com/search.html?search=tomato%20pasta&sortby=Rating"  
     - `webhook_url`: e.g., your webhook URL such as "https://webhook.site/your-id"  
     - `base_url`: "https://realfood.tesco.com/search.html"  
   - Connect output of MCP Client "List all tools for Bright Data" to this node.

4. **Add MCP Client Node ("Bright Data MCP Client For Recipe Extract")**  
   - Type: MCP Client  
   - Credentials: Use same MCP Client credentials.  
   - Parameters:  
     - Tool: "scrape_as_html"  
     - Operation: "executeTool"  
     - Tool Parameters: JSON string: `{ "url": "{{ $json.url }}" }`  
   - Connect output of Set node to this node.

5. **Add Chain LLM Node ("Paginated Data Extract")**  
   - Type: Chain LLM (LangChain)  
   - Parameters:  
     - Prompt Text: `"Extract the paginated data from the provided content. \n\nHere's the content : {{ $json.result.content[0].text }}\n\nAppend the link with the Base URL as {{ $('Set the Recipe Extract URL').item.json.base_url }}"`  
     - Model: Use "OpenAI Chat Model for Paginated Data Extract" (next step)  
     - Retry on fail: Enabled  
     - Has output parser: Enabled  
   - Connect output of MCP Client "Bright Data MCP Client For Recipe Extract" to this node.

6. **Add OpenAI Chat Model Node ("OpenAI Chat Model for Paginated Data Extract")**  
   - Type: OpenAI LLM Chat Model  
   - Credentials: Configure with your OpenAI API account.  
   - Model: "gpt-4o-mini"  
   - Connect internally as AI model for the Chain LLM node.

7. **Add Structured Output Parser Node ("Structured Output Parser for Paginated Data")**  
   - Type: Output Parser Structured  
   - JSON Schema: Provide example schema with `page_number` (int) and `link` (string).  
   - Connect internally as parser for the Chain LLM node.

8. **Add Code Node ("Code to output the array of paginated info")**  
   - Type: Code  
   - JavaScript code: `return $input.first().json.output`  
   - Connect output of Chain LLM "Paginated Data Extract" to this node.

9. **Add Split In Batches Node ("Loop Over Items")**  
   - Type: Split In Batches  
   - Default batch size 1.  
   - Connect output of Code node to this node.

10. **Add MCP Client Node ("Bright Data MCP Client For Recipe Extract Within The Loop")**  
    - Type: MCP Client  
    - Credentials: Same MCP Client credentials.  
    - Parameters:  
      - Tool: "scrape_as_html"  
      - Operation: "executeTool"  
      - Tool Parameters: `{ "url": "{{ $json.link }}" }` (from current item in loop)  
    - Connect first output of "Loop Over Items" to this node.

11. **Add Wait Node ("Wait")**  
    - Type: Wait  
    - Parameters: 10 seconds delay  
    - Connect second output of "Loop Over Items" to this node.  
    - Connect output of Wait back to "Bright Data MCP Client For Recipe Extract Within The Loop" node (loop back).

12. **Add Chain LLM Node ("Structured Recipe Data Extract")**  
    - Type: Chain LLM (LangChain)  
    - Parameters:  
      - Prompt Text: `"Extract search data. Here's the content : \nOutput the data in JSON"`  
      - Model: Use "OpenAI Chat Model for Structured Data Extract" (next step)  
      - Retry on fail: Enabled  
      - Has output parser: Enabled  
    - Connect output of "Bright Data MCP Client For Recipe Extract Within The Loop" to this node.

13. **Add OpenAI Chat Model Node ("OpenAI Chat Model for Structured Data Extract")**  
    - Type: OpenAI LLM Chat Model  
    - Credentials: Use your OpenAI API account.  
    - Model: "gpt-4o-mini"  
    - Connect internally as model to the above Chain LLM node.

14. **Add Structured Output Parser Node ("Structured Output Parser for Recipe Data Extract")**  
    - Type: Output Parser Structured  
    - JSON Schema: Provide detailed schema for recipe data objects including title, url, category, rating, serves, dietaryOptions, isHealthy, etc.  
    - Connect internally as parser for Chain LLM node.

15. **Add Function Node ("Create a binary data")**  
    - Type: Function  
    - JavaScript code:  
      ```js
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect output of "Structured Recipe Data Extract" to this node.

16. **Add Read/Write File Node ("Write the structured content to disk")**  
    - Type: Read/Write File  
    - Parameters:  
      - Operation: Write  
      - File Name: `d:\Recipe-Structured-Data-{{ $('Loop Over Items').item.json.page_number }}.json`  
    - Connect output of Function node to this node.

17. **Add HTTP Request Node ("Webhook Notification for Data Extract Within the Loop")**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `{{ $('Set the Recipe Extract URL').item.json.webhook_url }}`  
      - Method: POST  
      - Content Type: multipart-form-data  
      - Body Parameter: Name: `recipe_content`, Value: `{{ $json.output.toJsonString() }}`  
    - Connect output of "Structured Recipe Data Extract" to this node.  
    - Connect output of this node to "Loop Over Items" node to continue loop.

18. **Add Sticky Notes** as needed for labeling blocks and disclaimers.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it uses the community MCP Client node.                    | See Sticky Note2 in workflow                                                                            |
| Deals with Recipe data extraction by utilizing Bright Data MCP and OpenAI GPT-4o mini LLM.                      | Sticky Note3                                                                                           |
| OpenAI GPT-4o mini model is used for structured data extraction tasks.                                          | Sticky Note6                                                                                           |
| Workflow includes a 10-second wait between paginated requests to avoid rate-limiting and IP bans.                | Wait node configuration                                                                                |
| Bright Data Logo embedded in Sticky Note5 for branding purposes.                                                | https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png                        |
| Please ensure to update the webhook URL and input search URL as per your requirements before running workflow.  | Sticky Note3                                                                                           |

---

**Disclaimer:**  
The provided workflow is an automated recipe data extraction pipeline built with n8n, strictly complying with content policies. It processes only legal and publicly available data.

---