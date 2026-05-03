Extract and Analyze Web Data with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/extract-and-analyze-web-data-with-bright-data---google-gemini-8714


# Extract and Analyze Web Data with Bright Data & Google Gemini

### 1. Workflow Overview

This n8n workflow is designed for structured data extraction and analysis from web content using Bright Data's Web Unlocker API combined with Google Gemini AI models. It automates the process of fetching web data, cleansing and converting it into textual format, performing topic extraction, sentiment analysis, and clustering emerging trends by location and category. The workflow also outputs structured JSON files and sends notifications via webhooks for real-time monitoring.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Initialization:** Manual trigger entry point with setup of target URL and Bright Data zone.
- **1.2 Web Data Extraction:** Fetching raw web content in markdown format using Bright Data API.
- **1.3 Markdown Processing and Text Extraction:** Converting markdown content to plain textual data using Google Gemini LLM.
- **1.4 Topic Extraction:** Analyzing textual data to extract topics with structured metadata.
- **1.5 Trend Clustering by Location and Category:** Analyzing data to identify emerging trends clustered by geography and domain.
- **1.6 Sentiment Analysis:** Performing sentiment analysis on extracted topics.
- **1.7 Output and Notification:** Writing structured JSON files to disk and sending webhook notifications with results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
Starts the workflow manually and sets the target web URL and Bright Data zone parameters.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set URL and Bright Data Zone  
- Sticky Note (instructional)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution.  
  - Config: No parameters; triggers downstream flow.  
  - Inputs: None  
  - Outputs: To "Set URL and Bright Data Zone"  
  - Edge cases: No input; careful not to trigger unintentionally.

- **Set URL and Bright Data Zone**  
  - Type: Set Node  
  - Role: Defines parameters for the target URL (`https://www.bbc.com/news/world`) and Bright Data zone (`web_unlocker1`).  
  - Config: Assigns static strings for `url` and `zone`.  
  - Inputs: From manual trigger  
  - Outputs: To "Perform Bright Data Web Request"  
  - Edge cases: URL or zone misconfiguration will cause downstream API failures.

- **Sticky Note**  
  - Content: Workflow overview and setup instructions.  
  - Role: Documentation within the editor.  
  - No inputs/outputs.

---

#### 1.2 Web Data Extraction

**Overview:**  
Makes a POST request to Bright Data API to retrieve the raw markdown content of the specified web page via the Web Unlocker product.

**Nodes Involved:**  
- Perform Bright Data Web Request

**Node Details:**

- **Perform Bright Data Web Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Bright Data API endpoint to extract web data.  
  - Config:  
    - URL: https://api.brightdata.com/request  
    - Method: POST  
    - Headers: Uses HTTP Header Authentication credential.  
    - Body Parameters: Passes `zone`, `url` (appended with product and method query), `format` as raw, `data_format` as markdown.  
  - Inputs: From "Set URL and Bright Data Zone"  
  - Outputs: To "Markdown to Textual Data Extractor"  
  - Edge cases: API authentication issues, network timeouts, malformed URLs, API rate limits.

---

#### 1.3 Markdown Processing and Text Extraction

**Overview:**  
Converts markdown data retrieved from Bright Data into clean textual data using Google Gemini LLM via Langchain integration.

**Nodes Involved:**  
- Markdown to Textual Data Extractor  
- Google Gemini Chat Model for Data Extract  
- Sticky Note1 (LLM usage info)  
- Initiate a Webhook Notification for Markdown to Textual Data Extraction

**Node Details:**

- **Markdown to Textual Data Extractor**  
  - Type: Langchain LLM Chain (Custom)  
  - Role: Parses and cleans markdown content to plain text only, removing links, scripts, CSS, or any other formatting.  
  - Config: Prompt instructs the LLM to output only pure textual data from the markdown input.  
  - Inputs: From "Perform Bright Data Web Request" (raw markdown)  
  - Outputs: To:  
    - "Topic Extractor with the structured response"  
    - "Trends by location and category with the structured response"  
    - "Initiate a Webhook Notification for Markdown to Textual Data Extraction"  
  - Edge cases: LLM misinterpretation, malformed markdown, latency.

- **Google Gemini Chat Model for Data Extract**  
  - Type: Langchain Google Gemini LM Chat  
  - Role: Provides the language model backend for the markdown extraction chain node.  
  - Config: Uses Gemini 2.0 Flash Exp model with configured API credentials.  
  - Inputs: AI language model requests from "Markdown to Textual Data Extractor"  
  - Outputs: Back to "Markdown to Textual Data Extractor"  
  - Edge cases: API quota or authentication errors.

