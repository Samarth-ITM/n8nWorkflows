Generate Company Stories from LinkedIn with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/generate-company-stories-from-linkedin-with-bright-data---google-gemini-3540


# Generate Company Stories from LinkedIn with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the generation of company stories from LinkedIn profiles by leveraging Bright Data’s web scraping API and Google Gemini’s AI language model. It is designed to streamline the process of extracting company data from LinkedIn, transforming it into a structured narrative, and delivering the output via webhook for further use.

**Target Use Cases:**  
- Marketing professionals creating company narratives for campaigns  
- Sales teams summarizing potential clients  
- Content creators generating company-based stories or articles  
- Recruiters obtaining concise company overviews  

**Logical Blocks:**  
- **1.1 Input Acquisition:** Receives the LinkedIn company URL manually or via trigger and sets it for processing.  
- **1.2 Data Extraction via Bright Data:** Initiates a scraping job on Bright Data, monitors its progress, and downloads the snapshot data.  
- **1.3 Data Parsing and Story Generation:** Uses AI (Google Gemini) to extract structured company information from raw data and generate a detailed company story.  
- **1.4 Summarization:** Produces a concise summary of the detailed story using AI summarization chains.  
- **1.5 Output Delivery:** Sends the detailed story and summary to configured webhook endpoints for consumption or further automation.  
- **1.6 Control Flow and Error Handling:** Includes checks for scraping errors, waiting for data readiness, and conditional branching to ensure robustness.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Acquisition

- **Overview:**  
  This block initiates the workflow manually and sets the LinkedIn company URL to be processed.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set LinkedIn URL  

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually for testing or execution  
    - Configuration: No parameters; triggers workflow on manual activation  
    - Inputs: None  
    - Outputs: Connects to Set LinkedIn URL node  
    - Edge Cases: None; manual trigger ensures controlled start  

  - **Set LinkedIn URL**  
    - Type: Set  
    - Role: Defines the LinkedIn company URL to be scraped  
    - Configuration: Sets a string variable `url` with the LinkedIn company profile URL (default: `https://il.linkedin.com/company/bright-data`)  
    - Inputs: From manual trigger  
    - Outputs: Connects to Perform LinkedIn Web Request node  
    - Edge Cases: URL must be valid and accessible; incorrect URL leads to scraping failure  

---

#### 1.2 Data Extraction via Bright Data

- **Overview:**  
  This block triggers the Bright Data scraping job, monitors its progress, and downloads the scraped company profile data once ready.

- **Nodes Involved:**  
  - Perform LinkedIn Web Request  
  - Set Snapshot Id  
  - Check Snapshot Status  
  - If (condition on snapshot status)  
  - Check on the errors  
  - Wait for 30 seconds  
  - Download Snapshot  

