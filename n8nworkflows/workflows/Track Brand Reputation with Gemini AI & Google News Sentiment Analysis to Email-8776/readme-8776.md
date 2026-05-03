Track Brand Reputation with Gemini AI & Google News Sentiment Analysis to Email

https://n8nworkflows.xyz/workflows/track-brand-reputation-with-gemini-ai---google-news-sentiment-analysis-to-email-8776


# Track Brand Reputation with Gemini AI & Google News Sentiment Analysis to Email

### 1. Workflow Overview

This workflow automates the process of tracking brand reputation by performing sentiment analysis on recent news related to a specific brand or company. It leverages Google Sheets as the input source for brand keywords and email addresses, Google News RSS feeds to gather relevant news headlines, Google Gemini AI (PaLM) for advanced sentiment analysis, and Gmail to deliver a detailed, formatted sentiment report via email.

Logical blocks include:

- **1.1 Input Reception:** Detects new rows in a Google Sheets document containing brand keywords and recipient emails.
- **1.2 News Retrieval:** Fetches recent news headlines related to the brand from Google News RSS feeds.
- **1.3 Data Preparation:** Processes and aggregates news snippets for analysis.
- **1.4 AI Sentiment Analysis:** Sends aggregated news to Google Gemini AI for a witty, nuanced sentiment report.
- **1.5 Post-Processing:** Parses the AI response into readable paragraphs.
- **1.6 Email Delivery:** Sends the sentiment analysis report to the specified email address.
- **1.7 User Guidance:** Provides instructions on setting up and running the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Watches a Google Sheets document for new rows added, each containing a brand keyword and an email address to send the sentiment report.
- **Nodes Involved:** Google Sheets Trigger, Limit
- **Node Details:**

  - **Google Sheets Trigger**
    - Type: Trigger (Google Sheets)
    - Role: Initiates workflow when a new row is added to the specified sheet.
    - Configuration: Polls every minute; monitors sheet “page1” (gid=0) in a specified Google Sheets document URL.
    - Expressions:
      - Uses the Google Sheets OAuth2 credentials.
    - Inputs: None (trigger)
    - Outputs: JSON object with new row data including `keyword` and `email`.
    - Edge cases: Google Sheets API authentication failures; rate limits; new row format mismatches.
  
  - **Limit**
    - Type: Limit node
    - Role: Limits the number of items passed downstream to only the last added item.
    - Configuration: Keeps "lastItems" (default 1).
    - Inputs: From Google Sheets Trigger.
    - Outputs: Passes the most recent new row data.
    - Edge cases: Empty input if no new rows detected; ensures only one trigger per run.

#### 2.2 News Retrieval

- **Overview:** Uses the brand keyword from the sheet to fetch related news headlines from Google News RSS feed.
- **Nodes Involved:** RSS Read, Edit Fields, Aggregate
- **Node Details:**

  - **RSS Read**
    - Type: RSS Feed Read
    - Role: Retrieves news headlines matching the brand keyword.
    - Configuration:
      - URL dynamically set to `https://news.google.com/rss/search?q={{keyword}}&hl=en&gl=US&ceid=US:en`.
      - Uses expression: `$json.keyword` from the trigger node.
    - Inputs: From Limit node.
    - Outputs: Array of news items with fields like `contentSnippet`.
    - Edge cases: Empty or malformed keyword, RSS feed rate limits, network errors.
  
  - **Edit Fields**
    - Type: Set node
    - Role: Prepares news snippet field for aggregation.
    - Configuration:
      - Sets a single field `contentSnippet` with the news snippet text from each RSS item.
      - Uses expression: `$json.contentSnippet`.
    - Inputs: From RSS Read.
    - Outputs: Modified news items with standardized field.
    - Edge cases: Missing or empty `contentSnippet` in RSS feed.
  
  - **Aggregate**
    - Type: Aggregate node
    - Role: Combines all news snippets into a single aggregated dataset for AI input.
    - Configuration:
      - Aggregates by concatenating all `contentSnippet` values.
    - Inputs: From Edit Fields.
    - Outputs: Single item containing all concatenated snippets.
    - Edge cases: No news items found leads to empty aggregation, which may affect AI input.

#### 2.3 AI Sentiment Analysis

