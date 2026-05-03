DNB Company Search & Extract with Bright Data and OpenAI 4o mini

https://n8nworkflows.xyz/workflows/dnb-company-search---extract-with-bright-data-and-openai-4o-mini-4821


# DNB Company Search & Extract with Bright Data and OpenAI 4o mini

---

### 1. Workflow Overview

This workflow automates the extraction of structured company data from DNB (Dun & Bradstreet) using Bright Data’s MCP (Massive Cloud Proxy) Client tools for web search and scraping, combined with OpenAI GPT-4o mini LLMs for advanced natural language processing and data structuring.

**Target Use Cases:**  
- Automated business intelligence gathering on companies listed in DNB databases.  
- Extraction of company profiles, URLs, and detailed structured data in JSON format.  
- Notification and storage of extracted data for downstream processing or alerting.

**Logical Blocks:**  
- **1.1 Input Initialization and Trigger:** Manual trigger and input field setup for search query and webhook URL.  
- **1.2 Bright Data MCP Tool Utilization:** Listing available tools, executing a search engine query, and scraping webpage content as Markdown.  
- **1.3 URL Extraction via LLM:** Using OpenAI GPT-4o mini to extract URLs from search results with a structured output parser.  
- **1.4 Detailed Structured Data Extraction via LLM:** Using OpenAI GPT-4o mini to extract comprehensive company profiles in structured JSON format from scraped content.  
- **1.5 Data Handling and Output:** Converting JSON to binary, saving the output file locally, and sending a webhook notification with the extracted data.  
- **1.6 Supporting Elements:** Sticky notes for documentation, disclaimers, and configuration hints.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Initialization and Trigger

**Overview:**  
This block starts the workflow manually and sets input parameters required for the search and notification webhook.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Sticky Note (Documentation)  
- Set input fields

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual trigger  
  - *Role:* Entry point to start the workflow manually during testing or execution.  
  - *Config:* No parameters required.  
  - *Connections:* Outputs to “List all tools for Bright Data”.  
  - *Failures:* None expected.  
  - *Version:* n8n v1.x compatible.

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides contextual instructions about the workflow, including reminder to update search query and webhook URL, with a link to webhook.site for testing.  
  - *Content Highlights:*  
    - Deals with DNB data extract using Bright Data MCP Search and Markdown scraper.  
    - Reminder for user to update query and webhook URL before production.  
  - *No connections or execution impact.*

- **Set input fields**  
  - *Type:* Set node  
  - *Role:* Defines key input variables:  
    - `search` (string): Query to send to the search engine (default: "dnb starbucks url").  
    - `webhook_notification_url` (string): URL to send notifications (default points to webhook.site test URL).  
  - *Connections:* Input from “List all tools for Bright Data”, outputs to “MCP Client for Search Engine”.  
  - *Edge Cases:* Ensure that query string and webhook URL are valid; malformed URLs or empty queries can break downstream calls.

---

#### 1.2 Bright Data MCP Tool Utilization

**Overview:**  
This block interfaces with Bright Data MCP Client to list tools, perform a search engine query, and scrape the resulting URL's content in Markdown format.

**Nodes Involved:**  
- List all tools for Bright Data  
- MCP Client for Search Engine  
- DNB URL Data Extract Using LLM (connected downstream but logically part of LLM processing)  
- Bright Data MCP Client For DNB

**Node Details:**  

- **List all tools for Bright Data**  
  - *Type:* MCP Client (community node)  
  - *Role:* Lists all available Bright Data MCP tools (for info/debug). No parameters set.  
  - *Credentials:* MCP Client (STDIO) account configured.  
  - *Connections:* Outputs to “Set input fields”.  
  - *Failures:* MCP API auth failure, network issues.

- **MCP Client for Search Engine**  
  - *Type:* MCP Client  
  - *Role:* Executes a search query on Google via Bright Data’s "search_engine" tool.  
  - *Parameters:*  
    - `query`: dynamic expression from `search` input field.  
    - `engine`: fixed to "google".  
  - *Credentials:* MCP Client (STDIO) account.  
  - *Input:* From “Set input fields”.  
  - *Output:* Search results with URLs to next node.  
  - *Failures:* Query syntax errors, API limits, malformed input.

