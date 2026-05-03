Google Search Engine Results Page Extraction and Summarization with Bright Data

https://n8nworkflows.xyz/workflows/google-search-engine-results-page-extraction-and-summarization-with-bright-data-3533


# Google Search Engine Results Page Extraction and Summarization with Bright Data

### 1. Workflow Overview

This workflow automates the extraction, cleaning, summarization, and formatting of Google Search Engine Results Pages (SERPs) using Bright Data’s proxy-based API and Google Gemini AI models. It targets professionals and teams needing real-time, structured insights from Google Search results without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Query Setup**  
  Receives manual trigger input and sets the Google Search query and proxy zone parameters.

- **1.2 Google Search Execution via Bright Data**  
  Sends a programmatic Google Search request through Bright Data’s Web Unlocker API using proxy authentication.

- **1.3 Data Extraction and Cleaning**  
  Uses an LLM-powered Information Extractor node to clean raw HTML/CSS/JS from the search results and extract structured textual content.

- **1.4 Summarization of Search Results**  
  Applies a summarization chain powered by Google Gemini Flash model to generate concise summaries of the extracted search data.

- **1.5 AI Agent Formatting and Webhook Delivery**  
  An AI Agent formats the summarized search results into a JSON-compatible structure and sends the final output to a configured webhook endpoint.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Query Setup

- **Overview:**  
  This block initiates the workflow manually and sets the search query and proxy zone parameters for the Google Search request.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Google Search Query

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually for testing or ad-hoc runs.  
    - Configuration: No parameters; triggers workflow execution on demand.  
    - Input: None  
    - Output: Triggers the next node, "Set Google Search Query".  
    - Edge Cases: None typical; manual trigger requires user interaction.

  - **Set Google Search Query**  
    - Type: Set  
    - Role: Defines the search query string and the Bright Data proxy zone to be used.  
    - Configuration:  
      - `search_query`: Default value "Bright Data" (can be customized).  
      - `zone`: Default value "serp_api1" (Bright Data Web Unlocker zone).  
    - Input: Trigger from manual node.  
    - Output: Passes parameters to the HTTP Request node for Google Search.  
    - Edge Cases: Incorrect or empty query strings may cause no or irrelevant search results.

---

#### 2.2 Google Search Execution via Bright Data

- **Overview:**  
  Executes the Google Search request using Bright Data’s proxy API to bypass restrictions and scrape SERP data programmatically.

- **Nodes Involved:**  
  - Perform Google Search Request

- **Node Details:**

  - **Perform Google Search Request**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Bright Data’s API endpoint to perform the Google Search.  
    - Configuration:  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Body Parameters:  
        - `zone`: Dynamic from input (e.g., "serp_api1")  
        - `url`: Google Search URL constructed with encoded `search_query`  
        - `format`: "raw" (returns raw HTML content)  
      - Authentication: Header Authentication using Bright Data credentials configured in n8n.  
      - Headers: Set via generic header auth credentials.  
    - Input: Receives `search_query` and `zone` from previous node.  
    - Output: Raw HTML response from Google Search results.  
    - Edge Cases:  
      - Authentication failure if credentials are invalid.  
      - Network timeouts or API rate limits from Bright Data.  
      - Invalid zone or malformed URL parameters.  
      - Google blocking or CAPTCHA challenges despite proxy.

---

#### 2.3 Data Extraction and Cleaning

- **Overview:**  
  Cleans the raw HTML response to extract meaningful textual content using a Large Language Model (LLM) based Information Extractor.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Google Search Data Extractor

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides LLM capabilities to assist in extracting and cleaning data.  
    - Configuration:  
      - Model: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Gemini API key (PaLM API)  
    - Input: Receives raw HTML data from HTTP Request node.  
    - Output: Feeds cleaned data into the Information Extractor node.  
    - Edge Cases: API quota limits, latency, or model errors.

  - **Google Search Data Extractor**  
    - Type: LangChain Information Extractor  
    - Role: Uses LLM to strip HTML, CSS, and scripts, producing clean textual data.  
    - Configuration:  
      - Input text: Raw HTML from Google Search response.  
      - System prompt: Expert HTML extractor to remove markup and extract text.  
      - Output attribute: `textual_response` containing cleaned text.  
    - Input: Raw HTML from HTTP Request, LLM assistance from Gemini Chat Model.  
    - Output: Cleaned textual search results passed to Summarization Chain.  
    - Edge Cases:  
      - Failure to clean complex or obfuscated HTML.  
      - LLM misinterpretation or incomplete extraction.

---

#### 2.4 Summarization of Search Results

- **Overview:**  
  Generates a concise summary of the cleaned search results using a summarization chain powered by Google Gemini.

- **Nodes Involved:**  
  - Summarization Chain  
  - Google Gemini Chat Model For Summarization

