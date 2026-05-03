Generate Content Strategy Reports Analyzing Reddit, YouTube & X with Gemini

https://n8nworkflows.xyz/workflows/generate-content-strategy-reports-analyzing-reddit--youtube---x-with-gemini-5375


# Generate Content Strategy Reports Analyzing Reddit, YouTube & X with Gemini

### 1. Workflow Overview

This workflow, titled **"Generate Content Strategy Reports Analyzing Reddit, YouTube & X with Gemini"**, is designed to automate the process of collecting, filtering, analyzing, and reporting content trends related to a user-provided keyword across multiple social media platforms: Reddit, YouTube, and X (formerly Twitter). It leverages third-party APIs and scraping tools to gather raw data, applies AI models (specifically Google Gemini) to pre-filter and deeply analyze content, and finally synthesizes a comprehensive HTML report. The report is delivered via Gmail and a Feishu group chat, with archival of results into Google Sheets.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures user input keyword and sets analysis parameters.
- **1.2 Data Collection:** Queries Reddit, YouTube, and X for relevant content using official APIs and third-party scraping services.
- **1.3 Data Formatting & Merging:** Normalizes and merges data from disparate sources into a unified format.
- **1.4 AI Pre-filtering:** Uses an AI language model to preliminarily filter content for relevance.
- **1.5 Filtering Decision:** Routes data based on AI decisions, handling errors or irrelevant content.
- **1.6 Deep AI Analysis:** Performs in-depth structured analysis on filtered content to identify insights.
- **1.7 Report Synthesis:** Aggregates deep analysis results and synthesizes a final, formatted HTML report.
- **1.8 Notification & Archiving:** Sends the report via Gmail and Feishu, and archives data into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures the core keyword input from a user via a form trigger and sets initial analysis parameters including keyword, search date, and a unique analysis ID.

**Nodes Involved:**  
- Form Trigger  
- Analysis Parameters  

**Node Details:**  

- **Form Trigger**  
  - *Type:* formTrigger  
  - *Role:* Entry point capturing keyword input from a user form titled "选题捕手".  
  - *Configuration:* Single required field named "keyword" with placeholder "e.g.AI".  
  - *Connections:* Outputs to "Analysis Parameters".  
  - *Failure cases:* Missing required input; webhook issues.

- **Analysis Parameters**  
  - *Type:* set  
  - *Role:* Sets and normalizes the keyword, current date, and generates a unique analysis ID using timestamp and keyword.  
  - *Key expressions:*  
    - `keyword` extracted from form data or defaults to 'AI'.  
    - `search_date` as current date (yyyy-MM-dd).  
    - `analysis_id` as timestamp + sanitized keyword.  
  - *Connections:* Outputs to data collection nodes (YouTube, Reddit, X).  
  - *Failure cases:* Expression evaluation errors.

---

#### 1.2 Data Collection

**Overview:**  
Fetches recent content related to the keyword from Reddit, YouTube, and X. Supports multiple data sources for X, including official API and third-party scraping tools.

**Nodes Involved:**  
- Reddit: Search Posts  
- Format Reddit Data  
- YouTube: Search Videos  
- Split Out  
- Format YouTube Data  
- X: Search Tweets  
- Parse Twitter Data  
- Apify抓取x推文 (Apify X Tweets)  
- ScrapingBee抓取x推文 (ScrapingBee X Tweets)  
- twitterapi抓取x推文 (TwitterAPI X Tweets)  
- Sticky Note1 (Documentation for X scraping tools)  

**Node Details:**  

- **Reddit: Search Posts**  
  - *Type:* reddit node  
  - *Role:* Searches Reddit posts matching the keyword.  
  - *Credentials:* Reddit OAuth2.  
  - *Config:* Searches all Reddit locations.  
  - *Connections:* Output to "Format Reddit Data".  
  - *Failure cases:* OAuth errors, quota limits, network errors.