- **Node Details:**  

  - **Perform LinkedIn Web Request**  
    - Type: HTTP Request  
    - Role: Sends POST request to Bright Data API to trigger scraping of the LinkedIn URL  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON array with the URL to scrape  
      - Query Parameters: `dataset_id` (Bright Data dataset), `include_errors`=true  
      - Authentication: Header Auth (Bright Data API key)  
    - Inputs: Receives `url` from Set LinkedIn URL  
    - Outputs: Returns JSON with `snapshot_id` for the scraping job  
    - Edge Cases: API auth errors, invalid URL, rate limits, network timeouts  

  - **Set Snapshot Id**  
    - Type: Set  
    - Role: Extracts and stores the `snapshot_id` from the previous response for status checking  
    - Configuration: Sets variable `snapshot_id` from incoming JSON  
    - Inputs: From Perform LinkedIn Web Request  
    - Outputs: Connects to Check Snapshot Status  
    - Edge Cases: Missing or invalid snapshot_id leads to failure downstream  

  - **Check Snapshot Status**  
    - Type: HTTP Request  
    - Role: Polls Bright Data API to check if the scraping snapshot is ready  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Method: GET  
      - Authentication: Header Auth  
    - Inputs: Receives `snapshot_id` from Set Snapshot Id or Wait node  
    - Outputs: Returns status JSON (e.g., `ready`)  
    - Edge Cases: API errors, snapshot not found, network issues  

  - **If**  
    - Type: If (conditional)  
    - Role: Checks if snapshot status equals `ready`  
    - Configuration: Condition: `status == "ready"`  
    - Inputs: From Check Snapshot Status  
    - Outputs:  
      - True branch: proceeds to Check on the errors  
      - False branch: loops back to Wait for 30 seconds  
    - Edge Cases: Status other than `ready` or unexpected values  

  - **Check on the errors**  
    - Type: If  
    - Role: Verifies if scraping returned any errors (`errors == 0`)  
    - Configuration: Condition: `errors.toString() == "0"`  
    - Inputs: From If node (True branch)  
    - Outputs:  
      - True branch: proceeds to Download Snapshot  
      - False branch: loops back to Wait for 30 seconds  
    - Edge Cases: Non-zero errors indicate scraping issues, triggering retry  

  - **Wait for 30 seconds**  
    - Type: Wait  
    - Role: Delays workflow to allow scraping to complete before rechecking status  
    - Configuration: Wait time set to 30 seconds  
    - Inputs: From If (False branch) or Check on the errors (False branch)  
    - Outputs: Loops back to Check Snapshot Status  
    - Edge Cases: Excessive wait times if scraping is delayed or stuck  

  - **Download Snapshot**  
    - Type: HTTP Request  
    - Role: Downloads the scraped data snapshot in JSON format  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Query Parameter: `format=json`  
      - Authentication: Header Auth  
    - Inputs: From Check on the errors (True branch)  
    - Outputs: Provides raw scraped data JSON to next block  
    - Edge Cases: Snapshot not ready, API errors, timeout  

---

#### 1.3 Data Parsing and Story Generation

- **Overview:**  
  This block processes the raw scraped data to extract structured company information and generate a detailed company story using Google Gemini AI.

- **Nodes Involved:**  
  - LinkedIn Data Extractor  
  - Google Gemini Chat Model1  

- **Node Details:**  

  - **LinkedIn Data Extractor**  
    - Type: Information Extractor (LangChain)  
    - Role: Uses AI to parse raw scraped JSON data and produce a detailed company story in JSON format  
    - Configuration:  
      - Input Text: Prompt instructing to write a complete story using the provided company info JSON (`{{ $json.input }}`)  
      - System Prompt: "You are an expert data formatter"  
      - Output Attribute: `company_story` (required)  
      - AI Model: Receives input from Google Gemini Chat Model1  
    - Inputs: Raw scraped data from Download Snapshot  
    - Outputs: Detailed company story JSON to Concise Summary Generator and Webhook Notifier for Data Extractor  
    - Edge Cases: AI parsing errors, incomplete data, malformed JSON input  

  - **Google Gemini Chat Model1**  
    - Type: AI Language Model (Google Gemini)  
    - Role: Provides AI capabilities to LinkedIn Data Extractor for information extraction and story generation  
    - Configuration:  
      - Model Name: `models/gemini-2.0-flash-exp` (experimental flash model)  
      - Credentials: Google Gemini API key (PaLM)  
    - Inputs: Feeds AI model with raw data and prompt from LinkedIn Data Extractor  
    - Outputs: AI-generated structured story data  
    - Edge Cases: API quota limits, auth errors, response delays  

---

#### 1.4 Summarization

- **Overview:**  
  This block condenses the detailed company story into a concise summary using an AI summarization chain.

- **Nodes Involved:**  
  - Concise Summary Generator  
  - Default Data Loader  
  - Recursive Character Text Splitter  
  - Google Gemini Chat Model  

