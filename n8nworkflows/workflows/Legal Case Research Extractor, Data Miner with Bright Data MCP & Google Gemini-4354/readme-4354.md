Legal Case Research Extractor, Data Miner with Bright Data MCP & Google Gemini

https://n8nworkflows.xyz/workflows/legal-case-research-extractor--data-miner-with-bright-data-mcp---google-gemini-4354


# Legal Case Research Extractor, Data Miner with Bright Data MCP & Google Gemini

### 1. Workflow Overview

This workflow, titled **"Legal Case Research Extractor, Data Miner with Bright Data MCP & Google Gemini"**, automates the extraction and processing of legal case data from the website CourtListener. It is designed to:

- Scrape legal case listings and detailed case content.
- Use advanced scraping tools (Bright Data MCP Client) and AI models (OpenAI GPT-4o variant & Google Gemini) for structured data extraction and content summarization.
- Loop over extracted cases for detailed content retrieval.
- Send notifications via webhook and save data locally.

The workflow consists of the following logical blocks:

1.1 **Trigger and Initialization**  
- Manual trigger to start the workflow.  
- Setup of initial URLs and parameters.

1.2 **Legal Case Listing Scraping**  
- Use Bright Data MCP Client to scrape the main legal case search page.  
- AI processing to extract structured case data from raw scraped content.

1.3 **Iterative Detailed Case Scraping and Processing**  
- Loop over each extracted case link.  
- Scrape detailed case pages using Bright Data MCP within the loop.  
- Use Google Gemini AI to convert HTML content into textual data.  
- Send webhook notifications with extracted textual data.  
- Save each case's content as a JSON file.

1.4 **Supporting and Utility Functions**  
- Data transformations (e.g., creating binary data).  
- Wait node to manage pacing between loop iterations.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Initialization

**Overview:**  
Starts the workflow manually and sets all necessary URLs and parameters for the scraping process.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- List all tools for Bright Data  
- Set the Legal Case Research URL  
- Sticky Notes (for informational purposes)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: Default manual trigger with no parameters.  
  - Input/Output: None / Triggers "List all tools for Bright Data".  
  - Edge cases: No input failures. Requires user action.

- **List all tools for Bright Data**  
  - Type: MCP Client (Bright Data)  
  - Role: Lists available scraping tools from Bright Data MCP platform to confirm connectivity.  
  - Configuration: Uses MCP Client API credentials (MCP Client (STDIO) account).  
  - Input: Manual trigger output.  
  - Output: Triggers "Set the Legal Case Research URL".  
  - Edge cases: Possible API authentication errors or network issues.

- **Set the Legal Case Research URL**  
  - Type: Set node  
  - Role: Defines key variables:  
    - `url`: CourtListener search query URL for IT laws on cybercrime.  
    - `webhook_url`: Webhook.site URL for notifications.  
    - `base_url`: Base domain for CourtListener.  
  - Configuration: Hardcoded strings for URLs.  
  - Input: MCP Client output.  
  - Output: Triggers "Bright Data MCP Client For Legal Case Research".  
  - Edge cases: URL formatting errors if changed manually.

- **Sticky Notes (Informational)**  
  - Provide context about the workflow, usage instructions, disclaimers, and branding logos.  
  - No technical role.

---

#### 1.2 Legal Case Listing Scraping

**Overview:**  
Scrapes the CourtListener search results page for legal cases and extracts structured case data using AI.

**Nodes Involved:**  
- Bright Data MCP Client For Legal Case Research  
- Google Gemini Chat Model For Case Data Extract  
- Structured Output Parser  
- Case Extractor  
- Code to output the collection of cases

**Node Details:**

- **Bright Data MCP Client For Legal Case Research**  
  - Type: MCP Client  
  - Role: Scrapes the main search result page HTML from the provided URL (`url` variable).  
  - Configuration: Uses the `scrape_as_html` tool with URL parameter from `Set the Legal Case Research URL`.  
  - Input: URL set node output.  
  - Output: Raw HTML content under `result.content[0].text`.  
  - Edge cases:  
    - Scraping failures from network issues or anti-bot blocks.  
    - Rate limiting or API quota limits.

