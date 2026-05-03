Extract, Summarize & Analyze Amazon Price Drops with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/extract--summarize---analyze-amazon-price-drops-with-bright-data---google-gemini-4611


# Extract, Summarize & Analyze Amazon Price Drops with Bright Data & Google Gemini

---

### 1. Workflow Overview

This workflow automates the extraction, summarization, and sentiment analysis of Amazon product price drops by leveraging Bright Data's MCP Client for web scraping and Google's Gemini Large Language Model (LLM) for natural language processing tasks. It is designed for ecommerce analysts, price monitoring teams, or marketing specialists who need structured insights on price drops alongside summarized content and sentiment information.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization**: Manual start and setup of input parameters such as target URLs and webhook endpoints.
- **1.2 Price Drop Data Extraction via Bright Data MCP Client**: Scraping the daily Amazon price drop listings and extracting raw markdown data.
- **1.3 Structured Data Parsing with Google Gemini LLM**: Processing scraped markdown content to extract structured product information.
- **1.4 Iterative Detailed Data Extraction and AI Processing Loop**: For each product, fetch detailed markdown data, then perform summarization and sentiment analysis using Google Gemini.
- **1.5 Data Aggregation and Output Handling**: Consolidating all processed information, updating Google Sheets, and sending webhook notifications.
- **1.6 Supporting Utility and Informational Notes**: Sticky notes providing disclaimers, usage instructions, and branding.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview**: This block initiates the workflow manually and sets essential input fields such as the URL for price drops and a notification webhook URL.
- **Nodes Involved**: 
  - When clicking ‘Test workflow’
  - Bright Data MCP Client List Tools
  - Set input fields
  - Sticky Note3, Sticky Note4, Sticky Note2, Sticky Note5 (informative notes)

- **Node Details**:

  - **When clicking ‘Test workflow’**  
    - **Type**: Manual Trigger  
    - **Role**: Starts workflow on manual command  
    - **Connections**: Outputs to Bright Data MCP Client List Tools  
    - **Edge Cases**: Workflow will not run without manual triggering.

  - **Bright Data MCP Client List Tools**  
    - **Type**: MCP Client (Bright Data)  
    - **Role**: Accesses MCP Client tools list (possibly for validation or status)  
    - **Credentials**: Uses MCP Client API credentials (STDIO account)  
    - **Connections**: Outputs to Set input fields  
    - **Edge Cases**: Authentication failures or connectivity issues with Bright Data API.

  - **Set input fields**  
    - **Type**: Set Node  
    - **Role**: Defines two key string variables:
      - `price_drop_url` set to "https://camelcamelcamel.com/top_drops?t=daily"
      - `webhook_notification_url` set to a webhook.site URL for notifications  
    - **Connections**: Outputs to MCP Client for Price Drop Data Extract  
    - **Edge Cases**: Incorrect URLs or missing values will break downstream scraping and notifications.

  - **Sticky Notes (3,4,2,5)**  
    - Provide important disclaimers, note about Google Gemini LLM usage for structured extraction, Bright Data MCP Client node availability (self-hosted only), and branding logo.  
    - These do not affect execution but serve as documentation and user guidance.

---

#### 2.2 Price Drop Data Extraction via Bright Data MCP Client

- **Overview**: This block scrapes the daily Amazon price drop listings page as markdown using the Bright Data MCP Client.
- **Nodes Involved**:  
  - MCP Client for Price Drop Data Extract  
  - Structure Data Extract Using LLM  
  - Google Gemini Chat Model (for structured extraction)  
  - Structured Output Parser  
  - Split Out  

