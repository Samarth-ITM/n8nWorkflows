Indeed Data Scraper & Summarization with Airtable, Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/indeed-data-scraper---summarization-with-airtable--bright-data---google-gemini-3703


# Indeed Data Scraper & Summarization with Airtable, Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the extraction, summarization, and forwarding of company profile data from Indeed.com. It is designed primarily for recruiters, HR teams, market researchers, founders, investors, and no-code users who want to automate the process of gathering structured company insights without manual scraping.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Triggering the workflow manually and setting up initial parameters such as the Bright Data Web Unlocker zone.

- **1.2 Data Retrieval from Airtable**  
  Fetching company URLs from an Airtable base that contains Indeed company profile links.

- **1.3 Iterative Processing and Validation**  
  Looping over each Airtable record, waiting briefly between requests, and verifying that the company link is present before proceeding.

- **1.4 Web Scraping via Bright Data Web Unlocker**  
  Sending requests to Bright Data’s Web Unlocker API to scrape the raw HTML content of the Indeed company profile pages.

- **1.5 Data Extraction and Transformation**  
  Converting the raw markdown data from Bright Data into textual data, summarizing it using Google Gemini LLM, and formatting the output.

- **1.6 AI Agent Processing and Output Forwarding**  
  Using an AI agent to finalize formatting and send the summarized data to a specified webhook endpoint for downstream use.

- **1.7 Optional HTML Conversion and Notification**  
  Converting markdown to HTML and sending an additional webhook notification with the HTML response (optional, for monitoring or display purposes).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets the Bright Data Web Unlocker zone parameter for subsequent scraping requests.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Bright Data Zone (Set Node)  
  - Sticky Note (Instructional)

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user demand.  
    - Config: No parameters; simple manual trigger.  
    - Connections: Outputs to Set Bright Data Zone.  
    - Edge Cases: None; manual trigger only.

  - **Set Bright Data Zone**  
    - Type: Set Node  
    - Role: Defines the Bright Data zone parameter used for API requests.  
    - Config: Sets a string variable `zone` with value `"web_unlocker1"`.  
    - Connections: Outputs to Airtable node.  
    - Edge Cases: If the zone name is incorrect or not configured in Bright Data, scraping will fail.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Instructions about using Bright Data Web Unlocker, Airtable base setup, and webhook URL update.  
    - Role: Documentation aid for users.

---

#### 2.2 Data Retrieval from Airtable

- **Overview:**  
  Retrieves a list of company profile URLs from a specified Airtable base and table.

- **Nodes Involved:**  
  - Airtable  
  - Loop Over Items (SplitInBatches)  
  - Sticky Note2 (Data sample)

- **Node Details:**  
  - **Airtable**  
    - Type: Airtable Node  
    - Role: Searches and fetches records from Airtable base "Indeed" and table "Table 1".  
    - Config: Uses Airtable Personal Access Token credentials.  
    - Output: List of records containing fields such as `Tab` (company name) and `Link` (Indeed URL).  
    - Connections: Outputs to Loop Over Items.  
    - Edge Cases: API rate limits, invalid credentials, or empty tables.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes Airtable records one by one to avoid overloading APIs or hitting rate limits.  
    - Config: Default batch size (1).  
    - Connections: Outputs to Wait node (for pacing) and further processing.  
    - Edge Cases: Empty input list results in no processing.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Content: Sample JSON data from Airtable showing expected record structure.  
    - Role: Reference for users to understand input data format.

---

#### 2.3 Iterative Processing and Validation

- **Overview:**  
  Controls the pacing of requests and ensures only records with valid Indeed links proceed to scraping.

- **Nodes Involved:**  
  - Wait  
  - If Link field is not empty

- **Node Details:**  
  - **Wait**  
    - Type: Wait Node  
    - Role: Pauses workflow execution for 10 seconds between processing each record to avoid API throttling or bans.  
    - Config: Fixed 10-second delay.  
    - Connections: Outputs to If Link field is not empty.  
    - Edge Cases: Long workflows if many records; can be adjusted.

  - **If Link field is not empty**  
    - Type: If Node  
    - Role: Checks if the `Link` field in the current record is not empty before proceeding.  
    - Config: Condition: `Link` string is not empty.  
    - Connections: True branch to Perform Indeed Web Request; False branch ends processing for that item.  
    - Edge Cases: Missing or malformed URLs cause skipping.

