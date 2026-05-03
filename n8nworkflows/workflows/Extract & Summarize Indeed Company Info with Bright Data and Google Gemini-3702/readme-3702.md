Extract & Summarize Indeed Company Info with Bright Data and Google Gemini

https://n8nworkflows.xyz/workflows/extract---summarize-indeed-company-info-with-bright-data-and-google-gemini-3702


# Extract & Summarize Indeed Company Info with Bright Data and Google Gemini

### 1. Workflow Overview

This workflow automates the extraction and summarization of company profile information from Indeed using Bright Data’s Web Unlocker and Google Gemini’s large language model (LLM). It targets recruiters, HR teams, job seekers, and market researchers who need quick, summarized insights about companies without manual browsing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Query Setup**: Manual trigger and setting the Indeed company search query and Bright Data zone.
- **1.2 Data Extraction via Bright Data Web Unlocker**: Sending a POST request to Bright Data’s API to scrape the Indeed company profile page.
- **1.3 Data Transformation & Parsing**: Converting the raw markdown response into structured textual data.
- **1.4 Summarization with Google Gemini LLM**: Using Google Gemini models to summarize the extracted textual data.
- **1.5 AI Agent Formatting & Webhook Forwarding**: Formatting the summarized data and pushing it to a specified webhook endpoint.
- **1.6 Optional HTML Conversion & Notification**: Converting markdown to HTML and sending it to a webhook for notification or visualization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Query Setup

- **Overview:**  
  This block initiates the workflow manually and sets the search parameters for the Indeed company profile extraction, including the company name and Bright Data zone.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Indeed Search Query (Set Node)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers workflow on manual action.  
    - Inputs: None  
    - Outputs: Connects to Set Indeed Search Query  
    - Edge Cases: None specific, but manual trigger requires user action.

  - **Set Indeed Search Query**  
    - Type: Set  
    - Role: Defines the search query and Bright Data zone parameters.  
    - Configuration:  
      - `search_query`: Default value "Starbucks" (can be customized)  
      - `zone`: Default Bright Data zone "web_unlocker1" (must match configured zone)  
    - Inputs: From Manual Trigger  
    - Outputs: To Perform Indeed Web Request  
    - Edge Cases: Incorrect or empty search query or zone will cause scraping failure.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides instructions and reminders about setting the search query and webhook URL.  
    - Content Highlights:  
      - Reminder to set Indeed search query and webhook notification URL.  
      - Explains usage of Bright Data Web Unlocker and LLM chains.

#### 2.2 Data Extraction via Bright Data Web Unlocker

- **Overview:**  
  This block sends a POST request to Bright Data’s Web Unlocker API to scrape the Indeed company profile page using the search query and zone set previously.

- **Nodes Involved:**  
  - Perform Indeed Web Request (HTTP Request)  
  - Sticky Note1 (Instructional)

- **Node Details:**

  - **Perform Indeed Web Request**  
    - Type: HTTP Request  
    - Role: Sends POST request to Bright Data API to scrape Indeed company profile page.  
    - Configuration:  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Authentication: Header Auth (Generic HTTP Header Auth)  
      - Body Parameters:  
        - `zone`: from input JSON (e.g., "web_unlocker1")  
        - `url`: Constructed Indeed company URL using encoded search query, e.g., `https://www.indeed.com/cmp/Starbucks?product=unlocker&method=api`  
        - `format`: "raw"  
        - `data_format`: "markdown"  
      - Header Parameters: Configured via credentials  
    - Inputs: From Set Indeed Search Query  
    - Outputs: To Markdown to Textual Data Extractor and Convert Markdown to HTML  
    - Edge Cases:  
      - Authentication failure if credentials invalid.  
      - API rate limits or quota exceeded.  
      - Invalid zone or malformed URL causing scraping failure.  
      - Network timeouts or Bright Data service downtime.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains usage of Google Gemini Flash Exp model and AI chains for data extraction and summarization.  
    - Content Highlights:  
      - Mentions Basic LLM Chain Data Extractor and Summarization Chain.  
      - AI Agent formats and pushes results to webhook.

#### 2.3 Data Transformation & Parsing

- **Overview:**  
  Converts the raw markdown response from Bright Data into structured textual data for further processing by the LLM.

- **Nodes Involved:**  
  - Markdown to Textual Data Extractor (Chain LLM)  