- **Format Reddit Data**  
  - *Type:* set  
  - *Role:* Normalizes Reddit post data into unified structure with fields: source_weight=0.7, source_type='reddit', content (title + selftext), url, engagement_score (ups).  
  - *Connections:* Output to "Merge: All Sources".  
  - *Failure cases:* Missing fields in input data.

- **YouTube: Search Videos**  
  - *Type:* httpRequest  
  - *Role:* Queries YouTube Data API v3 for videos matching the keyword published within last 7 days.  
  - *Credentials:* YouTube API key (httpQueryAuth).  
  - *Config:* Parameters include "part=snippet", "type=video", "order=relevance", "maxResults=15".  
  - *Connections:* Output to "Split Out".  
  - *Failure cases:* API quota exceeded, invalid API key, network errors.

- **Split Out**  
  - *Type:* splitOut  
  - *Role:* Splits YouTube API response array of video items into individual items for processing.  
  - *Connections:* Output to "Format YouTube Data".  

- **Format YouTube Data**  
  - *Type:* set  
  - *Role:* Normalizes YouTube video data into unified structure with fields: source_weight=0.8, source_type='youtube', content (title + description), url (constructed YouTube link), channel.  
  - *Connections:* Output to "Merge: All Sources".  

- **X: Search Tweets**  
  - *Type:* twitter node (Twitter OAuth2 API)  
  - *Role:* Searches recent tweets matching the keyword.  
  - *Credentials:* Twitter OAuth2 API (X).  
  - *Connections:* Output to "Parse Twitter Data".  
  - *Failure cases:* Rate limits, authentication errors.

- **Parse Twitter Data**  
  - *Type:* code  
  - *Role:* Normalizes tweet JSON into unified structure: source_weight=0.9, source_type='twitter', content=text, url constructed from tweet ID, engagement_score calculated from public_metrics (likes + retweets + replies + quotes).  
  - *Connections:* Output to "Merge: All Sources".  
  - *Edge cases:* Missing public_metrics, malformed tweet data, error handling included.

- **Apify抓取x推文**, **ScrapingBee抓取x推文**, **twitterapi抓取x推文**  
  - *Type:* httpRequest  
  - *Role:* Alternative third-party scraping tools to fetch tweets from X.  
  - *Credentials:* API keys for respective services.  
  - *Sticky Note1:* Documents that these are optional third-party tools with limited free quota, and configuration details reference official docs.  
  - *Connections:* These nodes are standalone and not connected downstream in the current workflow context (potential for extension or comparison).  
  - *Failure cases:* API limits, proxy issues, authentication errors.

---

#### 1.3 Data Formatting & Merging

**Overview:**  
Merges content from Reddit, YouTube, and X into a unified stream and batches them for further processing.

**Nodes Involved:**  
- Merge: All Sources  
- Loop Over Items  

**Node Details:**  

- **Merge: All Sources**  
  - *Type:* merge  
  - *Role:* Merges three input streams (Reddit, YouTube, Twitter) into one unified output.  
  - *Connections:* Outputs to "Loop Over Items".  
  - *Failure cases:* Missing input streams, data format mismatch.

- **Loop Over Items**  
  - *Type:* splitInBatches  
  - *Role:* Batches merged content into groups of 10 items for batch AI processing.  
  - *Connections:* Outputs batch 1 to nowhere (empty), batch 2 to "AI Pre-filtering".  
  - *Note:* Possibly designed to separate initial batch for testing or asynchronous processing.  
  - *Failure cases:* Batch size misconfiguration.

---

#### 1.4 AI Pre-filtering

**Overview:**  
Uses an AI language model (Google Gemini via LangChain node) to pre-filter content, marking each item as "YES" or "NO" for relevance based on defined criteria.

**Nodes Involved:**  
- AI Pre-filtering  
- Pre-filter Content (lmChatGoogleGemini)  
- Parse AI Filter Results  

**Node Details:**  

