Email News Briefing by Keyword from Bright Data with AI Summary

https://n8nworkflows.xyz/workflows/email-news-briefing-by-keyword-from-bright-data-with-ai-summary-4570


# Email News Briefing by Keyword from Bright Data with AI Summary

### 1. Workflow Overview

This workflow automates the process of generating an email news briefing based on a user-provided keyword. It integrates with Bright Data’s API to search and retrieve recent news articles from Reuters related to the keyword, processes and cleans the returned data, summarizes the key points using AI (Google Gemini language model), and finally sends a formatted HTML email report to a predefined recipient.

The logical blocks of the workflow are:

- **1.1 Input Reception:** Receive the keyword input from the user via a web form.
- **1.2 Trigger Data Retrieval from Bright Data:** Initiate a dataset query with the keyword and poll the API until results are ready.
- **1.3 Data Processing and Cleaning:** Retrieve the snapshot data, parse, clean, and sort the news articles.
- **1.4 AI Summary Generation:** Use Google Gemini models to generate a concise news briefing summary.
- **1.5 Email Report Construction and Sending:** Convert the summary to HTML, build styled HTML content, and send the email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects the search keyword from a user-submitted web form to start the news retrieval process.

**Nodes Involved:**  
- When User Completes Form

**Node Details:**  
- **When User Completes Form**  
  - Type: Form trigger node  
  - Role: Captures user input via a web form.  
  - Configuration:  
    - Form titled "Search from Reuters by keyword".  
    - Single required field "Keyword" with placeholder example "e.g. 'energy shutdown'".  
    - Ignores bots to prevent spam.  
    - Responds with last node output.  
  - Inputs: External HTTP webhook trigger when form submitted.  
  - Outputs: Keyword data as JSON for downstream nodes.  
  - Edge Cases: Requires keyword field; empty submissions blocked. Form must be publicly accessible and webhook URL correctly configured.

---

#### 2.2 Trigger Data Retrieval from Bright Data

**Overview:**  
This block sends a POST request to Bright Data’s API to trigger a dataset search for news articles matching the keyword, then polls the API until the snapshot data is ready.

**Nodes Involved:**  
- HTTP Request- Post API call to Bright Data  
- Wait - Polling Bright Data  
- Snapshot Progress  
- If - Checking status of Snapshot - if data is ready or not  
- HTTP Request - Getting data from Bright Data  

**Node Details:**  

- **HTTP Request- Post API call to Bright Data**  
  - Type: HTTP Request (POST)  
  - Role: Initiates the search on Bright Data by sending the keyword and parameters.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - Body parameters: keyword (from form input), sort by "relevance" (customizable, see sticky note).  
    - Query parameters: dataset_id fixed to "gd_lyptx9h74wtlvpnfu", type "discover_new", discover_by "keyword", include_errors "true".  
    - Headers: Authorization with Bearer token (user must replace `YOUR_API_KEY` with valid key).  
  - Inputs: From form trigger node.  
  - Outputs: Snapshot ID for progress polling.  
  - Edge Cases: Auth failures if API key invalid; keyword empty (blocked upstream); API errors or rate limiting.  

- **Wait - Polling Bright Data**  
  - Type: Wait node  
  - Role: Pauses the workflow 15 seconds between polling attempts.  
  - Configuration: Wait time 15 seconds, does not execute once, loops while waiting.  
  - Inputs: After POST request trigger.  
  - Outputs: Triggers snapshot progress check.  
  - Edge Cases: If API is slow, may require longer wait; too frequent polling risks rate limiting.  

- **Snapshot Progress**  
  - Type: HTTP Request (GET)  
  - Role: Checks the status of the Bright Data snapshot progress.  
  - Configuration:  
    - URL built dynamically using snapshot_id returned from POST call.  
    - Authorization header with Bearer token.  
  - Inputs: Wait node output.  
  - Outputs: JSON with status field ("running" or other).  
  - Edge Cases: Auth error; invalid snapshot_id; network timeout.

- **If - Checking status of Snapshot - if data is ready or not**  
  - Type: If condition node  
  - Role: Checks if snapshot status is "running".  
  - Configuration: Condition equals "running".  
  - Inputs: Snapshot Progress node output.  
  - Outputs:  
    - If true: loops back to Wait - Polling Bright Data to poll again.  
    - If false: proceeds to get snapshot data.  
  - Edge Cases: Unexpected status values; expression errors.

