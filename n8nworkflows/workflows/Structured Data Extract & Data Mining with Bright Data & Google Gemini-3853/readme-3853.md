Structured Data Extract & Data Mining with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/structured-data-extract---data-mining-with-bright-data---google-gemini-3853


# Structured Data Extract & Data Mining with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow, **Structured Data Extract & Data Mining with Bright Data & Google Gemini**, automates the extraction, analysis, and distribution of structured insights from semi-structured web content such as markdown or HTML. It is designed for professionals like content analysts, SEO researchers, AI developers, and marketers who require scalable, AI-driven data mining and trend detection.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Manual trigger and setting the target URL and Bright Data zone for web scraping.
- **1.2 Data Extraction via Bright Data Web Unlocker**: Sending a request to Bright Data’s API to retrieve raw markdown content from the specified URL.
- **1.3 Markdown to Text Conversion**: Using an AI-powered LangChain node to convert markdown content into clean textual data.
- **1.4 AI-Driven Data Analysis**: Leveraging Google Gemini models and LangChain Information Extractor nodes to perform:
  - Topic extraction with structured output.
  - Trend analysis clustered by location and category with structured output.
- **1.5 Notification and Persistence**: Sending structured AI insights to external systems via webhook notifications and saving JSON outputs to disk files.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

**Overview:**  
This block starts the workflow manually and sets the key input parameters: the URL to scrape and the Bright Data Web Unlocker zone.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Set URL and Bright Data Zone (Set node)  
- Sticky Note (Instructional)  
- Sticky Note1 (Instructional)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually for testing or execution.  
  - Configuration: No parameters; simply triggers downstream nodes.  
  - Input: None  
  - Output: To “Set URL and Bright Data Zone” node  
  - Edge cases: None typical; manual trigger requires user action.

- **Set URL and Bright Data Zone**  
  - Type: Set node  
  - Role: Defines workflow variables `url` and `zone` for the target website and Bright Data zone name.  
  - Configuration:  
    - `url` set to `"https://www.bbc.com/news/world"` (default example)  
    - `zone` set to `"web_unlocker1"` (example Bright Data zone)  
  - Input: From manual trigger  
  - Output: To “Perform Bright Data Web Request”  
  - Edge cases: URL or zone misconfiguration leads to failed scraping requests.

- **Sticky Note** and **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Provide important instructions and context about workflow setup and AI model usage.  
  - Configuration: Static content with setup notes and model info.  
  - Input/Output: None (informational only)  
  - Edge cases: None

---

#### 1.2 Data Extraction via Bright Data Web Unlocker

**Overview:**  
This block performs the actual web scraping by sending a POST request to Bright Data’s Web Unlocker API to retrieve the raw markdown content of the target URL.

**Nodes Involved:**  
- Perform Bright Data Web Request (HTTP Request)

**Node Details:**  

- **Perform Bright Data Web Request**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Bright Data API to unlock and fetch webpage content in markdown format.  
  - Configuration:  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body parameters:  
      - `zone`: from previous node (`zone`)  
      - `url`: from previous node (`url`) appended with `?product=unlocker&method=api`  
      - `format`: `"raw"`  
      - `data_format`: `"markdown"`  
    - Authentication: Header Auth with Bearer token (configured in credentials)  
  - Input: From “Set URL and Bright Data Zone” node  
  - Output: JSON response containing raw markdown content under `data` property  
  - Edge cases:  
    - Authentication failure if token invalid or expired  
    - API rate limits or network timeouts  
    - Invalid zone or URL causing empty or error responses

---

#### 1.3 Markdown to Text Conversion

**Overview:**  
This block converts the raw markdown content into clean textual data using an AI-powered LangChain node specialized in markdown parsing.

**Nodes Involved:**  
- Markdown to Textual Data Extractor (LangChain LLM Chain)  
- Google Gemini Chat Model for Data Extract (LangChain Google Gemini LM Chat)

**Node Details:**  