- **Initiate a Webhook Notification for Markdown to Textual Data Extraction**  
  - Type: HTTP Request  
  - Role: Sends the cleaned text output to a webhook URL for monitoring/logging.  
  - Config: POST request to specified webhook.site URL with extracted text in the body.  
  - Inputs: From "Markdown to Textual Data Extractor"  
  - Outputs: None  
  - Edge cases: Webhook endpoint down, network errors.

- **Sticky Note1**  
  - Content: Details about Google Gemini Flash Exp usage and LLM chain purpose for data extraction and sentiment analysis.  
  - Role: Documentation.

---

#### 1.4 Topic Extraction

**Overview:**  
Performs topic analysis on the cleaned textual data to identify key topics, confidence scores, summaries, and associated keywords as a structured JSON array.

**Nodes Involved:**  
- Topic Extractor with the structured response  
- Google Gemini Chat Model for Sentiment Analyzer  
- Initiate a Webhook Notification for AI Sentiment Analyzer  
- Create a binary file for topics  
- Write the topics file to disk

**Node Details:**

- **Topic Extractor with the structured response**  
  - Type: Langchain Information Extractor  
  - Role: Analyzes text to extract topic modeling data with schema validation: topic, score, summary, keywords.  
  - Config:  
    - System prompt: "You are an expert data analyst."  
    - Input text: From cleaned text (`$('Perform Bright Data Web Request').item.json.data`)  
    - Schema: JSON schema defining the structure and types of output.  
  - Inputs: From "Markdown to Textual Data Extractor"  
  - Outputs: To:  
    - "Initiate a Webhook Notification for AI Sentiment Analyzer"  
    - "Create a binary file for topics"  
  - Edge cases: Schema validation failure, incomplete topic extraction, LLM errors.

- **Google Gemini Chat Model for Sentiment Analyzer**  
  - Type: Langchain Google Gemini LM Chat  
  - Role: Provides LLM backend for sentiment analysis on topic extraction output.  
  - Config: Gemini 2.0 Flash Exp model with credentials.  
  - Inputs: AI language model requests from "Topic Extractor with the structured response"  
  - Outputs: Back to "Topic Extractor with the structured response"  
  - Edge cases: API errors, quotas.

- **Initiate a Webhook Notification for AI Sentiment Analyzer**  
  - Type: HTTP Request  
  - Role: Sends topic extraction results (summary) to webhook URL for notification.  
  - Config: POST with JSON body containing summary.  
  - Inputs: From "Topic Extractor with the structured response"  
  - Outputs: None  
  - Edge cases: Network or webhook endpoint failure.

- **Create a binary file for topics**  
  - Type: Function  
  - Role: Converts JSON topic extraction result into a base64-encoded binary file for writing.  
  - Config: Uses Buffer to encode JSON string.  
  - Inputs: From "Topic Extractor with the structured response"  
  - Outputs: To "Write the topics file to disk"  
  - Edge cases: JSON stringify errors.

- **Write the topics file to disk**  
  - Type: Read/Write File  
  - Role: Writes the binary topics JSON file to local disk path `d:\topics.json`.  
  - Config: Operation "write" with fixed filename.  
  - Inputs: From "Create a binary file for topics"  
  - Outputs: None  
  - Edge cases: File system permissions, path invalid.

---

#### 1.5 Trend Clustering by Location and Category

**Overview:**  
Clusters emerging trends found in the textual content by geographic location and category, generating structured trend data with scores and mention keywords.

**Nodes Involved:**  
- Trends by location and category with the structured response  
- Google Gemini Chat Model  
- Initiate a Webhook Notification for trends by location and category  
- Create a binary data for tends  
- Write the trends file to disk

**Node Details:**

- **Trends by location and category with the structured response**  
  - Type: Langchain Information Extractor  
  - Role: Analyzes the same raw textual data to cluster trends by location and category with detailed structured output.  
  - Config:  
    - System prompt: "You are an expert data analyst."  
    - Input text: raw markdown content (`$('Perform Bright Data Web Request').item.json.data`)  
    - Schema: JSON schema describing location, category, and array of trends with scores and mentions.  
  - Inputs: From "Markdown to Textual Data Extractor"  
  - Outputs: To:  
    - "Initiate a Webhook Notification for trends by location and category"  
    - "Create a binary data for tends"  
  - Edge cases: Schema mismatch, LLM failures.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini LM Chat  
  - Role: LLM backend for trend clustering analysis.  
  - Config: Gemini 2.0 Flash Exp model with credentials.  
  - Inputs: AI requests from "Trends by location and category with the structured response"  
  - Outputs: Back to the same node  
  - Edge cases: API limits.