- **Node Details**:

  - **MCP Client for Price Drop Data Extract**  
    - **Type**: MCP Client (Bright Data)  
    - **Role**: Executes the `scrape_as_markdown` tool with the URL from `price_drop_url`  
    - **Credentials**: MCP Client API (STDIO account)  
    - **Input**: `price_drop_url` string set in previous block  
    - **Output**: Raw markdown data representing the price drop listings  
    - **Edge Cases**: Web scraping failures, rate limits, or invalid URL.

  - **Google Gemini Chat Model**  
    - **Type**: LangChain Google Gemini LLM Node  
    - **Role**: LLM for extracting structured data from raw markdown text  
    - **Model**: `models/gemini-2.0-flash-exp`  
    - **Credentials**: Google Palm API account  
    - **Connections**: Input from MCP Client node via chain LLM node, outputs to Structured Output Parser  
    - **Edge Cases**: API rate limits, malformed input text, or LLM response errors.

  - **Structured Output Parser**  
    - **Type**: LangChain Structured Output Parser  
    - **Role**: Parses Gemini LLM output into JSON structured format with fields like `id`, `title`, `price`, `savings`, and `link`  
    - **Input**: Raw LLM output JSON text  
    - **Output**: Clean structured JSON objects for downstream processing  
    - **Edge Cases**: Parsing errors if LLM output deviates from expected schema.

  - **Structure Data Extract Using LLM**  
    - **Type**: LangChain Chain LLM Node with output parser  
    - **Role**: Wraps the above two nodes into a single step to extract structured data from raw markdown  
    - **Connections**: Inputs markdown from MCP Client, outputs parsed structured objects to Split Out node  
    - **Retry**: Enabled on failure for robustness.

  - **Split Out**  
    - **Type**: Split Out Node  
    - **Role**: Splits the array of structured product data into individual items for batch processing  
    - **Output**: Feeds into Loop Over Items node

---

#### 2.3 Iterative Detailed Data Extraction and AI Processing Loop

- **Overview**: For each extracted product, this block fetches detailed markdown data from the individual product link, then runs summarization and sentiment analysis via Google Gemini LLM.
- **Nodes Involved**:  
  - Loop Over Items (splitInBatches)  
  - Wait (delay node)  
  - MCP Client for Price Drop Data Extract Within a Loop  
  - Summarize Content  
  - Google Gemini Chat Model for Summarize Content  
  - Sentiment Analysis  
  - Google Gemini Chat Model for Sentiment Analysis  
  - Merge  
  - Sticky Note1, Sticky Note (informative notes)

- **Node Details**:

  - **Loop Over Items**  
    - **Type**: splitInBatches  
    - **Role**: Processes products one by one or in batches for rate limiting and controlled execution.  
    - **Output**: Branch 1 to MCP Client for detailed extraction; Branch 2 to Wait node.

  - **Wait**  
    - **Type**: Wait Node  
    - **Role**: Delays execution for 10 seconds to avoid overloading scraping or API services  
    - **Input**: From Loop Over Items branch 2  
    - **Output**: Connects back to detailed MCP Client node to pace requests  
    - **Edge Cases**: Delay needed to prevent throttling or bans.

  - **MCP Client for Price Drop Data Extract Within a Loop**  
    - **Type**: MCP Client (Bright Data)  
    - **Role**: Scrapes individual product page markdown from each product `link` field  
    - **Credentials**: MCP Client API (STDIO account)  
    - **Input**: Product `link` from current item  
    - **Output**: Raw markdown content of product details  
    - **Edge Cases**: Broken links, rate limits, or extraction failure.

  - **Summarize Content**  
    - **Type**: LangChain Chain Summarization Node  
    - **Role**: Summarizes large textual content from product markdown into concise insights  
    - **Chunking Mode**: Advanced to handle long texts  
    - **Retry**: Enabled  
    - **Input**: Text from MCP Client detailed extraction  
    - **Output**: Summarized content passed to Merge node

  - **Google Gemini Chat Model for Summarize Content**  
    - **Type**: LangChain Google Gemini LLM  
    - **Role**: Language model running summarization prompt for the content summarizer node  
    - **Credentials**: Google Palm API  
    - **Model**: `models/gemini-2.0-flash-exp`

  - **Sentiment Analysis**  
    - **Type**: LangChain Information Extractor Node  
    - **Role**: Performs sentiment analysis on the summarized content text using a manual JSON schema with fields: sentiment (enum), sentimentScore (numeric), topics (array of strings)  
    - **Input**: Summarized text from product markdown  
    - **Retry**: Enabled  
    - **Output**: Sentiment data passed to Merge node

  - **Google Gemini Chat Model for Sentiment Analysis**  
    - **Type**: LangChain Google Gemini LLM  
    - **Role**: Runs sentiment analysis prompt against the summarized text  
    - **Credentials**: Google Palm API  
    - **Model**: `models/gemini-2.0-flash-exp`

  - **Merge**  
    - **Type**: Merge Node  
    - **Role**: Combines outputs of summarization and sentiment analysis into a single data item  
    - **Input**: Two inputs — summarization output and sentiment analysis output  
    - **Output**: Feeds into Aggregate node for final consolidation

  - **Sticky Note1 & Sticky Note**  
    - Provide descriptive labels for this block about looping and output data handling.

