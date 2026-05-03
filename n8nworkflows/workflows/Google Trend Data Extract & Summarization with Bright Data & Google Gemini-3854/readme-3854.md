Google Trend Data Extract & Summarization with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/google-trend-data-extract---summarization-with-bright-data---google-gemini-3854


# Google Trend Data Extract & Summarization with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the extraction, structuring, summarization, and distribution of Google Trends data using Bright Data's Web Unlocker and Google Gemini AI models. It is designed for professionals such as market researchers, content strategists, SEO analysts, growth hackers, and AI engineers who require automated, scalable insights from Google Trends data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Manual trigger to start the workflow and setting the target URL and Bright Data zone for web scraping.

- **1.2 Data Extraction via Bright Data Web Unlocker**: Performs a web request to Bright Data’s API to scrape Google Trends data in markdown format.

- **1.3 Markdown to Plain Text Conversion**: Converts the scraped markdown content into clean, textual data suitable for further processing.

- **1.4 Structured Data Extraction**: Parses the plain text into structured JSON format representing Google Trends topics and descriptions.

- **1.5 AI Summarization**: Uses Google Gemini AI to generate a human-readable summary of the structured trend data.

- **1.6 Distribution and Persistence**: Sends the summary via Gmail, triggers webhook notifications, and saves the structured data to disk for archiving.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets the key parameters: the URL to scrape and the Bright Data Web Unlocker zone.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set URL and Bright Data Zone (Set Node)  
  - Sticky Note (Instructional)  
  - Sticky Note1 (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers workflow execution on demand.  
    - Inputs: None  
    - Outputs: Connects to "Set URL and Bright Data Zone"  
    - Edge Cases: None (manual trigger)  

  - **Set URL and Bright Data Zone**  
    - Type: Set Node  
    - Role: Defines workflow variables `url` and `zone` for scraping.  
    - Configuration:  
      - `url`: Set to `"https://trends.google.com/trends/explore?gprop=youtube&hl=en-US"` (Google Trends YouTube property, English US)  
      - `zone`: Set to `"web_unlocker1"` (Bright Data zone name)  
    - Inputs: From manual trigger  
    - Outputs: To "Perform Bright Data Web Request"  
    - Edge Cases: URL or zone misconfiguration leads to failed scraping requests.  

  - **Sticky Note & Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provide important instructions and context about the workflow, including setup notes and AI model usage.  
    - Inputs/Outputs: None (informational)  

#### 2.2 Data Extraction via Bright Data Web Unlocker

- **Overview:**  
  This block sends a POST request to Bright Data’s Web Unlocker API to extract raw Google Trends data in markdown format.

- **Nodes Involved:**  
  - Perform Bright Data Web Request (HTTP Request)  

- **Node Details:**

  - **Perform Bright Data Web Request**  
    - Type: HTTP Request  
    - Role: Calls Bright Data API to scrape the target URL using the specified zone.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.brightdata.com/request`  
      - Body Parameters:  
        - `zone`: From previous node (`zone` variable)  
        - `url`: Concatenated with `?product=unlocker&method=api`  
        - `format`: `raw`  
        - `data_format`: `markdown` (requests markdown output)  
      - Authentication: Header Authentication with Bearer token (configured in credentials)  
    - Inputs: From "Set URL and Bright Data Zone"  
    - Outputs: To "Markdown to Textual Data Extractor"  
    - Edge Cases:  
      - Authentication failure (invalid token)  
      - API rate limits or network timeouts  
      - Invalid zone or URL causing empty or error responses  

#### 2.3 Markdown to Plain Text Conversion

- **Overview:**  
  Converts the markdown content extracted by Bright Data into clean textual data, removing links, scripts, CSS, or other non-text elements.

- **Nodes Involved:**  
  - Markdown to Textual Data Extractor (Langchain Chain LLM)  
  - Initiate a Webhook Notification for Markdown to Textual Data Extraction (HTTP Request)  

- **Node Details:**

  - **Markdown to Textual Data Extractor**  
    - Type: Langchain Chain LLM Node  
    - Role: Uses a prompt-driven AI chain to parse markdown into plain text without additional commentary or formatting.  
    - Configuration:  
      - Prompt: "You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc."  
      - Input Text: `{{ $json.data }}` (markdown content from Bright Data)  
      - Message Role: "You are a markdown expert"  
      - Model: Google Gemini Flash Exp (configured via linked AI model node)  
    - Inputs: From "Perform Bright Data Web Request"  
    - Outputs: To "Initiate a Webhook Notification for Markdown to Textual Data Extraction" and "Structured Data Extractor"  
    - Edge Cases:  
      - AI model errors or timeouts  
      - Unexpected markdown formats causing parsing errors  

  - **Initiate a Webhook Notification for Markdown to Textual Data Extraction**  
    - Type: HTTP Request  
    - Role: Sends the cleaned textual data to an external webhook endpoint for monitoring or integration.  
    - Configuration:  
      - Method: POST  
      - URL: `https://webhook.site/3c36d7d1-de1b-4171-9fd3-643ea2e4dd76` (example endpoint)  
      - Body Parameter: `content` set to `{{ $json.text }}` (plain text output)  
    - Inputs: From "Markdown to Textual Data Extractor"  
    - Outputs: None  
    - Edge Cases:  
      - Webhook endpoint unavailability or errors  
      - Network timeouts  

#### 2.4 Structured Data Extraction

- **Overview:**  
  Parses the plain text into structured JSON objects representing Google Trends topics and their descriptions, suitable for AI summarization.

- **Nodes Involved:**  
  - Structured Data Extractor (Langchain Information Extractor)  
  - Google Gemini Chat Model for Structured Data Extract (AI Model Node)  
  - Create a binary data (Function)  

- **Node Details:**

  - **Structured Data Extractor**  
    - Type: Langchain Information Extractor  
    - Role: Extracts structured JSON data from textual input using a defined schema.  
    - Configuration:  
      - Input Text: `Extract the Google Trend Data in JSON. Here's the content: {{ $json.text }}`  
      - Schema Type: Manual  
      - Input Schema: JSON schema defining an array of objects with properties:  
        - `topics` (string)  
        - `desc` (string)  
      - Model: Google Gemini Flash Exp (via linked AI model node)  
    - Inputs: From "Markdown to Textual Data Extractor"  
    - Outputs: To "Create a binary data" and "Summarize Google Trends"  
    - Edge Cases:  
      - AI extraction errors or schema mismatches  
      - Missing or malformed input text  

  - **Google Gemini Chat Model for Structured Data Extract**  
    - Type: Langchain AI Model Node  
    - Role: Provides the AI model interface for the Structured Data Extractor node.  
    - Configuration:  
      - Model Name: `models/gemini-2.0-flash-exp`  
      - Credentials: Google Gemini (PaLM) API account  
    - Inputs: Connected internally to "Structured Data Extractor"  
    - Outputs: To "Structured Data Extractor"  
    - Edge Cases: API key invalid or quota exceeded  

  - **Create a binary data**  
    - Type: Function Node  
    - Role: Converts the structured JSON data into a base64-encoded binary format for file writing.  
    - Configuration:  
      - JavaScript code encodes the JSON stringified data into base64 under binary property `data`.  
    - Inputs: From "Structured Data Extractor"  
    - Outputs: To "Write the file to disk"  
    - Edge Cases: Data encoding errors (unlikely)  

#### 2.5 AI Summarization

- **Overview:**  
  Generates a concise, human-readable summary of the structured Google Trends data using an advanced summarization chain powered by Google Gemini AI.

- **Nodes Involved:**  
  - Summarize Google Trends (Langchain Chain Summarization)  
  - Google Gemini Chat Model for Summarization (AI Model Node)  

- **Node Details:**

  - **Summarize Google Trends**  
    - Type: Langchain Chain Summarization  
    - Role: Processes the structured data to produce a summarized text output.  
    - Configuration:  
      - Chunking Mode: Advanced (handles large inputs by chunking)  
      - Model: Google Gemini Flash Exp (via linked AI model node)  
    - Inputs: From "Structured Data Extractor"  
    - Outputs: To "Initiate a Webhook Notification for Summarization", "Send Summary to Gmail"  
    - Edge Cases: AI summarization errors, rate limits, or incomplete data  

  - **Google Gemini Chat Model for Summarization**  
    - Type: Langchain AI Model Node  
    - Role: Provides the AI model interface for the Summarize Google Trends node.  
    - Configuration:  
      - Model Name: `models/gemini-2.0-flash-exp`  
      - Credentials: Google Gemini (PaLM) API account  
    - Inputs: Connected internally to "Summarize Google Trends"  
    - Outputs: To "Summarize Google Trends"  
    - Edge Cases: API failures or quota issues  

#### 2.6 Distribution and Persistence

- **Overview:**  
  Sends the AI-generated summary via email, triggers webhook notifications for external integrations, and saves the structured data to disk for archival purposes.

- **Nodes Involved:**  
  - Initiate a Webhook Notification for Summarization (HTTP Request)  
  - Send Summary to Gmail (Gmail Node)  
  - Write the file to disk (Read/Write File)  

- **Node Details:**

  - **Initiate a Webhook Notification for Summarization**  
    - Type: HTTP Request  
    - Role: Sends the summarized text to an external webhook endpoint for integration with other tools (e.g., Slack, Notion).  
    - Configuration:  
      - Method: POST  
      - URL: `https://webhook.site/3c36d7d1-de1b-4171-9fd3-643ea2e4dd76` (example endpoint)  
      - Body Parameter: `content` set to `{{ $json.response.text }}` (summary text)  
    - Inputs: From "Summarize Google Trends"  
    - Outputs: None  
    - Edge Cases: Webhook endpoint errors or network issues  

  - **Send Summary to Gmail**  
    - Type: Gmail Node  
    - Role: Sends the summary via email to a designated recipient.  
    - Configuration:  
      - Recipient: `ranjancse@gmail.com` (configurable)  
      - Subject: `"Google Trends Summary"` (static, can be customized)  
      - Message Body: `{{ $json.response.text }}` (summary text)  
      - Credentials: Gmail OAuth2 account configured  
    - Inputs: From "Summarize Google Trends"  
    - Outputs: None  
    - Edge Cases: Authentication errors, email sending failures  

  - **Write the file to disk**  
    - Type: Read/Write File  
    - Role: Saves the structured JSON data to a local file for archiving or auditing.  
    - Configuration:  
      - Operation: Write  
      - File Name: `d:\google-trends.json` (Windows path, can be customized)  
    - Inputs: From "Create a binary data"  
    - Outputs: None  
    - Edge Cases: File system permission errors, path invalidity  

---

### 3. Summary Table

| Node Name                                       | Node Type                          | Functional Role                               | Input Node(s)                     | Output Node(s)                                             | Sticky Note                                                                                                                               |
|------------------------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                   | Manual Trigger                   | Workflow entry point                           | None                             | Set URL and Bright Data Zone                                | This workflow deals with the structured data extraction by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Information Extraction, Summarization Chain are being used to demonstrate the usage of the N8N AI capabilities. Please make sure to set the web URL of your interest within the "Set URL and Bright Data Zone" node and update the Webhook Notification URL. |
| Set URL and Bright Data Zone                    | Set Node                        | Sets URL and Bright Data zone for scraping    | When clicking ‘Test workflow’    | Perform Bright Data Web Request                             |                                                                                                                                           |
| Perform Bright Data Web Request                  | HTTP Request                    | Calls Bright Data API to scrape Google Trends | Set URL and Bright Data Zone     | Markdown to Textual Data Extractor                          |                                                                                                                                           |
| Markdown to Textual Data Extractor               | Langchain Chain LLM             | Converts markdown to plain text                | Perform Bright Data Web Request  | Initiate a Webhook Notification for Markdown to Textual Data Extraction, Structured Data Extractor | LLM Usages: Google Gemini Flash Exp model is used. Basic LLM Chain Data Extractor. Information Extraction for structured data extraction. Summarization Chain for summary. |
| Initiate a Webhook Notification for Markdown to Textual Data Extraction | HTTP Request                    | Sends plain text data to webhook               | Markdown to Textual Data Extractor | None                                                       |                                                                                                                                           |
| Structured Data Extractor                        | Langchain Information Extractor | Extracts structured JSON from plain text      | Markdown to Textual Data Extractor | Create a binary data, Summarize Google Trends              |                                                                                                                                           |
| Create a binary data                            | Function                       | Converts JSON to base64 binary for file save  | Structured Data Extractor        | Write the file to disk                                      |                                                                                                                                           |
| Write the file to disk                          | Read/Write File                | Saves structured data to disk                   | Create a binary data             | None                                                       |                                                                                                                                           |
| Summarize Google Trends                         | Langchain Chain Summarization  | Summarizes structured trend data               | Structured Data Extractor        | Initiate a Webhook Notification for Summarization, Send Summary to Gmail |                                                                                                                                           |
| Initiate a Webhook Notification for Summarization | HTTP Request                    | Sends summary to webhook                        | Summarize Google Trends          | None                                                       |                                                                                                                                           |
| Send Summary to Gmail                           | Gmail                         | Sends summary email                             | Summarize Google Trends          | None                                                       |                                                                                                                                           |
| Google Gemini Chat Model for Data Extract      | Langchain AI Model Node        | AI model interface for markdown to text node  | Linked internally to Markdown to Textual Data Extractor | Markdown to Textual Data Extractor                          |                                                                                                                                           |
| Google Gemini Chat Model for Structured Data Extract | Langchain AI Model Node        | AI model interface for structured data extractor | Linked internally to Structured Data Extractor | Structured Data Extractor                                  |                                                                                                                                           |
| Google Gemini Chat Model for Summarization     | Langchain AI Model Node        | AI model interface for summarization node      | Linked internally to Summarize Google Trends | Summarize Google Trends                                    |                                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node "Set URL and Bright Data Zone"**  
   - Type: Set  
   - Parameters:  
     - `url`: `"https://trends.google.com/trends/explore?gprop=youtube&hl=en-US"` (or your target Google Trends URL)  
     - `zone`: `"web_unlocker1"` (your Bright Data Web Unlocker zone name)  
   - Connect output from Manual Trigger node.

3. **Create HTTP Request Node "Perform Bright Data Web Request"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic Header Auth with Bearer token (configure credentials with your Bright Data token)  
   - Body Parameters (form-data or JSON):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}?product=unlocker&method=api`  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Connect input from "Set URL and Bright Data Zone".

4. **Create Langchain Chain LLM Node "Markdown to Textual Data Extractor"**  
   - Type: Langchain Chain LLM  
   - Prompt:  
     ```
     You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.

     {{ $json.data }}
     ```  
   - Message Role: "You are a markdown expert"  
   - Model: Link to Google Gemini Chat Model node (to be created next)  
   - Connect input from "Perform Bright Data Web Request".

5. **Create Langchain AI Model Node "Google Gemini Chat Model for Data Extract"**  
   - Type: Langchain AI Model Node  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Google Gemini (PaLM) API key or equivalent  
   - Connect as AI model for "Markdown to Textual Data Extractor".

6. **Create HTTP Request Node "Initiate a Webhook Notification for Markdown to Textual Data Extraction"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Your webhook endpoint (e.g., `https://webhook.site/your-unique-url`)  
   - Body Parameter: `content` set to `={{ $json.text }}`  
   - Connect input from "Markdown to Textual Data Extractor".

