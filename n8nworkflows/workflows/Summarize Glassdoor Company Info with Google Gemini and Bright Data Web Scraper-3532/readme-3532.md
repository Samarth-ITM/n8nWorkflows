Summarize Glassdoor Company Info with Google Gemini and Bright Data Web Scraper

https://n8nworkflows.xyz/workflows/summarize-glassdoor-company-info-with-google-gemini-and-bright-data-web-scraper-3532


# Summarize Glassdoor Company Info with Google Gemini and Bright Data Web Scraper

### 1. Workflow Overview

This workflow automates the extraction, summarization, and delivery of Glassdoor company reviews using the Bright Data Web Scraper API and Google Gemini AI model. It is designed for HR professionals, employer branding teams, and analysts who need real-time, actionable insights from employee reviews without manual data sifting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Trigger:** Manual trigger to start the workflow.
- **1.2 Glassdoor Data Extraction:** Initiates a scraping request to Bright Data’s Glassdoor dataset, polls for completion, and downloads the snapshot data.
- **1.3 Data Preparation and Summarization:** Processes the downloaded data, splits text for AI consumption, and sends it to Google Gemini for summarization.
- **1.4 Output Delivery:** Sends the summarized insights to a configured webhook endpoint for further use or notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  This block provides a manual trigger to start the entire workflow on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually initiate the workflow.  
    - Configuration: No parameters; triggers workflow execution when clicked.  
    - Inputs: None  
    - Outputs: Connected to the HTTP Request to Glassdoor node.  
    - Edge Cases: None, as it is manual.  
    - Notes: Serves as the user-initiated start.

#### 2.2 Glassdoor Data Extraction

- **Overview:**  
  This block handles the interaction with Bright Data’s API to request Glassdoor data, monitor the scraping job status, and download the completed snapshot.

- **Nodes Involved:**  
  - HTTP Request to Glassdoor  
  - Set Snapshot Id  
  - Check Snapshot Status  
  - If  
  - Wait for 30 seconds  
  - Download the Snapshot Response  
  - Sticky Note (Glassdoor data extraction explanation)