- **AI Pre-filtering**  
  - *Type:* chainLlm (LangChain)  
  - *Role:* Runs a custom prompt instructing to judge each content item's relevance with a strict JSON output format.  
  - *Config:* Batch size 20, prompt includes keyword and content array.  
  - *Connections:* Output to "Parse AI Filter Results".  
  - *Failure cases:* Timeout, API errors, malformed AI output.

- **Pre-filter Content**  
  - *Type:* lmChatGoogleGemini  
  - *Role:* Alternative or subsequent AI filtering using Google Gemini 2.0-flash model.  
  - *Credentials:* Google Palm API credentials.  
  - *Connections:* Output to "AI Pre-filtering" (loop).  
  - *Failure cases:* API quota, auth failure.

- **Parse AI Filter Results**  
  - *Type:* code  
  - *Role:* Parses AI JSON string output from batches, aggregates all filtered items into a flat array with error handling on JSON parsing.  
  - *Connections:* Output to "IF: Is Content Relevant".  
  - *Failure cases:* JSON parse errors, empty response.

---

#### 1.5 Filtering Decision

**Overview:**  
Filters content items based on AI pre-filter decision; proceeds with "YES" items and handles errors or "NO" in a separate flow.

**Nodes Involved:**  
- IF: Is Content Relevant  
- Aggregate: Relevant Items  
- Handle Filter Errors  

**Node Details:**  

- **IF: Is Content Relevant**  
  - *Type:* if  
  - *Role:* Checks if AI decision equals "YES".  
  - *Connections:*  
    - True branch: to "Aggregate: Relevant Items"  
    - False branch: to "Handle Filter Errors"  
  - *Failure cases:* Missing decision field.

- **Aggregate: Relevant Items**  
  - *Type:* aggregate  
  - *Role:* Combines all relevant items for deep analysis.  
  - *Connections:* Output to "AI Deep Analysis".  

- **Handle Filter Errors**  
  - *Type:* code  
  - *Role:* Logs and formats error information with timestamp and analysis ID for error paths.  
  - *Connections:* None downstream (end of error path).  
  - *Failure cases:* Errors within error handling code.

---

#### 1.6 Deep AI Analysis

**Overview:**  
Conducts a structured deep analysis on the filtered content using a custom LangChain prompt and Google Gemini, extracting detailed metadata and insights.

**Nodes Involved:**  
- AI Deep Analysis (chainLlm)  
- Deep Analysis (lmChatGoogleGemini)  
- Structure Analysis Result  
- Aggregate: Deep Analysis Results  

**Node Details:**  

- **AI Deep Analysis**  
  - *Type:* chainLlm  
  - *Role:* Runs a prompt that requests extraction of core info, sentiment, trending potential, topic, key arguments, and new topic suggestions in a strict JSON format.  
  - *Connections:* Output to "Structure Analysis Result".  
  - *Failure cases:* Timeout, prompt misinterpretation.

- **Deep Analysis**  
  - *Type:* lmChatGoogleGemini  
  - *Role:* Runs the same or complementary deep AI analysis with Google Gemini 2.0-flash.  
  - *Credentials:* Google Palm API.  
  - *Connections:* Output to "Structure Analysis Result".  

- **Structure Analysis Result**  
  - *Type:* code  
  - *Role:* Parses AI JSON output, cleans markdown, matches AI results with original aggregated data, enriches with source info and timestamps. Logs errors on failure.  
  - *Connections:* Output to "Aggregate: Deep Analysis Results".  
  - *Failure cases:* JSON parse errors, format mismatch.

- **Aggregate: Deep Analysis Results**  
  - *Type:* aggregate  
  - *Role:* Combines all deep analysis results into one array for final synthesis.  
  - *Connections:* Output to "AI: Synthesize Final Report".  

---

#### 1.7 Report Synthesis

**Overview:**  
Synthesizes the final content strategy report as HTML using AI, formats the payload, compiles the final text, and sends notifications.