---

#### 2.4 Data Aggregation and Output Handling

- **Overview**: Collects all processed product data, aggregates it, updates a Google Sheet, and sends a webhook notification.
- **Nodes Involved**:  
  - Aggregate  
  - Update Google Sheets  
  - Webhook Notification for Price Drop Info  
  - Sticky Note (output data handling)

- **Node Details**:

  - **Aggregate**  
    - **Type**: Aggregate Node  
    - **Role**: Aggregates all processed item data into a single collection  
    - **Input**: Merged summarization and sentiment items per product  
    - **Output**: Feeds into Google Sheets update and webhook notification

  - **Update Google Sheets**  
    - **Type**: Google Sheets Node  
    - **Role**: Appends or updates the Google Sheet with aggregated product price drop info  
    - **Parameters**:  
      - Document ID: linked to a specific Google Sheet for storing price drop info  
      - Operation: appendOrUpdate  
      - Mapping: outputs JSON stringified data into a single column named `output`  
    - **Credentials**: Google Sheets OAuth2  
    - **Edge Cases**: OAuth token expiration, insufficient permissions, or sheet not found.

  - **Webhook Notification for Price Drop Info**  
    - **Type**: HTTP Request Node  
    - **Role**: Sends aggregated data as JSON to a user-defined webhook URL  
    - **Parameters**: POST request with body parameter `response` containing JSON stringified data  
    - **URL**: Dynamic, from `webhook_notification_url` input field  
    - **Edge Cases**: Webhook endpoint failure, timeouts, or network issues.

  - **Sticky Note**  
    - Marks this block as output data handling for clarity.

---

### 3. Summary Table