- **HTTP Request - Getting data from Bright Data**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves the completed snapshot data in JSON format once ready.  
  - Configuration:  
    - URL uses snapshot_id.  
    - Query parameter `format=json`.  
    - Authorization header.  
  - Inputs: If node output when snapshot ready.  
  - Outputs: Raw news data JSON.  
  - Edge Cases: Auth failure; snapshot expired; empty or malformed data.

---

#### 2.3 Data Processing and Cleaning

**Overview:**  
Processes the raw JSON news data, filters out invalid entries, sorts news by publication date descending, and selects top 10 articles with cleaned fields.

**Nodes Involved:**  
- Code - Parse and Clean JSON Data

**Node Details:**  
- **Code - Parse and Clean JSON Data**  
  - Type: Code node (Python)  
  - Role: Parses incoming JSON array, validates date fields, sorts by descending publication date, extracts relevant fields and joins topics list into a string.  
  - Configuration:  
    - Python code iterates over all news, attempts to parse ISO 8601 date strings, skips invalid dates.  
    - Sorts news by date descending.  
    - Selects top 10 news items.  
    - Extracts fields: headline, url, author, publication_date, type, content, keyword, topics (comma-separated).  
  - Inputs: Raw snapshot JSON from Bright Data.  
  - Outputs: JSON with cleaned array under key "news".  
  - Edge Cases: Date parsing failures (handled by skipping); missing fields (fields may be null or empty); empty news list results in empty output.

---

#### 2.4 AI Summary Generation

**Overview:**  
Generates a summarized news briefing text from cleaned news articles using Google Gemini chat and chain language models, focusing on key events, themes, and sentiment related to the keyword.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Google Gemini - Summary Analisys  
- Markdown  

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini Chat)  
  - Role: Optional intermediate conversational model (used as an AI language step, chained to the next).  
  - Configuration: ModelName "models/gemini-2.0-flash".  
  - Inputs: (Connected from Code - Parse and Clean JSON Data indirectly via chain).  
  - Outputs: AI processed text.  
  - Edge Cases: Model API errors; quota limits.

- **Google Gemini - Summary Analisys**  
  - Type: Chain Language Model (LangChain integration)  
  - Role: Main summarization step with a detailed prompt guiding the AI to create a concise news briefing.  
  - Configuration:  
    - Prompt includes instructions to focus on important events, recurring themes, overall sentiment.  
    - References the user’s original keyword from the form node.  
    - Output is a brief integrated summary including date ranges and source links.  
  - Inputs: Cleaned news JSON from Code node and keyword from form node.  
  - Outputs: Markdown text summary under `text` key.  
  - Edge Cases: AI model failures; incomplete data causing weak summary; prompt errors.

- **Markdown**  
  - Type: Markdown node  
  - Role: Converts the AI-generated markdown summary into HTML for email formatting.  
  - Configuration:  
    - Mode set to markdownToHtml.  
    - Input field is AI summary text.  
    - Output key "html".  
  - Inputs: AI summary text JSON.  
  - Outputs: HTML formatted summary.  
  - Edge Cases: Markdown parsing errors; empty input.

---

#### 2.5 Email Report Construction and Sending

**Overview:**  
Builds a styled HTML email from the AI summary HTML content and sends the report to a predefined email address.

**Nodes Involved:**  
- Code - Build HTML  
- Email Report  

**Node Details:**  

- **Code - Build HTML**  
  - Type: Code node (JavaScript)  
  - Role: Wraps the raw HTML from the Markdown node in a complete HTML document with inline CSS styling for fonts, colors, and layout.  
  - Configuration:  
    - Styles for body font, headings, links, lists, spacing.  
    - Inserts AI summary HTML into body.  
  - Inputs: HTML from Markdown node.  
  - Outputs: Full HTML document under "html" key.  
  - Edge Cases: Malformed input HTML causing rendering issues.

- **Email Report**  
  - Type: Email Send node  
  - Role: Sends the final email report with summary content.  
  - Configuration:  
    - Subject: Includes the keyword from the form (e.g. "Your N8N report about Reuters News by keyword: energy shutdown").  
    - To email: hardcoded recipient "your-mail@gmail.com" (user should change).  
    - From email: "n8n-mail@example.com" (replace with valid sender).  
    - HTML body: from Code - Build HTML node.  
  - Inputs: Full HTML email content.  
  - Outputs: Email delivery status.  
  - Edge Cases: SMTP auth failures; invalid email addresses; network errors.

