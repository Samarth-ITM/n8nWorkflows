Daily Solana News Tracker with GPT-4.1-mini Weekly Summaries in Google Sheets

https://n8nworkflows.xyz/workflows/daily-solana-news-tracker-with-gpt-4-1-mini-weekly-summaries-in-google-sheets-9874


# Daily Solana News Tracker with GPT-4.1-mini Weekly Summaries in Google Sheets

### 1. Workflow Overview

This workflow automates the daily tracking of Solana-related news articles and generates AI-powered weekly summaries stored in Google Sheets. Its primary use case is for investors or analysts needing a consolidated, duplicate-free record of Solana news with concise weekly insights produced by GPT-4.1-mini.

The workflow logic is grouped into three major blocks:

- **1.1 Daily News Retrieval and Processing:**  
  Triggered every day to fetch latest Solana news from CryptoPanic API, format and deduplicate articles, validate data completeness, and append new unique items to the "Raw Data" Google Sheet.

- **1.2 Weekly Summary Generation:**  
  Triggered only on Mondays to retrieve accumulated weekly news, aggregate article descriptions, send them to GPT-4.1-mini for a factual summary, and append the AI-generated summary to the "Weekly Summary" Google Sheet.

- **1.3 Supporting Documentation and Notes:**  
  Sticky notes explaining API setup, duplicate detection, data validation, storing articles, AI summarization, cost considerations, and a quick start guide.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Daily News Retrieval and Processing

**Overview:**  
Fetches daily Solana news from CryptoPanic, extracts relevant fields, removes duplicates by checking existing Google Sheets entries, filters out incomplete data, and appends validated new articles to the "Raw Data" sheet.

**Nodes Involved:**  
- Run Every Day at 08:00 PT  
- Get Solana News  
- Format By Items  
- Check For Duplicates  
- Keep Non-matches  
- Get Rid Of Empties  
- Append Valid Items  
- Sticky Notes: API, Duplicates, Validation, Store Articles

**Node Details:**

- **Run Every Day at 08:00 PT**  
  - Type: Schedule Trigger  
  - Config: Fires daily at 07:55 UTC (8:00 PT)  
  - Role: Initiates daily workflow run  
  - Inputs: None  
  - Outputs: Connects to "Get Solana News"  
  - Failure Modes: Scheduler downtime or misconfiguration

- **Get Solana News**  
  - Type: HTTP Request  
  - Config: GET request to CryptoPanic API endpoint filtering for Solana news; requires user to replace `[your token]` with a valid API key  
  - Outputs: Raw JSON data with news posts  
  - Failure Modes: API key missing/invalid, network errors, rate limits, malformed JSON  
  - Notes: See Sticky Note - API for setup instructions and alternatives

- **Format By Items**  
  - Type: Code (JavaScript)  
  - Config: Extracts `title`, `description`, and `published_at` from each news item in the API response and flattens them into separate items  
  - Inputs: Raw API response items  
  - Outputs: Cleaned array of news items with required fields  
  - Failure Modes: Unexpected API structure causing undefined fields or errors in JS code

- **Check For Duplicates**  
  - Type: Google Sheets (Lookup)  
  - Config: Searches "Raw Data" sheet for existing articles matching the current item's `title`  
  - Inputs: Formatted news items  
  - Outputs: Matches and non-matches for merging  
  - Failure Modes: Google Sheets API errors, permission issues, large dataset performance degradation  
  - Notes: Sticky Note - Duplicates explains the duplicate detection logic

- **Keep Non-matches**  
  - Type: Merge  
  - Config: Combines input streams, keeping only items that did not match existing titles (i.e., new articles)  
  - Inputs: Outputs of "Format By Items" and "Check For Duplicates"  
  - Outputs: Unique new articles only  
  - Failure Modes: Misconfiguration leading to loss of data or incorrect filtering

- **Get Rid Of Empties**  
  - Type: Code (JavaScript)  
  - Config: Filters out items missing any of the required fields (`title`, `description`, `published_at`)  
  - Inputs: Unique articles  
  - Outputs: Validated articles ready for storage  
  - Failure Modes: Logic errors causing valid articles removal or empty outputs  
  - Notes: Sticky Note - Validation explains data validation logic

