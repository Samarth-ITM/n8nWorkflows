Automated Product Review Monitoring with Sentiment Analysis via Decodo, Gemini & Telegram

https://n8nworkflows.xyz/workflows/automated-product-review-monitoring-with-sentiment-analysis-via-decodo--gemini---telegram-11632


# Automated Product Review Monitoring with Sentiment Analysis via Decodo, Gemini & Telegram

### 1. Workflow Overview

This workflow automates the monitoring of product reviews from Amazon URLs, performing sentiment analysis and summarization, then sending alerts via Telegram and logging results in Google Sheets. It targets e-commerce owners, product managers, reputation and CX teams, and marketplace sellers who want hands-off, scheduled review monitoring with alerts for critical sentiment changes.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Input Retrieval:** Periodic triggering of the workflow to fetch product URLs from a Google Sheet.
- **1.2 Review Scraping & Transformation:** Iteratively scrape Amazon reviews using Decodo for each URL, then convert raw JSON reviews into formatted text.
- **1.3 Sentiment Analysis & Summarization:** Analyze sentiment of the compiled reviews using AI models and generate a summary.
- **1.4 Alerts & Data Storage:** Send Telegram alerts if needed and append the processed data (reviews, sentiment, URL) back into a Google Sheet for historical tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Retrieval

**Overview:**  
This block triggers the workflow on a schedule, retrieves product URLs from a Google Sheet, and prepares for batch processing of each URL.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- Loop Over Items  
- Sticky Note1  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution at regular intervals (default daily).  
  - Config: Uses default schedule with no custom interval specified (implies daily or default trigger).  
  - Input: None  
  - Output: Triggers "Get row(s) in sheet" node.  
  - Edge cases: Workflow won't start if scheduling misconfigured or n8n instance offline.

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Reads product URLs from the first sheet ("gid=0") of a specified Google Spreadsheet.  
  - Config: Reads all rows; document and sheet ID configured via credentials and parameters.  
  - Important: Google Sheets OAuth2 credentials required with read access.  
  - Input: Trigger from Schedule Trigger  
  - Output: Passes rows to "Loop Over Items" to batch process URLs.  
  - Edge cases: Permission errors if OAuth token invalid; empty sheets yield no URLs.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Splits input rows into batches of 5 URLs to process incrementally.  
  - Config: batchSize=5  
  - Input: Rows from Google Sheets  
  - Output: Sends each batch to "Decodo" for review scraping.  
  - Edge cases: If no rows, loop does not execute; batch size too large could cause timeouts.

- **Sticky Note1**  
  - Content: Advises to keep one URL per row in Google Sheet and ensure n8n has sheet access.  
  - Role: Documentation, no execution role.

---

#### 2.2 Review Scraping & Transformation

**Overview:**  
For each product URL, this block scrapes Amazon reviews using Decodo, then reformats the raw JSON data into a concise, readable text summary per review.

**Nodes Involved:**  
- Decodo  
- Code in JavaScript  
- Sticky Note2  
- Sticky Note3  

**Node Details:**  

- **Decodo**  
  - Type: Decodo (custom node)  
  - Role: Scrapes Amazon product reviews given a URL.  
  - Config: Operation set to "amazon" with dynamic URL from input JSON.  
  - Credentials: Requires Decodo API credentials (API key).  
  - Input: Batch URLs from Loop Over Items  
  - Output: Raw JSON review data per product with reviewer name, rating, review content.  
  - Edge cases: API errors (auth, rate limits), invalid URL input, no reviews found.

- **Code in JavaScript**  
  - Type: Code node (JavaScript)  
  - Role: Transforms Decodo’s JSON review array into formatted text strings of the form: "<author> - <rating> - <content>".  
  - Key expressions: Maps over `results[0].content.results.reviews` to extract and join reviews with double newlines.  
  - Output: Object with two properties: `reviews` (all reviews concatenated as text) and `url` (original product URL).  
  - Input: Raw JSON from Decodo, plus URL from Google Sheets node.  
  - Edge cases: Missing or malformed JSON structure may cause errors; empty reviews array returns empty string.