---

**Sticky Note Content:**  
- Visible on the workflow near the initial HTTP request node:  
  "You can customize the sorting filter to 'newest' or 'oldest' if you prefer. However, it's recommended to keep it set to 'relevance' - the results will be sorted by the most recent founding dates in the next steps anyway."

---

### 3. Summary Table

| Node Name                                 | Node Type                            | Functional Role                               | Input Node(s)                               | Output Node(s)                             | Sticky Note                                                                                                                        |
|-------------------------------------------|------------------------------------|-----------------------------------------------|---------------------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When User Completes Form                   | Form Trigger                       | Receive user keyword input                      | (External HTTP trigger)                      | HTTP Request- Post API call to Bright Data  |                                                                                                                                    |
| HTTP Request- Post API call to Bright Data| HTTP Request (POST)                | Trigger Bright Data dataset search              | When User Completes Form                     | Wait - Polling Bright Data                   | You can customize the sorting filter to "newest" or "oldest" if you prefer. However, it's recommended to keep it set to "relevance".|
| Wait - Polling Bright Data                 | Wait                              | Pause between polling attempts                   | HTTP Request- Post API call to Bright Data  | Snapshot Progress                            |                                                                                                                                    |
| Snapshot Progress                          | HTTP Request (GET)                 | Check snapshot status                             | Wait - Polling Bright Data                   | If - Checking status of Snapshot             |                                                                                                                                    |
| If - Checking status of Snapshot - if data is ready or not | If                              | Branch based on snapshot readiness               | Snapshot Progress                            | Wait - Polling Bright Data (if running) / HTTP Request - Getting data from Bright Data (if ready) |                                                                                                                                    |
| HTTP Request - Getting data from Bright Data | HTTP Request (GET)               | Retrieve snapshot news JSON data                   | If - Checking status of Snapshot             | Code - Parse and Clean JSON Data             |                                                                                                                                    |
| Code - Parse and Clean JSON Data           | Code (Python)                     | Clean, filter, and sort news articles             | HTTP Request - Getting data from Bright Data | Google Gemini - Summary Analisys             |                                                                                                                                    |
| Google Gemini Chat Model                    | AI Language Model (Chat)          | Optional AI intermediate processing               | Code - Parse and Clean JSON Data             | Google Gemini - Summary Analisys (ai_languageModel input) |                                                                                                                                    |
| Google Gemini - Summary Analisys            | Chain Language Model (LangChain)  | Generate concise news briefing summary             | Code - Parse and Clean JSON Data / Google Gemini Chat Model | Markdown                                    |                                                                                                                                    |
| Markdown                                   | Markdown                         | Convert AI summary markdown to HTML                | Google Gemini - Summary Analisys             | Code - Build HTML                            |                                                                                                                                    |
| Code - Build HTML                          | Code (JavaScript)                 | Build styled full HTML email content               | Markdown                                    | Email Report                                 |                                                                                                                                    |
| Email Report                              | Email Send                      | Send final email report to user                     | Code - Build HTML                            | (Terminal node)                              |                                                                                                                                    |
| Sticky Note                               | Sticky Note                      | Informational note on sorting configuration        | N/A                                         | N/A                                         | You can customize the sorting filter to "newest" or "oldest" if you prefer. However, it's recommended to keep it set to "relevance".|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**  
   - Type: `When User Completes Form`  
   - Configure the webhook (unique URL generated by n8n).  
   - Form title: "Search from Reuters by keyword"  
   - Add one required field: Label "Keyword", placeholder "e.g. 'energy shutdown'".  
   - Enable "Ignore Bots" to reduce spam inputs.  
   - Response mode: "lastNode".

2. **Create an HTTP Request Node to Trigger Bright Data Search:**  
   - Name: `HTTP Request- Post API call to Bright Data`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: `Authorization: Bearer YOUR_API_KEY` (replace with valid API key)  
   - Query Parameters:  
     - `dataset_id`: "gd_lyptx9h74wtlvpnfu"  
     - `type`: "discover_new"  
     - `discover_by`: "keyword"  
     - `include_errors`: "true"  
   - Body Parameters:  
     - `keyword`: expression `={{ $json["Keyword"] }}` from form input  
     - `sort`: "relevance" (optionally "newest" or "oldest")  
   - Connect output from form trigger to this node’s input.