- **Node Details:**

  - **Markdown to Textual Data Extractor**  
    - Type: Chain LLM (Langchain)  
    - Role: Analyzes markdown input and converts it into clean textual data.  
    - Configuration:  
      - Prompt: "You are a markdown expert" with input text from `data` field of HTTP response.  
      - Input Expression: `=You need to analyze the below markdown and convert to textual data.\n\n{{ $json.data }}`  
    - Inputs: From Perform Indeed Web Request (HTTP Request)  
    - Outputs: To Indeed Summarization  
    - Edge Cases:  
      - Failure if input markdown is malformed or empty.  
      - LLM API errors or timeouts.

#### 2.4 Summarization with Google Gemini LLM

- **Overview:**  
  Summarizes the textual data extracted from Indeed company profiles to produce a concise, human-readable summary.

- **Nodes Involved:**  
  - Indeed Summarization (Chain Summarization)  
  - Google Gemini Chat Model For Summarization (LM Chat Google Gemini)

- **Node Details:**

  - **Google Gemini Chat Model For Summarization**  
    - Type: LM Chat Google Gemini  
    - Role: Provides the language model backend for summarization.  
    - Configuration:  
      - Model: "models/gemini-2.0-flash-exp" (Google Gemini Flash Exp model)  
      - Credentials: Google Gemini (PaLM) API key  
    - Inputs: Connected as AI language model for Indeed Summarization  
    - Outputs: To Indeed Summarization  
    - Edge Cases:  
      - API key invalid or expired.  
      - Model unavailability or quota exceeded.  
      - Latency or timeouts.

  - **Indeed Summarization**  
    - Type: Chain Summarization (Langchain)  
    - Role: Uses the Google Gemini model to summarize the textual data.  
    - Configuration: Default summarization chain options.  
    - Inputs: From Markdown to Textual Data Extractor  
    - Outputs: To Indeed Expert AI Agent and Initiate a Webhook Notification for Summarization  
    - Edge Cases: Same as Google Gemini Chat Model.

#### 2.5 AI Agent Formatting & Webhook Forwarding

- **Overview:**  
  An AI agent formats the summarized data and sends it to a specified webhook endpoint for downstream consumption.

- **Nodes Involved:**  
  - Indeed Expert AI Agent (Langchain Agent)  
  - Google Gemini Chat Model for AI Agent (LM Chat Google Gemini)  
  - Initiate a Webhook Request (Langchain Tool HTTP Request)  
  - Initiate a Webhook Notification for Summarization (HTTP Request)

- **Node Details:**

  - **Google Gemini Chat Model for AI Agent**  
    - Type: LM Chat Google Gemini  
    - Role: Provides LLM for the AI agent to format and prepare the final output.  
    - Configuration: Same as other Gemini nodes.  
    - Inputs: AI language model input for Indeed Expert AI Agent  
    - Outputs: To Indeed Expert AI Agent  
    - Edge Cases: Same as other Gemini nodes.

  - **Indeed Expert AI Agent**  
    - Type: Langchain Agent  
    - Role: Formats the search result and pushes it via HTTP Request tool.  
    - Configuration:  
      - Text prompt includes the textual data extracted from Markdown to Textual Data Extractor node.  
      - Uses AI tool to send HTTP request.  
    - Inputs: From Indeed Summarization and Google Gemini Chat Model for AI Agent  
    - Outputs: None (terminal node for AI agent)  
    - Edge Cases:  
      - LLM errors or malformed prompt.  
      - HTTP request failures.

  - **Initiate a Webhook Request**  
    - Type: Langchain Tool HTTP Request  
    - Role: Sends the formatted summary and search result to the configured webhook URL.  
    - Configuration:  
      - URL: `https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467` (placeholder, customizable)  
      - Method: POST  
      - Body Parameters:  
        - `search_summary`: summary text from AI agent response  
        - `search_result`: raw or formatted search result (optional)  
    - Inputs: From Indeed Expert AI Agent (AI tool)  
    - Outputs: To Indeed Expert AI Agent (no further nodes)  
    - Edge Cases:  
      - Webhook URL unreachable or incorrect.  
      - Network errors or timeouts.

  - **Initiate a Webhook Notification for Summarization**  
    - Type: HTTP Request  
    - Role: Sends the summarized text directly to a webhook for notification purposes.  
    - Configuration:  
      - URL: same webhook as above (placeholder)  
      - Method: POST  
      - Body Parameter: `summary` from Indeed Summarization response text  
    - Inputs: From Indeed Summarization  
    - Outputs: None  
    - Edge Cases: Same as Initiate a Webhook Request.

#### 2.6 Optional HTML Conversion & Notification

- **Overview:**  
  Converts the raw markdown response into HTML and sends it to a webhook for visualization or notification.

- **Nodes Involved:**  
  - Convert Markdown to HTML (Markdown Node)  
  - Initiate a Webhook Notification for Markdown to HTML Response (HTTP Request)