- **Node Details:**

  - **Google Gemini Chat Model For Summarization**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides LLM summarization capabilities.  
    - Configuration:  
      - Model: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Gemini API key  
    - Input: Receives cleaned text from the Information Extractor.  
    - Output: Feeds into the Summarization Chain node.  
    - Edge Cases: API limits, summarization quality variance.

  - **Summarization Chain**  
    - Type: LangChain Summarization Chain  
    - Role: Processes input text to produce a concise summary.  
    - Configuration: Default summarization options.  
    - Input: Cleaned text from extractor and LLM model.  
    - Output: Summary text passed to AI Agent for formatting.  
    - Edge Cases: Overly brief or vague summaries; failure on very large inputs.

---

#### 2.5 AI Agent Formatting and Webhook Delivery

- **Overview:**  
  The AI Agent formats the summarized search results into a structured JSON format and sends the final output to a webhook endpoint for downstream consumption.

- **Nodes Involved:**  
  - Google Search Expert AI Agent  
  - Google Gemini Chat Model1  
  - Webhook HTTP Request

- **Node Details:**

  - **Google Gemini Chat Model1**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Supports the AI Agent with language model capabilities.  
    - Configuration:  
      - Model: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Gemini API key  
    - Input: Receives prompt and data from AI Agent node.  
    - Output: Returns formatted JSON-compatible text.  
    - Edge Cases: Model errors or API limits.

  - **Google Search Expert AI Agent**  
    - Type: LangChain Agent  
    - Role: Acts as a virtual assistant to understand, format, and prepare search results for delivery.  
    - Configuration:  
      - Prompt: Defines role as Google Search Expert to format results and prepare webhook payload.  
      - Input text: Injects cleaned and summarized search results dynamically.  
    - Input: Receives summarized search results.  
    - Output: Sends formatted data to Webhook HTTP Request node.  
    - Edge Cases: Formatting errors, incomplete data, or webhook failures.

  - **Webhook HTTP Request**  
    - Type: LangChain HTTP Request Tool  
    - Role: Sends the final structured summary and search results to a configured webhook endpoint.  
    - Configuration:  
      - URL: Default set to `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7` (replaceable).  
      - Method: POST  
      - Body: Includes `search_summary` and `search_result` fields with AI Agent output.  
      - Sends full body payload.  
    - Input: Receives formatted JSON from AI Agent.  
    - Output: None (terminal node).  
    - Edge Cases: Network errors, invalid webhook URL, payload size limits.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                                  | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                               |
|-------------------------------|--------------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                       | Entry point to start workflow manually          | None                          | Set Google Search Query          |                                                                                                           |
| Set Google Search Query        | Set                                  | Defines search query and proxy zone              | When clicking ‘Test workflow’  | Perform Google Search Request    |                                                                                                           |
| Perform Google Search Request  | HTTP Request                         | Executes Google Search via Bright Data API       | Set Google Search Query        | Google Search Data Extractor     |                                                                                                           |
| Google Gemini Chat Model       | LangChain Google Gemini Chat Model  | Provides LLM support for data extraction         | Perform Google Search Request  | Google Search Data Extractor     |                                                                                                           |
| Google Search Data Extractor   | LangChain Information Extractor     | Cleans raw HTML to extract textual data          | Perform Google Search Request, Google Gemini Chat Model | Summarization Chain             |                                                                                                           |
| Google Gemini Chat Model For Summarization | LangChain Google Gemini Chat Model  | Provides LLM summarization capabilities           | Google Search Data Extractor   | Summarization Chain             |                                                                                                           |
| Summarization Chain            | LangChain Summarization Chain        | Summarizes cleaned search results                 | Google Search Data Extractor, Google Gemini Chat Model For Summarization | Google Search Expert AI Agent    |                                                                                                           |
| Google Gemini Chat Model1      | LangChain Google Gemini Chat Model  | Supports AI Agent formatting                       | Google Search Expert AI Agent  | Google Search Expert AI Agent    |                                                                                                           |
| Google Search Expert AI Agent  | LangChain Agent                     | Formats search results and prepares webhook payload | Summarization Chain           | Webhook HTTP Request            |                                                                                                           |
| Webhook HTTP Request           | LangChain HTTP Request Tool          | Sends final formatted data to webhook             | Google Search Expert AI Agent  | None                           |                                                                                                           |
| Sticky Note                   | Sticky Note                         | Notes on Bright Data Google Search SERP usage     | None                          | None                           | ## Bright Data Google Search SERP (Search Engine Results Page)\n\nDeals with the Google Search using the Bright Data Web Scraper API.\n\nThe Information Extraction, Summarization and AI Agent are being used to demonstrate the usage of the N8N AI capabilities.\n\n**Please make sure to Set the Google Search Query and update the Webhook Notification URL** |
| Sticky Note1                  | Sticky Note                         | Notes on LLM usage and AI capabilities             | None                          | None                           | ## LLM Usages\n\nGoogle Gemini Flash Exp model is being used.\n\nGoogle Search Data Extractor using the n8n Infromation Extractor node.\n\nSummarization Chain is being used for the summarization of search results.\n\nThe AI Agent formats the search result and pushes it to the Webhook via HTTP Request |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node: "Set Google Search Query"**  
   - Type: Set  
   - Parameters:  
     - `search_query` (string): Default "Bright Data" (customizable)  
     - `zone` (string): Default "serp_api1" (Bright Data Web Unlocker zone)  
   - Connect Manual Trigger → Set Node.

