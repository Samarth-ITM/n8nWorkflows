Extract & Summarize Wikipedia Data with Bright Data and Gemini AI

https://n8nworkflows.xyz/workflows/extract---summarize-wikipedia-data-with-bright-data-and-gemini-ai-3539


# Extract & Summarize Wikipedia Data with Bright Data and Gemini AI

### 1. Workflow Overview

This workflow automates the extraction and summarization of Wikipedia article data by leveraging Bright Data’s Web Unlocker API for scraping, Google Gemini AI models for natural language processing, and webhook notifications for downstream integration. It is designed for users needing structured, human-readable, and summarized Wikipedia content delivered automatically.

**Target Use Cases:**
- Researchers requiring structured Wikipedia data regularly.
- Data engineers enriching datasets or knowledge bases.
- Content creators automating fact-checking or content sourcing.
- Automation enthusiasts integrating Wikipedia data into external systems.

**Logical Blocks:**

- **1.1 Trigger Block:** Initiates the workflow manually or on schedule.
- **1.2 Setup Wikipedia Scraping Parameters:** Defines the target Wikipedia URL and Bright Data zone.
- **1.3 Wikipedia Content Retrieval:** Calls Bright Data Web Unlocker API to scrape raw Wikipedia HTML.
- **1.4 Data Extraction with LLM:** Uses a Google Gemini AI model to convert raw HTML into human-readable text.
- **1.5 Content Summarization:** Summarizes the extracted Wikipedia content using a Google Gemini summarization model.
- **1.6 Webhook Notification:** Sends the summarized content to a configured webhook endpoint for further processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Starts the workflow execution either manually or on a schedule.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)
- **Node Details:**
  - **Type:** Manual Trigger  
  - **Role:** Entry point for manual workflow execution.  
  - **Configuration:** No parameters; triggers workflow on user action.  
  - **Inputs:** None  
  - **Outputs:** Connects to `Set Wikipedia URL with Bright Data Zone` node.  
  - **Edge Cases:** None typical; manual trigger depends on user action.

#### 2.2 Setup Wikipedia Scraping Parameters

- **Overview:** Sets the Wikipedia article URL and Bright Data zone parameters for scraping.
- **Nodes Involved:**  
  - `Set Wikipedia URL with Bright Data Zone`
- **Node Details:**
  - **Type:** Set Node  
  - **Role:** Defines workflow variables `url` (Wikipedia article URL) and `zone` (Bright Data Web Unlocker zone).  
  - **Configuration:**  
    - `url` set to `https://en.wikipedia.org/wiki/Cloud_computing?product=unlocker&method=api` (default example).  
    - `zone` set to `web_unlocker1` (Bright Data zone identifier).  
  - **Inputs:** From manual trigger node.  
  - **Outputs:** To `Wikipedia Web Request` node.  
  - **Edge Cases:** URL must be valid and accessible; zone must be correctly configured in Bright Data account.

#### 2.3 Wikipedia Content Retrieval

- **Overview:** Sends a POST request to Bright Data’s Web Unlocker API to scrape raw HTML content of the Wikipedia page.
- **Nodes Involved:**  
  - `Wikipedia Web Request`
- **Node Details:**
  - **Type:** HTTP Request  
  - **Role:** Calls Bright Data API with authentication to retrieve raw Wikipedia page content.  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://api.brightdata.com/request`  
    - Body Parameters:  
      - `zone`: from previous node (`$json.zone`)  
      - `url`: from previous node (`$json.url`)  
      - `format`: `"raw"` (requests raw HTML content)  
    - Authentication: Generic HTTP Header Auth using pre-configured credentials (`Header Auth account`).  
  - **Inputs:** From `Set Wikipedia URL with Bright Data Zone`.  
  - **Outputs:** To `LLM Data Extractor`.  
  - **Edge Cases:**  
    - Authentication failure if credentials invalid or expired.  
    - API rate limits or quota exceeded.  
    - Network timeouts or unreachable API.  
    - Invalid zone or URL causing API errors.

