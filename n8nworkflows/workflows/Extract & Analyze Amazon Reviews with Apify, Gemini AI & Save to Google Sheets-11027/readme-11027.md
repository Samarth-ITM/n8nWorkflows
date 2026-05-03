Extract & Analyze Amazon Reviews with Apify, Gemini AI & Save to Google Sheets

https://n8nworkflows.xyz/workflows/extract---analyze-amazon-reviews-with-apify--gemini-ai---save-to-google-sheets-11027


# Extract & Analyze Amazon Reviews with Apify, Gemini AI & Save to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and AI-driven analysis of Amazon product reviews, specifically targeting negative feedback to identify product issues and improvement suggestions. It leverages an Apify actor to scrape reviews, uses Google Gemini AI to analyze each review’s sentiment and categorize complaints, formats AI output into structured JSON, stores the results in a Google Sheet, and finally sends a Slack notification upon completion.

Logical blocks:

- **1.1 Input & Configuration:** Manual trigger and configuration of parameters such as the Apify actor ID and maximum reviews to scrape.
- **1.2 Data Extraction:** Runs the Apify actor to scrape Amazon reviews based on predefined criteria.
- **1.3 Iterative AI Analysis:** Splits reviews into batches and for each review sends content to Google Gemini for sentiment analysis and categorization.
- **1.4 Data Formatting:** Parses the AI’s raw text response into clean JSON format.
- **1.5 Storage:** Creates or accesses a Google Sheet and appends the structured analysis data.
- **1.6 Completion Notification:** Sends a Slack message to notify that the review analysis batch is complete.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration

- **Overview:** Initiates the workflow manually and sets key parameters such as the Apify actor ID and maximum number of reviews to fetch.
- **Nodes Involved:** Manual Trigger, Workflow Configuration

**Manual Trigger**

- Type: Manual trigger node to start the workflow on demand.
- Config: Default (no parameters).
- Inputs: None.
- Outputs: Connected to Workflow Configuration.
- Edge cases: None typical; user must trigger manually.

**Workflow Configuration**

- Type: Set node to define static workflow parameters.
- Config:  
  - `maxReviews`: Number, set to 10 (limits reviews fetched).  
  - `apifyActorId`: String, set to "junglee/amazon-reviews-scraper" (Apify actor used).
- Inputs: Manual Trigger.
- Outputs: Connected to "Run an Actor and get dataset".
- Edge cases: Incorrect actor ID or invalid maxReviews value can cause Apify failures.

---

#### 2.2 Data Extraction

- **Overview:** Executes Apify actor to scrape Amazon reviews from a specified product URL, filtering for 1 and 2-star ratings to focus on critical feedback.
- **Nodes Involved:** Run an Actor and get dataset

**Run an Actor and get dataset**

- Type: Apify node running an actor and fetching its dataset.
- Config:  
  - Actor ID: `R8WeJwLuzLZ6g4Bkk` (linked to `junglee/amazon-reviews-scraper`).  
  - Operation: Run actor and retrieve dataset.  
  - Custom JSON body includes:  
    - `startUrls`: Product URL `"https://www.amazon.co.jp/dp/B09JDGYSQW"`.  
    - `maxReviews`: 10 (parameterized).  
    - `filterByRatings`: Only 1-star and 2-star reviews.  
    - Proxy configuration: Apify proxy enabled.
- Inputs: Workflow Configuration.
- Outputs: Connected to Loop Over Reviews.
- Edge cases:  
  - Authentication errors with Apify OAuth2.  
  - Network or proxy failures.  
  - Actor crashes or invalid URLs.
- Requirements: Valid Apify OAuth2 credentials with access to the actor.

---

#### 2.3 Iterative AI Analysis

- **Overview:** Iterates over each review individually, sending each review’s text to Google Gemini AI to analyze sentiment, categorize issues, summarize user complaints, and suggest improvements.
- **Nodes Involved:** Loop Over Reviews, Message a model

**Loop Over Reviews**