- **Node Details:**  

  - **Concise Summary Generator**  
    - Type: Chain Summarization (LangChain)  
    - Role: Generates a concise summary from the detailed company story  
    - Configuration:  
      - Summarization prompts instruct AI to write a concise summary of the input text (`{{ $json.output.company_story }}`)  
      - Operation Mode: Document Loader (uses Default Data Loader)  
    - Inputs: Receives detailed story from LinkedIn Data Extractor (main) and AI model from Google Gemini Chat Model (ai_languageModel)  
    - Outputs: Sends summary text to Webhook Notifier for Summary Generator  
    - Edge Cases: AI summarization errors, incomplete input data  

  - **Default Data Loader**  
    - Type: Document Default Data Loader (LangChain)  
    - Role: Loads text documents for summarization  
    - Configuration: Default options, no special parameters  
    - Inputs: Receives text chunks from Recursive Character Text Splitter  
    - Outputs: Provides documents to Concise Summary Generator  
    - Edge Cases: Empty or malformed text input  

  - **Recursive Character Text Splitter**  
    - Type: Text Splitter (LangChain)  
    - Role: Splits large text into manageable chunks with overlap for AI processing  
    - Configuration:  
      - Chunk Overlap: 100 characters  
      - Default chunk size (implicit)  
    - Inputs: Receives detailed story text from LinkedIn Data Extractor  
    - Outputs: Splits text to Default Data Loader  
    - Edge Cases: Very large texts may cause performance issues  

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini)  
    - Role: Provides AI summarization capabilities  
    - Configuration:  
      - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21` (experimental flash thinking model)  
      - Credentials: Google Gemini API key (PaLM)  
    - Inputs: Receives text chunks from Default Data Loader  
    - Outputs: Provides summarized text to Concise Summary Generator  
    - Edge Cases: API limits, auth errors, latency  

---

#### 1.5 Output Delivery

- **Overview:**  
  This block sends the generated detailed story and concise summary to configured webhook endpoints for further use or integration.

- **Nodes Involved:**  
  - Webhook Notifier for Data Extractor  
  - Webhook Notifier for Summary Generator  

- **Node Details:**  

  - **Webhook Notifier for Data Extractor**  
    - Type: HTTP Request  
    - Role: Sends the detailed company story JSON to a webhook URL  
    - Configuration:  
      - Method: POST  
      - URL: Configured webhook endpoint (default: `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7`)  
      - Body: Sends `response` parameter containing detailed story JSON (`{{ $json.output }}`)  
    - Inputs: From LinkedIn Data Extractor  
    - Outputs: None  
    - Edge Cases: Webhook endpoint unreachable, network errors  

  - **Webhook Notifier for Summary Generator**  
    - Type: HTTP Request  
    - Role: Sends the concise summary text to a webhook URL  
    - Configuration:  
      - Method: POST  
      - URL: Same webhook endpoint as above  
      - Body: Sends `response` parameter containing summary text (`{{ $json.response.text }}`)  
    - Inputs: From Concise Summary Generator  
    - Outputs: None  
    - Edge Cases: Webhook failures, network timeouts  

---

#### 1.6 Control Flow and Error Handling

- **Overview:**  
  This block ensures the workflow handles errors gracefully and waits appropriately for data readiness.

- **Nodes Involved:**  
  - If  
  - Check on the errors  
  - Wait for 30 seconds  

- **Node Details:**  
  These nodes are integrated within the Data Extraction block as described above, controlling retries and error checks.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                                  | Input Node(s)                    | Output Node(s)                          | Sticky Note                                                                                              |
|--------------------------------|--------------------------------------|-------------------------------------------------|---------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                       | Workflow manual start trigger                    | None                            | Set LinkedIn URL                      | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Set LinkedIn URL               | Set                                  | Sets LinkedIn company URL                         | When clicking ‘Test workflow’   | Perform LinkedIn Web Request          | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Perform LinkedIn Web Request   | HTTP Request                        | Triggers Bright Data scraping job                 | Set LinkedIn URL                | Set Snapshot Id                      | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Set Snapshot Id                | Set                                  | Stores snapshot_id for polling                    | Perform LinkedIn Web Request    | Check Snapshot Status                 | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Check Snapshot Status          | HTTP Request                        | Polls Bright Data for snapshot readiness          | Set Snapshot Id / Wait for 30s  | If                                  | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| If                           | If                                   | Checks if snapshot status is ready                | Check Snapshot Status           | Check on the errors / Wait for 30s    | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Check on the errors            | If                                   | Checks for scraping errors                         | If (True branch)                | Download Snapshot / Wait for 30s      | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Wait for 30 seconds            | Wait                                 | Waits before rechecking snapshot status           | If (False branch), Check on errors (False branch) | Check Snapshot Status                 | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Download Snapshot              | HTTP Request                        | Downloads scraped data snapshot                    | Check on the errors (True)      | LinkedIn Data Extractor               | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| LinkedIn Data Extractor        | Information Extractor (LangChain)  | Extracts structured company info and story        | Download Snapshot               | Concise Summary Generator, Webhook Notifier for Data Extractor | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Google Gemini Chat Model1      | AI Language Model (Google Gemini)  | AI engine for data extraction and story generation | LinkedIn Data Extractor (AI input) | LinkedIn Data Extractor              | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |
| Recursive Character Text Splitter | Text Splitter (LangChain)          | Splits text into chunks for AI processing          | LinkedIn Data Extractor         | Default Data Loader                  | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |
| Default Data Loader            | Document Loader (LangChain)         | Loads text chunks for summarization                | Recursive Character Text Splitter | Concise Summary Generator            | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |
| Google Gemini Chat Model       | AI Language Model (Google Gemini)  | AI engine for summarization                         | Default Data Loader             | Concise Summary Generator            | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |
| Concise Summary Generator      | Chain Summarization (LangChain)    | Generates concise summary from detailed story     | LinkedIn Data Extractor, Google Gemini Chat Model | Webhook Notifier for Summary Generator | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |
| Webhook Notifier for Data Extractor | HTTP Request                    | Sends detailed story to webhook                     | LinkedIn Data Extractor         | None                              |                                                                                                        |
| Webhook Notifier for Summary Generator | HTTP Request                  | Sends concise summary to webhook                    | Concise Summary Generator       | None                              |                                                                                                        |
| Sticky Note                   | Sticky Note                        | Notes on LinkedIn data extraction                   | None                          | None                              | Deals with the LinkedIn data extraction using the Bright Data Web Scrapper API.                         |
| Sticky Note1                  | Sticky Note                        | Notes on LLM usage and summarization chain          | None                          | None                              | LLM Usages: Google Gemini Flash Exp model is being used. Information extraction and summarization.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow  

2. **Create a Set Node**  
   - Name: `Set LinkedIn URL`  
   - Purpose: Define the LinkedIn company profile URL  
   - Parameters:  
     - Field: `url`  
     - Value: e.g., `https://il.linkedin.com/company/bright-data`  
   - Connect input from Manual Trigger  