#### 2.4 Data Extraction with LLM

- **Overview:** Uses Google Gemini AI (Gemini 2.0 Pro Exp model) to parse raw HTML and produce human-readable Wikipedia content.
- **Nodes Involved:**  
  - `LLM Data Extractor`  
  - `Google Gemini Chat Model2` (AI model node feeding into LLM Data Extractor)
- **Node Details:**
  - **LLM Data Extractor:**  
    - Type: LangChain LLM Chain Node  
    - Role: Converts raw HTML data into formatted, human-readable text.  
    - Configuration:  
      - Input text: raw HTML from previous node (`$json.data`)  
      - Prompt: "You are an expert Data Formatter. Make sure to format the data in a human readable manner. Please output the human readable content without your own thoughts."  
      - Output parser enabled to structure response.  
    - Inputs: From `Wikipedia Web Request` (raw HTML) and AI model node (`Google Gemini Chat Model2`) as language model.  
    - Outputs: To `Concise Summary Generator`.  
    - Edge Cases:  
      - AI model API errors or rate limits.  
      - Unexpected HTML structure causing poor extraction.  
      - Expression or prompt evaluation failures.  
  - **Google Gemini Chat Model2:**  
    - Type: Google Gemini Chat Model Node  
    - Role: Provides AI language model backend for data extraction.  
    - Configuration:  
      - Model: `models/gemini-2.0-pro-exp`  
      - Credentials: Google Gemini API key (PaLM API account)  
    - Inputs: None (used as AI model for LLM Data Extractor)  
    - Outputs: To `LLM Data Extractor`.  
    - Edge Cases: API quota, authentication errors, model unavailability.

#### 2.5 Content Summarization

- **Overview:** Summarizes the human-readable Wikipedia content into a concise summary using Google Gemini Flash Exp model.
- **Nodes Involved:**  
  - `Concise Summary Generator`  
  - `Google Gemini Chat Model For Summarization` (AI model node feeding into Concise Summary Generator)
- **Node Details:**
  - **Concise Summary Generator:**  
    - Type: LangChain Summarization Chain Node  
    - Role: Produces a concise summary from the extracted Wikipedia text.  
    - Configuration:  
      - Summarization prompt: "Write a concise summary of the following:\n\n\"{text}\"\n"  
      - Chunking mode: Advanced (handles large text by chunking)  
    - Inputs: From `LLM Data Extractor` (human-readable content) and AI model node (`Google Gemini Chat Model For Summarization`).  
    - Outputs: To `Summary Webhook Notifier`.  
    - Edge Cases:  
      - AI model errors or rate limiting.  
      - Chunking failures if text too large or malformed.  
  - **Google Gemini Chat Model For Summarization:**  
    - Type: Google Gemini Chat Model Node  
    - Role: Provides AI language model backend for summarization.  
    - Configuration:  
      - Model: `models/gemini-2.0-flash-exp`  
      - Credentials: Google Gemini API key (PaLM API account)  
    - Inputs: None (used as AI model for summarization chain)  
    - Outputs: To `Concise Summary Generator`.  
    - Edge Cases: Same as above for AI model nodes.

#### 2.6 Webhook Notification

- **Overview:** Sends the summarized Wikipedia content to a user-defined webhook URL for integration with external systems.
- **Nodes Involved:**  
  - `Summary Webhook Notifier`
