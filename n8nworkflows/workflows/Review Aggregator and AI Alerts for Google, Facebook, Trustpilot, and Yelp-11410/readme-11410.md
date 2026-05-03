Review Aggregator and AI Alerts for Google, Facebook, Trustpilot, and Yelp

https://n8nworkflows.xyz/workflows/review-aggregator-and-ai-alerts-for-google--facebook--trustpilot--and-yelp-11410


# Review Aggregator and AI Alerts for Google, Facebook, Trustpilot, and Yelp

### 1. Workflow Overview

This workflow automates the aggregation, AI-driven sentiment analysis, alerting, and weekly reporting of customer reviews from four major platforms: Google, Facebook, Trustpilot, and Yelp. Designed for businesses monitoring their online reputation, it fetches new reviews daily, normalizes diverse data formats into a unified structure, uses AI (GPT-4) to analyze sentiment and key issues, and triggers Slack alerts for negative reviews. Additionally, it logs all processed reviews to Google Sheets and generates a comprehensive weekly report every Monday, which is emailed to management.

The workflow is logically divided into these blocks:

- **1.1 Schedule and Configuration:** Daily trigger and loading of all API endpoints, credentials, and destination parameters.
- **1.2 Review Collection:** Fetch latest reviews from each platform via respective API calls.
- **1.3 Data Normalization and Merging:** Combine all reviews and normalize their structure for consistent downstream processing.
- **1.4 AI Sentiment Analysis:** Send normalized reviews to GPT-4 for sentiment scoring, categorization, and summary generation.
- **1.5 Alerting and Logging:** Send Slack alerts for negative reviews and log all enriched reviews into Google Sheets.
- **1.6 Weekly Reporting:** On Mondays, aggregate the past week’s reviews, generate metrics, get AI to create an executive summary, and email the report.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule and Configuration

**Overview:**  
Initiates the workflow daily at 9:00 AM and loads all essential configuration values (API URLs, Slack channel, email, Google Sheets IDs).

**Nodes Involved:**  
- Daily Schedule - 09:00  
- Workflow Configuration

**Node Details:**

- **Daily Schedule - 09:00**  
  - Type: Schedule Trigger  
  - Runs once daily at 9:00 AM local time.  
  - Output: Triggers the workflow start.  
  - Failure modes: Misconfigured timezone or n8n instance downtime could prevent triggering.

- **Workflow Configuration**  
  - Type: Set Node  
  - Holds key configuration variables as static strings to be replaced by user:  
    - API endpoints for Google My Business, Facebook Graph API, Trustpilot, Yelp Fusion  
    - Slack channel ID for alerts  
    - Manager email for weekly reports  
    - Google Sheets document ID and sheet name for logging  
  - Output: Provides these configs as JSON for downstream use.  
  - Edge cases: Missing or incorrect values will cause API request failures downstream.

---

#### 1.2 Review Collection

**Overview:**  
Fetches reviews updated or created in the past day from each review platform’s API using HTTP request nodes.

**Nodes Involved:**  
- Get Google Reviews  
- Get Facebook Reviews  
- Get Trustpilot Reviews  
- Get Yelp Reviews

**Node Details:**

- **Get Google Reviews**  
  - HTTP Request  
  - URL dynamically set from configuration  
  - Query parameter: `updated_after` set to ISO timestamp 1 day ago  
  - Header: Authorization Bearer from `GOOGLE_API_KEY` environment variable  
  - Expects JSON response  
  - Potential failures: API key invalid, rate limits, malformed endpoint URL.

- **Get Facebook Reviews**  
  - HTTP Request  
  - URL from config  
  - Query parameters: `since` (Unix timestamp 1 day ago), `access_token` from env var `FACEBOOK_ACCESS_TOKEN`  
  - JSON response expected  
  - Edge cases: Token expiration, permission errors, API changes.

- **Get Trustpilot Reviews**  
  - HTTP Request  
  - URL from config  
  - Query param: `createdSince` ISO timestamp 1 day ago  
  - Header: Authorization Bearer from `TRUSTPILOT_API_KEY` env var  
  - JSON response expected  
  - Edge cases: API throttling, key validity.