7. **Create Langchain Information Extractor Node "Structured Data Extractor"**  
   - Type: Langchain Information Extractor  
   - Input Text:  
     ```
     Extract the Google Trend Data in JSON.

     Here's the content:

     {{ $json.text }}
     ```  
   - Schema Type: Manual  
   - Input Schema:  
     ```json
     {
       "type": "array",
       "properties": {
         "topics": { "type": "string" },
         "desc": { "type": "string" }
       }
     }
     ```  
   - Model: Link to Google Gemini Chat Model for Structured Data Extract node  
   - Connect input from "Markdown to Textual Data Extractor".

8. **Create Langchain AI Model Node "Google Gemini Chat Model for Structured Data Extract"**  
   - Type: Langchain AI Model Node  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Google Gemini (PaLM) API key  
   - Connect as AI model for "Structured Data Extractor".

9. **Create Function Node "Create a binary data"**  
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
   - Connect input from "Structured Data Extractor".

10. **Create Read/Write File Node "Write the file to disk"**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: `d:\google-trends.json` (adjust path as needed)  
    - Connect input from "Create a binary data".

11. **Create Langchain Chain Summarization Node "Summarize Google Trends"**  
    - Type: Langchain Chain Summarization  
    - Chunking Mode: Advanced  
    - Model: Link to Google Gemini Chat Model for Summarization node  
    - Connect input from "Structured Data Extractor".