- **Markdown to Textual Data Extractor**  
  - Type: LangChain Chain LLM node  
  - Role: Parses markdown content into plaintext, removing links, scripts, CSS, and other non-text elements.  
  - Configuration:  
    - Prompt instructs to analyze markdown and output textual data only, no additional commentary or formatting.  
    - Message context: “You are a markdown expert”  
    - Input text: `{{ $json.data }}` from Bright Data response  
  - Input: From “Perform Bright Data Web Request” node  
  - Output: Clean plaintext content for further AI processing  
  - Edge cases:  
    - Expression failures if input markdown is missing or malformed  
    - AI model errors or timeouts

- **Google Gemini Chat Model for Data Extract**  
  - Type: LangChain Google Gemini LM Chat node  
  - Role: Provides AI language model capabilities to assist the markdown to text extraction chain.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key configured under “Google Gemini(PaLM) Api account”  
  - Input: Feeds into “Markdown to Textual Data Extractor” as AI language model  
  - Output: AI-processed textual data  
  - Edge cases: API key invalid, quota exceeded, or network issues

---

#### 1.4 AI-Driven Data Analysis

**Overview:**  
This block performs advanced AI analysis on the cleaned textual data to extract topics and identify trends clustered by location and category, producing structured JSON outputs.

**Nodes Involved:**  
- Topic Extractor with the structured response (Information Extractor)  
- Google Gemini Chat Model for Sentiment Analyzer (LangChain Google Gemini LM Chat)  
- Trends by location and category with the structured response (Information Extractor)  
- Google Gemini Chat Model (LangChain Google Gemini LM Chat)

**Node Details:**  

- **Topic Extractor with the structured response**  
  - Type: LangChain Information Extractor  
  - Role: Extracts key topics, confidence scores, summaries, and keywords from the cleaned text.  
  - Configuration:  
    - Input text: Uses raw markdown data from Bright Data request (`{{ $('Perform Bright Data Web Request').item.json.data }}`)  
    - System prompt: “You are an expert data analyst.”  
    - Schema: JSON schema defining an array of topic objects with properties: topic (string), score (0-1 number), summary (string), keywords (string array)  
  - Input: From “Markdown to Textual Data Extractor” node  
  - Output: Structured JSON array of topics  
  - Edge cases: Schema validation errors if AI output does not conform; API failures

- **Google Gemini Chat Model for Sentiment Analyzer**  
  - Type: LangChain Google Gemini LM Chat node  
  - Role: Supports the Topic Extractor node by providing AI language model capabilities for sentiment and topic analysis.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key  
  - Input: Feeds into “Topic Extractor with the structured response”  
  - Output: AI-enhanced topic extraction  
  - Edge cases: API key or network issues

- **Trends by location and category with the structured response**  
  - Type: LangChain Information Extractor  
  - Role: Analyzes content to cluster emerging trends by geographic location and category, outputting structured JSON.  
  - Configuration:  
    - Input text: Raw markdown data from Bright Data request  
    - System prompt: “You are an expert data analyst.”  
    - Schema: JSON schema defining an array of objects with location, category, and an array of trends (each with trend label, score, summary, mentions)  
  - Input: From “Markdown to Textual Data Extractor” node  
  - Output: Structured JSON array of trends clustered by location and category  
  - Edge cases: Schema validation errors; AI response inconsistencies

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini LM Chat node  
  - Role: Provides AI language model support for the trends extraction node.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key  
  - Input: Feeds into “Trends by location and category with the structured response”  
  - Output: AI-enhanced trend extraction  
  - Edge cases: API or network failures

---

#### 1.5 Notification and Persistence

**Overview:**  
This block sends the AI-extracted insights to external endpoints via webhooks and saves the structured JSON data to local disk files for persistence and audit.

**Nodes Involved:**  
- Initiate a Webhook Notification for Markdown to Textual Data Extraction (HTTP Request)  
- Initiate a Webhook Notification for AI Sentiment Analyzer (HTTP Request)  
- Initiate a Webhook Notification for trends by location and category (HTTP Request)  
- Create a binary file for topics (Function)  
- Write the topics file to disk (Read/Write File)  
- Create a binary data for tends (Function)  
- Write the trends file to disk (Read/Write File)

**Node Details:**  