- **Get Yelp Reviews**  
  - HTTP Request  
  - URL from config  
  - Query param: `start_date` formatted as `yyyy-MM-dd` (1 day ago)  
  - Header: Authorization Bearer with placeholder token (replace with actual)  
  - JSON response expected  
  - Edge cases: Token missing or incorrect, API changes.

---

#### 1.3 Data Normalization and Merging

**Overview:**  
Combines reviews from all platforms into a single list and normalizes their disparate data structures into a unified format with consistent fields.

**Nodes Involved:**  
- Merge All Reviews  
- Normalize All Reviews

**Node Details:**

- **Merge All Reviews**  
  - Type: Merge  
  - Combines four inputs (Google, Facebook, Trustpilot, Yelp reviews) by position (concatenation) into one array.  
  - Failure modes: If any input is missing or empty, downstream steps may receive incomplete data.

- **Normalize All Reviews**  
  - Type: Code Node (JavaScript)  
  - For each review item, detects platform-specific fields and maps to a normalized schema including:  
    - platform, rating, text, created_at, reviewer_name, review_url, location  
  - Supports Google, Facebook, Trustpilot, Yelp formats explicitly; falls back to generic mapping for unknown formats.  
  - Output: Array of normalized review objects.  
  - Edge cases: Unexpected API response structures may cause missing or incorrect fields.

---

#### 1.4 AI Sentiment Analysis

**Overview:**  
Prepares each review’s text for AI processing, sends it to GPT-4 to analyze sentiment, key issues/highlights, and summary, then parses and categorizes AI output.

**Nodes Involved:**  
- Prepare AI Input  
- Sentiment Analysis (GPT-4)  
- Categorize & Parse AI Output

**Node Details:**

- **Prepare AI Input**  
  - Set Node  
  - Converts each normalized review into a JSON string containing key fields (platform, location, rating, text, reviewer_name, created_at) under `reviewData`.  
  - Outputs enriched JSON for GPT-4 input.

- **Sentiment Analysis (GPT-4)**  
  - OpenAI Node (GPT-4 model)  
  - Prompt instructs AI to return JSON with: sentiment, sentiment_score (-1 to 1), main_issues, main_highlights, suggested_response_tone, and summary.  
  - Sends reviewData as input content.  
  - Credentials: OpenAI API key configured.  
  - Failure modes: API rate limits, invalid prompt formatting, AI service downtime.

- **Categorize & Parse AI Output**  
  - Code Node  
  - Parses AI JSON response.  
  - Categorizes review into one of four categories (service, price, product, experience) based on keyword matching in the original review text.  
  - Adds flags like `is_negative` based on sentiment.  
  - Outputs enriched review objects with AI insights and category.  
  - Edge cases: Malformed AI JSON response or missing fields.

---

#### 1.5 Alerting and Logging

**Overview:**  
Checks if review sentiment is negative, sends Slack alerts for such reviews, and logs all reviews with AI insights into Google Sheets.

**Nodes Involved:**  
- Check If Negative  
- Alert - Negative Review  
- Log Review to Sheet

**Node Details:**

- **Check If Negative**  
  - If Node  
  - Condition: `is_negative` field equals true.  
  - Routes negative reviews to Slack alert; all reviews proceed to logging.  
  - Edge cases: Missing or malformed sentiment flag could misroute data.

- **Alert - Negative Review**  
  - Slack node with webhook  
  - Sends formatted alert message to configured Slack channel, including platform, location, rating, reviewer, summary, main issues, suggested response tone, and review URL.  
  - Uses Slack API credentials.  
  - Failure modes: Slack API errors, invalid channel ID, credential issues.

- **Log Review to Sheet**  
  - Google Sheets node (append or update)  
  - Logs review fields including date, rating, summary, category, location, platform, raw text, sentiment, URL, and sentiment score.  
  - Uses service account authentication.  
  - Sheet and document IDs configured from workflow config.  
  - Edge cases: Sheet permission errors, schema mismatch.

---

#### 1.6 Weekly Reporting

**Overview:**  
Runs every Monday to fetch the last 7 days of reviews from Sheets, aggregates statistics, generates an AI-written executive summary, and emails it to management.