3. **Create HTTP Request Node: "Perform Google Search Request"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.brightdata.com/request`  
     - Method: POST  
     - Body Parameters:  
       - `zone`: Use expression to get from previous node `{{$json["zone"]}}`  
       - `url`: Construct Google Search URL with encoded query: `https://www.google.com/search?q={{ encodeURI($json["search_query"]) }}`  
       - `format`: "raw"  
     - Authentication: Generic Header Authentication  
   - Credentials: Configure Bright Data Header Auth credentials with your Bright Data API key.  
   - Connect Set Node → HTTP Request Node.

4. **Create LangChain Google Gemini Chat Model Node: "Google Gemini Chat Model"**  
   - Type: LangChain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: "models/gemini-2.0-flash-exp"  
   - Credentials: Google Gemini (PaLM) API key.  
   - Connect HTTP Request → Google Gemini Chat Model (as ai_languageModel input).

5. **Create LangChain Information Extractor Node: "Google Search Data Extractor"**  
   - Type: Information Extractor  
   - Parameters:  
     - Text: Use expression to get raw HTML from HTTP Request node `{{$json["data"]}}`  
     - System Prompt Template: "You are an expert HTML extractor. Your job is to analyze the search result and strip out the html, css, scripts and produce a textual data."  
     - Output Attribute: `textual_response`  
   - Connect HTTP Request → Information Extractor (main input)  
   - Connect Google Gemini Chat Model → Information Extractor (ai_languageModel input).

6. **Create LangChain Google Gemini Chat Model Node: "Google Gemini Chat Model For Summarization"**  
   - Same configuration as previous Gemini Chat Model node.  
   - Connect Information Extractor → Google Gemini Chat Model For Summarization (ai_languageModel input).

7. **Create LangChain Summarization Chain Node: "Summarization Chain"**  
   - Type: Summarization Chain  
   - Parameters: Default options.  
   - Connect Information Extractor → Summarization Chain (main input)  
   - Connect Google Gemini Chat Model For Summarization → Summarization Chain (ai_languageModel input).

8. **Create LangChain Agent Node: "Google Search Expert AI Agent"**  
   - Type: Agent  
   - Parameters:  
     - Prompt Type: Define  
     - Text:  
       ```
       You are an expert Google Search Expert. You need to format the search result and push it to the Webhook via HTTP Request. Here is the search result - {{ $('Google Search Data Extractor').item.json.output.textual_response }}
       ```  
   - Connect Summarization Chain → Google Search Expert AI Agent (main input).

9. **Create LangChain Google Gemini Chat Model Node: "Google Gemini Chat Model1"**  
   - Same configuration as other Gemini Chat Model nodes.  
   - Connect Google Search Expert AI Agent → Google Gemini Chat Model1 (ai_languageModel input).

10. **Create LangChain HTTP Request Tool Node: "Webhook HTTP Request"**  
    - Type: HTTP Request Tool  
    - Parameters:  
      - URL: Set your webhook URL (default example: `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7`)  
      - Method: POST  
      - Send Body: true  
      - Body Parameters:  
        - `search_summary`: Expression `={{ $json.response.text }}` (from AI Agent output)  
        - `search_result`: (empty or as needed)  
    - Connect Google Search Expert AI Agent → Webhook HTTP Request (ai_tool input).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses Bright Data’s Web Unlocker API to programmatically perform Google Search queries via proxy, enabling scraping without manual intervention.                                                                  | https://brightdata.com/                                                                             |
| Google Gemini Flash Exp model (PaLM API) is used for all LLM tasks including data extraction, summarization, and AI agent formatting.                                                                                          | Google Gemini API (PaLM)                                                                            |
| To customize the search query, modify the "Set Google Search Query" node’s `search_query` parameter.                                                                                                                             | Workflow customization instructions                                                                |
| The webhook URL in "Webhook HTTP Request" node must be updated to your target endpoint for receiving the structured search results and summaries.                                                                              | Replace default webhook.site URL                                                                    |
| The workflow demonstrates n8n’s AI capabilities integrating LangChain nodes with external APIs for advanced automation of search data processing.                                                                             | n8n AI integration                                                                                   |
| For proxy setup, create a Web Unlocker zone in Bright Data’s dashboard under Proxies & Scraping → Scraping Solutions → Web Unlocker API.                                                                                        | https://brightdata.com/                                                                               |
| Credentials setup: Bright Data Header Auth credentials and Google Gemini API credentials must be configured in n8n before running the workflow.                                                                                 | n8n Credentials configuration                                                                       |
| This workflow can be extended to accept dynamic input from Google Sheets, Airtable, or forms, and output results to Slack, CRM, or Google Docs by replacing the input and webhook nodes accordingly.                             | Customization ideas                                                                                  |

---

This document provides a comprehensive reference to understand, reproduce, and customize the Google Search Engine Results Page Extraction and Summarization workflow using Bright Data and Google Gemini AI models in n8n.