- **Append Valid Items**  
  - Type: Google Sheets (Append)  
  - Config: Appends validated new articles to "Raw Data" sheet with columns: `date`, `title`, `descripton` (note the intentional misspelling), and a blank `summary` column  
  - Inputs: Valid articles  
  - Outputs: Confirmation of appended rows  
  - Failure Modes: Google Sheets API errors, permission issues, schema mismatch  
  - Notes: Sticky Note - Store explains columns and storage details

---

#### 2.2 Weekly Summary Generation

**Overview:**  
Runs only on Mondays, reads all stored articles from the past week, aggregates their descriptions into a single text blob, sends it to GPT-4.1-mini for summarization, and saves the AI-generated summary in the "Weekly Summary" sheet.

**Nodes Involved:**  
- Process If Monday  
- Get News  
- Aggregate  
- Summarize News  
- Append Summary  
- Sticky Notes: Read News, Aggregate, AI Summary, Save Summary

**Node Details:**

- **Process If Monday**  
  - Type: If Condition  
  - Config: Checks if current day of week is Monday (`weekday == 2` in n8n's numbering)  
  - Inputs: Triggered from appending new items daily  
  - Outputs: Proceeds only on Monday to generate summary  
  - Failure Modes: Date/time misconfigurations causing missed summary runs

- **Get News**  
  - Type: Google Sheets (Read)  
  - Config: Reads all rows from "Raw Data" sheet (gid=0) to collect the week's articles  
  - Inputs: Trigger from "Process If Monday"  
  - Outputs: Full dataset of stored articles  
  - Failure Modes: Large data volume causing timeouts or API errors

- **Aggregate**  
  - Type: Aggregate  
  - Config: Concatenates the `descripton` field of all articles into one string  
  - Inputs: Articles from Google Sheets  
  - Outputs: Single aggregated text string containing all descriptions  
  - Failure Modes: Null or missing description fields, aggregation errors  
  - Notes: Sticky Note - Aggregate explains the purpose

- **Summarize News**  
  - Type: OpenAI (LangChain)  
  - Config: Uses GPT-4.1-mini model with low temperature (0.2) and max tokens 200 to produce a concise factual summary (2-3 sentences) of the aggregated descriptions; system prompt instructs no hype or price predictions, output as JSON with a `summary` field  
  - Inputs: Aggregated descriptions  
  - Outputs: JSON summary text  
  - Failure Modes: API key issues, rate limits, malformed prompts, model failures  
  - Credentials: Requires configured OpenAI API credentials  
  - Notes: Sticky Note - AI Summary details model and output format

- **Append Summary**  
  - Type: Google Sheets (Append)  
  - Config: Appends summary to "Weekly Summary" sheet with columns: `Date` (current date in dd/MM/yyyy) and `Summary` (AI-generated text)  
  - Inputs: AI summary JSON  
  - Outputs: Confirmation of append operation  
  - Failure Modes: Google Sheets API errors or schema mismatches  
  - Notes: Sticky Note - Save Summary explains storage details

---

#### 2.3 Supporting Documentation and Notes

**Overview:**  
Provides users with setup guidance, cost estimates, and explanations of key workflow parts through sticky notes dispersed throughout the canvas.

**Nodes Involved:**  
- Sticky Note - API  
- Sticky Note - Duplicates  
- Sticky Note - Validation  
- Sticky Note - Store  
- Sticky Note - Read News  
- Sticky Note - Aggregate  
- Sticky Note - AI Summary  
- Sticky Note - Save Summary  
- Sticky Note - Quick Start  
- Sticky Note - Cost

**Node Details:**  
Each sticky note contains markdown-formatted content explaining the respective topic such as API setup, duplicate detection logic, data validation, storage columns, AI summarization details, cost estimates (~$0.001 per summary), and a quick start checklist.

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                             | Input Node(s)                       | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------|--------------------------|--------------------------------------------|-----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Run Every Day at 08:00 PT| Schedule Trigger         | Initiates daily workflow run                | None                              | Get Solana News                 |                                                                                                |
| Get Solana News          | HTTP Request             | Fetches Solana news from CryptoPanic API   | Run Every Day at 08:00 PT          | Format By Items                 | ## 🌐 NEWS API SOURCE<br>**CryptoPanic API**<br>Fetches latest Solana news.<br>Setup instructions and alternatives listed. |
| Format By Items          | Code                     | Extracts and flattens news article fields  | Get Solana News                   | Check For Duplicates, Keep Non-matches | ## 🔍 DUPLICATE DETECTION<br>Checks if article title exists in sheet "Raw Data".              |
| Check For Duplicates     | Google Sheets (Lookup)   | Finds existing articles by title           | Format By Items                   | Keep Non-matches                |                                                                                                |
| Keep Non-matches         | Merge                    | Filters to new unique articles              | Format By Items, Check For Duplicates | Get Rid Of Empties            |                                                                                                |
| Get Rid Of Empties       | Code                     | Removes articles missing required fields   | Keep Non-matches                  | Append Valid Items              | ## 🧹 DATA VALIDATION<br>Removes items with missing fields.                                    |
| Append Valid Items       | Google Sheets (Append)   | Appends new validated articles to sheet    | Get Rid Of Empties                | Process If Monday               | ## 💾 STORE ARTICLES<br>Appends new articles to "Raw Data" sheet (date, title, description).   |
| Process If Monday        | If Condition             | Checks if today is Monday                    | Append Valid Items                | Get News                       |                                                                                                |
| Get News                 | Google Sheets (Read)     | Reads all stored news articles for the week| Process If Monday                 | Aggregate                      | ## 📖 READ ALL ARTICLES<br>Retrieves accumulated news from "Raw Data" sheet.                   |
| Aggregate               | Aggregate                | Concatenates all article descriptions       | Get News                        | Summarize News                 | ## 🔗 AGGREGATE ALL ARTICLE DESCRIPTIONS                                                   |
| Summarize News           | OpenAI (LangChain)       | Generates weekly summary using GPT-4.1-mini| Aggregate                       | Append Summary                 | ## 🤖 AI SUMMARIZATION<br>Produces 2-3 sentence factual summary with key takeaways.           |
| Append Summary           | Google Sheets (Append)   | Stores AI-generated weekly summary          | Summarize News                  | None                          | ## 📝 SAVE WEEKLY SUMMARY<br>Appends summary to "Weekly Summary" sheet with date and text.    |
| Sticky Note - API        | Sticky Note              | Explains news API source and setup          | None                           | None                          | See above                                                                                      |
| Sticky Note - Duplicates | Sticky Note              | Explains duplicate detection logic          | None                           | None                          | See above                                                                                      |
| Sticky Note - Validation | Sticky Note              | Explains data validation step                | None                           | None                          | See above                                                                                      |
| Sticky Note - Store      | Sticky Note              | Explains storage of new articles             | None                           | None                          | See above                                                                                      |
| Sticky Note - Read News  | Sticky Note              | Explains retrieval of all articles for summary| None                       | None                          | See above                                                                                      |
| Sticky Note - Aggregate  | Sticky Note              | Explains aggregation of article descriptions| None                          | None                          | See above                                                                                      |
| Sticky Note - AI Summary | Sticky Note              | Explains AI summarization step               | None                           | None                          | See above                                                                                      |
| Sticky Note - Save Summary| Sticky Note             | Explains saving of weekly summary             | None                           | None                          | See above                                                                                      |
| Sticky Note - Quick Start| Sticky Note              | Provides quick start instructions             | None                           | None                          | ## 🚀 QUICK START<br>Stepwise setup checklist for API, Google Sheets, OpenAI, activation.     |
| Sticky Note - Cost       | Sticky Note              | Explains estimated costs and data volume      | None                           | None                          | ## 💰 COST & PERFORMANCE<br>API and OpenAI cost estimates.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Run Every Day at 08:00 PT" node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 07:55 UTC (equivalent to 8:00 PT)  

2. **Create "Get Solana News" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://cryptopanic.com/api/v1/posts/?auth_token=[your token]&currencies=sol&public=true`  
   - Replace `[your token]` with your CryptoPanic API key  
   - Output: JSON response of news posts  

3. **Create "Format By Items" node**  
   - Type: Code (JavaScript)  
   - Code: Extract `title`, `description`, `published_at` from each post in `results` field, output array of items with these fields  
   - Input: Connect from "Get Solana News"  

4. **Create "Check For Duplicates" node**  
   - Type: Google Sheets (Lookup)  
   - Connect to your Google Sheets credential  
   - Document ID: Your Google Sheet with news data  
   - Sheet: "Raw Data" (gid=0)  
   - Lookup Column: `title`  
   - Lookup Value: Expression `={{ $json.title }}`  
   - Input: Connect from "Format By Items"  

5. **Create "Keep Non-matches" node**  
   - Type: Merge  
   - Mode: Combine  
   - Join Mode: Keep Non-matches  
   - Fields to Match: `title`  
   - Inputs: Connect first input to "Format By Items" output, second input to "Check For Duplicates" output  

6. **Create "Get Rid Of Empties" node**  
   - Type: Code (JavaScript)  
   - Code: Filter items to keep only those with non-empty `title`, `description`, and `published_at`  
   - Input: Connect from "Keep Non-matches"  

7. **Create "Append Valid Items" node**  
   - Type: Google Sheets (Append)  
   - Connect to Google Sheets credential  
   - Document ID: Same as above  
   - Sheet: "Raw Data" (gid=0)  
   - Columns Mapping:  
     - date : `={{ $json.published_at }}`  
     - title : `={{ $json.title }}`  
     - descripton : `={{ $json.description }}` (note the spelling)  
     - summary : leave empty or blank  
   - Operation: Append  
   - Input: Connect from "Get Rid Of Empties"  

8. **Create "Process If Monday" node**  
   - Type: If  
   - Condition: Check if current weekday equals 2 (Monday) using expression `={{ $now.weekday.toString() }}`  
   - Input: Connect from "Append Valid Items"  

9. **Create "Get News" node**  
   - Type: Google Sheets (Read)  
   - Document ID: Same as above  
   - Sheet: "Raw Data" (gid=0)  
   - Input: Connect from "Process If Monday" true branch  

10. **Create "Aggregate" node**  
    - Type: Aggregate  
    - Aggregate field: `descripton` (collect all descriptions into a single string)  
    - Input: Connect from "Get News"  

11. **Create "Summarize News" node**  
    - Type: OpenAI (LangChain)  
    - Model ID: `gpt-4.1-mini`  
    - Max Tokens: 200  
    - Temperature: 0.2  
    - System Prompt: "You are a helpful, intelligent crypto news analyst"  
    - User Prompt Template:  
      ```
      Your task is to take as an input Solana news at {{ $json.descripton }} and summarize all the descriptions into a one 2-3 factual sentence summary including key takeaways for investors to understand.

      No hype, no price predictions, fancy language. Your tone of voice should be spartan.

      Output in JSON:

      { "summary":""}
      ```  
    - Credential: Connect your OpenAI API credentials  
    - Input: Connect from "Aggregate"  

12. **Create "Append Summary" node**  
    - Type: Google Sheets (Append)  
    - Document ID: Same as above  
    - Sheet: "Weekly Summary" (gid=942745497)  
    - Columns Mapping:  
      - Date: `={{ $now.toFormat("dd/MM/yyyy") }}`  
      - Summary: `={{ $json.message.content.summary }}`  
    - Operation: Append  
    - Input: Connect from "Summarize News"  

13. **Add Sticky Notes** for documentation as per workflow for API setup, duplicate detection, validation, storage, reading news, aggregation, AI summarization, saving summary, quick start guide, and cost info.

14. **Credentials Setup:**  
    - Configure Google Sheets OAuth2 credentials with access to your spreadsheet.  
    - Configure OpenAI API key with GPT-4.1-mini model access.  

15. **Test Workflow:**  
    - Run manually first to verify API connectivity, data flow, and Google Sheets writes.  
    - Activate the workflow for daily automated runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| CryptoPanic API key is free; replace `[your token]` in the HTTP Request node URL to authenticate.                                                          | CryptoPanic API: https://cryptopanic.com                                                                    |
| Google Sheet must have two tabs: "Raw Data" with columns (date, title, descripton, summary) and "Weekly Summary" with (Date, Summary).                     | Google Sheets setup                                                                                           |
| GPT-4.1-mini usage costs approximately $0.001 per summary, estimated $0.05/month for weekly summaries.                                                     | OpenAI pricing                                                                                                |
| Workflow designed to minimize duplicates and ensure data completeness by validating essential fields.                                                     | Data quality assurance                                                                                        |
| Summaries are factual, concise, and free of hype or speculation, suitable for investor insight.                                                            | AI summarization guidelines                                                                                   |
| Workflow is inactive by default; activate after configuration and testing.                                                                                  | Workflow status                                                                                               |
| For alternative news sources, consider CoinGecko, NewsAPI, or custom RSS feeds by modifying the HTTP Request node accordingly.                            | Alternative API options                                                                                        |
| Useful for investors, analysts, or enthusiasts tracking Solana news with automated AI insights.                                                             | Use case                                                                                                      |

---

**Disclaimer:**  
The content provided is derived solely from an automated workflow built in n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected material. All data processed is legal and publicly available.