**Nodes Involved:**  
- Check If Monday (Weekly Report)  
- Get Last 7 Days Reviews  
- Aggregate Weekly Stats  
- Calculate Weekly Metrics  
- Generate Weekly Summary (GPT-4)  
- Send Weekly Email Report

**Node Details:**

- **Check If Monday (Weekly Report)**  
  - If Node  
  - Checks current weekday equals 1 (Monday)  
  - Only triggers weekly reporting sequence on Mondays.

- **Get Last 7 Days Reviews**  
  - Google Sheets node  
  - Queries Google Sheet for reviews with `date` greater than or equal to 7 days ago.  
  - Uses service account credentials.  
  - Edge cases: No data for week, query failures.

- **Aggregate Weekly Stats**  
  - Aggregate Node  
  - Aggregates fields: platform, sentiment, category, main_issues, main_highlights.  
  - Produces grouped data for metric calculation.

- **Calculate Weekly Metrics**  
  - Code Node  
  - Calculates counts per platform, sentiment breakdown percentages, and top 5 issues and highlights by frequency from aggregated data.  
  - Outputs structured metrics JSON.

- **Generate Weekly Summary (GPT-4)**  
  - OpenAI Node (GPT-4)  
  - Prompt instructs AI to write a professional executive summary including sentiment trends, platform insights, top wins/issues, and actionable recommendations.  
  - Input: JSON metrics from previous node.  
  - Output: AI-generated textual summary.

- **Send Weekly Email Report**  
  - Gmail Node  
  - Sends an HTML formatted email to the configured manager email address.  
  - Includes metrics, platform breakdown, top highlights and issues, and AI-generated summary.  
  - Uses service account authentication.  
  - Edge cases: Email sending failures, invalid recipient email.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                                      | Input Node(s)                                  | Output Node(s)                               | Sticky Note                                                                                                                                            |