3. **Create an HTTP Request Node**  
   - Name: `Perform LinkedIn Web Request`  
   - Purpose: Trigger Bright Data scraping job  
   - Parameters:  
     - URL: `https://api.brightdata.com/datasets/v3/trigger`  
     - Method: POST  
     - Body Type: JSON  
     - Body Content: `[{"url": "{{ $json.url }}"}]`  
     - Query Parameters:  
       - `dataset_id`: your Bright Data dataset ID (e.g., `gd_l1vikfnt1wgvvqz95w`)  
       - `include_errors`: `true`  
     - Authentication: Generic Header Auth with Bright Data API key  
   - Connect input from Set LinkedIn URL  

4. **Create a Set Node**  
   - Name: `Set Snapshot Id`  
   - Purpose: Extract `snapshot_id` from previous response  
   - Parameters:  
     - Set variable `snapshot_id` to `{{$json.snapshot_id}}`  
   - Connect input from Perform LinkedIn Web Request  

5. **Create an HTTP Request Node**  
   - Name: `Check Snapshot Status`  
   - Purpose: Poll Bright Data for snapshot readiness  
   - Parameters:  
     - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
     - Method: GET  
     - Authentication: Generic Header Auth with Bright Data API key  
   - Connect input from Set Snapshot Id and later from Wait node  

6. **Create an If Node**  
   - Name: `If`  
   - Purpose: Check if snapshot status is `ready`  
   - Condition: `{{$json.status}} == "ready"`  
   - Connect input from Check Snapshot Status  

7. **Create an If Node**  
   - Name: `Check on the errors`  
   - Purpose: Check if `errors == 0` in snapshot response  
   - Condition: `{{$json.errors.toString()}} == "0"`  
   - Connect input from If node (True branch)  

8. **Create a Wait Node**  
   - Name: `Wait for 30 seconds`  
   - Purpose: Delay before rechecking snapshot status  
   - Parameters: Wait 30 seconds  
   - Connect input from If node (False branch) and Check on the errors (False branch)  
   - Output connects back to Check Snapshot Status  