- **Google Gemini Chat Model For Case Data Extract**  
  - Type: LangChain AI model (Google Gemini)  
  - Role: Processes scraped HTML content to extract legal cases in a structured format.  
  - Configuration: Uses "models/gemini-2.0-flash-exp" Google Gemini chat model.  
  - Input: Raw scraped content passed as prompt text.  
  - Output: AI-generated structured data passed to output parser.  
  - Edge cases: AI model response errors, timeouts, or malformed outputs.

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses the AI response into JSON objects with schema sample having `Id`, `Link`, and `Title`.  
  - Configuration: JSON schema example provided to guide parsing.  
  - Input: AI model output.  
  - Output: Structured JSON case list.  
  - Edge cases: Parsing failures if AI output deviates from schema.

- **Case Extractor**  
  - Type: LangChain Chain LLM  
  - Role: Further processes parsed data, enforcing extraction logic on the structured content.  
  - Configuration: Prompt instructs expert structured data extraction on the parsed text.  
  - Input: Parsed structured JSON cases.  
  - Output: Final structured case data array.  
  - Edge cases: AI failures, retry enabled.

- **Code to output the collection of cases**  
  - Type: Code (JavaScript)  
  - Role: Converts first input JSON's `.output` property to the output of this node. Essentially flattens or passes on data.  
  - Configuration: `return $input.first().json.output`  
  - Input: Case Extractor output.  
  - Output: Triggers "Loop Over Items".  
  - Edge cases: None significant unless input missing `.output` property.

---

#### 1.3 Iterative Detailed Case Scraping and Processing

**Overview:**  
For each extracted case, scrapes detailed pages, extracts textual data using AI, sends notifications, and writes data to disk.

**Nodes Involved:**  
- Loop Over Items  
- Wait  
- Bright Data MCP Client For Legal Case Research Within Loop  
- Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop  
- HTML to Textual Data Extract Within Loop  
- Create a binary data for LinkedIn company info extract  
- Webhook Notification for HTML to Textual Data Extract Within the Loop  
- Write the case content to disk

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Loops through each case object from the collection to process individually.  
  - Configuration: Default batch size (assumed 1 per iteration).  
  - Input: Code node output (array of cases).  
  - Output: Main output triggers "Wait" on second output and also the nested scraping loop.  
  - Edge cases: Empty input array resulting in no iterations.

- **Wait**  
  - Type: Wait  
  - Role: Delays subsequent loop iterations by 10 seconds to avoid rate limiting or API overload.  
  - Configuration: 10 seconds delay.  
  - Input: Loop Over Items second output (used for pacing).  
  - Output: Connects back to "Bright Data MCP Client For Legal Case Research Within Loop" (creating pacing loop).  
  - Edge cases: Workflow timeout if batches too large or external API rate limits persist.

- **Bright Data MCP Client For Legal Case Research Within Loop**  
  - Type: MCP Client  
  - Role: Scrapes each individual case page HTML using the case's `Link` appended to base URL.  
  - Configuration: Uses `scrape_as_html` tool with URL constructed as `base_url` + `/` + case's `Link`.  
  - Input: Current case JSON from Loop Over Items.  
  - Output: Raw HTML content under `result.content[0].text`.  
  - Edge cases: Failures due to invalid URLs, network errors, anti-bot measures.

- **Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop**  
  - Type: LangChain AI model (Google Gemini)  
  - Role: Converts raw HTML content of each case detail page into clean textual data.  
  - Configuration: Uses Gemini chat model "models/gemini-2.0-flash-exp".  
  - Input: Raw scraped HTML from MCP client.  
  - Output: Textual data passed to "HTML to Textual Data Extract Within Loop".  
  - Edge cases: AI timeouts or malformed responses.

- **HTML to Textual Data Extract Within Loop**  
  - Type: LangChain Chain LLM  
  - Role: Processes AI model output to extract or refine textual case content.  
  - Configuration: Prompt instructs extraction of textual content from HTML.  
  - Input: AI model output.  
  - Output: Triggers both "Create a binary data for LinkedIn company info extract" and "Webhook Notification for HTML to Textual Data Extract Within the Loop".  
  - Edge cases: AI failures, retry enabled.

- **Create a binary data for LinkedIn company info extract**  
  - Type: Function  
  - Role: Encodes JSON case content into base64 binary data for file storage compatibility.  
  - Configuration: Converts JSON to string, then to base64 buffer stored as binary data.  
  - Input: Textual data JSON.  
  - Output: Triggers "Write the case content to disk".  
  - Edge cases: Encoding errors if input invalid.

