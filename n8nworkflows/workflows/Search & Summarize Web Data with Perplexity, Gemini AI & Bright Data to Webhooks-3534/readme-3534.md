Search & Summarize Web Data with Perplexity, Gemini AI & Bright Data to Webhooks

https://n8nworkflows.xyz/workflows/search---summarize-web-data-with-perplexity--gemini-ai---bright-data-to-webhooks-3534


# Search & Summarize Web Data with Perplexity, Gemini AI & Bright Data to Webhooks

### 1. Workflow Overview

This n8n workflow automates the process of performing Google searches via Perplexity.ai using Bright Data's proxy scraping service, then extracts, cleans, summarizes, formats, and finally sends the results to a configurable webhook endpoint.

**Target Use Cases:**  
- Professionals and teams needing automated, real-time insights from Google search results.  
- Use cases requiring transform raw search data into structured summaries or JSON-formatted reports, then delivery to CRM, Slack, Google Sheets, or other target systems via webhooks.

**Logical Blocks:**

- **1.1 Input Reception & Search Trigger:** Starts the workflow and triggers a Perplexity search request via Bright Data API.  
- **1.2 Search Snapshot Monitoring:** Monitors the search execution status via snapshot polling until ready, including error checks and delays.  
- **1.3 Data Download & Extraction:** Downloads the search results snapshot JSON, then extracts readable plain text from embedded HTML content using AI-powered cleaning.  
- **1.4 Data Summarization:** Splits extracted text into manageable chunks, then summarizes the entire content using Google Gemini AI via a Summarization Chain.  
- **1.5 Result Delivery:** Sends the final summarized and structured output to a configured webhook endpoint.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Search Trigger

**Overview:**  
This block manually triggers the workflow and initiates a Perplexity search query by invoking Bright Data’s dataset trigger API with configured query parameters.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Perplexity Search Request  
- Set Snapshot Id  

**Node Details:**  

- *When clicking ‘Test workflow’*  
  - Type: Manual trigger node  
  - Role: Starts workflow manually for testing or ad hoc runs  
  - Configuration: Default, no parameters  
  - Inputs: none  
  - Outputs: Perplexity Search Request

- *Perplexity Search Request*  
  - Type: HTTP Request (POST)  
  - Role: Sends search trigger request to Bright Data's Perplexity API to start a search job  
  - Configuration:  
    - URL: https://api.brightdata.com/datasets/v3/trigger  
    - Method: POST  
    - Body (JSON): Contains URL (perplexity.ai), prompt (search query), and country param (e.g. US)  
    - QueryParams: dataset_id (Bright Data dataset), include_errors=true  
    - Authentication: Header Auth with Bright Data credentials  
  - Key expressions: Uses $json to extract data for snapshot_id on response  
  - Inputs: Manual Trigger  
  - Outputs: Set Snapshot Id

- *Set Snapshot Id*  
  - Type: Set node  
  - Role: Saves the snapshot_id returned from search trigger for downstream polling  
  - Configuration: Assigns variable `snapshot_id` with the value from API response  
  - Inputs: Perplexity Search Request  
  - Outputs: Check Snapshot Status

**Edge cases / Failures:**  
- API auth invalid or expired credentials (fail HTTP request).  
- Empty or malformed response missing snapshot_id.  
- Network timeouts.

---

#### 2.2 Search Snapshot Monitoring

**Overview:**  
Polls Bright Data's API to monitor the progress of the triggered search snapshot until the status is "ready", with wait delays between polls and error handling.

**Nodes Involved:**  
- Check Snapshot Status  
- If (status check)  
- Wait  
- Check on the errors  

**Node Details:**  

- *Check Snapshot Status*  
  - Type: HTTP Request (GET)  
  - Role: Queries Bright Data API for snapshot progress status using snapshot_id  
  - Configuration:  
    - URL composed dynamically with snapshot_id, e.g. https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}  
    - Authentication: Header Auth credentials  
  - Inputs: Set Snapshot Id or Wait node  
  - Outputs: If node

- *If*  
  - Type: If conditional node  
  - Role: Checks if snapshot status from API response equals "ready" (string comparison)  
  - Configuration: Combinator AND with the single condition: `{{$json.status}} === "ready"`  
  - Inputs: Check Snapshot Status  
  - Outputs:  
    - True -> Check on the errors  
    - False -> Wait node

- *Wait*  
  - Type: Wait node  
  - Role: Pauses execution for 30 seconds before next polling attempt  
  - Configuration: 30 seconds delay  
  - Inputs: If node (when status != ready)  
  - Outputs: Check Snapshot Status (loops polling)