|----------------------------|----------------------------------|-----------------------------------------------------|-----------------------------------------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule - 09:00      | Schedule Trigger                 | Triggers workflow daily at 9:00 AM                   |                                               | Workflow Configuration                        | ## Schedule & Config  Daily trigger and core settings: review sources, business IDs, API keys, and destination Sheet/Slack/email. Update before live. |
| Workflow Configuration      | Set                             | Loads configuration variables for API endpoints etc. | Daily Schedule - 09:00                         | Get Google Reviews, Get Facebook Reviews, Get Trustpilot Reviews, Get Yelp Reviews | ## Schedule & Config                                                                                                                                   |
| Get Google Reviews          | HTTP Request                    | Fetches Google reviews updated in last day           | Workflow Configuration                         | Merge All Reviews                             | ## Fetch Reviews  Pulls the latest reviews from Google, Facebook, Trustpilot, and Yelp. Each node targets one platform with your business/location IDs. |
| Get Facebook Reviews        | HTTP Request                    | Fetches Facebook reviews from last day                | Workflow Configuration                         | Merge All Reviews                             | ## Fetch Reviews                                                                                                                                       |
| Get Trustpilot Reviews      | HTTP Request                    | Fetches Trustpilot reviews created since yesterday    | Workflow Configuration                         | Merge All Reviews                             | ## Fetch Reviews                                                                                                                                       |
| Get Yelp Reviews            | HTTP Request                    | Fetches Yelp reviews from last day                     | Workflow Configuration                         | Merge All Reviews                             | ## Fetch Reviews                                                                                                                                       |
| Merge All Reviews           | Merge                           | Combines all platform reviews into one list           | Get Google Reviews, Get Facebook Reviews, Get Trustpilot Reviews, Get Yelp Reviews | Normalize All Reviews                         | ## Merge & Normalize Combines reviews from all platforms and converts them into a single, consistent format.                                            |
| Normalize All Reviews       | Code                            | Normalizes review data into a unified format           | Merge All Reviews                              | Prepare AI Input, Check If Monday (Weekly Report) | ## Merge & Normalize                                                                                                                                   |
| Prepare AI Input            | Set                             | Formats review data as JSON string for AI input       | Normalize All Reviews                          | Sentiment Analysis (GPT-4)                    | ## AI Sentiment  Sends each review to AI to detect sentiment, score, key issues/highlights, and a short summary.                                         |
| Sentiment Analysis (GPT-4)  | OpenAI (GPT-4)                  | Analyzes sentiment and extracts key insights          | Prepare AI Input                              | Categorize & Parse AI Output                  | ## AI Sentiment                                                                                                                                         |
| Categorize & Parse AI Output| Code                            | Parses AI JSON response and categorizes reviews        | Sentiment Analysis (GPT-4)                     | Check If Negative                             | ## AI Sentiment                                                                                                                                         |
| Check If Negative           | If                              | Checks if review sentiment is negative                 | Categorize & Parse AI Output                    | Alert - Negative Review, Log Review to Sheet | ## Alerts & Logging  Sends a Slack alert for negative reviews and logs all reviews plus AI insights into Google Sheets.                                   |
| Alert - Negative Review     | Slack                           | Sends Slack alert for negative reviews                  | Check If Negative                              | Log Review to Sheet                           | ## Alerts & Logging                                                                                                                                     |
| Log Review to Sheet         | Google Sheets                   | Logs all reviews and AI insights into Google Sheets    | Check If Negative, Alert - Negative Review    |                                              | ## Alerts & Logging                                                                                                                                     |
| Check If Monday (Weekly Report)| If                            | Checks if today is Monday to trigger weekly report     | Normalize All Reviews                          | Get Last 7 Days Reviews                      | ## Weekly Report  On Mondays, aggregates the last 7 days, builds stats, AI summary, and emails report.                                                  |
| Get Last 7 Days Reviews     | Google Sheets                   | Fetches reviews from last 7 days for weekly report     | Check If Monday (Weekly Report)                | Aggregate Weekly Stats                        | ## Weekly Report                                                                                                                                         |
| Aggregate Weekly Stats      | Aggregate                      | Aggregates review fields for metric calculation         | Get Last 7 Days Reviews                        | Calculate Weekly Metrics                      | ## Weekly Report                                                                                                                                         |
| Calculate Weekly Metrics    | Code                            | Computes counts, percentages, top issues/highlights     | Aggregate Weekly Stats                         | Generate Weekly Summary (GPT-4)               | ## Weekly Report                                                                                                                                         |
| Generate Weekly Summary (GPT-4)| OpenAI (GPT-4)               | Generates executive summary based on weekly metrics    | Calculate Weekly Metrics                       | Send Weekly Email Report                      | ## Weekly Report                                                                                                                                         |
| Send Weekly Email Report    | Gmail                          | Emails the weekly summary report to management          | Generate Weekly Summary (GPT-4)                |                                              | ## Weekly Report                                                                                                                                         |
| Sticky Note                 | Sticky Note                    | Workflow overview and usage instructions                |                                               |                                              | ## How it works  This workflow automatically monitors your reviews across Google, Facebook, Trustpilot, and Yelp. See full note content in node details. |
| Sticky Note1                | Sticky Note                    | Schedule and configuration explanation                   |                                               |                                              | ## Schedule & Config                                                                                                                                   |
| Sticky Note2                | Sticky Note                    | Explanation of review fetching nodes                     |                                               |                                              | ## Fetch Reviews                                                                                                                                       |
| Sticky Note3                | Sticky Note                    | Explanation of merging and normalization                  |                                               |                                              | ## Merge & Normalize                                                                                                                                   |
| Sticky Note4                | Sticky Note                    | Explanation of AI sentiment analysis section              |                                               |                                              | ## AI Sentiment                                                                                                                                         |
| Sticky Note5                | Sticky Note                    | Explanation of alerting and logging section               |                                               |                                              | ## Alerts & Logging                                                                                                                                     |
| Sticky Note6                | Sticky Note                    | Explanation of weekly reporting section                    |                                               |                                              | ## Weekly Report                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: *Daily Schedule - 09:00*  
   - Type: Schedule Trigger  
   - Configure to run daily at hour 9 (9:00 AM local time).

2. **Create a Set Node for Configuration**  
   - Name: *Workflow Configuration*  
   - Add string fields:  
     - `googleApiUrl`  
     - `facebookApiUrl`  
     - `trustpilotApiUrl`  
     - `yelpApiUrl`  
     - `slackChannel` (Slack channel ID)  
     - `managerEmail` (email for weekly report)  
     - `sheetsDocumentId` (Google Sheets ID)  
     - `sheetsSheetName` (sheet name, e.g., "Reviews")  
   - Connect *Daily Schedule - 09:00* → *Workflow Configuration*