- **Node Details:**  

  - **HTTP Request to Glassdoor**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Bright Data API to trigger scraping of a specific Glassdoor company page.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON array with Glassdoor URL (default example: Apple UK Glassdoor page)  
      - Query Parameters: dataset_id for Bright Data Glassdoor dataset, include_errors=true  
      - Authentication: Header Authentication with Bright Data API key  
    - Inputs: Manual Trigger node  
    - Outputs: Set Snapshot Id node  
    - Edge Cases: API authentication failure, invalid URL, dataset_id errors, network timeouts.

  - **Set Snapshot Id**  
    - Type: Set  
    - Role: Extracts and stores the snapshot_id from the HTTP Request response for subsequent polling.  
    - Configuration: Assigns `snapshot_id` from JSON response.  
    - Inputs: HTTP Request to Glassdoor  
    - Outputs: Check Snapshot Status  
    - Edge Cases: Missing or malformed snapshot_id in response.

  - **Check Snapshot Status**  
    - Type: HTTP Request  
    - Role: Polls Bright Data API to check the progress status of the scraping job using the snapshot_id.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Method: GET  
      - Authentication: Header Authentication  
    - Inputs: Set Snapshot Id, Wait for 30 seconds (looped)  
    - Outputs: If node  
    - Edge Cases: API errors, snapshot_id invalid, network issues, rate limiting.

  - **If**  
    - Type: If  
    - Role: Checks if the snapshot status returned by the polling is "ready".  
    - Configuration: Condition: `$json.status == "ready"` (case-sensitive, strict)  
    - Inputs: Check Snapshot Status  
    - Outputs:  
      - True: Download the Snapshot Response  
      - False: Wait for 30 seconds  
    - Edge Cases: Unexpected status values, missing status field.

  - **Wait for 30 seconds**  
    - Type: Wait  
    - Role: Pauses workflow execution for 30 seconds before re-polling snapshot status.  
    - Configuration: 30 seconds delay  
    - Inputs: If (False branch)  
    - Outputs: Check Snapshot Status  
    - Edge Cases: Workflow timeout if polling takes too long.

  - **Download the Snapshot Response**  
    - Type: HTTP Request  
    - Role: Downloads the completed snapshot data in JSON format from Bright Data API.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $('HTTP Request to Glassdoor').item.json.snapshot_id }}`  
      - Query Parameter: format=json  
      - Timeout: 10 seconds  
      - Authentication: Header Authentication  
    - Inputs: If (True branch)  
    - Outputs: Summarization of Glassdoor Response  
    - Edge Cases: Timeout, invalid snapshot_id, incomplete data.

  - **Sticky Note (Glassdoor data extraction explanation)**  
    - Content: Explains the use of Bright Data Web Scraper API for Glassdoor data extraction and mentions the use of the summarization chain for AI capabilities.

#### 2.3 Data Preparation and Summarization

- **Overview:**  
  This block prepares the downloaded Glassdoor data for AI summarization by loading and splitting the text, then sends it to Google Gemini for generating a concise summary.

- **Nodes Involved:**  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Google Gemini Chat Model  
  - Summarization of Glassdoor Response  
  - Sticky Note1 (LLM usage explanation)

- **Node Details:**  

  - **Recursive Character Text Splitter**  
    - Type: Text Splitter (Recursive Character)  
    - Role: Splits large text documents into smaller chunks with overlap to optimize AI processing.  
    - Configuration: Chunk overlap set to 100 characters  
    - Inputs: None directly connected; used internally by Default Data Loader  
    - Outputs: Default Data Loader (ai_textSplitter)  
    - Edge Cases: Improper splitting if input text is malformed or empty.

  - **Default Data Loader**  
    - Type: Document Default Data Loader  
    - Role: Loads the downloaded JSON data and prepares it for summarization.  
    - Configuration: Default options (no custom settings)  
    - Inputs: Recursive Character Text Splitter (ai_textSplitter)  
    - Outputs: Summarization of Glassdoor Response (ai_document)  
    - Edge Cases: Data format issues, empty documents.

  - **Google Gemini Chat Model**  
    - Type: Language Model (Google Gemini)  
    - Role: Processes text chunks and generates a summary using Google Gemini experimental model.  
    - Configuration:  
      - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21` (experimental flash thinking model)  
      - Credentials: Google Palm API key configured under credentials  
    - Inputs: Summarization of Glassdoor Response (ai_languageModel)  
    - Outputs: Summarization of Glassdoor Response  
    - Edge Cases: API key invalid, rate limits, model errors, network failures.

  - **Summarization of Glassdoor Response**  
    - Type: Chain Summarization  
    - Role: Orchestrates the summarization chain combining document loading, splitting, and language model summarization.  
    - Configuration: Operation mode set to documentLoader  
    - Inputs: Default Data Loader (ai_document), Google Gemini Chat Model (ai_languageModel), Download the Snapshot Response (main)  
    - Outputs: Configure Webhook Notification  
    - Edge Cases: Chain failures, empty input, AI model errors.

  - **Sticky Note1 (LLM usage explanation)**  
    - Content: Notes the use of Google Gemini Flash Experimental model and the summarization chain for content summarization.

#### 2.4 Output Delivery

- **Overview:**  
  This block sends the AI-generated summary to a configured webhook endpoint for notifications or integration with HR dashboards.

- **Nodes Involved:**  
  - Configure Webhook Notification