- **Bright Data MCP Client For DNB**  
  - *Type:* MCP Client  
  - *Role:* Scrapes a single webpage URL with advanced content extraction returning Markdown.  
  - *Parameters:*  
    - `url`: extracted dynamically from previous node output (`output.url`).  
  - *Credentials:* MCP Client (STDIO) account.  
  - *Input:* From “DNB URL Data Extract Using LLM”.  
  - *Outputs:* Markdown content for LLM processing.  
  - *Notes:* Has a sticky note describing this scraper usage.  
  - *Failures:* Invalid URLs, scraping timeouts, content structure changes.

---

#### 1.3 URL Extraction via LLM

**Overview:**  
This block uses OpenAI GPT-4o mini to extract URLs from the search engine results text, parsing output in a structured JSON format.

**Nodes Involved:**  
- DNB URL Data Extract Using LLM  
- OpenAI Chat Model for URL Data Extract  
- Structured Output Parser for URL

**Node Details:**  

- **DNB URL Data Extract Using LLM**  
  - *Type:* Langchain chainLlm node  
  - *Role:* Defines prompt to extract URLs for DNB from the search engine content.  
  - *Prompt:* `"Extract the URLs for DNB  {{ $json.result.content[0].text }}"`.  
  - *Has output parser:* True, uses a structured output parser node.  
  - *Retry on fail:* Enabled.  
  - *Input:* From “MCP Client for Search Engine”.  
  - *Output:* Parsed URLs in JSON.  
  - *Failures:* LLM API errors, prompt failures, parsing errors.

- **OpenAI Chat Model for URL Data Extract**  
  - *Type:* Langchain LM Chat OpenAI node  
  - *Role:* Uses GPT-4o mini model for the above LLM node.  
  - *Credentials:* OpenAI API.  
  - *Input:* Connected as language model to “DNB URL Data Extract Using LLM”.  
  - *Failures:* API quota, network, or auth errors.

- **Structured Output Parser for URL**  
  - *Type:* Langchain output parser structured  
  - *Role:* Enforces JSON schema for a URL field to parse LLM output correctly.  
  - *Input:* Connected to “DNB URL Data Extract Using LLM” output parser.  
  - *Failures:* Output format mismatch or invalid JSON.

---

#### 1.4 Detailed Structured Data Extraction via LLM

**Overview:**  
This block uses OpenAI GPT-4o mini to extract detailed company profile information from the scraped Markdown content, outputting a highly structured JSON per a defined schema.

**Nodes Involved:**  
- DNB Structured Data Extract Using LLM  
- OpenAI Chat Model for DNB Structured Data Extract  
- Structured Output Parser for Structured Extract

**Node Details:**  

- **DNB Structured Data Extract Using LLM**  
  - *Type:* Langchain chainLlm node  
  - *Role:* Prompt extracts company profile from scraped Markdown content with instruction to output highly structured JSON.  
  - *Prompt:* `"Extract the Company Profile from {{ $json.result.content[0].text }} Output in a highly structured JSON format."`  
  - *Has output parser:* True, linked to structured JSON schema parser.  
  - *Retry on fail:* Enabled.  
  - *Input:* From “Bright Data MCP Client For DNB”.  
  - *Outputs:* Parsed structured company profile JSON.  
  - *Failures:* LLM errors, parsing failures, malformed schema compliance.

- **OpenAI Chat Model for DNB Structured Data Extract**  
  - *Type:* Langchain LM Chat OpenAI node  
  - *Role:* GPT-4o mini model assigned to the above node.  
  - *Credentials:* OpenAI API.  
  - *Input:* Connected as language model to “DNB Structured Data Extract Using LLM”.  
  - *Failures:* API limits, network issues.