**Nodes Involved:**  
- AI: Synthesize Final Report  
- Synthesis (lmChatGoogleGemini)  
- Format Report Payloads  
- Splicing final report  
- Send HTML Report  
- Send Feishu Card  

**Node Details:**  

- **AI: Synthesize Final Report**  
  - *Type:* agent (LangChain)  
  - *Role:* Creates a top-level HTML report from the aggregated deep analysis data, including clustering, trend prediction, and topic generation with formatted HTML tags.  
  - *Connections:* Output to "Format Report Payloads".  
  - *Failure cases:* API limit, malformed input.

- **Synthesis**  
  - *Type:* lmChatGoogleGemini  
  - *Role:* Alternative or complementary LLM synthesis using Gemini 2.5-flash model.  
  - *Credentials:* Google Palm API.  
  - *Connections:* Outputs to "AI: Synthesize Final Report" (loop).  

- **Format Report Payloads**  
  - *Type:* set  
  - *Role:* Sets report title (including keyword and date), report content (HTML), and a summary string of data counts.  
  - *Connections:* Outputs to Google Sheets, Splicing final report, and Feishu card nodes.  

- **Splicing final report**  
  - *Type:* set  
  - *Role:* Combines title, summary, and content into a plain text report version for email.  
  - *Connections:* Output to "Send HTML Report".  

- **Send HTML Report**  
  - *Type:* gmail  
  - *Role:* Sends the final report as an email with the subject set to report title and body containing the compiled report text.  
  - *Credentials:* Gmail OAuth2.  
  - *Failure cases:* Auth failure, email quota.

- **Send Feishu Card**  
  - *Type:* httpRequest  
  - *Role:* Sends an interactive card message to a Feishu group chat with report title and summary, including a note that full report was sent via Gmail.  
  - *Config:* Requires Feishu webhook URL configured in node parameters.  
  - *Failure cases:* Network errors, invalid webhook.

---

#### 1.8 Notification & Archiving

**Overview:**  
Archives the report data into a Google Sheets document for record-keeping.

**Nodes Involved:**  
- Archive Data  

**Node Details:**  

- **Archive Data**  
  - *Type:* googleSheets  
  - *Role:* Appends the report data to a specified Google Sheets document and sheet.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Parameters:* Document ID and sheet name are configured (hidden, expected to be set by the user).  
  - *Failure cases:* Auth errors, quota limits, invalid sheet.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                         | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                      |