9. **Create an HTTP Request Node**  
   - Name: `Download Snapshot`  
   - Purpose: Download scraped data once ready and error-free  
   - Parameters:  
     - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
     - Query Parameter: `format=json`  
     - Authentication: Generic Header Auth with Bright Data API key  
   - Connect input from Check on the errors (True branch)  

10. **Create a Google Gemini Chat Model Node**  
    - Name: `Google Gemini Chat Model1`  
    - Purpose: AI model for data extraction and story generation  
    - Parameters:  
      - Model Name: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key  
    - Connect as AI input to LinkedIn Data Extractor  

11. **Create an Information Extractor Node**  
    - Name: `LinkedIn Data Extractor`  
    - Purpose: Extract structured company story from raw data  
    - Parameters:  
      - Text: Prompt to write a story using JSON company info (`{{ $json.input }}`)  
      - System Prompt: "You are an expert data formatter"  
      - Output Attribute: `company_story`  
    - Connect input from Download Snapshot  
    - Connect AI input from Google Gemini Chat Model1  

12. **Create a Recursive Character Text Splitter Node**  
    - Name: `Recursive Character Text Splitter`  
    - Purpose: Split detailed story text for summarization  
    - Parameters: Chunk Overlap = 100  
    - Connect input from LinkedIn Data Extractor (main output)  

13. **Create a Default Data Loader Node**  
    - Name: `Default Data Loader`  
    - Purpose: Load text chunks for summarization  
    - Connect input from Recursive Character Text Splitter  

14. **Create a Google Gemini Chat Model Node**  
    - Name: `Google Gemini Chat Model`  
    - Purpose: AI model for summarization  
    - Parameters:  
      - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21`  
    - Credentials: Google Gemini API key  
    - Connect AI input from Default Data Loader  

15. **Create a Chain Summarization Node**  
    - Name: `Concise Summary Generator`  
    - Purpose: Generate concise summary from detailed story  
    - Parameters:  
      - Summarization prompt: "Write a concise summary of the following: {{ $json.output.company_story }}"  
      - Operation Mode: Document Loader  
    - Connect main input from LinkedIn Data Extractor  
    - Connect AI input from Google Gemini Chat Model  

16. **Create HTTP Request Node**  
    - Name: `Webhook Notifier for Data Extractor`  
    - Purpose: Send detailed story JSON to webhook  
    - Parameters:  
      - URL: Your webhook endpoint  
      - Method: POST  
      - Body Parameter: `response` = `{{ $json.output }}`  
    - Connect input from LinkedIn Data Extractor  

17. **Create HTTP Request Node**  
    - Name: `Webhook Notifier for Summary Generator`  
    - Purpose: Send concise summary text to webhook  
    - Parameters:  
      - URL: Your webhook endpoint  
      - Method: POST  
      - Body Parameter: `response` = `{{ $json.response.text }}`  
    - Connect input from Concise Summary Generator  

18. **Add Sticky Notes** (optional)  
    - Add notes on LinkedIn data extraction and LLM usage for documentation  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Deals with the LinkedIn data extraction using the Bright Data Web Scraper API. The information extraction and summarization demonstrate n8n AI usage.  | Workflow sticky note                                                                              |
| Google Gemini Flash Exp model is used for information extraction and summarization chains.                                                               | Workflow sticky note                                                                              |
| Setup instructions: Sign up at Bright Data, create Web Unlocker zone, configure header auth in n8n, obtain Google Gemini API key, update URLs accordingly. | Workflow description and setup section                                                          |
| Bright Data Web Unlocker API documentation: https://brightdata.com/                                                                                      | External resource                                                                                |
| Google Gemini (PaLM) API documentation: https://cloud.google.com/vertex-ai/docs/generative-ai                                                             | External resource                                                                                |
| Webhook.site used for testing webhook outputs: https://webhook.site/                                                                                     | Testing and debugging output delivery                                                           |

---

This completes the detailed structured reference document for the "Generate Company Stories from LinkedIn with Bright Data & Google Gemini" workflow, enabling advanced users and AI agents to understand, reproduce, and modify the workflow effectively.