---

#### 2.4 Web Scraping via Bright Data Web Unlocker

- **Overview:**  
  Sends a POST request to Bright Data’s Web Unlocker API to scrape the raw HTML content of the Indeed company profile page.

- **Nodes Involved:**  
  - Perform Indeed Web Request  
  - Sticky Note (Instructional)

- **Node Details:**  
  - **Perform Indeed Web Request**  
    - Type: HTTP Request Node  
    - Role: Calls Bright Data API with parameters to unlock and scrape the Indeed company page.  
    - Config:  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Headers: Uses HTTP Header Auth credentials for Bright Data.  
      - Body Parameters:  
        - `zone`: from Set Bright Data Zone node (`web_unlocker1`)  
        - `url`: constructed Indeed company URL from Airtable record, encoded  
        - `format`: `raw`  
        - `data_format`: `markdown`  
    - Connections: Outputs to Markdown to Textual Data Extractor and Convert Markdown to HTML nodes.  
    - Edge Cases: Authentication failure, rate limiting, invalid URLs, API downtime.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Notes about Bright Data Web Unlocker usage and AI capabilities demonstration.

---

#### 2.5 Data Extraction and Transformation

- **Overview:**  
  Converts the raw markdown scraped data into clean textual data, then summarizes it using Google Gemini LLM.

- **Nodes Involved:**  
  - Markdown to Textual Data Extractor  
  - Indeed Summarizer  
  - Google Gemini Chat Model For Summarization  
  - Sticky Note1 (LLM usage explanation)