- **Initiate a Webhook Notification for Markdown to Textual Data Extraction**  
  - Type: HTTP Request  
  - Role: Sends the cleaned textual content via POST to a configured webhook URL.  
  - Configuration:  
    - URL: Example webhook URL (`https://webhook.site/...`)  
    - Body parameter: `content` set to `{{ $json.text }}` (clean text)  
    - Sends JSON body  
  - Input: From “Markdown to Textual Data Extractor” node  
  - Output: None  
  - Edge cases: Webhook endpoint unreachable or rejects request

- **Initiate a Webhook Notification for AI Sentiment Analyzer**  
  - Type: HTTP Request  
  - Role: Sends AI topic extraction summary to webhook endpoint.  
  - Configuration:  
    - URL: Same example webhook URL  
    - Body parameter: `summary` set to `{{ $json.output }}` from topic extractor  
  - Input: From “Topic Extractor with the structured response” node  
  - Output: None  
  - Edge cases: Webhook failures

- **Initiate a Webhook Notification for trends by location and category**  
  - Type: HTTP Request  
  - Role: Sends structured trend analysis output to webhook endpoint.  
  - Configuration:  
    - URL: Same example webhook URL  
    - Body parameter: `summary` set to `{{ $json.output }}` from trends extractor  
  - Input: From “Trends by location and category with the structured response” node  
  - Output: None  
  - Edge cases: Webhook failures

- **Create a binary file for topics**  
  - Type: Function  
  - Role: Converts JSON topic data into a base64-encoded binary format for file writing.  
  - Configuration:  
    - Converts JSON to string with indentation, encodes to base64, assigns to binary property `data`  
  - Input: From “Topic Extractor with the structured response”  
  - Output: Binary data for file write  
  - Edge cases: JSON stringify errors if input malformed

- **Write the topics file to disk**  
  - Type: Read/Write File  
  - Role: Writes the binary topic data to local disk file `d:\topics.json`.  
  - Configuration:  
    - Operation: Write  
    - File path: `d:\topics.json`  
  - Input: From “Create a binary file for topics”  
  - Output: None  
  - Edge cases: File system permissions, disk space, path validity

- **Create a binary data for tends**  
  - Type: Function  
  - Role: Converts JSON trend data into base64-encoded binary for file writing.  
  - Configuration: Same as “Create a binary file for topics” but for trends data  
  - Input: From “Trends by location and category with the structured response”  
  - Output: Binary data for file write  
  - Edge cases: Same as above

- **Write the trends file to disk**  
  - Type: Read/Write File  
  - Role: Writes the binary trend data to local disk file `d:\trends.json`.  
  - Configuration:  
    - Operation: Write  
    - File path: `d:\trends.json`  
  - Input: From “Create a binary data for tends”  
  - Output: None  
  - Edge cases: Same as above

---

### 3. Summary Table