- **Node Details:**
  - **Type:** HTTP Request  
  - **Role:** Posts the summary text to a specified webhook endpoint.  
  - **Configuration:**  
    - Method: POST (default)  
    - URL: `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7` (example URL, user should replace)  
    - Body Parameters: JSON with field `summary` containing the summarized text (`{{$json.response.text}}`)  
    - Sends body as JSON payload.  
  - **Inputs:** From `Concise Summary Generator`.  
  - **Outputs:** None (terminal node).  
  - **Edge Cases:**  
    - Webhook URL invalid or unreachable.  
    - Network timeouts.  
    - Downstream system errors or rejections.

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                          | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                          |
|----------------------------------|----------------------------------|----------------------------------------|--------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                   | Workflow start trigger                  | None                           | Set Wikipedia URL with Bright Data Zone |                                                                                                    |
| Set Wikipedia URL with Bright Data Zone | Set                             | Defines Wikipedia URL and Bright Data zone | When clicking ‘Test workflow’  | Wikipedia Web Request             | Set the URL which you are interested to scrap the data                                             |
| Wikipedia Web Request             | HTTP Request                    | Scrapes raw Wikipedia HTML via Bright Data API | Set Wikipedia URL with Bright Data Zone | LLM Data Extractor               |                                                                                                    |
| Google Gemini Chat Model2         | Google Gemini Chat Model         | AI model for data extraction            | None (AI model node)            | LLM Data Extractor               | Google Gemini Flash Exp model is being used to demonstrate the data extraction and summarization aspects. Basic LLM Chain is being used for extracting the html to text. Summarization Chain is being used for summarization of the Wikipedia data. Note - Replace Google Gemini with the Open AI or suitable LLM providers of your choice. |
| LLM Data Extractor               | LangChain LLM Chain              | Converts raw HTML to human-readable text | Wikipedia Web Request, Google Gemini Chat Model2 | Concise Summary Generator       | Basic LLM Chain Data Extractor                                                                     |
| Google Gemini Chat Model For Summarization | Google Gemini Chat Model         | AI model for summarization              | None (AI model node)            | Concise Summary Generator       | Google Gemini Flash Exp model is being used to demonstrate the data extraction and summarization aspects. Basic LLM Chain is being used for extracting the html to text. Summarization Chain is being used for summarization of the Wikipedia data. Note - Replace Google Gemini with the Open AI or suitable LLM providers of your choice. |
| Concise Summary Generator        | LangChain Summarization Chain    | Generates concise summary from extracted text | LLM Data Extractor, Google Gemini Chat Model For Summarization | Summary Webhook Notifier        | Summarization Chain                                                                               |
| Summary Webhook Notifier         | HTTP Request                    | Sends summary to configured webhook    | Concise Summary Generator       | None                            | Please make sure to update the Wikipedia URL with Bright Data Zone. Also make sure to set the Webhook Notification URL. |
| Sticky Note                     | Sticky Note                     | Notes on workflow purpose and usage    | None                           | None                            | This template deals with the Wikipedia data extraction and summarization of content with the Bright Data. The LLM Data Extractor is responsible for producing a human readable content. The Concise Summary Generator node is responsible for generating the concise summary of the Wikipedia extracted info. Please make sure to update the Wikipedia URL with Bright Data Zone. Also make sure to set the Webhook Notification URL. |
| Sticky Note1                    | Sticky Note                     | Notes on LLM usage                      | None                           | None                            | Google Gemini Flash Exp model is being used to demonstrate the data extraction and summarization aspects. Basic LLM Chain is being used for extracting the html to text. Summarization Chain is being used for summarization of the Wikipedia data. Note - Replace Google Gemini with the Open AI or suitable LLM providers of your choice. |
| Sticky Note2                    | Sticky Note                     | Notes on Basic LLM Chain Data Extractor | None                           | None                            | Basic LLM Chain Data Extractor                                                                     |
| Sticky Note3                    | Sticky Note                     | Notes on Summarization Chain            | None                           | None                            | Summarization Chain                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.  
   - No parameters needed.

2. **Create Set Node: "Set Wikipedia URL with Bright Data Zone"**  
   - Type: Set  
   - Parameters:  
     - Assign variable `url` with the Wikipedia article URL (e.g., `https://en.wikipedia.org/wiki/Cloud_computing?product=unlocker&method=api`).  
     - Assign variable `zone` with Bright Data zone name (e.g., `web_unlocker1`).  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node: "Wikipedia Web Request"**  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: POST  
     - URL: `https://api.brightdata.com/request`  
     - Authentication: Generic HTTP Header Auth (configure credentials with Bright Data API key).  
     - Body Parameters (JSON):  
       - `zone`: Expression `{{$json.zone}}`  
       - `url`: Expression `{{$json.url}}`  
       - `format`: `"raw"`  
   - Connect output of Set Node to this node.