- **Sticky Note2**  
  - Content: Explains Decodo scrapes Amazon reviews and outputs structured JSON with reviewer name, rating, and content.

- **Sticky Note3**  
  - Content: Explains JavaScript node converts raw JSON into clean multi-line text string for further AI processing.

---

#### 2.3 Sentiment Analysis & Summarization

**Overview:**  
This block performs sentiment analysis on the concatenated reviews and generates a summarization of the overall feedback using AI language models.

**Nodes Involved:**  
- Sentiment Analyzer  
- Summarize Reviews  
- Google Gemini Chat Model  

**Node Details:**  

- **Sentiment Analyzer**  
  - Type: Langchain Sentiment Analysis  
  - Role: Detects overall sentiment category (positive, neutral, negative) from the compiled review text.  
  - Config: Input text dynamically set to `reviews` string from the JavaScript node.  
  - Input: Text from "Code in JavaScript" node.  
  - Output: JSON including sentiment category used for alerts and storage.  
  - Edge cases: AI model API errors, ambiguous sentiment, very short or noisy text input.

- **Summarize Reviews**  
  - Type: Langchain Chain Summarization  
  - Role: Produces a concise textual summary of all collected reviews.  
  - Input: Receives output from Sentiment Analyzer (can also accept Google Gemini outputs).  
  - Output: Summary text included in Telegram alerts.  
  - Edge cases: Model API rate limits or failures; quality depends on input text quality.

- **Google Gemini Chat Model**  
  - Type: Google Gemini AI Chat Model (Langchain integration)  
  - Role: Optional AI model that can be used to assist sentiment analysis and summarization.  
  - Credentials: Requires Google PaLM API credentials.  
  - Input: Receives review texts for AI processing.  
  - Output: Sends data to Sentiment Analyzer and Summarize Reviews as an alternative or complement.  
  - Edge cases: API quota limits, auth failures.

---

#### 2.4 Alerts & Data Storage

**Overview:**  
Sends Telegram alerts based on sentiment and summary, and stores all relevant data into a historical Google Sheet for record keeping.

**Nodes Involved:**  
- Alert Group  
- Store to Sheet  
- Sticky Note4  

**Node Details:**  

- **Alert Group**  
  - Type: Telegram node  
  - Role: Sends formatted alert messages to a Telegram group chat.  
  - Config: Message includes alert emoji, current date, sentiment category, product URL, and review summary.  
  - Credentials: Requires Telegram Bot API credentials.  
  - Input: Triggered after summarization completes, runs inside the batch loop.  
  - Output: None (terminal node for alert).  
  - Edge cases: Telegram API errors, invalid chat ID, network failures.

- **Store to Sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row to the "user review aggregations" sheet with the review text, sentiment, and URL.  
  - Config: Defines columns "Url", "Sentiment", and "List Review" with respective values from prior nodes.  
  - Credentials: Google Sheets OAuth2 with write access to the designated spreadsheet.  
  - Input: Runs per batch item after sentiment analysis.  
  - Output: None.  
  - Edge cases: Permission denied, quota exceeded, invalid cell mapping.

- **Sticky Note4**  
  - Content: Describes purpose of storing data in the sheet as a historical review database with fields: cleaned reviews, sentiment, URL.

---

### 3. Summary Table