- **Node Details:**  

  - **Configure Webhook Notification**  
    - Type: HTTP Request  
    - Role: Sends the summarized text to a webhook URL (e.g., Slack, Notion, or custom endpoint).  
    - Configuration:  
      - URL: `https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7` (example webhook for testing)  
      - Method: POST (default)  
      - Body: JSON with parameter `summary` set to the summarized text from AI response (`{{$json.response.text}}`)  
      - Sends body as JSON  
    - Inputs: Summarization of Glassdoor Response  
    - Outputs: None (end of workflow)  
    - Edge Cases: Webhook URL invalid, network failures, payload size limits.

---

### 3. Summary Table

| Node Name                      | Node Type                                  | Functional Role                               | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                      |
|--------------------------------|--------------------------------------------|-----------------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                             | Manual start of workflow                       | None                            | HTTP Request to Glassdoor       |                                                                                                 |
| HTTP Request to Glassdoor       | HTTP Request                              | Trigger Bright Data Glassdoor scraping job    | When clicking ‘Test workflow’   | Set Snapshot Id                 |                                                                                                 |
| Set Snapshot Id                | Set                                       | Extracts snapshot_id from scraping response   | HTTP Request to Glassdoor       | Check Snapshot Status           |                                                                                                 |
| Check Snapshot Status          | HTTP Request                              | Polls Bright Data API for scraping job status | Set Snapshot Id, Wait for 30s   | If                            |                                                                                                 |
| If                            | If                                        | Checks if scraping job status is "ready"      | Check Snapshot Status           | Download the Snapshot Response, Wait for 30 seconds |                                                                                                 |
| Wait for 30 seconds            | Wait                                      | Delay before re-polling status                 | If (False branch)               | Check Snapshot Status           |                                                                                                 |
| Download the Snapshot Response | HTTP Request                              | Downloads completed Glassdoor data snapshot   | If (True branch)                | Summarization of Glassdoor Response |                                                                                                 |
| Recursive Character Text Splitter | Text Splitter (Recursive Character)     | Splits text into chunks for AI processing     | None (used internally)          | Default Data Loader             |                                                                                                 |
| Default Data Loader            | Document Default Data Loader               | Loads and prepares data for summarization     | Recursive Character Text Splitter | Summarization of Glassdoor Response |                                                                                                 |
| Google Gemini Chat Model       | Language Model (Google Gemini)             | Generates summary using Google Gemini AI      | Summarization of Glassdoor Response | Summarization of Glassdoor Response | Gemini Experimental Model                                                                       |
| Summarization of Glassdoor Response | Chain Summarization                      | Orchestrates summarization chain               | Default Data Loader, Google Gemini Chat Model, Download the Snapshot Response | Configure Webhook Notification | LLM Usages: Google Gemini Flash Exp model is being used. Summarization Chain is used for summarization |
| Configure Webhook Notification | HTTP Request                              | Sends summary to webhook endpoint              | Summarization of Glassdoor Response | None                          |                                                                                                 |
| Sticky Note                   | Sticky Note                               | Explains Glassdoor data extraction and AI use | None                           | None                          | Deals with the Glassdoor data extraction by using the Bright Data Web Scrapper API. The summarization chain is being used to demonstrate the usage of the N8N AI capabilities. |
| Sticky Note1                  | Sticky Note                               | Notes on LLM usage                             | None                           | None                          | LLM Usages: Google Gemini Flash Exp model is being used. Summarization Chain is used for summarization |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add HTTP Request Node: HTTP Request to Glassdoor**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     [
       {
         "url": "https://www.glassdoor.co.uk/Overview/Working-at-Apple-EI_IE1138.11,16.htm"
       }
     ]
     ```  
   - Query Parameters:  
     - dataset_id: `gd_l7j0bx501ockwldaqf`  
     - include_errors: `true`  
   - Authentication: Generic Header Authentication with Bright Data API key credentials  
   - Connect output to Set Snapshot Id node.

3. **Add Set Node: Set Snapshot Id**  
   - Type: Set  
   - Purpose: Extract `snapshot_id` from HTTP Request response JSON.  
   - Assignments:  
     - Variable: `snapshot_id`  
     - Value: `={{ $json.snapshot_id }}`  
   - Connect output to Check Snapshot Status node.

4. **Add HTTP Request Node: Check Snapshot Status**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Authentication: Generic Header Authentication with Bright Data API key  
   - Connect output to If node.

5. **Add If Node**  
   - Type: If  
   - Condition: Check if `$json.status == "ready"` (string equals, case-sensitive)  
   - True branch connects to Download the Snapshot Response node.  
   - False branch connects to Wait for 30 seconds node.

6. **Add Wait Node: Wait for 30 seconds**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect output back to Check Snapshot Status node (loop).

7. **Add HTTP Request Node: Download the Snapshot Response**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $('HTTP Request to Glassdoor').item.json.snapshot_id }}`  
   - Query Parameter: format = json  
   - Timeout: 10 seconds  
   - Authentication: Generic Header Authentication with Bright Data API key  
   - Connect output to Summarization of Glassdoor Response node.