3. **Create four HTTP Request nodes to fetch reviews:**  
   - *Get Google Reviews*  
     - URL: `={{ $('Workflow Configuration').first().json.googleApiUrl }}`  
     - Query param: `updated_after={{ $now.minus({ days: 1 }).toISO() }}`  
     - Header: Authorization Bearer with environment variable `GOOGLE_API_KEY`  
   - *Get Facebook Reviews*  
     - URL: `={{ $('Workflow Configuration').first().json.facebookApiUrl }}`  
     - Query params: `since={{ $now.minus({ days: 1 }).toUnixInteger() }}`, `access_token={{ FACEBOOK_ACCESS_TOKEN }}`  
   - *Get Trustpilot Reviews*  
     - URL: `={{ $('Workflow Configuration').first().json.trustpilotApiUrl }}`  
     - Query param: `createdSince={{ $now.minus({ days: 1 }).toISO() }}`  
     - Header: Authorization Bearer with env var `TRUSTPILOT_API_KEY`  
   - *Get Yelp Reviews*  
     - URL: `={{ $('Workflow Configuration').first().json.yelpApiUrl }}`  
     - Query param: `start_date={{ $now.minus({ days: 1 }).toFormat('yyyy-MM-dd') }}`  
     - Header: Authorization Bearer with your Yelp API token

4. **Connect Workflow Configuration outputs to each HTTP Request node**  
   - All four HTTP Request nodes run in parallel.

5. **Create a Merge node**  
   - Name: *Merge All Reviews*  
   - Mode: Combine by position  
   - Inputs: 4 (one from each review fetch node)  
   - Connect all four HTTP Request nodes → *Merge All Reviews*

6. **Create a Code Node to Normalize Reviews**  
   - Name: *Normalize All Reviews*  
   - Use JavaScript code to detect platform-specific fields and convert to uniform structure with keys: platform, rating, text, created_at, reviewer_name, review_url, location  
   - Connect *Merge All Reviews* → *Normalize All Reviews*

7. **Create a Set Node to Prepare AI Input**  
   - Name: *Prepare AI Input*  
   - Assign a string field `reviewData` with JSON stringification of the normalized review subset (platform, location, rating, text, reviewer_name, created_at)  
   - Connect *Normalize All Reviews* → *Prepare AI Input*

8. **Create an OpenAI GPT-4 Node for Sentiment Analysis**  
   - Name: *Sentiment Analysis (GPT-4)*  
   - Model: GPT-4  
   - Instructions prompt: Analyze review sentiment, score (-1 to 1), main issues/highlights, suggested response tone, summary; return valid JSON only  
   - Input: Use `reviewData` field from Prepare AI Input node  
   - Connect *Prepare AI Input* → *Sentiment Analysis (GPT-4)*  
   - Configure OpenAI API credentials

9. **Create a Code Node to Parse and Categorize AI Output**  
   - Name: *Categorize & Parse AI Output*  
   - Parse JSON content from AI response  
   - Categorize review based on keyword matching in original text into `service`, `price`, `product`, or `experience`  
   - Add flags like `is_negative`  
   - Connect *Sentiment Analysis (GPT-4)* → *Categorize & Parse AI Output*

10. **Create an If Node to Check for Negative Sentiment**  
    - Name: *Check If Negative*  
    - Condition: `is_negative` equals `true`  
    - Connect *Categorize & Parse AI Output* → *Check If Negative*

11. **Create a Slack Node for Negative Review Alerts**  
    - Name: *Alert - Negative Review*  
    - Send formatted alert message including platform, location, rating, reviewer, summary, main issues, suggested response tone, review URL  
    - Use Slack credentials and channel from config  
    - Connect *Check If Negative* (true output) → *Alert - Negative Review*