- **Initiate a Webhook Notification for trends by location and category**  
  - Type: HTTP Request  
  - Role: Sends trend clustering results to webhook URL.  
  - Config: POST request with summary in body.  
  - Inputs: From "Trends by location and category with the structured response"  
  - Outputs: None  
  - Edge cases: Network webhook failures.

- **Create a binary data for tends**  
  - Type: Function  
  - Role: Encodes JSON trend clustering data into base64 binary for writing.  
  - Config: Buffer encoding of JSON stringify.  
  - Inputs: From "Trends by location and category with the structured response"  
  - Outputs: To "Write the trends file to disk"  
  - Edge cases: JSON stringify issues.

- **Write the trends file to disk**  
  - Type: Read/Write File  
  - Role: Writes trends JSON file to disk path `d:\trends.json`.  
  - Config: Write operation with fixed filename.  
  - Inputs: From "Create a binary data for tends"  
  - Outputs: None  
  - Edge cases: File permissions, invalid path.

---

#### 1.6 Sentiment Analysis

**Overview:**  
Performs sentiment analysis on the extracted topics using Google Gemini LLM to generate sentiment summaries.

**Nodes Involved:**  
- Google Gemini Chat Model for Sentiment Analyzer  
- Topic Extractor with the structured response (input source)  
- Initiate a Webhook Notification for AI Sentiment Analyzer

**Node Details:**

- **Google Gemini Chat Model for Sentiment Analyzer**  
  - See details in block 1.4 above.

- **Topic Extractor with the structured response**  
  - Also acts as input provider for sentiment analysis.

- **Initiate a Webhook Notification for AI Sentiment Analyzer**  
  - See details in block 1.4 above.

---

#### 1.7 Output and Notification

**Overview:**  
Handles saving JSON results to local disk files and notifying external systems via webhooks.

**Nodes Involved:**  
- Create a binary file for topics  
- Write the topics file to disk  
- Create a binary data for tends  
- Write the trends file to disk  
- Initiate Webhook Notification nodes (3 instances)

**Node Details:**

- **Create a binary file for topics** and **Create a binary data for tends**  
  - Types: Function nodes  
  - Role: Encode JSON outputs into base64 binary format for file writing.  
  - Edge cases: JSON encoding failure.

- **Write the topics file to disk** and **Write the trends file to disk**  
  - Types: File Read/Write nodes  
  - Role: Write JSON data to fixed local disk paths.  
  - Edge cases: File system access errors.

- **Initiate Webhook Notification nodes**  
  - Type: HTTP Request nodes  
  - Role: Send extracted text, topics summary, and trends summary to webhook.site endpoints for monitoring.  
  - Edge cases: Network issues, endpoint availability.

---

### 3. Summary Table