- **Node Details:**

  - **Convert Markdown to HTML**  
    - Type: Markdown Node  
    - Role: Converts markdown data from Bright Data response into HTML format.  
    - Configuration:  
      - Mode: markdownToHtml  
      - Input: `data` field from HTTP response  
    - Inputs: From Perform Indeed Web Request  
    - Outputs: To Initiate a Webhook Notification for Markdown to HTML Response  
    - Edge Cases: Malformed markdown input.

  - **Initiate a Webhook Notification for Markdown to HTML Response**  
    - Type: HTTP Request  
    - Role: Sends the HTML formatted response to a webhook endpoint.  
    - Configuration:  
      - URL: same webhook placeholder  
      - Method: POST  
      - Body Parameter: `html_response` containing HTML data  
    - Inputs: From Convert Markdown to HTML  
    - Outputs: None  
    - Edge Cases: Webhook failures or network issues.

---

### 3. Summary Table

| Node Name                                  | Node Type                        | Functional Role                                   | Input Node(s)                        | Output Node(s)                                         | Sticky Note                                                                                          |
|--------------------------------------------|---------------------------------|--------------------------------------------------|------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’               | Manual Trigger                  | Entry point to start workflow manually            | None                               | Set Indeed Search Query                                |                                                                                                    |
| Set Indeed Search Query                      | Set                            | Sets search query and Bright Data zone            | When clicking ‘Test workflow’      | Perform Indeed Web Request                             | Reminder to set Indeed search query and webhook URL                                                |
| Perform Indeed Web Request                   | HTTP Request                   | Sends POST request to Bright Data Web Unlocker API | Set Indeed Search Query             | Markdown to Textual Data Extractor, Convert Markdown to HTML | Explains usage of Google Gemini and AI chains                                                      |
| Markdown to Textual Data Extractor           | Chain LLM (Langchain)          | Converts markdown to textual data                  | Perform Indeed Web Request          | Indeed Summarization                                  |                                                                                                    |
| Indeed Summarization                         | Chain Summarization (Langchain) | Summarizes textual data using Google Gemini LLM   | Markdown to Textual Data Extractor  | Indeed Expert AI Agent, Initiate a Webhook Notification for Summarization |                                                                                                    |
| Google Gemini Chat Model For Summarization  | LM Chat Google Gemini          | Provides LLM backend for summarization             | Indeed Summarization (AI languageModel) | Indeed Summarization                                  |                                                                                                    |
| Indeed Expert AI Agent                       | Langchain Agent                | Formats summary and pushes to webhook              | Indeed Summarization, Google Gemini Chat Model for AI Agent | None                                                  |                                                                                                    |
| Google Gemini Chat Model for AI Agent       | LM Chat Google Gemini          | Provides LLM for AI agent formatting                | Indeed Expert AI Agent (AI languageModel) | Indeed Expert AI Agent                                |                                                                                                    |
| Initiate a Webhook Request                   | Langchain Tool HTTP Request    | Sends formatted summary to webhook                  | Indeed Expert AI Agent (AI tool)   | None                                                  |                                                                                                    |
| Initiate a Webhook Notification for Summarization | HTTP Request                   | Sends summary text to webhook for notification     | Indeed Summarization                | None                                                  |                                                                                                    |
| Convert Markdown to HTML                     | Markdown Node                 | Converts markdown to HTML                            | Perform Indeed Web Request          | Initiate a Webhook Notification for Markdown to HTML Response |                                                                                                    |
| Initiate a Webhook Notification for Markdown to HTML Response | HTTP Request                   | Sends HTML response to webhook                       | Convert Markdown to HTML            | None                                                  |                                                                                                    |
| Sticky Note                                  | Sticky Note                   | Instructional note                                  | None                               | None                                                  | Deals with Indeed scraping using Bright Data Web Unlocker; reminder to set search query and webhook |
| Sticky Note1                                 | Sticky Note                   | Instructional note                                  | None                               | None                                                  | Explains LLM usage including Google Gemini Flash Exp model and AI chains                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start of workflow.

2. **Add Set Node**  
   - Name: "Set Indeed Search Query"  
   - Add two string fields:  
     - `search_query` (default: "Starbucks")  
     - `zone` (default: "web_unlocker1")  
   - Connect output of Manual Trigger to this node.

3. **Add HTTP Request Node for Bright Data**  
   - Name: "Perform Indeed Web Request"  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic HTTP Header Auth (configure credentials with Bright Data API key)  
   - Body Parameters (form or JSON):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `=https://www.indeed.com/cmp/{{ encodeURI($json.search_query) }}?product=unlocker&method=api`  
     - `format`: "raw"  
     - `data_format`: "markdown"  
   - Connect output of Set Indeed Search Query to this node.