- **Webhook Notification for HTML to Textual Data Extract Within the Loop**  
  - Type: HTTP Request  
  - Role: Sends extracted textual case content to a configured webhook URL (e.g., webhook.site) for external notifications or integrations.  
  - Configuration: Multipart form-data POST with field `case_content` containing textual content. URL from `webhook_url` variable.  
  - Input: Textual data JSON.  
  - Output: Triggers Loop Over Items (to continue loop).  
  - Edge cases: Network failures, webhook endpoint unavailability.

- **Write the case content to disk**  
  - Type: Read/Write File  
  - Role: Saves extracted case content to local disk as JSON files named using case `Id`.  
  - Configuration: Files saved to `d:\Case-<Id>.json`.  
  - Input: Binary encoded JSON data from function.  
  - Output: None (end of branch).  
  - Edge cases: File system permission errors, path availability issues.

---

#### 1.4 Supporting and Utility Functions

**Overview:**  
Contains informational notes and minor utility nodes for workflow control.

**Nodes Involved:**  
- Sticky Notes (Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5)

**Node Details:**  
- Provide disclaimers, branding, usage instructions, and AI usage notes.  
- Sticky Note2 highlights that the MCP Client node is community contributed and only available on n8n self-hosted.  
- Sticky Note3 reminds users to update URLs and webhook endpoints as needed.

---

### 3. Summary Table

| Node Name                                                  | Node Type                          | Functional Role                                          | Input Node(s)                                   | Output Node(s)                                                       | Sticky Note                                                                                                                                                                    |
|------------------------------------------------------------|----------------------------------|----------------------------------------------------------|------------------------------------------------|--------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                              | Manual Trigger                   | Manual start trigger                                     | None                                           | List all tools for Bright Data                                     |                                                                                                                                                                               |
| List all tools for Bright Data                             | MCP Client                      | Lists available Bright Data tools                        | When clicking ‘Test workflow’                   | Set the Legal Case Research URL                                   |                                                                                                                                                                               |
| Set the Legal Case Research URL                            | Set                              | Sets initial URL and webhook parameters                  | List all tools for Bright Data                  | Bright Data MCP Client For Legal Case Research                    |                                                                                                                                                                               |
| Bright Data MCP Client For Legal Case Research             | MCP Client                      | Scrapes main legal case search page                      | Set the Legal Case Research URL                 | Google Gemini Chat Model For Case Data Extract                     | Scrape a single webpage URL with advanced options for content extraction and get back the results in MarkDown language.                                                       |
| Google Gemini Chat Model For Case Data Extract             | LangChain AI (Google Gemini)    | Extracts structured case data from raw HTML              | Bright Data MCP Client For Legal Case Research  | Structured Output Parser                                          |                                                                                                                                                                               |
| Structured Output Parser                                   | LangChain Output Parser          | Parses AI output into JSON structured format             | Google Gemini Chat Model For Case Data Extract  | Case Extractor                                                   |                                                                                                                                                                               |
| Case Extractor                                            | LangChain Chain LLM              | Refines and structures case extraction                    | Structured Output Parser                         | Code to output the collection of cases                            |                                                                                                                                                                               |
| Code to output the collection of cases                    | Code                            | Outputs final structured case list                        | Case Extractor                                  | Loop Over Items                                                  |                                                                                                                                                                               |
| Loop Over Items                                           | SplitInBatches                  | Iterates over each extracted case                         | Code to output the collection of cases          | Wait (batch pacing), Bright Data MCP Client For Legal Case Research Within Loop | Bright Data Legal Case Research Scraper Loop through and perform the data extraction using MCP and LLMs                                                                        |
| Wait                                                     | Wait                            | Delays loop iteration by 10 seconds                       | Loop Over Items (second output)                  | Bright Data MCP Client For Legal Case Research Within Loop       |                                                                                                                                                                               |
| Bright Data MCP Client For Legal Case Research Within Loop | MCP Client                      | Scrapes individual case detail pages                      | Loop Over Items                                | Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop | Scrape a single webpage URL with advanced options for content extraction and get back the results in MarkDown language.                                                       |
| Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop | LangChain AI (Google Gemini)    | Converts HTML content to textual data                      | Bright Data MCP Client For Legal Case Research Within Loop | HTML to Textual Data Extract Within Loop                          |                                                                                                                                                                               |
| HTML to Textual Data Extract Within Loop                  | LangChain Chain LLM              | Extracts/refines textual content from HTML                | Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop | Create a binary data for LinkedIn company info extract, Webhook Notification for HTML to Textual Data Extract Within the Loop |                                                                                                                                                                               |
| Create a binary data for LinkedIn company info extract    | Function                        | Encodes JSON data into base64 binary for storage          | HTML to Textual Data Extract Within Loop         | Write the case content to disk                                   |                                                                                                                                                                               |
| Webhook Notification for HTML to Textual Data Extract Within the Loop | HTTP Request                   | Sends extracted textual content to external webhook       | HTML to Textual Data Extract Within Loop         | Loop Over Items                                                 |                                                                                                                                                                               |
| Write the case content to disk                            | Read/Write File                 | Writes case content JSON file to disk                      | Create a binary data for LinkedIn company info extract | None                                                           |                                                                                                                                                                               |
| Sticky Note                                               | Sticky Note                    | Workflow title and branding                               | None                                           | None                                                           | ## Bright Data Legal Case Research Scraper                                                                                                                                     |
| Sticky Note1                                              | Sticky Note                    | Explains loop and extraction logic                        | None                                           | None                                                           | ## Bright Data Legal Case Research Scraper Loop through and perform the data extraction using MCP and LLMs                                                                     |
| Sticky Note2                                              | Sticky Note                    | Disclaimer about MCP Client node availability             | None                                           | None                                                           | ## Disclaimer This template is only available on n8n self-hosted as it's making use of the community node for MCP Client.                                                       |
| Sticky Note3                                              | Sticky Note                    | Usage notes and reminders                                 | None                                           | None                                                           | ## Note Deals with the Legal Case data extraction by utilizing the Bright Data MCP and OpenAI GPT 4o LLM. Please make sure to set the input fields node with the Legal case URL. Please make sure to update the Webhook Notification URL of your interest |
| Sticky Note4                                              | Sticky Note                    | AI usage explanation                                     | None                                           | None                                                           | ## LLM Usages OpenAI 4o mini LLM is being utilized for the structured data extraction handling.                                                                                  |
| Sticky Note5                                              | Sticky Note                    | Branding logo                                            | None                                           | None                                                           | ## Logo ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger (default settings).