| Node Name                                    | Node Type                                    | Functional Role                                     | Input Node(s)                            | Output Node(s)                                                 | Sticky Note                                                                                                           |
|----------------------------------------------|----------------------------------------------|----------------------------------------------------|-----------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                 | Manual Trigger                               | Workflow manual start                              | -                                       | Bright Data MCP Client List Tools                              |                                                                                                                       |
| Bright Data MCP Client List Tools             | MCP Client (Bright Data)                      | Access MCP Client tools                            | When clicking ‘Test workflow’            | Set input fields                                               |                                                                                                                       |
| Set input fields                              | Set Node                                    | Define input URLs and webhook                      | Bright Data MCP Client List Tools        | MCP Client for Price Drop Data Extract                         | **Note:** Please make sure to set the input fields                                                                    |
| MCP Client for Price Drop Data Extract        | MCP Client (Bright Data)                      | Scrape Amazon price drop listing page markdown    | Set input fields                         | Structure Data Extract Using LLM                               |                                                                                                                       |
| Google Gemini Chat Model                      | LangChain Google Gemini LLM                   | LLM for structured data extraction                 | MCP Client for Price Drop Data Extract   | Structured Output Parser                                      | Google Gemini LLM is used for structured data extraction                                                              |
| Structured Output Parser                      | LangChain Structured Output Parser            | Parse LLM output into structured JSON              | Google Gemini Chat Model                 | Structure Data Extract Using LLM                              |                                                                                                                       |
| Structure Data Extract Using LLM              | LangChain Chain LLM                           | Extract structured data from markdown              | MCP Client for Price Drop Data Extract   | Split Out                                                     |                                                                                                                       |
| Split Out                                    | Split Out Node                              | Split product array into individual items          | Structure Data Extract Using LLM         | Loop Over Items                                               |                                                                                                                       |
| Loop Over Items                              | splitInBatches                              | Batch processing of product items                   | Split Out                               | MCP Client for Price Drop Data Extract Within a Loop, Wait    | Loop and Extract Data: Perform Summarization & Sentiment Analysis                                                    |
| Wait                                         | Wait Node                                   | Delay between item processing                        | Loop Over Items (branch 2)               | MCP Client for Price Drop Data Extract Within a Loop          |                                                                                                                       |
| MCP Client for Price Drop Data Extract Within a Loop | MCP Client (Bright Data)                      | Scrape individual product markdown                  | Loop Over Items (branch 1)               | Summarize Content, Sentiment Analysis                         |                                                                                                                       |
| Google Gemini Chat Model for Summarize Content | LangChain Google Gemini LLM                   | LLM for summarization prompt                        | Summarize Content                       | Summarize Content                                            |                                                                                                                       |
| Summarize Content                            | LangChain Chain Summarization                 | Summarize detailed product content                  | MCP Client for Price Drop Data Extract Within a Loop | Merge                                                   |                                                                                                                       |
| Google Gemini Chat Model for Sentiment Analysis | LangChain Google Gemini LLM                   | LLM for sentiment analysis prompt                   | Sentiment Analysis                     | Sentiment Analysis                                          |                                                                                                                       |
| Sentiment Analysis                           | LangChain Information Extractor               | Extract sentiment and topics from summarized text  | MCP Client for Price Drop Data Extract Within a Loop | Merge                                                   |                                                                                                                       |
| Merge                                        | Merge Node                                  | Combine summarization and sentiment outputs         | Summarize Content, Sentiment Analysis    | Aggregate                                                    |                                                                                                                       |
| Aggregate                                    | Aggregate Node                              | Aggregate all processed product data                | Merge                                  | Update Google Sheets, Webhook Notification for Price Drop Info |                                                                                                                       |
| Update Google Sheets                         | Google Sheets                               | Append or update price drop data in Google Sheets  | Aggregate                              | Loop Over Items                                              |                                                                                                                       |
| Webhook Notification for Price Drop Info    | HTTP Request                               | Send aggregated data to webhook endpoint            | Aggregate                              | -                                                            |                                                                                                                       |
| Sticky Note2                                 | Sticky Note                                 | Disclaimer about MCP Client node availability        | -                                       | -                                                            | This template is only available on n8n self-hosted as it's making use of the community node for MCP Client.             |
| Sticky Note3                                 | Sticky Note                                 | Workflow notes on price drop extraction and AI usage | -                                       | -                                                            | Deals with extraction of price drop info, structured data extraction, sentiment analysis, and summarization.            |
| Sticky Note4                                 | Sticky Note                                 | Note on LLM usage                                   | -                                       | -                                                            | Google Gemini LLM is being utilized for the structured data extraction handling.                                       |
| Sticky Note5                                 | Sticky Note                                 | Branding logo                                      | -                                       | -                                                            | ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)                            |
| Sticky Note1                                 | Sticky Note                                 | Loop and extract data description                    | -                                       | -                                                            | Perform Summarization & Sentiment Analysis                                                                           |
| Sticky Note                                  | Sticky Note                                 | Output data handling label                           | -                                       | -                                                            | Output Data Handling                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start workflow manually.

2. **Add Bright Data MCP Client List Tools Node**  
   - Type: MCP Client (Bright Data)  
   - Credential: MCP Client API (STDIO account)  
   - Connect manual trigger output to this node.

3. **Add Set Node for Input Fields**  
   - Type: Set  
   - Assign two string fields:
     - `price_drop_url`: `"https://camelcamelcamel.com/top_drops?t=daily"`  
     - `webhook_notification_url`: `"https://webhook.site/24878284-919d-4d39-bff0-5f36bfae17b6"`  
   - Connect Bright Data MCP Client List Tools node output to this node.

4. **Add MCP Client Node for Initial Price Drop Data Extraction**  
   - Type: MCP Client (Bright Data)  
   - Operation: `executeTool` with toolName `scrape_as_markdown`  
   - Tool parameters: `{ "url": "{{ $json.price_drop_url }}" }`  
   - Credential: MCP Client API (STDIO account)  
   - Connect Set node output to this node.

5. **Add LangChain Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini LLM  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credential: Google Palm API  
   - Connect MCP Client node output to this node (through Chain LLM node).