4. **Add Markdown Node**  
   - Name: "Convert Markdown to HTML"  
   - Mode: markdownToHtml  
   - Input: `={{ $json.data }}` from Perform Indeed Web Request  
   - Connect output of Perform Indeed Web Request to this node.

5. **Add HTTP Request Node for Markdown HTML Notification**  
   - Name: "Initiate a Webhook Notification for Markdown to HTML Response"  
   - Method: POST  
   - URL: Set your webhook URL (e.g., https://webhook.site/your-url)  
   - Body Parameter: `html_response` = `={{ $json.data }}` from Convert Markdown to HTML  
   - Connect output of Convert Markdown to HTML to this node.

6. **Add Chain LLM Node**  
   - Name: "Markdown to Textual Data Extractor"  
   - Prompt: "You are a markdown expert"  
   - Input Text: `=You need to analyze the below markdown and convert to textual data.\n\n{{ $json.data }}` from Perform Indeed Web Request  
   - Connect output of Perform Indeed Web Request to this node.

7. **Add LM Chat Google Gemini Node**  
   - Name: "Google Gemini Chat Model For Summarization"  
   - Model: "models/gemini-2.0-flash-exp"  
   - Credentials: Configure Google Gemini (PaLM) API credentials  
   - Connect as AI language model input to "Indeed Summarization" node (next step).

8. **Add Chain Summarization Node**  
   - Name: "Indeed Summarization"  
   - Connect output of "Markdown to Textual Data Extractor" to this node.  
   - Connect AI language model input from "Google Gemini Chat Model For Summarization".

9. **Add HTTP Request Node for Summarization Notification**  
   - Name: "Initiate a Webhook Notification for Summarization"  
   - Method: POST  
   - URL: Your webhook URL  
   - Body Parameter: `summary` = `={{ $json.response.text }}` from Indeed Summarization  
   - Connect output of Indeed Summarization to this node.

10. **Add LM Chat Google Gemini Node for AI Agent**  
    - Name: "Google Gemini Chat Model for AI Agent"  
    - Model: "models/gemini-2.0-flash-exp"  
    - Credentials: Same as above  
    - Connect as AI language model input to "Indeed Expert AI Agent".

11. **Add Langchain Agent Node**  
    - Name: "Indeed Expert AI Agent"  
    - Prompt:  
      `You are an Indeed Expert. You need to format the search result and push it to the Webhook via HTTP Request. Here is the search result - {{ $('Markdown to Textual Data Extractor').item.json.text }}`  
    - Connect AI language model input from "Google Gemini Chat Model for AI Agent".  
    - Connect main input from "Indeed Summarization".

12. **Add Langchain Tool HTTP Request Node**  
    - Name: "Initiate a Webhook Request"  
    - URL: Your webhook URL  
    - Method: POST  
    - Body Parameters:  
      - `search_summary`: `={{ $json.response.text }}` (from AI agent output)  
      - `search_result`: (optional, raw or formatted data)  
    - Connect AI tool input from "Indeed Expert AI Agent".

13. **Add Sticky Notes**  
    - Add two sticky notes with the content provided, placed near relevant nodes for documentation and reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Deals with the Indeed Company web scraping by utilizing Bright Data Web Unlocker Product. The Basic LLM Chain, Summarization and AI Agent demonstrate n8n AI capabilities. Please set the Indeed search query and webhook URL. | Sticky Note near input and query setup nodes.                                                  |
| Google Gemini Flash Exp model is used for data extraction and summarization. The AI Agent formats and pushes results to webhook via HTTP Request.                                                        | Sticky Note near LLM and AI processing nodes.                                                  |
| Bright Data setup instructions: Sign up at https://brightdata.com/, create Web Unlocker zone under Proxies & Scraping, configure Header Auth credentials in n8n.                                        | Setup section in workflow description.                                                         |
| Google Gemini API setup: Configure Google Gemini (PaLM) API credentials in n8n or access via Vertex AI or proxy.                                                                                          | Setup section in workflow description.                                                         |
| Webhook endpoints can be customized to Slack, Notion, Airtable, or any custom webhook for downstream processing.                                                                                         | Customization notes in workflow description.                                                   |
| Workflow tags: Engineering, AI, HR.                                                                                                                                                                      | Metadata for categorization.                                                                    |

---

This documentation provides a detailed, stepwise understanding of the "Extract & Summarize Indeed Company Info with Bright Data and Google Gemini" workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.