3. **Create a Wait Node for Polling:**  
   - Name: `Wait - Polling Bright Data`  
   - Type: Wait  
   - Set wait time to 15 seconds.  
   - Connect output of HTTP POST request node here.

4. **Create HTTP Request Node to Check Snapshot Progress:**  
   - Name: `Snapshot Progress`  
   - Type: HTTP Request (GET)  
   - URL: Expression: `=https://api.brightdata.com/datasets/v3/progress/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}`  
   - Headers: `Authorization: Bearer YOUR_API_KEY`  
   - Connect output of Wait node here.

5. **Create an If Node to Check Snapshot Status:**  
   - Name: `If - Checking status of Snapshot - if data is ready or not`  
   - Condition: Check if `$json.status` equals `"running"` (case sensitive)  
   - Connect output of Snapshot Progress node here.  
   - If true (status = running), connect back to `Wait - Polling Bright Data` to poll again.  
   - If false, proceed to next step.

6. **Create HTTP Request Node to Get Snapshot Data:**  
   - Name: `HTTP Request - Getting data from Bright Data`  
   - Type: HTTP Request (GET)  
   - URL: Expression: `=https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query parameter: `format=json`  
   - Headers: `Authorization: Bearer YOUR_API_KEY`  
   - Connect "false" output of If node here.

7. **Create a Python Code Node to Parse and Clean JSON Data:**  
   - Name: `Code - Parse and Clean JSON Data`  
   - Language: Python  
   - Paste the provided code that filters news by valid publication date, sorts descending, takes top 10, and extracts key fields including topics as comma-separated string.  
   - Connect output of snapshot data node here.

8. **Create Google Gemini Chat Model Node (optional intermediate AI step):**  
   - Name: `Google Gemini Chat Model`  
   - Select model: "models/gemini-2.0-flash"  
   - Connect output of code node here.

9. **Create Google Gemini Chain Language Model Node (Summary):**  
   - Name: `Google Gemini - Summary Analisys`  
   - Configure prompt with instructions to generate consolidated summary focusing on important events, themes, sentiment, and including date range and source links.  
   - Reference the keyword from the form node via expression: `{{ $('When User Completes Form').first().json['Keyword'] }}`  
   - Connect output of code node and optionally Google Gemini Chat Model node here.

10. **Create Markdown Node to Convert Summary to HTML:**  
    - Name: `Markdown`  
    - Mode: markdownToHtml  
    - Input markdown text from Google Gemini - Summary Analisys output.  
    - Connect output of summary node here.

11. **Create JavaScript Code Node to Build Full HTML Email:**  
    - Name: `Code - Build HTML`  
    - Paste provided JavaScript code that wraps raw HTML with styles and full HTML structure.  
    - Connect output of Markdown node here.

12. **Create Email Send Node:**  
    - Name: `Email Report`  
    - Configure SMTP credentials or OAuth2 as needed for sending email.  
    - From: e.g. "n8n-mail@example.com" (replace with valid sender)  
    - To: e.g. "your-mail@gmail.com" (replace with recipient)  
    - Subject: Expression including keyword, e.g. `"Your N8N report about Reuters News by keyword: {{ $('When User Completes Form').first().json['Keyword'] }}"`  
    - HTML body: from Code - Build HTML output.  
    - Connect output of Code - Build HTML node here.

13. **Add Sticky Note:**  
    - Add a sticky note near the initial HTTP Request node to inform about sorting options:  
      "You can customize the sorting filter to 'newest' or 'oldest' if you prefer. However, it's recommended to keep it set to 'relevance' - the results will be sorted by the most recent founding dates in the next steps anyway."

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses Bright Data’s dataset API; users must obtain and set their own Bright Data API key in HTTP Request headers.| Bright Data API Documentation: https://brightdata.com/docs/datasets-api                            |
| The AI summarization uses Google Gemini model integrations which require configured credentials in n8n LangChain nodes.    | n8n Google Gemini Node Docs: https://docs.n8n.io/integrations/ai/google-gemini/                      |
| The email node requires valid SMTP credentials or OAuth2 configuration to send emails successfully.                         | n8n Email Node Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.emailSend/   |
| The form trigger node webhook URL must be accessible externally for users to submit keywords.                               | n8n Webhook Documentation: https://docs.n8n.io/integrations/builtin/triggers/n8n-nodes-base.webhook/|

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly sourced.