2. **Add MCP Client Node to List Tools**  
   - Name: "List all tools for Bright Data"  
   - Type: MCP Client (Bright Data)  
   - Credentials: Set MCP Client API credentials (MCP Client (STDIO) account).  
   - No parameters needed.  
   - Connect output of manual trigger to this node.

3. **Add Set Node to Define URLs**  
   - Name: "Set the Legal Case Research URL"  
   - Type: Set node  
   - Add fields:  
     - `url` = "https://www.courtlistener.com/?q=IT%20laws%20for%20cyber%20crime&type=o&order_by=dateFiled%20desc&stat_Published=on"  
     - `webhook_url` = "https://webhook.site/7b5380a0-0544-48dc-be43-0116cb2d52c2" (replace with your webhook)  
     - `base_url` = "https://www.courtlistener.com"  
   - Connect output of "List all tools for Bright Data" to this node.

4. **Add MCP Client Node to Scrape Legal Case Search Page**  
   - Name: "Bright Data MCP Client For Legal Case Research"  
   - Type: MCP Client  
   - Credentials: MCP Client API credentials (same as before).  
   - Parameters:  
     - Tool Name: "scrape_as_html"  
     - Operation: "executeTool"  
     - Tool Parameters: JSON with `"url": "{{ $json.url }}"` referencing the Set node.  
   - Connect output of Set node to this node.

5. **Add Google Gemini Chat Model Node for Case Data Extraction**  
   - Name: "Google Gemini Chat Model For Case Data Extract"  
   - Type: LangChain LLM Google Gemini Chat  
   - Credentials: Google Palm API credentials.  
   - Parameters:  
     - Model Name: "models/gemini-2.0-flash-exp"  
     - Text prompt: `"Extract the content in a structured format. Here's the content : {{ $json.result.content[0].text }}"`  
     - Add system message: "You are an expert structured data extractor."  
   - Connect output of MCP Client scrape node here.