8. **Add Recursive Character Text Splitter Node**  
   - Type: Text Splitter (Recursive Character)  
   - Chunk Overlap: 100 characters  
   - This node is used internally by Default Data Loader.

9. **Add Default Data Loader Node**  
   - Type: Document Default Data Loader  
   - Connect Recursive Character Text Splitter output to Default Data Loader’s ai_textSplitter input.

10. **Add Google Gemini Chat Model Node**  
    - Type: Language Model (Google Gemini)  
    - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21`  
    - Credentials: Google Palm API key configured in n8n credentials  
    - Connect output to Summarization of Glassdoor Response node’s ai_languageModel input.

11. **Add Chain Summarization Node: Summarization of Glassdoor Response**  
    - Type: Chain Summarization  
    - Operation Mode: documentLoader  
    - Connect Default Data Loader output to ai_document input.  
    - Connect Download the Snapshot Response main output to main input.  
    - Connect Google Gemini Chat Model output to ai_languageModel input.  
    - Connect output to Configure Webhook Notification node.

12. **Add HTTP Request Node: Configure Webhook Notification**  
    - Type: HTTP Request  
    - Method: POST (default)  
    - URL: Your webhook URL (e.g., `https://webhook.site/your-unique-id`)  
    - Body Parameters:  
      - Name: `summary`  
      - Value: `={{ $json.response.text }}` (the summarized text from AI response)  
    - Connect input from Summarization of Glassdoor Response node.

13. **Credential Setup**  
    - Configure Bright Data API key as Generic Header Authentication credential in n8n.  
    - Configure Google Palm API key for Google Gemini Chat Model node.  
    - Ensure webhook endpoint is accessible and ready to receive POST requests.

14. **Optional: Add Sticky Notes**  
    - Add notes explaining the Glassdoor data extraction process and AI summarization usage for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Sign up at [Bright Data](https://brightdata.com/) and create a Web Unlocker zone for Glassdoor scraping.       | Setup instructions for Bright Data API usage.                                                  |
| Google Gemini API key or access via Vertex AI is required for summarization.                                    | Credential setup for AI summarization.                                                         |
| Webhook or endpoint can be Slack, Notion, custom HR dashboard, or any HTTP receiver for summary notifications. | Integration options for receiving summarized insights.                                         |
| Customize summarization prompts to focus on culture, leadership, compensation, or exit motivations.             | Tailor AI output to specific HR or business intelligence needs.                                |
| Integrate with HR systems like BambooHR, Workday, SAP SuccessFactors, Google Sheets, or Airtable via APIs.      | Extend workflow to automate data flow into existing HR or BI platforms.                         |
| Workflow demonstrates n8n AI capabilities using summarization chain and Google Gemini experimental model.      | Useful for users exploring AI integration in n8n workflows.                                   |

---

This detailed reference document enables advanced users and AI agents to fully understand, reproduce, and customize the "Summarize Glassdoor Company Info with Google Gemini and Bright Data Web Scraper" workflow efficiently.