- **Overview:** Sends aggregated news snippets to Google Gemini AI to generate a detailed, entertaining sentiment analysis report.
- **Nodes Involved:** Message a model
- **Node Details:**

  - **Message a model**
    - Type: Google Gemini (PaLM) AI node
    - Role: Performs natural language processing and sentiment analysis using advanced AI.
    - Configuration:
      - Model: `models/gemini-2.5-flash` (Gemini 2.5 Flash)
      - Message prompt is a detailed template that instructs the AI to:
        - Summarize what the brand does.
        - Analyze the tone and sentiment of the news headlines.
        - Provide a sentiment score out of 10 for categories: Positive Perception, Trustworthiness, Overall Sentiment.
        - Offer actionable recommendations.
      - Uses expressions referencing:
        - Aggregated news snippets (`$json.contentSnippet`).
        - Brand keyword from the trigger node (`$('Google Sheets Trigger').item.json.keyword`).
    - Inputs: From Aggregate.
    - Outputs: AI-generated textual report.
    - Credentials: Requires Google Gemini API credentials.
    - Edge cases: API authentication issues, rate limits, malformed prompt or excessively long input, AI response errors.

#### 2.4 Post-Processing

- **Overview:** Parses the AI model's multi-paragraph response into an array of separate paragraphs for email formatting.
- **Nodes Involved:** Code in JavaScript
- **Node Details:**

  - **Code in JavaScript**
    - Type: Code node (JavaScript)
    - Role: Splits the AI response text into an array of paragraphs separated by blank lines.
    - Configuration:
      - Extracts the text from `$json.content.parts[0].text`.
      - Splits on double newlines `\n\n`, trims, and filters out empty strings.
      - Returns an object with a `paragraphs` array.
    - Inputs: From Message a model node.
    - Outputs: JSON with `paragraphs` array.
    - Edge cases: Unexpected AI response format, missing text field.

#### 2.5 Email Delivery

- **Overview:** Sends the formatted sentiment analysis report via Gmail to the email specified in the Google Sheets input.
- **Nodes Involved:** Send a message
- **Node Details:**

  - **Send a message**
    - Type: Gmail node (OAuth2)
    - Role: Sends an HTML email with the sentiment report.
    - Configuration:
      - Recipient email: Dynamic from `$('Google Sheets Trigger').item.json.email`.
      - Subject: Includes the brand keyword dynamically.
      - Message body: Joins the paragraphs array into HTML `<p>` tags for readable formatting.
      - Uses Gmail OAuth2 credentials.
    - Inputs: From Code in JavaScript.
    - Outputs: Confirmation of email sent.
    - Edge cases: Authentication failure, invalid email format, Gmail API rate limits, email delivery failure.

#### 2.6 User Guidance

- **Overview:** Provides instructions directly within the workflow for users setting up and running it.
- **Nodes Involved:** Sticky Note
- **Node Details:**

  - **Sticky Note**
    - Type: Sticky Note
    - Role: Displays comprehensive setup instructions and usage notes.
    - Content:
      - How to create the Google Sheets document with correct sheet and columns.
      - Instructions to paste the Google Sheets URL in the trigger node.
      - Credential setup requirements for Google Gemini API and Gmail.
      - Overview of workflow automation and expected behavior.
    - Inputs/Outputs: None (informational only).

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                         | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                           |
|---------------------|-------------------------------|---------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger| Google Sheets Trigger          | Initiates workflow on new sheet row   | None                   | Limit                    | 1. Create Google Sheets with sheet name "page1", columns "keyword" and "email". 2. Insert Google Sheets URL here.                   |
| Limit               | Limit                         | Limits to last new row only            | Google Sheets Trigger   | RSS Read                 | See above                                                                                                                           |
| RSS Read            | RSS Feed Read                 | Fetches news from Google News RSS     | Limit                  | Edit Fields              | See above                                                                                                                           |
| Edit Fields         | Set                          | Extracts and formats news snippet     | RSS Read               | Aggregate                | See above                                                                                                                           |
| Aggregate           | Aggregate                    | Aggregates all snippets into one      | Edit Fields            | Message a model          | See above                                                                                                                           |
| Message a model     | Google Gemini AI (PaLM)       | Performs AI sentiment analysis        | Aggregate              | Code in JavaScript       | 4. Setup Google Gemini (PaLM) API credentials in this node.                                                                        |
| Code in JavaScript  | Code                         | Parses AI response into paragraphs    | Message a model        | Send a message           | See above                                                                                                                           |
| Send a message      | Gmail (OAuth2)                | Sends sentiment report via email      | Code in JavaScript     | None                     | 5. Setup Gmail OAuth2 credentials in this node.                                                                                     |
| Sticky Note         | Sticky Note                  | Provides setup and usage instructions | None                   | None                     | Instructions for Google Sheets, API credentials, and workflow usage provided here.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**
   - Type: Google Sheets Trigger
   - Set event to "Row Added"
   - Poll interval: Every minute
   - Document URL: Paste your Google Sheets document URL
   - Sheet Name: `page1` (or use gid=0)
   - Connect OAuth2 credentials for Google Sheets.
   - Outputs JSON with fields `keyword` and `email`.