|----------------------------|--------------------------------|---------------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------|
| Form Trigger               | formTrigger                    | Receive keyword input                  | -                               | Analysis Parameters              |                                                                                                                  |
| Analysis Parameters        | set                            | Set keyword, date, analysis_id        | Form Trigger                    | YouTube: Search Videos, Reddit: Search Posts, X: Search Tweets |                                                                                                                  |
| YouTube: Search Videos     | httpRequest                    | Fetch YouTube videos                   | Analysis Parameters             | Split Out                       |                                                                                                                  |
| Split Out                  | splitOut                      | Split YouTube results                  | YouTube: Search Videos          | Format YouTube Data              |                                                                                                                  |
| Format YouTube Data        | set                            | Normalize YouTube data                 | Split Out                      | Merge: All Sources              |                                                                                                                  |
| Reddit: Search Posts       | reddit                        | Fetch Reddit posts                     | Analysis Parameters             | Format Reddit Data              |                                                                                                                  |
| Format Reddit Data         | set                            | Normalize Reddit data                  | Reddit: Search Posts            | Merge: All Sources              |                                                                                                                  |
| X: Search Tweets           | twitter                       | Fetch tweets from X                    | Analysis Parameters             | Parse Twitter Data              |                                                                                                                  |
| Parse Twitter Data         | code                          | Normalize tweet data                   | X: Search Tweets               | Merge: All Sources              |                                                                                                                  |
| Apify抓取x推文              | httpRequest                   | Third-party scraping for X tweets     | -                               | -                              | ## X Third-party scraping tools; limited free quota; see official docs for config.                               |
| ScrapingBee抓取x推文         | httpRequest                   | Third-party scraping for X tweets     | -                               | -                              | ## X Third-party scraping tools; limited free quota; see official docs for config.                               |
| twitterapi抓取x推文          | httpRequest                   | Third-party scraping for X tweets     | -                               | -                              | ## X Third-party scraping tools; limited free quota; see official docs for config.                               |
| Merge: All Sources         | merge                         | Merge all sources' normalized data    | Format Reddit Data, Format YouTube Data, Parse Twitter Data | Loop Over Items                |                                                                                                                  |
| Loop Over Items            | splitInBatches                | Batch items for AI processing          | Merge: All Sources              | AI Pre-filtering                |                                                                                                                  |
| AI Pre-filtering           | chainLlm                      | AI-based pre-filtering of content     | Loop Over Items                | Parse AI Filter Results         | ## llm node: "Just change it to your usual one."                                                                 |
| Pre-filter Content         | lmChatGoogleGemini            | AI pre-filter alternative with Gemini |                             | AI Pre-filtering                | ## llm node: "Just change it to your usual one."                                                                 |
| Parse AI Filter Results    | code                          | Parse AI JSON filter results           | AI Pre-filtering               | IF: Is Content Relevant         |                                                                                                                  |
| IF: Is Content Relevant    | if                            | Branch based on AI decision             | Parse AI Filter Results        | Aggregate: Relevant Items (YES), Handle Filter Errors (NO) |                                                                                                                  |
| Aggregate: Relevant Items  | aggregate                     | Aggregate relevant items                | IF: Is Content Relevant (YES)  | AI Deep Analysis               |                                                                                                                  |
| Handle Filter Errors       | code                          | Log & report filtering errors           | IF: Is Content Relevant (NO)   | -                              |                                                                                                                  |
| AI Deep Analysis           | chainLlm                      | Perform deep AI analysis on filtered content | Aggregate: Relevant Items        | Structure Analysis Result       |                                                                                                                  |
| Deep Analysis             | lmChatGoogleGemini            | Deep analysis alternative with Gemini  |                              | Structure Analysis Result       |                                                                                                                  |
| Structure Analysis Result  | code                          | Parse & enrich AI deep analysis results | AI Deep Analysis, Deep Analysis | Aggregate: Deep Analysis Results |                                                                                                                  |
| Aggregate: Deep Analysis Results | aggregate              | Combine all deep analysis results       | Structure Analysis Result       | AI: Synthesize Final Report     |                                                                                                                  |
| AI: Synthesize Final Report | agent                        | Generate final HTML report from analysis | Aggregate: Deep Analysis Results | Format Report Payloads         |                                                                                                                  |
| Synthesis                  | lmChatGoogleGemini            | Alternative synthesis with Gemini      | AI: Synthesize Final Report     | AI: Synthesize Final Report     |                                                                                                                  |
| Format Report Payloads     | set                            | Format report title, content, summary  | AI: Synthesize Final Report     | Archive Data, Splicing final report, Send Feishu Card |                                                                                                                  |
| Splicing final report      | set                            | Compile full plain text report          | Format Report Payloads          | Send HTML Report               |                                                                                                                  |
| Send HTML Report           | gmail                         | Email the final report                  | Splicing final report           | -                              |                                                                                                                  |
| Send Feishu Card           | httpRequest                   | Send report summary card to Feishu chat | Format Report Payloads          | -                              | ## URL configuration settings: Feishu webhook must be obtained and configured.                                  |
| Archive Data               | googleSheets                  | Append report data to Google Sheets    | Format Report Payloads          | -                              |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Input Trigger:**  
   - Add a **Form Trigger** node named "Form Trigger".  
   - Configure with a form titled "选题捕手" containing one required text field `keyword` with placeholder "e.g.AI".  
   - Save webhook ID for external invocation.