- *Check on the errors*  
  - Type: If conditional node  
  - Role: Checks if there are errors reported in the snapshot response (`errors` count equals 0)  
  - Configuration: Combinator AND for `{{$json.errors.toString()}} === "0"`  
  - Inputs: If node (when status == ready)  
  - Outputs:  
    - True (no errors) -> Download Snapshot  
    - False (errors present) -> Wait node (retry polling)

**Edge cases / Failures:**  
- Snapshot status never reaches "ready" causing infinite polling (manual workflow stop may be required).  
- Errors returned from Bright Data API snapshot (could be search failure).  
- Network issues or credential failures during repeated polling.

---

#### 2.3 Data Download & Extraction

**Overview:**  
Downloads the snapshot search result JSON file, then uses an AI-powered information extractor to clean up embedded HTML content into readable text.

**Nodes Involved:**  
- Download Snapshot  
- Google Gemini Chat Model1  
- Readable Data Extractor  

**Node Details:**  

- *Download Snapshot*  
  - Type: HTTP Request (GET)  
  - Role: Downloads the actual snapshot JSON data from Bright Data using snapshot_id  
  - Configuration:  
    - URL constructed dynamically: https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}  
    - Query parameter: format=json  
    - Authentication: Header Auth credentials  
    - Timeout: 10 seconds  
  - Inputs: Check on the errors (when no errors)  
  - Outputs: Readable Data Extractor

- *Google Gemini Chat Model1*  
  - Type: Langchain Google Gemini Chat Model  
  - Role: AI model used to assist with cleaning and extracting readable content from raw HTML  
  - Configuration:  
    - Model: models/gemini-2.0-flash-exp (experimental flash model)  
    - Credentials: Google Gemini API Key (PaLM)  
  - Inputs: Passed text content (`answer_html` field) from Download Snapshot  
  - Outputs: Readable Data Extractor

- *Readable Data Extractor*  
  - Type: Langchain Information Extractor  
  - Role: Uses Google Gemini AI to extract and clean readable content from noisy HTML  
  - Configuration:  
    - Input text from JSON field `answer_html`  
    - Specifies attribute "readable content" for cleaned output  
  - Inputs: Download Snapshot raw data + Gemini Chat Model1 for AI language understanding  
  - Outputs: Summarization of search result

**Edge cases / Failures:**  
- Raw JSON missing `answer_html` field leads to extractor failure.  
- AI model quota or key failures (Google PaLM).  
- HTML complexity causing incomplete extraction or malformed output.

---

#### 2.4 Data Summarization

**Overview:**  
This block splits large extracted text into manageable chunks, loads them as documents, and summarizes them using Google Gemini AI model through a summarization chain.

**Nodes Involved:**  
- Recursive Character Text Splitter  
- Default Data Loader  
- Google Gemini Chat Model (primary summarization model)  
- Summarization of search result  

**Node Details:**  

- *Recursive Character Text Splitter*  
  - Type: Langchain Text Splitter  
  - Role: Splits large texts into chunks with overlap to ensure context is preserved during summarization  
  - Configuration:  
    - Chunk Overlap: 100 characters  
    - Default chunk size (implied) - Recursive strategy  
  - Inputs: readable content from Readable Data Extractor  
  - Outputs: Default Data Loader

- *Default Data Loader*  
  - Type: Langchain Document Data Loader  
  - Role: Converts text chunks into document objects for summarization chain input  
  - Configuration: Default options  
  - Inputs: Recursive Character Text Splitter  
  - Outputs: Summarization of search result

- *Google Gemini Chat Model*  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Primary AI summarization engine producing concise summaries of loaded documents  
  - Configuration:  
    - Model: models/gemini-2.0-flash-thinking-exp-01-21 (experimental "flash thinking" model)  
    - Credentials: Google Gemini API key (PaLM)  
  - Inputs: Summarization chain input  
  - Outputs: Summarization of search result

- *Summarization of search result*  
  - Type: Langchain Summarization Chain  
  - Role: Manages the summarization process by feeding data loader documents into AI model  
  - Configuration: Operation mode set as document loader  
  - Inputs: Default Data Loader (documents) and Google Gemini Chat Model (AI)  
  - Outputs: Webhook Notifier

**Edge cases / Failures:**  
- AI summarization quotas or rate limits impacting completions.  
- Very large inputs triggering timeouts or incomplete chunk processing.  
- Model version or API breaking changes.

---

#### 2.5 Result Delivery

**Overview:**  
Sends the final summarized search result to a user-configured webhook URL for consumption by third-party apps, CRMs, or messaging services.