- Type: Split in Batches node to process reviews one by one.
- Config: Default batch size (1 review per batch).
- Inputs: Run an Actor and get dataset.
- Outputs:  
  - Main output 0: To Slack notification.  
  - Main output 1: To AI analysis node ("Message a model").
- Edge cases: Empty review sets may cause no loops; large volumes might impact rate limits.

**Message a model**

- Type: Google Gemini AI node (Langchain integration).
- Config:  
  - Model ID: `"models/gemini-2.0-flash-lite"`.  
  - Prompt instructs AI (in Japanese) to analyze the review and output JSON with:  
    - `sentiment_score` (1-5),  
    - `category` (one of Quality, Price, Function, Design, Shipping, Other),  
    - `summary` (max 30 Japanese characters),  
    - `improvement` (max 50 Japanese characters).  
- Inputs: Loop Over Reviews (each review’s JSON).
- Outputs: Connected to Code (Parse JSON) node.
- Edge cases:  
  - API authentication or quota errors.  
  - Unexpected AI output format causing JSON parse errors.

---

#### 2.4 Data Formatting

- **Overview:** Cleans and parses AI text output to ensure valid JSON format for downstream processing.
- **Nodes Involved:** Code (Parse JSON)1

**Code (Parse JSON)1**

- Type: Code node (JavaScript).
- Config:  
  - Extracts AI response text from multiple possible properties.  
  - Strips markdown code block delimiters like ```json.  
  - Parses cleaned string into JSON object, returns as node output.
- Inputs: Message a model.
- Outputs: Create spreadsheet.
- Edge cases:  
  - Malformed AI JSON output causing parse exceptions.  
  - Missing or unexpected properties in AI response.

---

#### 2.5 Storage

- **Overview:** Creates or accesses a Google Spreadsheet and appends each parsed JSON result as a new row.
- **Nodes Involved:** Create spreadsheet

**Create spreadsheet**

- Type: Google Sheets node.
- Config:  
  - Resource: Spreadsheet.  
  - (Details like Spreadsheet ID and sheet name must be configured externally).  
- Inputs: Code (Parse JSON)1.
- Outputs: Loop Over Reviews (to process next batch or finish).
- Edge cases:  
  - Google Sheets API authentication errors (OAuth2).  
  - Missing or incorrect spreadsheet ID.  
  - Quota or rate limits.

---

#### 2.6 Completion Notification

- **Overview:** Upon completing all review batches, sends a Slack message notification indicating how many reviews were processed.
- **Nodes Involved:** Slack - Send Completion Notification

**Slack - Send Completion Notification**

- Type: Slack node.
- Config:  
  - Text message (in Japanese) confirms completion and number of reviews processed.  
  - Channel ID or name must be specified by user.  
  - Authentication via OAuth2.
- Inputs: Loop Over Reviews (completion event).
- Outputs: None.
- Edge cases:  
  - Slack authentication or permission errors.  
  - Invalid channel ID.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                                   | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                                             |
|-------------------------------|-------------------------------|-------------------------------------------------|---------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                 | Manual Trigger                | Initiates workflow manually                      | -                         | Workflow Configuration              |                                                                                                                         |
| Workflow Configuration         | Set                          | Sets workflow parameters (maxReviews, actorId) | Manual Trigger            | Run an Actor and get dataset        |                                                                                                                         |
| Run an Actor and get dataset   | Apify                        | Runs Apify actor to scrape Amazon reviews       | Workflow Configuration    | Loop Over Reviews                   | 1. **Data Collection:** The workflow triggers the Apify actor (`junglee/amazon-reviews-scraper`) to fetch reviews from a specified Amazon product URL. It is currently configured to filter for 1 and 2-star reviews to focus on complaints. |
| Loop Over Reviews              | Split In Batches             | Processes each review individually               | Run an Actor and get dataset | Slack - Send Completion Notification, Message a model | 2. **AI Analysis:** It loops through each review and sends the content to Google Gemini. The AI determines a sentiment score (1-5), categorizes the issue (Quality, Design, Shipping, etc.), summarizes the complaint, and proposes a concrete improvement plan. |
| Message a model                | Google Gemini AI (Langchain) | Analyzes reviews with AI, outputs JSON results  | Loop Over Reviews         | Code (Parse JSON)1                 |                                                                                                                         |
| Code (Parse JSON)1             | Code                         | Parses AI raw text response into valid JSON     | Message a model           | Create spreadsheet                 | 3. **Formatting:** A Code node parses the AI's response to ensure it is in a clean JSON format.                         |
| Create spreadsheet            | Google Sheets                | Appends parsed data to Google Sheet              | Code (Parse JSON)1        | Loop Over Reviews                   | 4. **Storage:** The structured data is appended as a new row in a Google Sheet.                                        |
| Slack - Send Completion Notification | Slack                       | Sends Slack notification upon completion         | Loop Over Reviews (completion) | -                              | 5. **Notification:** A Slack message is sent to your specified channel to confirm the batch analysis is complete.       |
| Sticky Note                   | Sticky Note                  | Template description and audience info           | -                         | -                                  |                                                                                                                         |
| Sticky Note1                  | Sticky Note                  | Explains Data Collection block                    | -                         | -                                  | 1. **Data Collection:** The workflow triggers the Apify actor (`junglee/amazon-reviews-scraper`) to fetch reviews from a specified Amazon product URL. It is currently configured to filter for 1 and 2-star reviews to focus on complaints. |
| Sticky Note2                  | Sticky Note                  | Explains AI Analysis block                        | -                         | -                                  | 2. **AI Analysis:** It loops through each review and sends the content to Google Gemini. The AI determines a sentiment score (1-5), categorizes the issue (Quality, Design, Shipping, etc.), summarizes the complaint, and proposes a concrete improvement plan. |
| Sticky Note3                  | Sticky Note                  | Explains Formatting block                         | -                         | -                                  | 3. **Formatting:** A Code node parses the AI's response to ensure it is in a clean JSON format.                         |
| Sticky Note4                  | Sticky Note                  | Explains Storage block                            | -                         | -                                  | 4. **Storage:** The structured data is appended as a new row in a Google Sheet.                                        |
| Sticky Note5                  | Sticky Note                  | Explains Notification block                       | -                         | -                                  | 5. **Notification:** A Slack message is sent to your specified channel to confirm the batch analysis is complete.       |
| Sticky Note6                  | Sticky Note                  | Setup requirements and instructions               | -                         | -                                  | ## 🛠️ Requirements\n- **n8n** (Self-hosted or Cloud)\n- **Apify Account:** You need to rent the `junglee/amazon-reviews-scraper` actor.\n- **Google Cloud Account:** For accessing the Gemini (PaLM) API and Google Sheets API.\n- **Slack Account:** For receiving notifications.\n\n## 🚀 How to set up\n1. **Apify Config:** Enter your Apify API token in the credentials. In the \"Run an Actor\" node, update the `startUrls` to the Amazon product page you want to analyze.\n2. **Google Sheets:** Create a new Google Sheet with the following header columns: `sentiment_score`, `category`, `summary`, `improvement`. Copy the Spreadsheet ID into the Google Sheets node.\n3. **AI Prompt:** The \"Message a model\" node contains the prompt. It is currently set to output results in **Japanese**. If you need English output, simply translate the prompt text inside this node.\n4. **Slack:** Select the channel where you want to receive notifications in the Slack node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** — to start the workflow manually.

2. **Add a Set node (Workflow Configuration):**  
   - Add two variables:  
     - `maxReviews`: Number, set to 10.  
     - `apifyActorId`: String, set to `"junglee/amazon-reviews-scraper"`.  
   - Connect Manual Trigger output to this node.

3. **Add an Apify node (Run an Actor and get dataset):**  
   - Credentials: Configure Apify OAuth2 credentials with your API token.  
   - Operation: Select "Run actor and get dataset".  
   - Actor ID: Use the selected actor or parameterize with `apifyActorId`.  
   - Custom body JSON:  
     ```json
     {
       "startUrls": [{"url": "https://www.amazon.co.jp/dp/B09JDGYSQW"}],
       "maxReviews": 10,
       "filterByRatings": ["oneStar", "twoStars"],
       "proxyConfiguration": {"useApifyProxy": true}
     }
     ```  
   - Connect Workflow Configuration output to this node.

4. **Add a SplitInBatches node (Loop Over Reviews):**  
   - No special options, default batch size 1.  
   - Connect Apify node output to this node.

5. **Add a Google Gemini node (Message a model):**  
   - Credentials: Configure Google Cloud credentials with Gemini API access.  
   - Model: Set to `"models/gemini-2.0-flash-lite"`.  
   - Prompt message:  
     ```
     あなたは熟練の商品開発マーケターです。
     以下のAmazonレビューを分析し、情報をJSON形式のみで出力してください。
     {
       "sentiment_score": (1〜5の数値。1が最悪、5が最高),
       "category": ("Quality", "Price", "Function", "Design", "Shipping", "Other" から最も当てはまるものを1つ選択),
       "summary": (ユーザーの不満点を日本語で30文字以内に要約),
       "improvement": (具体的な改善案を日本語で50文字以内で提案)
     }
     ```
   - Connect Loop Over Reviews second output (index 1) to this node.

6. **Add a Code node (Code (Parse JSON)1):**  
   - JavaScript code:  
     ```js
     const responseData = $input.first().json;
     const aiResponse = responseData.text || responseData.output || (responseData.message && responseData.message.content);
     const cleanJson = aiResponse.replace(/```json|```/g, '').trim();
     return [{ json: JSON.parse(cleanJson) }];
     ```  
   - Connect Message a model output to this node.

7. **Add a Google Sheets node (Create spreadsheet):**  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Resource: Spreadsheet.  
   - Select or input your target spreadsheet ID (must have columns: `sentiment_score`, `category`, `summary`, `improvement`).  
   - Append the JSON data from Code node as new rows.  
   - Connect Code node output to this node.

8. **Connect Create spreadsheet output back to Loop Over Reviews (main output) to continue looping over batches.**

9. **Connect Loop Over Reviews main output index 0 to Slack node (Slack - Send Completion Notification):**  
   - Credentials: Set Slack OAuth2 credentials.  
   - Channel: Set the Slack channel ID or name where notifications should be sent.  
   - Message text:  
     ```
     ✅ Amazon レビュー分析が完了しました！

     処理したレビュー数: {{ $('Loop Over Reviews').item.json.batchSize }}
     Notionデータベースを確認してください。
     ```
   - Connect Loop Over Reviews main output 0 to Slack node.

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to help Product Managers, E-commerce Sellers, Market Researchers, and Customer Support Teams efficiently analyze customer feedback from Amazon reviews and generate actionable insights automatically. It eliminates manual review reading and data entry by combining Apify scraping, Google Gemini AI analysis, Google Sheets storage, and Slack notifications.                                                                                                                                    | Sticky Note overview at workflow start.                                                                                            |
| For setup, ensure you have: 1) Apify account with access to `junglee/amazon-reviews-scraper` actor, 2) Google Cloud project with Gemini (PaLM) API and Google Sheets API enabled, 3) Slack workspace with OAuth2 app configured for sending messages.                                                                                                                                                                                                                                                                    | Sticky Note6 instructions.                                                                                                         |
| The AI prompt is currently in Japanese; translate prompt text inside the Google Gemini node if English or other language output is needed.                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note6 instructions.                                                                                                         |
| Slack notification message includes placeholders for processed review count and directs users to check Notion database (likely external to this workflow). Adjust message as needed for your environment.                                                                                                                                                                                                                                                                                                                   | Slack node configuration.                                                                                                         |
| Apify actor `junglee/amazon-reviews-scraper` filters 1 and 2-star reviews by default to focus on negative feedback; modify filter in the Apify node’s custom body JSON if different rating ranges are desired.                                                                                                                                                                                                                                                                                                              | Apify node configuration.                                                                                                         |
| Code node parsing assumes AI outputs JSON enclosed in markdown code blocks; malformed outputs may cause parse errors. Consider adding error handling or fallback logic if needed.                                                                                                                                                                                                                                                                                                                                        | Code (Parse JSON)1 node details.                                                                                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.