2. **Set Analysis Parameters:**  
   - Add a **Set** node named "Analysis Parameters".  
   - Connect from "Form Trigger".  
   - Assign `keyword` from the form input (`{{$json.body.keyword || $json.query.keyword || 'AI'}}`).  
   - Assign `search_date` as current date in `yyyy-MM-dd` using `$now.toFormat('yyyy-MM-dd')`.  
   - Assign `analysis_id` as timestamp + sanitized keyword (`{{$now.toFormat('yyyyMMddHHmmss')}}_{{keyword.replace(' ', '_')}}`).

3. **Fetch Reddit Posts:**  
   - Add a **Reddit** node named "Reddit: Search Posts".  
   - Use OAuth2 credentials.  
   - Set operation to "search" with keyword from "Analysis Parameters".  
   - Connect from "Analysis Parameters".

4. **Format Reddit Data:**  
   - Add a **Set** node "Format Reddit Data".  
   - Map Reddit post fields:  
     - `source_weight`: 0.7  
     - `source_type`: "reddit"  
     - `content`: title + selftext  
     - `url`: post URL  
     - `engagement_score`: ups  
   - Connect from "Reddit: Search Posts".

5. **Fetch YouTube Videos:**  
   - Add an **HTTP Request** node "YouTube: Search Videos".  
   - Use YouTube API key credentials.  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query parameters: part=snippet, q=keyword, type=video, order=relevance, maxResults=15, publishedAfter=7 days ago ISO date.  
   - Connect from "Analysis Parameters".

6. **Split YouTube Items:**  
   - Add **Split Out** node "Split Out".  
   - Set fieldToSplitOut to "items".  
   - Connect from "YouTube: Search Videos".

7. **Format YouTube Data:**  
   - Add **Set** node "Format YouTube Data".  
   - Map fields:  
     - `source_weight`: 0.8  
     - `source_type`: "youtube"  
     - `content`: snippet.title + snippet.description  
     - `url`: constructed YouTube URL  
     - `channel`: snippet.channelTitle  
   - Connect from "Split Out".

8. **Fetch Tweets from X:**  
   - Add **Twitter** node "X: Search Tweets".  
   - Use OAuth2 credentials for X.  
   - Operation: search with keyword.  
   - Connect from "Analysis Parameters".

9. **Parse Twitter Data:**  
   - Add **Code** node "Parse Twitter Data".  
   - Implement JavaScript to calculate engagement and normalize fields:  
     - source_weight: 0.9  
     - source_type: "twitter"  
     - content: tweet text  
     - url: constructed from tweet ID  
     - engagement_score: sum of public_metrics  
   - Connect from "X: Search Tweets".

10. *(Optional)* Add third-party X scraping HTTP requests ("Apify抓取x推文", "ScrapingBee抓取x推文", "twitterapi抓取x推文") with respective API keys and configurations as needed.

11. **Merge All Sources:**  
    - Add **Merge** node "Merge: All Sources".  
    - Number of inputs: 3 (Reddit, YouTube, Twitter).  
    - Connect "Format Reddit Data", "Format YouTube Data", "Parse Twitter Data" as inputs.

12. **Batch Items:**  
    - Add **Split In Batches** node "Loop Over Items".  
    - Batch size: 10.  
    - Connect from "Merge: All Sources".

13. **AI Pre-filtering:**  
    - Add **Chain LLM** node "AI Pre-filtering".  
    - Use prompt to mark content as "YES" or "NO" for relevance with strict JSON output.  
    - Batch size: 20.  
    - Connect from second output of "Loop Over Items".

14. **Parse AI Filter Results:**  
    - Add **Code** node "Parse AI Filter Results".  
    - Flatten and parse AI JSON responses safely.  
    - Connect from "AI Pre-filtering".

15. **Decision Branch:**  
    - Add **If** node "IF: Is Content Relevant".  
    - Condition: `$json.decision == 'YES'`.  
    - Connect from "Parse AI Filter Results".