**Nodes Involved:**  
- Webhook Notifier  

**Node Details:**  

- *Webhook Notifier*  
  - Type: HTTP Request (POST)  
  - Role: Sends summarized data to configured webhook URL  
  - Configuration:  
    - URL: Default placeholder is https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7 (editable)  
    - Method: POST  
    - Body parameters: JSON field `response` populated with workflow final output (`$json.output`)  
    - No authentication by default (can be customized)  
  - Inputs: Summarization of search result  
  - Outputs: None (terminal node)

**Edge cases / Failures:**  
- Invalid or unreachable webhook URL (network errors, 4xx/5xx)  
- Payload too large or malformed webhook payloads  
- Webhook authentication needed but not configured

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                          | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                             |
|-----------------------------|--------------------------------------|----------------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                       | Start workflow manually                 | None                          | Perplexity Search Request    | ## Note  Deals with the Perplexity Search using the Bright Data Web Scrapper API. The information extraction and summarization demonstrate N8N AI capabilities. **Please make sure to update the Webhook Notification URL** |
| Perplexity Search Request    | HTTP Request (POST)                   | Trigger Perplexity search via Bright Data | When clicking ‘Test workflow’ | Set Snapshot Id             |                                                                                                       |
| Set Snapshot Id             | Set                                  | Assign snapshot_id variable             | Perplexity Search Request      | Check Snapshot Status        |                                                                                                       |
| Check Snapshot Status       | HTTP Request (GET)                    | Poll snapshot status from Bright Data   | Set Snapshot Id, Wait          | If                          |                                                                                                       |
| If                         | If                                   | Check if snapshot status == "ready"    | Check Snapshot Status          | Check on the errors, Wait    |                                                                                                       |
| Wait                       | Wait                                 | Delay between polling attempts          | If                            | Check Snapshot Status        |                                                                                                       |
| Check on the errors         | If                                   | Confirm no search errors in snapshot    | If                            | Download Snapshot, Wait      |                                                                                                       |
| Download Snapshot           | HTTP Request (GET)                    | Download search snapshot JSON data      | Check on the errors            | Readable Data Extractor      |                                                                                                       |
| Google Gemini Chat Model1   | Langchain Google Gemini Chat Model   | AI model assisting HTML-to-text cleaning | Download Snapshot             | Readable Data Extractor      | ## LLM Usages  Google Gemini Flash Exp model is used for formatting HTML to text and summarization.  |
| Readable Data Extractor     | Langchain Information Extractor      | Clean HTML content into readable text   | Download Snapshot, Google Gemini Chat Model1 | Summarization of search result |                                                                                                       |
| Recursive Character Text Splitter | Langchain Text Splitter           | Split text into chunks for summarization | Readable Data Extractor        | Default Data Loader          |                                                                                                       |
| Default Data Loader         | Langchain Document Data Loader       | Load text chunks as documents for AI    | Recursive Character Text Splitter | Summarization of search result |                                                                                                       |
| Google Gemini Chat Model    | Langchain Google Gemini Chat Model   | Main AI model generating summaries      | Summarization of search result (documents) | Summarization of search result |                                                                                                       |
| Summarization of search result | Langchain Summarization Chain        | Summarize documents using AI model      | Default Data Loader, Google Gemini Chat Model | Webhook Notifier             |                                                                                                       |
| Webhook Notifier           | HTTP Request (POST)                   | Send final summary to configured webhook | Summarization of search result | None                        |                                                                                                       |
| Sticky Note                | Sticky Note                          | Workflow note regarding data source and webhook URL | None                          | None                        | ## Note  Deals with the Perplexity Search using the Bright Data Web Scrapper API. The information extraction and summarization are done to demonstrate the usage of the N8N AI capabilities. **Please make sure to update the Webhook Notification URL** |
| Sticky Note1               | Sticky Note                          | Notes on LLM usage details               | None                          | None                        | ## LLM Usages  Google Gemini Flash Exp model is being used. Information extraction for formatting HTML to text and summarization chain for content summarization. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node "When clicking ‘Test workflow’"**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually

2. **Add HTTP Request Node "Perplexity Search Request"**  
   - Method: POST  
   - URL: https://api.brightdata.com/datasets/v3/trigger  
   - Body Type: JSON (raw) with parameters:  
     ```json
     [
       {
         "url": "https://www.perplexity.ai",
         "prompt": "tell me about BrightData",
         "country": "US"
       }
     ]
     ```  
   - Query Parameters:  
     - dataset_id = your Bright Data dataset ID (e.g., gd_m7dhdot1vw9a7gc1n)  
     - include_errors = true  
   - Authentication: Header Auth with Bright Data API key (create credentials with header auth)  
   - Connect output from manual trigger