4. **Create Google Gemini Chat Model Node: "Google Gemini Chat Model2"**  
   - Type: LangChain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: `models/gemini-2.0-pro-exp`  
     - Credentials: Google Gemini API key (PaLM API account).  
   - No input connections (used as AI model provider).

5. **Create LangChain LLM Chain Node: "LLM Data Extractor"**  
   - Type: LangChain LLM Chain  
   - Parameters:  
     - Text input: Expression `{{$json.data}}` (raw HTML from previous HTTP request).  
     - Prompt: "You are an expert Data Formatter. Make sure to format the data in a human readable manner. Please output the human readable content without your own thoughts."  
     - Enable output parser.  
   - Connect:  
     - Input from `Wikipedia Web Request` node (main input).  
     - AI Language Model: connect to `Google Gemini Chat Model2`.  
   - Output connects to next summarization node.

6. **Create Google Gemini Chat Model Node: "Google Gemini Chat Model For Summarization"**  
   - Type: LangChain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash-exp`  
     - Credentials: Google Gemini API key (PaLM API account).  
   - No input connections (used as AI model provider).

7. **Create LangChain Summarization Chain Node: "Concise Summary Generator"**  
   - Type: LangChain Summarization Chain  
   - Parameters:  
     - Summarization prompt:  
       ```
       Write a concise summary of the following:

       "{text}"
       ```  
     - Chunking mode: Advanced  
   - Connect:  
     - Input from `LLM Data Extractor` (main input).  
     - AI Language Model: connect to `Google Gemini Chat Model For Summarization`.  
   - Output connects to webhook notifier.

8. **Create HTTP Request Node: "Summary Webhook Notifier"**  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: POST (default)  
     - URL: Set to your webhook endpoint (default example: `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7`)  
     - Send Body: true  
     - Body Parameters: JSON with field `summary` set to expression `{{$json.response.text}}` (summary text).  
   - Connect input from `Concise Summary Generator`.

9. **Credentials Setup:**  
   - Configure Bright Data API credentials as Generic HTTP Header Auth with your Bright Data API key.  
   - Configure Google Gemini API credentials (PaLM API key) for both Gemini Chat Model nodes.

10. **Testing:**  
    - Trigger the workflow manually via the Manual Trigger node.  
    - Verify the summary is posted to the webhook endpoint.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This template deals with Wikipedia data extraction and summarization using Bright Data and Google Gemini AI. The LLM Data Extractor node produces human-readable content, and the Concise Summary Generator node creates a concise summary of the extracted info. | Workflow purpose and usage notes.                                                                  |
| Sign up at Bright Data and create a Web Unlocker zone under Scraping Solutions to enable Wikipedia scraping.                                                                                                                                                      | https://brightdata.com/                                                                            |
| Google Gemini API key (PaLM API) is required for AI processing. You may replace Google Gemini with OpenAI or other LLM providers as needed.                                                                                                                      | Replace Google Gemini with OpenAI or suitable LLM providers.                                       |
| Update the Wikipedia URL and Bright Data zone in the "Set Wikipedia URL with Bright Data Zone" node before running. Also update the webhook URL in "Summary Webhook Notifier" to your target endpoint.                                                               | Customization instructions.                                                                        |
| The workflow supports integration with Slack, Discord, Telegram, MS Teams, or internal APIs by changing the webhook URL in the final node.                                                                                                                      | Integration flexibility via webhook.                                                              |
| The workflow uses advanced chunking mode in summarization to handle large Wikipedia articles efficiently.                                                                                                                                                         | Summarization chain configuration detail.                                                         |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration to enable reproduction, customization, and troubleshooting by advanced users and automation agents alike.