12. **Create a Google Sheets Node to Log Reviews**  
    - Name: *Log Review to Sheet*  
    - Operation: Append or update rows  
    - Map fields: date, rating, summary, category, location, platform, raw_text, sentiment, review_url, sentiment_score  
    - Use Google Sheets credentials with service account  
    - Sheet name and document ID from config  
    - Connect both *Check If Negative* (false output) and *Alert - Negative Review* → *Log Review to Sheet*

13. **Create an If Node to Check if Today is Monday**  
    - Name: *Check If Monday (Weekly Report)*  
    - Condition: weekday equals 1 (Monday)  
    - Connect *Normalize All Reviews* → *Check If Monday (Weekly Report)*

14. **Create a Google Sheets Node to Get Last 7 Days Reviews**  
    - Name: *Get Last 7 Days Reviews*  
    - Query: Filter `date` >= 7 days ago ISO date  
    - Use same Google Sheets credentials and config  
    - Connect *Check If Monday* (true) → *Get Last 7 Days Reviews*

15. **Create Aggregate Node to Summarize Weekly Data**  
    - Name: *Aggregate Weekly Stats*  
    - Aggregate fields: platform, sentiment, category, main_issues, main_highlights  
    - Connect *Get Last 7 Days Reviews* → *Aggregate Weekly Stats*

16. **Create Code Node to Calculate Weekly Metrics**  
    - Name: *Calculate Weekly Metrics*  
    - Calculate total counts, sentiment breakdown percentages, top 5 issues and highlights by frequency  
    - Connect *Aggregate Weekly Stats* → *Calculate Weekly Metrics*

17. **Create OpenAI GPT-4 Node to Generate Weekly Summary**  
    - Name: *Generate Weekly Summary (GPT-4)*  
    - Prompt: Generate executive summary including sentiment trends, platform insights, top wins/issues, recommendations in professional tone  
    - Input: JSON metrics from previous node  
    - Connect *Calculate Weekly Metrics* → *Generate Weekly Summary (GPT-4)*  
    - Configure OpenAI API credentials

18. **Create Gmail Node to Send Weekly Email Report**  
    - Name: *Send Weekly Email Report*  
    - Recipient: manager email from config  
    - Subject: "Weekly Review Summary - {current date}"  
    - Body: HTML formatted with key metrics, platform breakdown, highlights/issues lists, AI summary  
    - Use Gmail credentials with service account  
    - Connect *Generate Weekly Summary (GPT-4)* → *Send Weekly Email Report*

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automatically monitors your reviews across Google, Facebook, Trustpilot, and Yelp. Each morning it fetches new reviews, normalizes the data, performs AI sentiment analysis, alerts on negatives, logs all data, and sends a weekly summary email on Mondays. | Sticky Note node content: "How it works" section                                                 |
| Setup instructions: Add credentials and business IDs/URLs for each platform; configure Slack, Google Sheets, and email credentials; adjust AI prompts and sentiment thresholds; set timezone and schedule; test before going live.                                    | Sticky Note node content: "Setup steps" section                                                  |
| For Slack alerts, ensure permissions and webhook configuration are correct. Slack message formatting uses markdown with bullet points and bold for clarity.                                                                                                        | Alert - Negative Review node details                                                             |
| Google Sheets logging requires a service account with editor permissions on the target Sheet. The appendOrUpdate operation uses `date` as the matching column.                                                                                                     | Log Review to Sheet node details                                                                 |
| AI nodes require OpenAI GPT-4 credentials and rely on structured JSON prompts and responses. Handle possible API rate limits or errors gracefully.                                                                                                                | Sentiment Analysis and Weekly Summary nodes                                                      |
| Weekly report email is sent via Gmail node using service account authentication; ensure correct scopes and email address configured.                                                                                                                               | Send Weekly Email Report node details                                                            |
| The workflow expects environment variables for API keys and tokens: `GOOGLE_API_KEY`, `FACEBOOK_ACCESS_TOKEN`, `TRUSTPILOT_API_KEY`, and Yelp token placeholder; substitute accordingly.                                                                           | Workflow Configuration and HTTP Request nodes                                                   |
| For detailed instructions on n8n usage and OpenAI node configuration, refer to official n8n documentation: https://docs.n8n.io/nodes/ and OpenAI API docs: https://platform.openai.com/docs/api-reference | External documentation links                                                                     |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.