- **Node Details:**  
  - **Markdown to Textual Data Extractor**  
    - Type: Chain LLM Node  
    - Role: Processes markdown data to extract clean textual content.  
    - Config: Uses prompt defining the role as a markdown expert analyzing the input markdown.  
    - Input: `data` field from Perform Indeed Web Request node.  
    - Output: Textual data for summarization.  
    - Connections: Outputs to Indeed Summarizer.  
    - Edge Cases: Parsing errors if markdown is malformed.

  - **Indeed Summarizer**  
    - Type: Chain Summarization Node  
    - Role: Summarizes the textual data into concise company insights.  
    - Config: Uses Google Gemini Chat Model For Summarization as the language model.  
    - Connections: Outputs to Indeed Expert AI Agent.  
    - Edge Cases: LLM API errors, rate limits.

  - **Google Gemini Chat Model For Summarization**  
    - Type: Google Gemini LLM Chat Node  
    - Role: Provides the underlying language model for summarization.  
    - Config: Model name `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini (PaLM) API key.  
    - Connections: Linked as AI language model for Indeed Summarizer.  
    - Edge Cases: API key invalid, quota exceeded.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Explains usage of Google Gemini Flash Exp model, summarization chain, and AI agent formatting.

---

#### 2.6 AI Agent Processing and Output Forwarding

- **Overview:**  
  An AI agent formats the summarized search result and sends it to a configured webhook endpoint for downstream consumption.

- **Nodes Involved:**  
  - Indeed Expert AI Agent  
  - Google Gemini Chat Model for AI Agent  
  - Webhook HTTP Request

- **Node Details:**  
  - **Indeed Expert AI Agent**  
    - Type: Langchain AI Agent Node  
    - Role: Formats the summarized text and prepares it for webhook transmission.  
    - Config: Prompt defines role as an Indeed Expert formatting the search result and pushing it to webhook.  
    - Input: Text from Indeed Summarizer (via Markdown to Textual Data Extractor).  
    - Connections: Outputs to Loop Over Items (to continue processing) and Webhook HTTP Request (as AI tool).  
    - Edge Cases: Agent prompt failures, API errors.

  - **Google Gemini Chat Model for AI Agent**  
    - Type: Google Gemini LLM Chat Node  
    - Role: Provides language model for AI Agent.  
    - Config: Same model as summarization (`models/gemini-2.0-flash-exp`).  
    - Credentials: Google Gemini API.  
    - Connections: AI language model for Indeed Expert AI Agent.  
    - Edge Cases: Same as other Google Gemini nodes.

  - **Webhook HTTP Request**  
    - Type: HTTP Request Node  
    - Role: Sends the formatted summary to a webhook endpoint.  
    - Config:  
      - URL: User-configured webhook URL (example: webhook.site URL)  
      - Method: POST  
      - Body: Includes `search_summary` from AI Agent response text and `search_result` (empty or additional data).  
    - Connections: Invoked as AI tool by Indeed Expert AI Agent.  
    - Edge Cases: Webhook endpoint down, network errors.

---

#### 2.7 Optional HTML Conversion and Notification

- **Overview:**  
  Converts the markdown scraped data to HTML and sends it to a webhook for notification or monitoring purposes.

- **Nodes Involved:**  
  - Convert Markdown to HTML  
  - Initiate a Webhook Notification for Markdown to HTML Response

- **Node Details:**  
  - **Convert Markdown to HTML**  
    - Type: Markdown Node  
    - Role: Converts markdown text from scraping into HTML format.  
    - Config: Mode set to `markdownToHtml`.  
    - Input: `data` field from Perform Indeed Web Request.  
    - Connections: Outputs to Initiate a Webhook Notification node.  
    - Edge Cases: Malformed markdown input.

  - **Initiate a Webhook Notification for Markdown to HTML Response**  
    - Type: HTTP Request Node  
    - Role: Sends the HTML response to a webhook endpoint for monitoring or display.  
    - Config:  
      - URL: Same webhook URL as main output or separate endpoint.  
      - Method: POST  
      - Body: Contains `html_response` field with HTML data.  
    - Edge Cases: Webhook failures, network issues.

---

### 3. Summary Table

| Node Name                             | Node Type                         | Functional Role                                  | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                                                       |
|-------------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’       | Manual Trigger                   | Starts the workflow manually                     |                               | Set Bright Data Zone                       | Deals with the Company web scraping by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Summarization and AI Agent are being used to demonstrate the usage of the n8n AI capabilities. Please make sure to connect to Airtable with the Base Table as "Indeed" and the default Table1 filled with the indeed links to scrape. Also make sure to update the Webhook Notification URL |
| Set Bright Data Zone                | Set Node                        | Sets Bright Data zone parameter                   | When clicking ‘Test workflow’ | Airtable                                  | See above                                                                                                                        |
| Airtable                           | Airtable Node                   | Retrieves company URLs from Airtable              | Set Bright Data Zone           | Loop Over Items                           | Airtable Table Data Sample: [{"id":"recCDNhVfdlc97cgf","createdTime":"2025-04-14T02:55:31.000Z","Tab":"Starbucks","Link":"https://www.indeed.com/cmp/Starbucks"}, ...] |
| Loop Over Items                    | SplitInBatches                  | Processes each Airtable record individually       | Airtable                      | Wait, (second output to Loop Over Items) | See above                                                                                                                        |
| Wait                              | Wait Node                      | Pauses 10 seconds between processing each record | Loop Over Items               | If Link field is not empty                |                                                                                                                                |
| If Link field is not empty         | If Node                        | Checks if company link is present                  | Wait                         | Perform Indeed Web Request                 |                                                                                                                                |
| Perform Indeed Web Request          | HTTP Request                   | Scrapes Indeed company page via Bright Data API   | If Link field is not empty    | Markdown to Textual Data Extractor, Convert Markdown to HTML | See first Sticky Note                                                                                                           |
| Markdown to Textual Data Extractor  | Chain LLM                      | Converts markdown to clean textual data           | Perform Indeed Web Request    | Indeed Summarizer                         | LLM Usages: Google Gemini Flash Exp model is used. Basic LLM Chain Data Extractor. Summarization Chain is used for summarization. AI Agent formats and pushes to Webhook. |
| Indeed Summarizer                  | Chain Summarization            | Summarizes textual company data                    | Markdown to Textual Data Extractor | Indeed Expert AI Agent                   | See above                                                                                                                        |
| Google Gemini Chat Model For Summarization | Google Gemini LLM Chat Node | Provides LLM for summarization                      |                             | Indeed Summarizer                         | See above                                                                                                                        |
| Indeed Expert AI Agent             | Langchain AI Agent             | Formats summary and prepares webhook payload      | Indeed Summarizer             | Loop Over Items, Webhook HTTP Request     | See above                                                                                                                        |
| Google Gemini Chat Model for AI Agent | Google Gemini LLM Chat Node | Provides LLM for AI Agent                           |                             | Indeed Expert AI Agent                     | See above                                                                                                                        |
| Webhook HTTP Request              | HTTP Request                   | Sends formatted summary to webhook endpoint       | Indeed Expert AI Agent (ai_tool) |                                         | See above                                                                                                                        |
| Convert Markdown to HTML           | Markdown Node                  | Converts markdown scraped data to HTML             | Perform Indeed Web Request    | Initiate a Webhook Notification for Markdown to HTML Response |                                                                                                                                |
| Initiate a Webhook Notification for Markdown to HTML Response | HTTP Request                   | Sends HTML response to webhook                      | Convert Markdown to HTML      |                                            |                                                                                                                                |
| Sticky Note                       | Sticky Note                   | Instructional note                                 |                               |                                            | See first Sticky Note                                                                                                           |
| Sticky Note1                      | Sticky Note                   | Explains LLM usage                                |                               |                                            | See LLM Usages note                                                                                                            |
| Sticky Note2                      | Sticky Note                   | Shows Airtable data sample                         |                               |                                            | See Airtable Table Data Sample                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Add a Set Node**  
   - Name: `Set Bright Data Zone`  
   - Purpose: Define a variable `zone` with value `"web_unlocker1"` (or your Bright Data zone).  
   - Connect output of Manual Trigger to this node.

3. **Add an Airtable Node**  
   - Name: `Airtable`  
   - Purpose: Fetch company records from Airtable.  
   - Configure credentials with your Airtable Personal Access Token.  
   - Set Base to your Airtable base containing Indeed company data (e.g., "Indeed").  
   - Set Table to the table containing company URLs (e.g., "Table 1").  
   - Connect output of Set Bright Data Zone to Airtable node.

4. **Add a SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Purpose: Process each Airtable record individually.  
   - Connect Airtable output to this node.

5. **Add a Wait Node**  
   - Name: `Wait`  
   - Purpose: Pause 10 seconds between processing each record to avoid rate limits.  
   - Set amount to 10 seconds.  
   - Connect output of Loop Over Items (first output) to Wait node.

6. **Add an If Node**  
   - Name: `If Link field is not empty`  
   - Purpose: Check if the current record has a non-empty `Link` field.  
   - Condition: `Link` string is not empty.  
   - Connect output of Wait node to this If node.

7. **Add an HTTP Request Node**  
   - Name: `Perform Indeed Web Request`  
   - Purpose: Scrape Indeed company page using Bright Data Web Unlocker API.  
   - Configure HTTP Header Auth credentials with Bright Data API key.  
   - Set URL to `https://api.brightdata.com/request` and method to POST.  
   - Body parameters:  
     - `zone`: expression referencing `zone` from Set Bright Data Zone node.  
     - `url`: construct Indeed company URL using Airtable `Link` field, URL-encoded.  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Connect True output of If node to this node.