- **Structured Output Parser for Structured Extract**  
  - *Type:* Langchain output parser structured  
  - *Role:* Custom manual JSON schema enforcing detailed company profile structure, including companyName, overview, contacts, financialData, and FAQ.  
  - *Input:* Connected as output parser to “DNB Structured Data Extract Using LLM”.  
  - *Failures:* JSON schema validation errors if LLM output deviates.

---

#### 1.5 Data Handling and Output

**Overview:**  
This block converts the structured JSON into a base64-encoded binary file, writes it to disk, and sends an HTTP webhook notification with the structured data.

**Nodes Involved:**  
- Create a binary data for Structured Data Extract  
- Write the structured content to disk  
- Initiate a Webhook Notification for Structured Data

**Node Details:**  

- **Create a binary data for Structured Data Extract**  
  - *Type:* Function node  
  - *Role:* Converts the JSON output of structured data into base64-encoded binary under `items[0].binary.data.data`.  
  - *Code:* Uses Node.js Buffer to encode JSON string.  
  - *Input:* From “DNB Structured Data Extract Using LLM”.  
  - *Output:* Binary data for file writing.  
  - *Failures:* Encoding errors if JSON is malformed.

- **Write the structured content to disk**  
  - *Type:* ReadWriteFile node  
  - *Role:* Writes the binary JSON file to local path `d:\DNB_Info.json`.  
  - *Parameters:* Operation "write", filename fixed.  
  - *Input:* From “Create a binary data for Structured Data Extract”.  
  - *Failures:* File system permission errors, invalid path.

- **Initiate a Webhook Notification for Structured Data**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a POST request to the configured webhook notification URL with the JSON structured company info in body parameter `dnb_company_info`.  
  - *Parameters:* URL dynamically read from the `webhook_notification_url` input field.  
  - *Input:* From “DNB Structured Data Extract Using LLM”.  
  - *Failures:* Network errors, invalid URL, HTTP failures.

---

#### 1.6 Supporting Elements (Sticky Notes & Metadata)

**Overview:**  
Non-executable nodes providing documentation, branding, and disclaimers.

**Nodes Involved:**  
- Sticky Note2 (disclaimer about community MCP node usage)  
- Sticky Note5 (Bright Data logo)  
- Sticky Note6 (LLM usage note)

**Node Details:**  

- **Sticky Note2**  
  - Notes that the workflow is only supported on n8n self-hosted instances due to community MCP node usage.

- **Sticky Note5**  
  - Displays Bright Data company logo for branding.

- **Sticky Note6**  
  - Notes that OpenAI 4o mini LLM is used for structured data extraction.

---

### 3. Summary Table