| Node Name            | Node Type                        | Functional Role                               | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                  |
|----------------------|---------------------------------|-----------------------------------------------|-----------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Triggers workflow on schedule                  | None                  | Get row(s) in sheet        |                                                                                              |
| Get row(s) in sheet  | Google Sheets                   | Reads product URLs from spreadsheet            | Schedule Trigger      | Loop Over Items            | "Keep one URL per row. Ensure Sheet permissions allow n8n access." (Sticky Note1)            |
| Loop Over Items       | SplitInBatches                  | Processes URLs in batches of 5                  | Get row(s) in sheet   | Decodo (batch 1), Loop Over Items (batch 2) |                                                                                              |
| Decodo                | Decodo Node                    | Scrapes Amazon reviews for each URL             | Loop Over Items       | Code in JavaScript         | "Scrapes Amazon product reviews based on URL. Outputs structured JSON." (Sticky Note2)      |
| Code in JavaScript    | Code                           | Transforms raw JSON reviews into formatted text | Decodo                | Sentiment Analyzer         | "Transforms Decodo JSON into clean text blocks: <author> – <rating> – <content>" (Sticky Note3) |
| Sentiment Analyzer    | Langchain Sentiment Analysis   | Detects overall sentiment of reviews            | Code in JavaScript, Google Gemini Chat Model | Store to Sheet, Summarize Reviews |                                                                                              |
| Summarize Reviews     | Langchain Summarization Chain  | Generates summary of reviews                     | Sentiment Analyzer, Google Gemini Chat Model | Alert Group                |                                                                                              |
| Alert Group           | Telegram                       | Sends Telegram alerts with sentiment and summary | Summarize Reviews     | Loop Over Items            |                                                                                              |
| Store to Sheet        | Google Sheets                  | Logs reviews, sentiment, and URL into sheet     | Sentiment Analyzer    | Loop Over Items            | "Saves records into 'user review aggregations' sheet for historical tracking." (Sticky Note4) |
| Google Gemini Chat Model | Google Gemini Chat Model     | Optional AI model for sentiment and summarization | Code in JavaScript    | Sentiment Analyzer, Summarize Reviews |                                                                                              |
| Sticky Note1          | Sticky Note                    | Documentation note on Google Sheets setup       | None                  | None                      | "Keep one URL per row. Ensure Sheet permissions allow n8n access."                           |
| Sticky Note2          | Sticky Note                    | Explains Decodo node output                       | None                  | None                      | "Scrapes Amazon product reviews and outputs structured JSON."                               |
| Sticky Note3          | Sticky Note                    | Explains JavaScript node's transformation        | None                  | None                      | "Transforms raw JSON into clean, multi-line text string."                                   |
| Sticky Note4          | Sticky Note                    | Explains purpose of storing results in sheet     | None                  | None                      | "Saves a new record into the 'user review aggregations' sheet with relevant fields."        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set it to trigger at desired intervals (e.g., daily).  
   - No special parameters needed.

2. **Add a Google Sheets node named "Get row(s) in sheet"**  
   - Operation: Read rows from sheet.  
   - Document ID: Set to your Google Sheets document containing product URLs.  
   - Sheet Name: Use the first sheet or specify sheet by gid=0.  
   - Credentials: Configure Google Sheets OAuth2 credentials with read access.  
   - Connect Schedule Trigger output to this node.

3. **Add a SplitInBatches node named "Loop Over Items"**  
   - Batch Size: 5 (to process URLs in manageable chunks).  
   - Connect output of "Get row(s) in sheet" to this node.

4. **Add a Decodo node named "Decodo"**  
   - Operation: "amazon" (scrapes Amazon reviews).  
   - URL parameter: Set as expression `={{ $json.url }}` to use URL from current batch item.  
   - Credentials: Set Decodo API credentials.  
   - Connect output of "Loop Over Items" (first output) to this node.

5. **Add a Code node named "Code in JavaScript"**  
   - Paste following JS code:

     ```javascript
     const reviews = $input.first().json.results[0].content.results.reviews.map(el => (
       `${el.review_from} - ${el.rating} - ${el.content}`
     ));
     return {
       reviews: reviews.join("\n\n"),
       url: $('Get row(s) in sheet').first().json.url
     };
     ```
   - Input: Connect output of "Decodo" to this node.