| Node Name                                                | Node Type                                    | Functional Role                                  | Input Node(s)                                  | Output Node(s)                                                         | Sticky Note                                                                                                                                                                                         |
|----------------------------------------------------------|----------------------------------------------|-------------------------------------------------|-----------------------------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                            | Manual Trigger                              | Manual start trigger                             | None                                          | Set URL and Bright Data Zone                                           |                                                                                                                                                                                                     |
| Set URL and Bright Data Zone                             | Set Node                                   | Define target URL and Bright Data zone          | When clicking ‘Test workflow’                  | Perform Bright Data Web Request                                        | Please make sure to set the web URL of your interest within this node and update the Webhook Notification URL                                                                                       |
| Perform Bright Data Web Request                          | HTTP Request                               | Fetch raw markdown web content via Bright Data  | Set URL and Bright Data Zone                    | Markdown to Textual Data Extractor                                    |                                                                                                                                                                                                     |
| Markdown to Textual Data Extractor                       | Langchain LLM Chain                        | Convert markdown to clean textual data          | Perform Bright Data Web Request                 | Topic Extractor with the structured response, Trends by location and category with the structured response, Initiate a Webhook Notification for Markdown to Textual Data Extraction | Google Gemini Flash Exp model is used here. Basic LLM Chain Data Extractor. Information Extraction for sentiment analysis.                                                                             |
| Google Gemini Chat Model for Data Extract                | Langchain Google Gemini LM Chat            | LLM backend for markdown text extraction         | Markdown to Textual Data Extractor (AI request) | Markdown to Textual Data Extractor (response)                         |                                                                                                                                                                                                     |
| Topic Extractor with the structured response             | Langchain Information Extractor            | Extract topics with scores, summaries, keywords | Markdown to Textual Data Extractor              | Initiate a Webhook Notification for AI Sentiment Analyzer, Create a binary file for topics  |                                                                                                                                                                                                     |
| Google Gemini Chat Model for Sentiment Analyzer          | Langchain Google Gemini LM Chat            | LLM backend for sentiment analysis               | Topic Extractor with the structured response (AI request) | Topic Extractor with the structured response (response)               |                                                                                                                                                                                                     |
| Initiate a Webhook Notification for Markdown to Textual Data Extraction | HTTP Request                               | Notify with cleaned text output                    | Markdown to Textual Data Extractor              | None                                                                  |                                                                                                                                                                                                     |
| Initiate a Webhook Notification for AI Sentiment Analyzer | HTTP Request                               | Notify with sentiment analysis summary            | Topic Extractor with the structured response   | None                                                                  |                                                                                                                                                                                                     |
| Trends by location and category with the structured response | Langchain Information Extractor            | Cluster emerging trends by location and category | Markdown to Textual Data Extractor              | Initiate a Webhook Notification for trends by location and category, Create a binary data for tends |                                                                                                                                                                                                     |
| Google Gemini Chat Model                                 | Langchain Google Gemini LM Chat            | LLM backend for trend clustering                   | Trends by location and category with the structured response (AI request) | Trends by location and category with the structured response (response) |                                                                                                                                                                                                     |
| Initiate a Webhook Notification for trends by location and category | HTTP Request                               | Notify with trend clustering summary               | Trends by location and category with the structured response | None                                                                  |                                                                                                                                                                                                     |
| Create a binary file for topics                          | Function                                   | Encode topics JSON to binary for file writing      | Topic Extractor with the structured response   | Write the topics file to disk                                         |                                                                                                                                                                                                     |
| Write the topics file to disk                            | Read/Write File                            | Save topics JSON file on disk                       | Create a binary file for topics                 | None                                                                  |                                                                                                                                                                                                     |
| Create a binary data for tends                           | Function                                   | Encode trends JSON to binary for file writing      | Trends by location and category with the structured response | Write the trends file to disk                                         |                                                                                                                                                                                                     |
| Write the trends file to disk                            | Read/Write File                            | Save trends JSON file on disk                       | Create a binary data for tends                  | None                                                                  |                                                                                                                                                                                                     |
| Sticky Note                                             | Sticky Note                               | Workflow overview and setup instructions           | None                                          | None                                                                  | This workflow deals with the structured data extraction by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Information Extraction, are used to demonstrate n8n AI capabilities.      |
| Sticky Note1                                            | Sticky Note                               | Explanation of LLM usage                            | None                                          | None                                                                  | Google Gemini Flash Exp model is used. Basic LLM Chain Data Extractor. Information Extraction is used for custom sentiment analysis with structured response.                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - No special parameters.

2. **Add a Set Node**  
   - Name: Set URL and Bright Data Zone  
   - Assign two string variables:  
     - `url`: e.g., `https://www.bbc.com/news/world`  
     - `zone`: e.g., `web_unlocker1`  
   - Connect input from manual trigger.

3. **Add HTTP Request Node for Bright Data API**  
   - Name: Perform Bright Data Web Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: HTTP Header Auth credential (set up with Bright Data API key)  
   - Body Parameters (form-data or JSON):  
     - `zone`: from Set node variable  
     - `url`: append `?product=unlocker&method=api` to `url` variable  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Connect input from Set node.

4. **Add Langchain LLM Chain Node for Markdown to Textual Data Conversion**  
   - Name: Markdown to Textual Data Extractor  
   - Model: Google Gemini Flash Exp via Langchain LLM integration  
   - Prompt: Instruct to analyze markdown and output only clean textual data without links or scripts.  
   - Connect input from HTTP Request node.  
   - Configure Google Gemini API credentials in the node.

5. **Add Google Gemini Chat Model Node**  
   - Name: Google Gemini Chat Model for Data Extract  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credential: Google Palm API account  
   - Connect AI model input/output to Langchain LLM Chain node.

6. **Add Langchain Information Extractor Node for Topic Extraction**  
   - Name: Topic Extractor with the structured response  
   - Input Text: Use the textual data from markdown extractor node.  
   - System prompt: "You are an expert data analyst."  
   - Define JSON schema with fields: topic (string), score (0-1), summary (string), keywords (array of strings).  
   - Connect input from Markdown to Textual Data Extractor.  