| Node Name                             | Node Type                         | Functional Role                                    | Input Node(s)                | Output Node(s)                           | Sticky Note                                                                                                                                                   |
|-------------------------------------|----------------------------------|---------------------------------------------------|------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’        | Manual Trigger                   | Manual start trigger                              |                              | List all tools for Bright Data          | Deals with the DNB data extract using Bright Data MCP Search and Markdown Web scraper. Update search query and webhook URL. Test using https://webhook.site/ |
| Sticky Note                         | Sticky Note                      | Workflow documentation                            |                              |                                         | Deals with the DNB data extract using Bright Data MCP Search and Markdown Web scraper. Update search query and webhook URL. Test using https://webhook.site/ |
| Set input fields                   | Set                             | Define input parameters: search query, webhook URL | List all tools for Bright Data | MCP Client for Search Engine             |                                                                                                                                                               |
| List all tools for Bright Data     | MCP Client                      | List available Bright Data MCP tools             | When clicking ‘Test workflow’| Set input fields                        |                                                                                                                                                               |
| MCP Client for Search Engine       | MCP Client                      | Perform Google search via Bright Data             | Set input fields              | DNB URL Data Extract Using LLM          |                                                                                                                                                               |
| DNB URL Data Extract Using LLM     | Langchain chainLlm              | Extract URLs from search results using LLM        | MCP Client for Search Engine  | Bright Data MCP Client For DNB           |                                                                                                                                                               |
| Bright Data MCP Client For DNB     | MCP Client                      | Scrape webpage URL content as Markdown             | DNB URL Data Extract Using LLM | DNB Structured Data Extract Using LLM   | Scrape a single webpage URL with advanced options for content extraction and get back results in Markdown language.                                          |
| DNB Structured Data Extract Using LLM | Langchain chainLlm           | Extract structured company profile using LLM      | Bright Data MCP Client For DNB | Create a binary data for Structured Data Extract, Initiate a Webhook Notification for Structured Data |                                                                                                                                                               |
| Create a binary data for Structured Data Extract | Function                 | Encode JSON structured data into base64 binary    | DNB Structured Data Extract Using LLM | Write the structured content to disk     |                                                                                                                                                               |
| Write the structured content to disk| ReadWriteFile                  | Save extracted structured data JSON to disk       | Create a binary data for Structured Data Extract |                                         |                                                                                                                                                               |
| Initiate a Webhook Notification for Structured Data | HTTP Request            | Send extracted data to webhook notification URL   | DNB Structured Data Extract Using LLM |                                         |                                                                                                                                                               |
| Sticky Note2                      | Sticky Note                      | Disclaimer about community MCP node usage          |                              |                                         | This template is only available on n8n self-hosted as it uses the community MCP Client node.                                                                 |
| Sticky Note5                      | Sticky Note                      | Branding (Bright Data logo)                         |                              |                                         | ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)                                                                    |
| Sticky Note6                      | Sticky Note                      | LLM usage note                                     |                              |                                         | OpenAI 4o mini LLM is being utilized for the structured data extraction handling.                                                                              |
| OpenAI Chat Model for URL Data Extract | Langchain LM Chat OpenAI     | LLM model for URL extraction                        |                              | DNB URL Data Extract Using LLM           |                                                                                                                                                               |
| Structured Output Parser for URL    | Langchain Output Parser Structured | Parse LLM output for URLs to structured JSON       |                              | DNB URL Data Extract Using LLM           |                                                                                                                                                               |
| OpenAI Chat Model for DNB Structured Data Extract | Langchain LM Chat OpenAI | LLM model for structured company profile extraction |                              | DNB Structured Data Extract Using LLM   |                                                                                                                                                               |
| Structured Output Parser for Structured Extract | Langchain Output Parser Structured | Parse LLM output for company profile JSON          |                              | DNB Structured Data Extract Using LLM   |                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Test workflow’` with default settings.

2. **Add a Sticky Note** near the trigger with the content:  
   _"Deals with the DNB (https://www.dnb.com/) data extract using the Bright Data MCP Search and Markdown Web scraper. Please update the search query and webhook notification URL. Test using https://webhook.site/."_

3. **Add a MCP Client node** named `List all tools for Bright Data`.  
   - No parameters needed.  
   - Configure credentials for MCP Client (STDIO) with valid Bright Data credentials.

4. **Connect the Manual Trigger output to `List all tools for Bright Data` input.**

5. **Add a Set node** named `Set input fields`.  
   - Define two string fields:  
     - `search` with default value `"dnb starbucks url"`  
     - `webhook_notification_url` with default value e.g., `"https://webhook.site/your-test-url"`  
   - No additional options.

6. **Connect `List all tools for Bright Data` output to `Set input fields` input.**

7. **Add MCP Client node** named `MCP Client for Search Engine`.  
   - Set toolName to `"search_engine"`.  
   - Operation: `"executeTool"`.  
   - Tool parameters JSON:  
     ```json
     {
       "query": "={{ $json.search }}",
       "engine": "google"
     }
     ```  
   - Use same MCP Client (STDIO) credentials.

8. **Connect `Set input fields` output to `MCP Client for Search Engine` input.**

9. **Add Langchain chainLlm node** named `DNB URL Data Extract Using LLM`.  
   - Set `Prompt` to:  
     `Extract the URLs for DNB  {{ $json.result.content[0].text }}`  
   - Enable output parser.  
   - Enable retry on failure.

10. **Add Langchain LM Chat OpenAI node** named `OpenAI Chat Model for URL Data Extract`.  
    - Model: `gpt-4o-mini`  
    - Assign OpenAI API credentials.

11. **Connect `OpenAI Chat Model for URL Data Extract` to `DNB URL Data Extract Using LLM` as the language model.**

12. **Add Langchain Output Parser Structured node** named `Structured Output Parser for URL`.  
    - Input a JSON schema example with a single field `"url": "url"` to enforce URL extraction format.

13. **Connect `Structured Output Parser for URL` to `DNB URL Data Extract Using LLM`'s output parser input.**

14. **Connect `MCP Client for Search Engine` output to `DNB URL Data Extract Using LLM` input.**

15. **Add MCP Client node** named `Bright Data MCP Client For DNB`.  
    - ToolName: `"scrape_as_markdown"`  
    - Operation: `"executeTool"`  
    - Tool parameters:  
      ```json
      {
        "url": "={{ $json.output.url }}"
      }
      ```  
    - Use same MCP Client credentials.

16. **Connect `DNB URL Data Extract Using LLM` output to `Bright Data MCP Client For DNB` input.**

17. **Add Langchain chainLlm node** named `DNB Structured Data Extract Using LLM`.  
    - Prompt:  
      ```
      Extract the Company Profile from {{ $json.result.content[0].text }}

      Output in a highly structured JSON format.
      ```  
    - Enable output parser.  
    - Enable retry on failure.

18. **Add Langchain LM Chat OpenAI node** named `OpenAI Chat Model for DNB Structured Data Extract`.  
    - Model: `gpt-4o-mini`  
    - Assign OpenAI API credentials.

19. **Connect `OpenAI Chat Model for DNB Structured Data Extract` as language model to `DNB Structured Data Extract Using LLM`.**

20. **Add Langchain Output Parser Structured node** named `Structured Output Parser for Structured Extract`.  
    - Paste the provided JSON schema for the company profile into manual schema input.

21. **Connect `Structured Output Parser for Structured Extract` to `DNB Structured Data Extract Using LLM`'s output parser input.**

22. **Connect `Bright Data MCP Client For DNB` output to `DNB Structured Data Extract Using LLM` input.**

23. **Add Function node** named `Create a binary data for Structured Data Extract`.  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```