| Node Name                                         | Node Type                          | Functional Role                                      | Input Node(s)                          | Output Node(s)                                               | Sticky Note                                                                                                           |
|--------------------------------------------------|----------------------------------|-----------------------------------------------------|--------------------------------------|--------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                     | Manual Trigger                   | Manual start of workflow                             | None                                 | Set URL and Bright Data Zone                                 | This workflow deals with the structured data extraction by utilizing Bright Data Web Unlocker Product.                 |
| Set URL and Bright Data Zone                      | Set                              | Defines URL and Bright Data zone parameters         | When clicking ‘Test workflow’        | Perform Bright Data Web Request                              | Please make sure to set the web URL of your interest within the "Set URL and Bright Data Zone" node and update webhook.|
| Perform Bright Data Web Request                    | HTTP Request                    | Fetches raw markdown content from Bright Data API   | Set URL and Bright Data Zone         | Markdown to Textual Data Extractor                           |                                                                                                                       |
| Markdown to Textual Data Extractor                 | LangChain Chain LLM             | Converts markdown to clean textual data              | Perform Bright Data Web Request       | Topic Extractor with the structured response, Initiate a Webhook Notification for Markdown to Textual Data Extraction, Trends by location and category with the structured response | LLM Usages: Google Gemini Flash Exp model is being used. Basic LLM Chain Data Extractor. Information Extraction used.  |
| Google Gemini Chat Model for Data Extract          | LangChain Google Gemini LM Chat | AI model for markdown to text extraction             | None (used by Markdown to Textual Data Extractor) | Markdown to Textual Data Extractor                           |                                                                                                                       |
| Topic Extractor with the structured response       | LangChain Information Extractor | Extracts topics with structured output                | Markdown to Textual Data Extractor   | Initiate a Webhook Notification for AI Sentiment Analyzer, Create a binary file for topics |                                                                                                                       |
| Google Gemini Chat Model for Sentiment Analyzer    | LangChain Google Gemini LM Chat | AI model supporting topic extraction and sentiment   | None (used by Topic Extractor)       | Topic Extractor with the structured response                |                                                                                                                       |
| Initiate a Webhook Notification for Markdown to Textual Data Extraction | HTTP Request                    | Sends clean text to webhook endpoint                  | Markdown to Textual Data Extractor   | None                                                        |                                                                                                                       |
| Initiate a Webhook Notification for AI Sentiment Analyzer | HTTP Request                    | Sends topic extraction summary to webhook             | Topic Extractor with the structured response | None                                                        |                                                                                                                       |
| Trends by location and category with the structured response | LangChain Information Extractor | Extracts trends clustered by location and category    | Markdown to Textual Data Extractor   | Initiate a Webhook Notification for trends by location and category, Create a binary data for tends |                                                                                                                       |
| Google Gemini Chat Model                            | LangChain Google Gemini LM Chat | AI model supporting trends extraction                  | None (used by Trends by location and category) | Trends by location and category with the structured response |                                                                                                                       |
| Initiate a Webhook Notification for trends by location and category | HTTP Request                    | Sends trend analysis output to webhook                 | Trends by location and category with the structured response | None                                                        |                                                                                                                       |
| Create a binary file for topics                     | Function                        | Converts topic JSON to base64 binary for file writing | Topic Extractor with the structured response | Write the topics file to disk                               |                                                                                                                       |
| Write the topics file to disk                        | Read/Write File                 | Saves topic data JSON file to disk                      | Create a binary file for topics      | None                                                        |                                                                                                                       |
| Create a binary data for tends                      | Function                        | Converts trend JSON to base64 binary for file writing | Trends by location and category with the structured response | Write the trends file to disk                               |                                                                                                                       |
| Write the trends file to disk                        | Read/Write File                 | Saves trend data JSON file to disk                      | Create a binary data for tends       | None                                                        |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node "Set URL and Bright Data Zone"**  
   - Type: Set  
   - Add two string fields:  
     - `url`: Set default to `"https://www.bbc.com/news/world"` (replace with target URL)  
     - `zone`: Set default to `"web_unlocker1"` (replace with your Bright Data zone name)  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node "Perform Bright Data Web Request"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic HTTP Header Auth with Bearer token (configure credentials with your Bright Data Web Unlocker token)  
   - Body parameters (form or JSON):  
     - `zone`: Expression `{{$json["zone"]}}`  
     - `url`: Expression `{{$json["url"] + "?product=unlocker&method=api"}}`  
     - `format`: `"raw"`  
     - `data_format`: `"markdown"`  
   - Connect output of "Set URL and Bright Data Zone" to this node.

4. **Create LangChain Chain LLM Node "Markdown to Textual Data Extractor"**  
   - Type: LangChain Chain LLM  
   - Prompt:  
     ```
     You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.

     {{ $json.data }}
     ```  
   - Add system message: “You are a markdown expert”  
   - Connect output of "Perform Bright Data Web Request" to this node.  
   - Under AI Language Model, create LangChain Google Gemini LM Chat node (next step) and link it here.

5. **Create LangChain Google Gemini LM Chat Node "Google Gemini Chat Model for Data Extract"**  
   - Type: LangChain Google Gemini LM Chat  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credentials: Configure with Google Gemini API key (Google Palm API)  
   - Connect this node as AI Language Model input to "Markdown to Textual Data Extractor".