6. **Add LangChain Structured Output Parser Node**  
   - Type: Output Parser Structured  
   - JSON Schema Example: Array of objects with `id`, `title`, `price`, `savings`, `link` fields (as per example)  
   - Connect Google Gemini Chat Model output to this node.

7. **Add Chain LLM Node to Wrap Structured Extraction**  
   - Type: Chain LLM Node with output parser  
   - Text to extract structured data from: `=Extract structured data from  {{ $json.result.content[0].text }}`  
   - Enable retry on fail  
   - Connect MCP Client node output to this node; connect Google Gemini Chat Model and Structured Output Parser as ai_languageModel and ai_outputParser respectively.

8. **Add Split Out Node**  
   - Type: Split Out  
   - Field to split out: `output` (array of structured product data)  
   - Connect Chain LLM node output to this node.

9. **Add SplitInBatches Node (Loop Over Items)**  
   - Type: splitInBatches  
   - Connect Split Out node output to here.

10. **Add Wait Node**  
    - Type: Wait  
    - Delay: 10 seconds  
    - Connect second output of splitInBatches (branch 2) to Wait node output.

11. **Add MCP Client Node for Detailed Product Markdown Extraction in Loop**  
    - Type: MCP Client (Bright Data)  
    - Operation: `executeTool` with toolName `scrape_as_markdown`  
    - Tool parameters: `{ "url": "{{ $json.link }}" }`  
    - Credential: MCP Client API (STDIO account)  
    - Connect first output of splitInBatches (branch 1) to this node.  
    - Connect Wait node output to this node to pace calls.

12. **Add LangChain Google Gemini Chat Model Node for Summarization**  
    - Type: LangChain Google Gemini LLM  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credential: Google Palm API

13. **Add Chain Summarization Node**  
    - Type: Chain Summarization  
    - Chunking Mode: Advanced  
    - Enable retry on fail  
    - Connect MCP Client detailed extraction output to this node.  
    - Connect Google Gemini Chat Model for Summarize Content as ai_languageModel for summarization.

14. **Add LangChain Google Gemini Chat Model Node for Sentiment Analysis**  
    - Type: LangChain Google Gemini LLM  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credential: Google Palm API

15. **Add Information Extractor Node for Sentiment Analysis**  
    - Type: Information Extractor  
    - Text: `=Perform sentiment analysis of  {{ $json.result.content[0].text }}`  
    - Input Schema (manual) with fields:
      - `sentiment` (enum: positive, neutral, negative)
      - `sentimentScore` (number from -1 to 1)
      - `topics` (array of strings)  
    - Enable retry on fail  
    - Connect MCP Client detailed extraction output to this node.  
    - Connect Google Gemini Chat Model for Sentiment Analysis as ai_languageModel.

16. **Add Merge Node**  
    - Type: Merge  
    - Connect summarization output and sentiment analysis output as inputs.  
    - Purpose: Combine both AI results per product.

17. **Add Aggregate Node**  
    - Type: Aggregate  
    - Aggregate all merged item data into one array  
    - Connect Merge node output to Aggregate node.

18. **Add Google Sheets Node**  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: `gid=0` (or desired sheet)  
    - Mapping: Map a column `output` to `={{ $json.data.toJsonString() }}`  
    - Credential: Google Sheets OAuth2  
    - Connect Aggregate output to this node.

19. **Add HTTP Request Node for Webhook Notification**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `={{ $('Set input fields').item.json.webhook_notification_url }}`  
    - Body Parameters: JSON with key `response` containing aggregated data stringified  
    - Connect Aggregate output to this node.

20. **Add Sticky Notes for Documentation**  
    - Add notes about workflow usage, disclaimers, LLM usage, and branding for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it uses the community MCP Client node.                          | Disclaimer Sticky Note2                                                                                            |
| Google Gemini LLM is used for structured data extraction and NLP tasks like summarization and sentiment analysis.     | Sticky Note4                                                                                                        |
| Deals with ecommerce price drop extraction, structured data parsing, summarization, and sentiment analysis.           | Sticky Note3                                                                                                        |
| Branding logo for Bright Data is included for visual identification.                                                  | ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow developed with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.

---