8. **Add a Chain LLM Node**  
   - Name: `Markdown to Textual Data Extractor`  
   - Purpose: Convert markdown scraped data to clean text.  
   - Configure prompt to analyze markdown as a markdown expert.  
   - Input: `data` field from Perform Indeed Web Request.  
   - Connect output of Perform Indeed Web Request to this node.

9. **Add a Chain Summarization Node**  
   - Name: `Indeed Summarizer`  
   - Purpose: Summarize textual data into concise company insights.  
   - Configure to use Google Gemini Chat Model For Summarization as language model.  
   - Connect output of Markdown to Textual Data Extractor to this node.

10. **Add a Google Gemini LLM Chat Node**  
    - Name: `Google Gemini Chat Model For Summarization`  
    - Purpose: Provide LLM for summarization.  
    - Set model name to `models/gemini-2.0-flash-exp`.  
    - Use Google Gemini (PaLM) API credentials.  
    - Connect as AI language model for Indeed Summarizer.

11. **Add a Langchain AI Agent Node**  
    - Name: `Indeed Expert AI Agent`  
    - Purpose: Format summarized data and prepare webhook payload.  
    - Configure prompt to act as Indeed Expert formatting the search result and pushing to webhook.  
    - Connect output of Indeed Summarizer to this node.