16. **Aggregate Relevant Items:**  
    - Add **Aggregate** node "Aggregate: Relevant Items".  
    - Connect True output from "IF: Is Content Relevant".

17. **Handle Filter Errors:**  
    - Add **Code** node "Handle Filter Errors".  
    - Connect False output from "IF: Is Content Relevant".

18. **Deep AI Analysis:**  
    - Add **Chain LLM** node "AI Deep Analysis".  
    - Prompt for detailed structured analysis with JSON output.  
    - Connect from "Aggregate: Relevant Items".

19. **Alternative Deep Analysis:**  
    - Add **lmChatGoogleGemini** node "Deep Analysis" with Gemini 2.0-flash model.  
    - Connect also from "Aggregate: Relevant Items".

20. **Structure Analysis Result:**  
    - Add **Code** node "Structure Analysis Result".  
    - Parse and enrich AI analysis result, match original items.  
    - Connect from both AI Deep Analysis and Deep Analysis nodes.

21. **Aggregate Deep Analysis Results:**  
    - Add **Aggregate** node "Aggregate: Deep Analysis Results".  
    - Connect from "Structure Analysis Result".

22. **Synthesize Final Report:**  
    - Add **Agent** node "AI: Synthesize Final Report".  
    - Prompt to generate HTML report with headings, lists, and bold text from deep analysis JSON.  
    - Connect from "Aggregate: Deep Analysis Results".

23. **Alternative Synthesis:**  
    - Add **lmChatGoogleGemini** node "Synthesis" with Gemini 2.5-flash.  
    - Connect from "AI: Synthesize Final Report".

24. **Format Report Payloads:**  
    - Add **Set** node "Format Report Payloads".  
    - Assign report_title, report_content (HTML), analysis_summary (counts).  
    - Connect from "AI: Synthesize Final Report".

25. **Splicing Final Report:**  
    - Add **Set** node "Splicing final report".  
    - Concatenate title, summary, content into final plain text report.  
    - Connect from "Format Report Payloads".

26. **Send HTML Report:**  
    - Add **Gmail** node "Send HTML Report".  
    - Use Gmail OAuth2 credentials.  
    - Subject: report_title.  
    - Message: final_report_text.  
    - Connect from "Splicing final report".

27. **Send Feishu Card:**  
    - Add **HTTP Request** node "Send Feishu Card".  
    - POST to Feishu webhook URL (configured in node).  
    - Payload includes interactive card with report_title and analysis_summary.  
    - Connect from "Format Report Payloads".

28. **Archive Data:**  
    - Add **Google Sheets** node "Archive Data".  
    - Append report data to configured sheet.  
    - Connect from "Format Report Payloads".

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| ## X Third-party scraping tools: X API has limited quota; third-party scraping tools (Apify, ScrapingBee, TwitterAPI) can be used as alternatives. Please check their official documentation for configuration. | Sticky Note1 near X scraping nodes.                                                                |
| ## llm node: Just change it to your usual one.                                                                    | Sticky Note near AI Pre-filtering and Pre-filter Content nodes, indicating LLM nodes are interchangeable. |
| ## URL configuration settings: Feishu Group Chat webhook must be created and webhook URL filled in the HTTP Request node. | Sticky Note2 near Send Feishu Card node.                                                           |
| The workflow is configured with OAuth2 credentials for Reddit, Twitter (X), Gmail, Google Sheets, and Google Palm API for Gemini models. Make sure to set these up correctly before running. | Credential setup requirement.                                                                      |
| The workflow relies heavily on strict JSON formatting in AI prompts and responses; errors in AI output parsing are handled but may cause workflow failures. | Important for error anticipation.                                                                  |
| The workflow uses batching and aggregation to handle large data volumes efficiently with AI models.               | Performance and scalability note.                                                                  |

---

**Disclaimer:**  
The provided description and analysis are based solely on the n8n workflow exported JSON. All configurations respect content policies and utilize only legal, public data sources.