6. **Create LangChain Information Extractor Node "Topic Extractor with the structured response"**  
   - Type: LangChain Information Extractor  
   - Text input: Expression referencing raw markdown content:  
     `{{ $('Perform Bright Data Web Request').item.json.data }}`  
   - System prompt: “You are an expert data analyst.”  
   - Schema: Use the provided JSON schema defining topics with topic, score, summary, keywords (copy from workflow)  
   - Connect output of "Markdown to Textual Data Extractor" to this node.  
   - Under AI Language Model, create LangChain Google Gemini LM Chat node (next step) and link it here.

7. **Create LangChain Google Gemini LM Chat Node "Google Gemini Chat Model for Sentiment Analyzer"**  
   - Same configuration as step 5.  
   - Connect as AI Language Model input to "Topic Extractor with the structured response".

8. **Create LangChain Information Extractor Node "Trends by location and category with the structured response"**  
   - Type: LangChain Information Extractor  
   - Text input: Expression referencing raw markdown content:  
     `{{ $('Perform Bright Data Web Request').item.json.data }}`  
   - System prompt: “You are an expert data analyst.”  
   - Schema: Use the provided JSON schema defining trends clustered by location and category (copy from workflow)  
   - Connect output of "Markdown to Textual Data Extractor" to this node.  
   - Under AI Language Model, create LangChain Google Gemini LM Chat node (next step) and link it here.

9. **Create LangChain Google Gemini LM Chat Node "Google Gemini Chat Model"**  
   - Same configuration as step 5.  
   - Connect as AI Language Model input to "Trends by location and category with the structured response".

10. **Create HTTP Request Node "Initiate a Webhook Notification for Markdown to Textual Data Extraction"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Set to your webhook endpoint (example: `https://webhook.site/...`)  
    - Body parameter: `content` set to `{{ $json.text }}`  
    - Connect output of "Markdown to Textual Data Extractor" to this node.

11. **Create HTTP Request Node "Initiate a Webhook Notification for AI Sentiment Analyzer"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Same webhook endpoint as above or different as needed  
    - Body parameter: `summary` set to `{{ $json.output }}` from topic extractor  
    - Connect output of "Topic Extractor with the structured response" to this node.

12. **Create HTTP Request Node "Initiate a Webhook Notification for trends by location and category"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Same webhook endpoint or as configured  
    - Body parameter: `summary` set to `{{ $json.output }}` from trends extractor  
    - Connect output of "Trends by location and category with the structured response" to this node.

13. **Create Function Node "Create a binary file for topics"**  
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
    - Connect output of "Topic Extractor with the structured response" to this node.

14. **Create Read/Write File Node "Write the topics file to disk"**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: `d:\topics.json` (adjust path as needed)  
    - Connect output of "Create a binary file for topics" to this node.

15. **Create Function Node "Create a binary data for tends"**  
    - Type: Function  
    - Code: Same as step 13 but connected to trends data.  
    - Connect output of "Trends by location and category with the structured response" to this node.

16. **Create Read/Write File Node "Write the trends file to disk"**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: `d:\trends.json` (adjust path as needed)  
    - Connect output of "Create a binary data for tends" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow deals with the structured data extraction by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Information Extraction, are used to demonstrate n8n AI capabilities. Please set the web URL and webhook URL. | Sticky Note on workflow start                                                                       |
| Google Gemini Flash Exp model is used for AI processing including data extraction and sentiment analysis.                                                                                                                            | Sticky Note1                                                                                        |
| Setup instructions for Bright Data Web Unlocker: Create a Web Unlocker zone, configure Header Auth credentials with Bearer token.                                                                                                   | Workflow description section                                                                        |
| Google Gemini API key or access via Vertex AI or proxy is required for AI nodes.                                                                                                                                                      | Workflow description section                                                                        |
| Webhook URLs can be customized to send notifications to Slack, internal APIs, Zapier, or Make for automation.                                                                                                                        | Workflow description section                                                                        |
| Output files are saved locally to disk but can be adapted to remote storage like FTP, S3, or GCS.                                                                                                                                     | Workflow description section                                                                        |
| Bright Data documentation: https://brightdata.com/                                                                                                                                                                                   | External resource                                                                                   |
| Google Gemini (PaLM) API documentation: https://developers.generativeai.google                                                                                                                                                | External resource                                                                                   |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node configurations, data flow, and integration points, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.