6. **Add a Langchain Sentiment Analyzer node named "Sentiment Analyzer"**  
   - Input Text: Set to expression `={{ $json.reviews }}` (from code node).  
   - Credentials: Add Langchain or OpenAI credentials as required by your instance.  
   - Connect output of "Code in JavaScript" to this node.

7. **Add a Langchain Chain Summarization node named "Summarize Reviews"**  
   - Use default summarization chain options or customize prompt if needed.  
   - Connect output of "Sentiment Analyzer" to this node.

8. **Add a Telegram node named "Alert Group"**  
   - Chat ID: Set to your Telegram group/channel ID (e.g., `-4869385284`).  
   - Text: Use template:

     ```
     🚨ALERT WARNING🚨
     Date: {{ DateTime.now().format('yyyy-MM-dd') }}
     Sentiment: {{ $('Sentiment Analyzer').item.json.sentimentAnalysis.category }}

     Url: {{ $('Code in JavaScript').item.json.url }}

     Summary: {{ $json.output.text }}
     ```
   - Credentials: Configure Telegram Bot API credentials.  
   - Connect output of "Summarize Reviews" to this node.

9. **Add a Google Sheets node named "Store to Sheet"**  
   - Operation: Append  
   - Document ID: Your spreadsheet ID for logging reviews.  
   - Sheet Name: The target sheet name or gid (e.g., `1499922583`).  
   - Columns: Define columns "Url", "Sentiment", "List Review" with values from previous nodes:

     - Url: `={{ $('Code in JavaScript').item.json.url }}`
     - Sentiment: `={{ $json.sentimentAnalysis.category }}`
     - List Review: `={{ $json.reviews }}`

   - Credentials: Use Google Sheets OAuth2 credentials with write access.  
   - Connect output of "Sentiment Analyzer" to this node.

10. **Connect "Loop Over Items" second output to "Decodo"**  
    - This enables batch iteration over URLs.

11. **Optionally, add a Google Gemini Chat Model node**  
    - Connect input from "Code in JavaScript" node.  
    - Connect outputs to "Sentiment Analyzer" and "Summarize Reviews" nodes for AI-enhanced processing.  
    - Configure Google PaLM API credentials.

12. **Add Sticky Notes as documentation nodes at appropriate locations**  
    - Add notes about Google Sheets setup, Decodo function, code transformation, and storage purpose.

13. **Ensure all credentials are properly configured and tested** before activating workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Automated review monitoring is ideal for e-commerce owners, product managers, reputation teams, and sellers to receive daily alerts and build a historical review log without coding.                                                         | Sticky Note main content                                                                        |
| Keep Google Sheets organized with one URL per row and ensure n8n has permission to read and write sheets.                                                                                                                                     | Sticky Note1                                                                                   |
| Decodo scrapes Amazon reviews and outputs structured JSON with author, rating, and content for further processing.                                                                                                                            | Sticky Note2                                                                                   |
| JavaScript node transforms Decodo raw data into clean text strings "<author> – <rating> – <content>" for sentiment analysis and summarization by AI models.                                                                                   | Sticky Note3                                                                                   |
| Results are saved in a dedicated Google Sheet ("user review aggregations") with columns for URL, sentiment, and review text, enabling historical analysis and tracking of review trends over time.                                               | Sticky Note4                                                                                   |
| Telegram alerts notify teams immediately when sentiment changes or negative feedback is detected, facilitating rapid response.                                                                                                                | Alert Group node configuration                                                                |
| Workflow leverages Langchain AI nodes and Google Gemini models for advanced sentiment detection and summarization, requiring appropriate API credentials and quotas.                                                                           | Langchain & Google Gemini nodes                                                                 |
| For optimal reliability, monitor API rate limits, handle empty or malformed data gracefully, and secure credentials properly.                                                                                                                 | General operational advice                                                                     |

---

**Disclaimer:** The provided content is exclusively generated from an n8n automation workflow. It complies with all content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.