12. **Create Langchain AI Model Node "Google Gemini Chat Model for Summarization"**  
    - Type: Langchain AI Model Node  
    - Model Name: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini (PaLM) API key  
    - Connect as AI model for "Summarize Google Trends".

13. **Create HTTP Request Node "Initiate a Webhook Notification for Summarization"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Your webhook endpoint (e.g., `https://webhook.site/your-unique-url`)  
    - Body Parameter: `content` set to `={{ $json.response.text }}`  
    - Connect input from "Summarize Google Trends".

14. **Create Gmail Node "Send Summary to Gmail"**  
    - Type: Gmail  
    - Send To: Your recipient email (e.g., `ranjancse@gmail.com`)  
    - Subject: `"Google Trends Summary"` (customize as needed)  
    - Message: `={{ $json.response.text }}`  
    - Credentials: Gmail OAuth2 credentials configured  
    - Connect input from "Summarize Google Trends".

15. **Connect all nodes as per the flow:**  
    - Manual Trigger → Set URL and Bright Data Zone → Perform Bright Data Web Request → Markdown to Textual Data Extractor →  
      - Initiate Webhook Notification for Markdown to Textual Data Extraction  
      - Structured Data Extractor →  
        - Create a binary data → Write the file to disk  
        - Summarize Google Trends →  
          - Initiate Webhook Notification for Summarization  
          - Send Summary to Gmail

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses Bright Data Web Unlocker API for scraping Google Trends data in markdown format.                                                                                                                          | https://brightdata.com/                                                                                  |
| Google Gemini Flash Exp model is used for all AI processing steps: markdown conversion, structured data extraction, and summarization.                                                                                       | Google Gemini API / Vertex AI                                                                            |
| Ensure to configure the Bright Data Header Authentication credential with your Bearer token for Web Unlocker API access.                                                                                                    | Credential setup in n8n with Generic Auth Type: Header Authentication                                    |
| Customize the webhook URLs to your own endpoints for integration with Slack, Notion, Zapier, or other automation tools.                                                                                                      | Webhook.site URLs in example nodes                                                                       |
| Gmail node requires OAuth2 credentials; configure with your Google account for sending emails.                                                                                                                               | Gmail OAuth2 credential setup                                                                            |
| To extend file storage beyond local disk, integrate with S3 or cloud storage nodes.                                                                                                                                           | n8n integrations                                                                                         |
| For dynamic email subjects or recipients, modify the Gmail node parameters with expressions like `Weekly Google Trends Summary – {{ $now.format('YYYY-MM-DD') }}` or multiple emails separated by commas.                      | n8n expression syntax                                                                                    |
| The workflow can be adapted to read input URLs from Google Sheets, Airtable, or other sources by replacing the manual trigger and Set node with appropriate input nodes.                                                      | n8n Google Sheets, Airtable nodes                                                                        |
| The AI prompt templates can be customized to extract different insights such as content ideas, trend shifts, or keyword popularity changes.                                                                                  | Modify prompt text in Langchain nodes                                                                    |

---

This comprehensive documentation enables understanding, reproduction, and modification of the "Google Trend Data Extract & Summarization with Bright Data & Google Gemini" workflow, covering all nodes, configurations, and integration points.