12. **Add a Google Gemini LLM Chat Node**  
    - Name: `Google Gemini Chat Model for AI Agent`  
    - Purpose: Provide LLM for AI Agent.  
    - Same model and credentials as summarization node.  
    - Connect as AI language model for Indeed Expert AI Agent.

13. **Add an HTTP Request Node**  
    - Name: `Webhook HTTP Request`  
    - Purpose: Send formatted summary to webhook endpoint.  
    - Set URL to your webhook endpoint (e.g., webhook.site URL).  
    - Method: POST.  
    - Body parameters:  
      - `search_summary`: from AI Agent response text.  
      - `search_result`: optional or empty.  
    - Connect AI tool output of Indeed Expert AI Agent to this node.

14. **Connect output of Indeed Expert AI Agent back to Loop Over Items**  
    - Purpose: Continue processing next record.

15. **Optional: Add a Markdown Node**  
    - Name: `Convert Markdown to HTML`  
    - Purpose: Convert markdown scraped data to HTML.  
    - Input: `data` from Perform Indeed Web Request.  
    - Connect output of Perform Indeed Web Request to this node.

16. **Optional: Add an HTTP Request Node**  
    - Name: `Initiate a Webhook Notification for Markdown to HTML Response`  
    - Purpose: Send HTML response to webhook for monitoring.  
    - Configure webhook URL and POST method.  
    - Body parameter: `html_response` with HTML data.  
    - Connect output of Convert Markdown to HTML to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Deals with the Company web scraping by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Summarization and AI Agent demonstrate n8n AI capabilities. | Sticky Note at workflow start.                                                                        |
| Google Gemini Flash Exp model is used for summarization and AI agent tasks.                                                        | Sticky Note1 describing LLM usage.                                                                    |
| Airtable base "Indeed" with table containing company names and Indeed URLs is required. Sample data provided in Sticky Note2.     | Sticky Note2 showing sample Airtable data.                                                           |
| Setup instructions: Sign up at Bright Data, create Web Unlocker zone, configure credentials in n8n for Bright Data, Google Gemini, and Airtable. Update webhook URL accordingly. | Workflow description section.                                                                          |
| Workflow is designed for recruiters, HR teams, market researchers, founders, investors, and no-code users automating company data extraction and summarization. | Workflow description section.                                                                          |
| For customization, users can extend scraper targets, modify Gemini prompts, or route outputs to other platforms like Google Sheets or CRMs. | Workflow description section.                                                                          |
| Bright Data Web Unlocker API documentation: https://brightdata.com/docs/web-unlocker-api                                           | External resource for API setup and troubleshooting.                                                  |
| Google Gemini (PaLM) API documentation: https://developers.generativeai.google/api/guides                                         | External resource for LLM API usage and limits.                                                       |
| Airtable API documentation: https://airtable.com/api                                                                          | External resource for Airtable integration and troubleshooting.                                       |

---

This document provides a complete, structured reference for understanding, reproducing, and modifying the Indeed Data Scraper & Summarization workflow using n8n, Bright Data, Airtable, and Google Gemini. It anticipates common failure points such as API authentication errors, rate limits, missing data, and network issues, and offers guidance on setup and customization.