6. **Add Structured Output Parser Node**  
   - Name: "Structured Output Parser"  
   - Type: LangChain Output Parser Structured  
   - Parameters: Provide JSON schema example with fields `Id`, `Link`, `Title`.  
   - Connect output of Google Gemini Chat Model node here.

7. **Add Case Extractor Node**  
   - Name: "Case Extractor"  
   - Type: LangChain Chain LLM  
   - Parameters: Prompt `"Extract the content in a structured format. Here's the content : {{ $json.result.content[0].text }}"`  
   - Set retry on fail enabled.  
   - Connect output parser node to this node.

8. **Add Code Node to Output Cases**  
   - Name: "Code to output the collection of cases"  
   - Type: Code (JavaScript)  
   - Code: `return $input.first().json.output`  
   - Connect Case Extractor output to this node.

9. **Add SplitInBatches Node to Loop Over Cases**  
   - Name: "Loop Over Items"  
   - Type: SplitInBatches  
   - Default settings (batch size 1).  
   - Connect Code node output here.

10. **Add Wait Node for Loop Pacing**  
    - Name: "Wait"  
    - Type: Wait  
    - Parameters: Wait for 10 seconds.  
    - Connect second output of Loop Over Items to Wait node.

11. **Add MCP Client Node to Scrape Each Case Detail Page**  
    - Name: "Bright Data MCP Client For Legal Case Research Within Loop"  
    - Type: MCP Client  
    - Credentials: MCP Client API credentials.  
    - Parameters:  
      - Tool Name: "scrape_as_html"  
      - Operation: "executeTool"  
      - Tool Parameters: `"url": "{{ $('Set the Legal Case Research URL').item.json.base_url }}/{{ $json.Link }}"`  
    - Connect first output of Loop Over Items to this node.  
    - Connect Wait node output back to this node (loop pacing).

12. **Add Google Gemini Chat Model Node for HTML to Textual Extract**  
    - Name: "Google Gemini Chat Model for HTML to Textual Data Extract Within the Loop"  
    - Type: LangChain LLM Google Gemini Chat  
    - Credentials: Google Palm API credentials.  
    - Parameters:  
      - Model Name: "models/gemini-2.0-flash-exp"  
      - Prompt: `=Extract html to textual content  {{ $json.result.content[0].text }}`  
    - Connect MCP Client within loop output to this node.

13. **Add Chain LLM Node for Textual Data Extraction**  
    - Name: "HTML to Textual Data Extract Within Loop"  
    - Type: LangChain Chain LLM  
    - Parameters: Prompt: `=Extract html to textual content  {{ $json.result.content[0].text }}`  
    - Retry on fail enabled.  
    - Connect Google Gemini chat model node output here.

14. **Add Function Node to Create Binary Data for Storage**  
    - Name: "Create a binary data for LinkedIn company info extract"  
    - Type: Function  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect Chain LLM output here.

15. **Add HTTP Request Node for Webhook Notification**  
    - Name: "Webhook Notification for HTML to Textual Data Extract Within the Loop"  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `={{ $('Set the Legal Case Research URL').item.json.webhook_url }}`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body parameters: field "case_content" set to `={{ $json.text }}`  
    - Connect Chain LLM output here (parallel to Function node).

16. **Add Read/Write File Node to Save Extracted Case Content**  
    - Name: "Write the case content to disk"  
    - Type: Read/Write File  
    - Parameters:  
      - Operation: Write  
      - File Name: `d:\Case-{{ $('Loop Over Items').item.json['Id'] }}.json`  
    - Connect Function node output here.

17. **Add Sticky Notes**  
    - Add multiple sticky notes for branding, disclaimers, usage instructions as per original workflow to improve maintainability and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                     |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| This workflow utilizes the Bright Data MCP Client community node, which is only available on n8n self-hosted.    | MCP Client Node availability note.                               |
| The AI models used include Google Gemini (PaLM) and OpenAI GPT-4o mini variant for structured data extraction.  | AI model usage information.                                      |
| Users must update the input URL for legal case search and webhook URLs for notification to their own endpoints. | Workflow customization instructions.                             |
| Branding logo for Bright Data is included as a sticky note with an image link: ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Branding.                                                         |

---

**Disclaimer:** The data processed and extracted by this workflow is from publicly accessible legal case information. All scraping and AI usage comply with relevant terms of service and legal requirements. The workflow is designed for legal research and data mining purposes only.