7. **Add Google Gemini Chat Model for Sentiment Analyzer**  
   - Name: Google Gemini Chat Model for Sentiment Analyzer  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credential: Google Palm API account  
   - Connect AI model input/output to Topic Extractor node.

8. **Add HTTP Request Node for Webhook Notification (Topics Sentiment)**  
   - Name: Initiate a Webhook Notification for AI Sentiment Analyzer  
   - Method: POST  
   - URL: your webhook notification endpoint (e.g., https://webhook.site/...)  
   - Send body with field `summary` containing sentiment output.  
   - Connect input from Topic Extractor node.

9. **Add Function Node to Encode Topics JSON as Binary**  
   - Name: Create a binary file for topics  
   - Code: Encode JSON output to base64 binary buffer.  
   - Connect input from Topic Extractor node.

10. **Add Read/Write File Node to Save Topics JSON**  
    - Name: Write the topics file to disk  
    - Operation: Write  
    - File Path: e.g., `d:\topics.json`  
    - Connect input from Function node.

11. **Add Langchain Information Extractor Node for Trends Clustering**  
    - Name: Trends by location and category with the structured response  
    - Input Text: Use the same markdown raw data input as topic extractor.  
    - System prompt: "You are an expert data analyst."  
    - Define JSON schema with location, category, and trends array (trend, score, summary, mentions).  
    - Connect input from Markdown to Textual Data Extractor.

12. **Add Google Gemini Chat Model Node for Trend Clustering**  
    - Name: Google Gemini Chat Model  
    - Model Name: `models/gemini-2.0-flash-exp`  
    - Credential: Google Palm API account  
    - Connect AI model input/output to Trends node.

13. **Add HTTP Request Node for Webhook Notification (Trends)**  
    - Name: Initiate a Webhook Notification for trends by location and category  
    - Method: POST  
    - URL: webhook endpoint  
    - Send body with field `summary` containing trend summary.  
    - Connect input from Trends node.

14. **Add Function Node to Encode Trends JSON as Binary**  
    - Name: Create a binary data for tends  
    - Code: Encode trends JSON output to base64 binary buffer.  
    - Connect input from Trends node.

15. **Add Read/Write File Node to Save Trends JSON**  
    - Name: Write the trends file to disk  
    - Operation: Write  
    - File Path: e.g., `d:\trends.json`  
    - Connect input from Function node.

16. **Add HTTP Request Node for Webhook Notification (Markdown Text)**  
    - Name: Initiate a Webhook Notification for Markdown to Textual Data Extraction  
    - Method: POST  
    - URL: webhook endpoint  
    - Send body with field `content` containing extracted textual data.  
    - Connect input from Markdown to Textual Data Extractor.

17. **Add Sticky Notes for documentation as needed**  
    - Provide workflow overview and LLM usage notes.

18. **Verify all credentials are set:**  
    - Google Palm API (Google Gemini) with API keys  
    - Bright Data HTTP Header Auth with API key  
    - Webhook URLs configured correctly.

19. **Set execution order properly:**  
    - Manual Trigger → Set URL → Bright Data Request → Markdown Extractor → (Topic Extractor + Trends Extractor + Webhook notifications) → Sentiment analysis → File writes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow deals with structured data extraction using Bright Data Web Unlocker product and demonstrates n8n AI capabilities including Basic LLM Chain and Information Extraction with Google Gemini models.                 | Workflow Sticky Note in n8n editor                                                                 |
| Google Gemini Flash Exp model is used for all LLM-related nodes, enabling advanced language understanding and extraction capabilities.                                                                                       | Workflow Sticky Note1 in n8n editor                                                                |
| Update the target web URL in the "Set URL and Bright Data Zone" node to customize the data source.                                                                                                                           | Workflow Sticky Note in n8n editor                                                                 |
| Remember to update webhook URLs according to your notification infrastructure to receive real-time updates.                                                                                                                  | Workflow Sticky Note in n8n editor                                                                 |
| For local file writing, ensure the n8n process has appropriate file system permissions to write to the specified disk paths (e.g., `d:\topics.json`, `d:\trends.json`).                                                         | General system configuration note                                                                 |
| Bright Data API may impose rate limits or require specific API key permissions; verify that your Bright Data account and API key have access to the Web Unlocker product and API method used in this workflow.                 | Bright Data API documentation                                                                     |
| Google Gemini (PaLM) API credentials must be configured with access to the specified models; monitor usage quotas and error logs to handle rate limit or authentication issues.                                                | Google PaLM API documentation                                                                     |
| Webhook.site URLs used here are for demonstration; replace with your own endpoints for production use.                                                                                                                       | https://webhook.site/                                                                              |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.