24. **Connect `DNB Structured Data Extract Using LLM` output to the Function node input.**

25. **Add ReadWriteFile node** named `Write the structured content to disk`.  
    - Operation: Write  
    - Filename: `d:\DNB_Info.json` (adjust as needed)  
    - Input: binary data from Function node.

26. **Connect Function node output to ReadWriteFile node input.**

27. **Add HTTP Request node** named `Initiate a Webhook Notification for Structured Data`.  
    - Method: POST  
    - URL: Dynamic expression: `={{ $('Set input fields').item.json.webhook_notification_url }}`  
    - Send body with parameter:  
      - Name: `dnb_company_info`  
      - Value: `={{ $json.output }}` (the structured company info JSON)  

28. **Connect `DNB Structured Data Extract Using LLM` output also to HTTP Request node input (parallel connection).**

29. **Add sticky notes as needed for disclaimers and branding:**  
    - Note about self-hosted requirement due to MCP Client community node.  
    - Bright Data logo.  
    - Note about OpenAI 4o mini usage.

30. **Validate connections and credentials are properly set for MCP Client and OpenAI API.**

31. **Test workflow using “Test workflow” button. Update input fields as needed for real queries and webhook URLs.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                      |
|------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Deals with the DNB data extract using Bright Data MCP Search and Markdown Web scraper. Please update the search query and webhook notification URL. Test using https://webhook.site/. | Workflow main sticky note.                                         |
| This template is only available on n8n self-hosted as it uses the community MCP Client node. | Disclaimer sticky note.                                             |
| OpenAI 4o mini LLM is being utilized for the structured data extraction handling. | LLM usage sticky note.                                             |
| Bright Data logo image.                                                       | https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal or offensive elements. All handled data is legal and publicly accessible.