2. **Add Limit node:**
   - Type: Limit
   - Configure to keep the last added item only.
   - Connect input from Google Sheets Trigger.
   - Connect output to RSS Read node.

3. **Add RSS Read node:**
   - Type: RSS Feed Read
   - URL parameter: Set dynamically using expression:
     `='https://news.google.com/rss/search?q=' + $json.keyword + '&hl=en&gl=US&ceid=US:en'`
   - Connect input from Limit node.
   - Output to Edit Fields node.

4. **Add Set node ("Edit Fields"):**
   - Type: Set
   - Configure to assign a field named `contentSnippet` with expression: `$json.contentSnippet`
   - Connect input from RSS Read.
   - Output to Aggregate node.

5. **Add Aggregate node:**
   - Type: Aggregate
   - Configure to aggregate the field `contentSnippet` by concatenation.
   - Connect input from Edit Fields.
   - Output to Message a model node.

6. **Add Google Gemini AI node ("Message a model"):**
   - Type: Google Gemini (PaLM) AI node
   - Select model `models/gemini-2.5-flash`.
   - Message content: Use a templated prompt that includes:
     - News headlines: `{{ $json.contentSnippet }}`
     - Target brand: `{{ $('Google Sheets Trigger').item.json.keyword }}`
     - Instructions for sentiment analysis and formatting as described above.
   - Provide Google Gemini API credentials.
   - Connect input from Aggregate node.
   - Output to Code in JavaScript node.

7. **Add Code node ("Code in JavaScript"):**
   - Type: Code
   - JavaScript code to:
     - Extract AI response text: `const input = $json.content.parts[0].text;`
     - Split text into paragraphs by double newlines.
     - Return object with `paragraphs` array.
   - Connect input from Message a model.
   - Output to Send a message node.

8. **Add Gmail node ("Send a message"):**
   - Type: Gmail (OAuth2)
   - Set recipient email dynamically: `{{ $('Google Sheets Trigger').item.json.email }}`
   - Subject: `{{ $('Google Sheets Trigger').item.json.keyword }} Sentiment Analysis on news`
   - Message: Use HTML by joining paragraphs with `<p>` tags:
     `{{ $json.paragraphs.map(p => `<p>${p}</p>`).join('') }}`
   - Connect Gmail OAuth2 credentials.
   - Connect input from Code node.

9. **Add Sticky Note:**
   - Add a sticky note to the canvas.
   - Paste detailed instructions on:
     - Google Sheets setup
     - Credential setup for Google Gemini API and Gmail
     - Workflow automation overview

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow requires Google Sheets with a sheet named "page1" and columns "keyword" and "email".              | Referenced in Sticky Note node and key for input data structure.                                            |
| Google Gemini API (PaLM) credentials must be configured for the AI sentiment analysis node.                    | Credentials setup required in “Message a model” node.                                                        |
| Gmail OAuth2 credentials are required for sending emails securely via the Gmail node.                           | Credentials setup required in “Send a message” node.                                                         |
| Workflow runs automatically upon new row addition to Google Sheets and sends comprehensive sentiment reports.  | Enables near real-time brand reputation tracking through email alerts.                                       |
| Uses Google News RSS feed to fetch news relevant to brand keywords dynamically.                                 | RSS URL dynamically generated using the brand keyword from Google Sheets.                                   |
| Report formatting includes paragraph separation for readability and includes sentiment scores and actionable recommendations. | Prompt instructs AI to produce structured and easy-to-read output.                                           |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automation workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.