3. **Add Set Node "Set Snapshot Id"**  
   - Add field `snapshot_id` with value expression: `{{$json["snapshot_id"]}}`  
   - Connect output from Perplexity Search Request

4. **Add HTTP Request Node "Check Snapshot Status"**  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{$json["snapshot_id"]}}`  
   - Authentication: Header Auth (same as above)  
   - Connect output from Set Snapshot Id (and later from Wait node for looping)

5. **Add If Node "If" for Checking Status**  
   - Condition: Check if `{{$json.status}}` equals `"ready"` (case-sensitive, strict)  
   - Connect output from Check Snapshot Status

6. **Add Wait Node "Wait"**  
   - Delay: 30 seconds  
   - Connect output from "If" node's "False" branch (status !== ready)  
   - Connect output of Wait back to Check Snapshot Status (loop)

7. **Add If Node "Check on the errors"**  
   - Condition: Check if `{{$json.errors.toString()}}` equals `"0"` (i.e., no errors)  
   - Connect output from If node's "True" branch (snapshot ready)

8. **Connect "Check on the errors"**  
   - True branch: Connect to "Download Snapshot" node  
   - False branch: Connect back to "Wait" node to retry polling

9. **Add HTTP Request Node "Download Snapshot"**  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{$json["snapshot_id"]}}`  
   - Query param: format = json  
   - Authentication: Header Auth  
   - Timeout: 10 seconds  
   - Connect output from "Check on the errors" True

10. **Add Google Gemini Chat Model Node "Google Gemini Chat Model1"**  
    - Model Name: `models/gemini-2.0-flash-exp` (experimental flash)  
    - Credentials: Google PaLM/Gemini API key configured  
    - This node is used in AI language processing of HTML data for extraction  
    - Connect output from "Download Snapshot"

11. **Add Information Extractor Node "Readable Data Extractor"**  
    - Input Text: `{{$json["answer_html"]}}` (HTML data from snapshot)  
    - Attributes: Add attribute named "readable content"  
    - AI Model: Connect this node to "Google Gemini Chat Model1" input  
    - Connect output from "Download Snapshot" and AI connection from "Google Gemini Chat Model1"

12. **Add Recursive Character Text Splitter Node**  
    - Chunk Overlap: 100 characters  
    - Connect output from "Readable Data Extractor" (readable content)

13. **Add Default Data Loader Node**  
    - Standard document loader with default options  
    - Connect output from Text Splitter

14. **Add Google Gemini Chat Model Node "Google Gemini Chat Model"**  
    - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21`  
    - Credentials: Google PaLM/Gemini API key  
    - Connect for AI summarization task

15. **Add Summarization Chain Node "Summarization of search result"**  
    - Operation Mode: document loader  
    - Connect document input from Default Data Loader  
    - Connect AI model input from Google Gemini Chat Model  
    - Output will be final summarized content

16. **Add HTTP Request Node "Webhook Notifier"**  
    - Method: POST  
    - URL: Set your webhook destination URL (default is https://webhook.site/...; change as applicable)  
    - Body Parameters: JSON with key "response" set as `{{$json.output}}` (the summarization result)  
    - No authentication by default (can add custom if needed)  
    - Connect output from Summarization Chain node

17. **Add Sticky Note Nodes (optional)**  
    - Add notes explaining usage and instructions for updating the webhook URL and LLM usage

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                | Context or Link                                                |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow relies on Bright Data Web Unlocker API for proxy-based SERP Google search automation.                                                                                              | Bright Data: https://brightdata.com/                           |
| Google Gemini AI (PaLM) used as experimental flash and flash-thinking models for HTML extraction and summarization. Credentials required in n8n.                                            | Google Vertex AI docs or PaLM API                             |
| Update the webhook endpoint in "Webhook Notifier" node to integrate with third-party apps (Slack, CRM, Google Sheets, etc.).                                                                | Webhook.site example URL is a placeholder                      |
| Can be customized for multi-language summarization, tone adjustments, and input triggers from data sources like Google Sheets or Airtable.                                                 | Workflow description section                                   |
| Sticky notes inside the workflow describe the Perplexity search and AI model usages for better context.                                                                                      | Contains instructions and remarks                              |

---

This structured documentation enables users and AI agents to fully comprehend, reproduce, and extend the workflow’s capabilities, with attention to potential failure points and integration details.