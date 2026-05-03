Monitor Furusato Nozei Market Trends with Google News, AI Analysis and Slack Reporting

https://n8nworkflows.xyz/workflows/monitor-furusato-nozei-market-trends-with-google-news--ai-analysis-and-slack-reporting-11128


# Monitor Furusato Nozei Market Trends with Google News, AI Analysis and Slack Reporting

### 1. Workflow Overview

This workflow automates the monitoring and analysis of market trends related to Japan’s "Furusato Nozei" (Hometown Tax) system by leveraging Google News, Google Trends data, AI-driven analysis, and Slack notifications. It is designed for marketers, municipal researchers, and analysts to stay updated on relevant news and search trends without manual effort.

The workflow is logically divided into these key blocks:

- **1.1 Scheduled Input Reception:** Periodically triggers the workflow and fetches recent news articles about Furusato Nozei from Google News RSS.
- **1.2 AI Processing of News:** Formats and summarizes the news articles using an AI agent, extracting key market trends and a primary search keyword.
- **1.3 Keyword Validation and Configuration:** Cleans and verifies the extracted keyword, setting a default if none is found.
- **1.4 Search Trend Data Retrieval:** Uses the validated keyword to query Google Trends via SerpApi, retrieving timeseries data for market interest.
- **1.5 Trend Analysis and Report Generation:** Analyzes the Google Trends data with AI to produce a concise market report.
- **1.6 Reporting to Slack:** Formats the final report and sends it as a message to a specified Slack channel for team visibility.

This modular approach ensures clear data flow and easy maintenance or extension.


---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Reception

- **Overview:**  
  This block triggers the workflow daily at 9:00 AM JST and fetches the latest news articles related to "ふるさと納税" (Furusato Nozei) from Google News RSS feed.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS Read  
  - ニュース整形 (News Formatting)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution daily at 9:00 AM JST.  
    - Configuration: Interval trigger set to hour 9 (9:00 AM).  
    - Inputs: None  
    - Outputs: Triggers RSS Read node.  
    - Edge Cases: If time zone settings differ or node is disabled, workflow won't trigger.  

  - **RSS Read**  
    - Type: RSS Feed Read  
    - Role: Fetches latest news articles from Google News RSS for "ふるさと納税".  
    - Configuration: URL set to Japanese Google News RSS with query for Furusato Nozei, region Japan, language Japanese.  
    - Inputs: Trigger from Schedule node.  
    - Outputs: Passes raw news items to ニュース整形 node.  
    - Edge Cases: RSS feed downtime, network issues, or format changes may cause failure or empty results.  

  - **ニュース整形 (News Formatting)**  
    - Type: Set node  
    - Role: Extracts and concatenates the titles and snippets of up to 5 news articles into a formatted string for AI input.  
    - Configuration: Uses JavaScript expression to slice first 5 items and join title + snippet separated by "---".  
    - Inputs: News items from RSS Read.  
    - Outputs: Single string containing formatted news list to AI Agent.  
    - Edge Cases: Less than 5 news items available; empty or malformed content fields.  


#### 2.2 AI Processing of News

- **Overview:**  
  Uses an AI agent to analyze the formatted news list, extracting three key trends and a primary search keyword relevant to "Furusato Nozei".  

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model (AI language model backend)  
  - Code in JavaScript  
  - If (Keyword validation)  
  - Workflow Configuration

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain AI Agent  
    - Role: Analyzes news list text to identify market trends and extract a single keyword in a strict JSON format.  
    - Configuration: Custom system prompt instructing the AI to output exactly the specified JSON with 3 trend summaries and one keyword; fallback keyword "ふるさと納税" if none found.  
    - Inputs: News list string from ニュース整形.  
    - Outputs: Raw AI text output (expected JSON string).  
    - Edge Cases: AI may output malformed JSON or unexpected format; requires parsing and fallback handling.  
    - Sub-workflow: Uses OpenRouter Chat Model as language model.  

  - **OpenRouter Chat Model**  
    - Type: Langchain LM Chat OpenRouter  
    - Role: Provides OpenRouter-based language model service for AI Agent nodes.  
    - Configuration: Requires OpenRouter API key credential setup.  
    - Inputs: AI Agent node requests.  
    - Outputs: AI-generated textual responses.  
    - Edge Cases: API key missing, rate limits, or network errors.  

  - **Code in JavaScript**  
    - Type: Code node executing JavaScript  
    - Role: Parses AI Agent’s JSON output to structured data; cleans and validates the extracted keyword string; sets default keyword if empty.  
    - Configuration: Parses JSON, trims whitespace, replaces newlines, defaults keyword to "ふるさと納税" if empty.  
    - Inputs: AI Agent outputs with raw text.  
    - Outputs: Adds `parsedData` object with `summary` and `keyword` fields.  
    - Edge Cases: JSON parse failure (sets error flag), empty keyword fallback.  

  - **If**  
    - Type: Conditional node  
    - Role: Checks if parsed keyword is non-empty to proceed; otherwise stops workflow.  
    - Configuration: Condition checks trimmed `parsedData.keyword` is not empty.  
    - Inputs: Output from JavaScript code node.  
    - Outputs:  
      - True branch: proceeds to Workflow Configuration.  
      - False branch: No Operation node (halts further execution).  
    - Edge Cases: Keyword empty/null leads to no further processing.  

  - **Workflow Configuration**  
    - Type: Set node  
    - Role: Prepares variables for downstream Google Trends query: keyword, country code `jp`, language code `ja`.  
    - Configuration: Assigns keyword from parsed data (trimmed and cleaned), country fixed to "jp", language fixed to "ja".  
    - Inputs: If node true branch.  
    - Outputs: To Google Trends API node.  


#### 2.3 Search Trend Data Retrieval

- **Overview:**  
  Using the validated keyword, this block fetches historical search interest data from Google Trends via SerpApi.  

- **Nodes Involved:**  
  - Google Trends API  
  - Aggregate Research Data

- **Node Details:**

  - **Google Trends API**  
    - Type: HTTP Request node  
    - Role: Calls SerpApi’s Google Trends endpoint to retrieve timeseries data for the keyword in Japan.  
    - Configuration:  
      - URL: `https://serpapi.com/search`  
      - Query parameters: engine=google_trends, q=keyword, data_type=TIMESERIES, geo=JP, api_key (SerpApi key required)  
    - Inputs: Workflow Configuration output (keyword).  
    - Outputs: Raw Google Trends data JSON.  
    - Edge Cases: Missing or invalid SerpApi key, API quota limits, network errors, empty or malformed data.  

  - **Aggregate Research Data**  
    - Type: Aggregate node  
    - Role: Aggregates all data items from Google Trends API response into a single item for downstream processing.  
    - Configuration: Aggregate all item data into one.  
    - Inputs: Google Trends API output.  
    - Outputs: Single aggregated data item to Report AI Agent.  
    - Edge Cases: Input data empty or inconsistent.  


#### 2.4 Trend Analysis and Report Generation

- **Overview:**  
  This block uses AI to analyze the Google Trends timeseries data and generate a concise markdown report summarizing search volume trends and market insights.

- **Nodes Involved:**  
  - Report AI Agent  
  - OpenRouter Chat Model1  
  - Format Final Report

- **Node Details:**

  - **Report AI Agent**  
    - Type: Langchain AI Agent  
    - Role: Receives aggregated Google Trends data and keyword; outputs a markdown report analyzing trend dynamics over the past 12 months.  
    - Configuration:  
      - System message prompts for a market analyst tone, includes keyword and instructions for concise ~200 character summary.  
      - Output is markdown text only.  
    - Inputs: Aggregated Google Trends data, keyword.  
    - Outputs: Markdown report string.  
    - Edge Cases: AI output format deviations, API errors.  
    - Sub-workflow: Uses OpenRouter Chat Model1.  

  - **OpenRouter Chat Model1**  
    - Type: Langchain LM Chat OpenRouter  
    - Role: Provides language model backend for Report AI Agent.  
    - Configuration: Requires OpenRouter API key credential.  
    - Inputs: Report AI Agent requests.  
    - Outputs: AI-generated markdown text.  
    - Edge Cases: Same as previous OpenRouter node.  

  - **Format Final Report**  
    - Type: Set node  
    - Role: Packages the AI report output together with keyword and current timestamp into a structured format with fields: `report`, `keyword`, `timestamp`.  
    - Inputs: Report AI Agent output.  
    - Outputs: To Slack message node.  
    - Edge Cases: Timestamp generation failure (unlikely).  


#### 2.5 Reporting to Slack

- **Overview:**  
  Sends the formatted market trend report as a message to a predefined Slack channel for team awareness.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: Slack node  
    - Role: Posts the final report message to Slack channel "C09V7LVKTGD" (named "ふるさと納税").  
    - Configuration:  
      - Text content from `slack_text` field (mapped from formatted report).  
      - OAuth2 authentication with Slack credentials required.  
    - Inputs: Format Final Report node output.  
    - Outputs: Message delivery confirmation or error.  
    - Edge Cases: Slack OAuth token missing/expired, channel ID invalid, network errors.  


---

### 3. Summary Table

| Node Name              | Node Type                             | Functional Role                              | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                      |
|------------------------|-------------------------------------|----------------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                    | Triggers workflow daily at 9:00 AM JST       | -                        | RSS Read                 | **Schedule:** Enable the `Schedule Trigger` node to run at your preferred time (default 9:00)  |
| RSS Read               | RSS Feed Read                      | Fetches latest Furusato Nozei news from RSS  | Schedule Trigger          | ニュース整形               | **Check the RSS Feed:** The `RSS Read` node is pre-configured for "ふるさと納税".               |
| ニュース整形            | Set                               | Formats news items for AI input               | RSS Read                 | AI Agent                 |                                                                                                |
| AI Agent               | Langchain AI Agent                 | Summarizes news and extracts keyword          | ニュース整形               | Code in JavaScript       |                                                                                                |
| OpenRouter Chat Model  | LM Chat OpenRouter                 | AI language model backend                      | AI Agent (ai_languageModel) | AI Agent                 | **Configure Credentials:** Add your OpenRouter API key to the Chat Model nodes.                |
| Code in JavaScript     | Code                              | Parses AI JSON output, cleans keyword         | AI Agent                 | If                       |                                                                                                |
| If                     | If                                | Checks for non-empty keyword to continue      | Code in JavaScript       | Workflow Configuration / No Operation |                                                                                                |
| No Operation, do nothing | No Operation                      | Halts workflow if no valid keyword            | If (false branch)        | -                        |                                                                                                |
| Workflow Configuration | Set                               | Sets keyword, country, language for trends API | If (true branch)         | Google Trends API        | **Regional Settings:** Pre-set for Japan (`jp`/`ja`). Adjust here if needed.                    |
| Google Trends API      | HTTP Request                      | Fetches Google Trends timeseries data         | Workflow Configuration   | Aggregate Research Data  | **Configure Credentials:** Add your SerpApi key to this node.                                 |
| Aggregate Research Data | Aggregate                        | Aggregates Google Trends data into one item   | Google Trends API        | Report AI Agent          |                                                                                                |
| Report AI Agent        | Langchain AI Agent                 | Analyzes trends data, outputs markdown report | Aggregate Research Data  | Format Final Report      |                                                                                                |
| OpenRouter Chat Model1 | LM Chat OpenRouter                 | AI language model backend                      | Report AI Agent (ai_languageModel) | Report AI Agent          | **Configure Credentials:** Add your OpenRouter API key to the Chat Model nodes.                |
| Format Final Report    | Set                               | Packages report, keyword, timestamp           | Report AI Agent          | Send a message           |                                                                                                |
| Send a message         | Slack                             | Sends final report message to Slack channel   | Format Final Report      | -                        | **Configure Credentials:** Connect your Slack account here.                                   |
| Sticky Note1           | Sticky Note                       | Describes overall workflow purpose and requirements | -                        | -                        | ## How it works… (See full sticky note content in section 5)                                  |
| Sticky Note2           | Sticky Note                       | Notes regional settings (Japan-specific)      | -                        | -                        |                                                                                                |
| Sticky Note3           | Sticky Note                       | Notes schedule trigger configuration          | -                        | -                        |                                                                                                |
| Sticky Note4           | Sticky Note                       | Notes RSS Feed setup                           | -                        | -                        |                                                                                                |
| Sticky Note5           | Sticky Note                       | Notes SerpApi key configuration                | -                        | -                        |                                                                                                |
| Sticky Note6           | Sticky Note                       | Notes Slack credential configuration           | -                        | -                        |                                                                                                |
| Sticky Note            | Sticky Note                       | Notes OpenRouter API key configuration         | -                        | -                        |                                                                                                |


---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 9:00 AM (local Japan time preferable).  

2. **Create RSS Read Node**  
   - Type: RSS Feed Read  
   - URL: `https://news.google.com/rss/search?q=ふるさと納税&hl=ja&gl=JP&ceid=JP:ja`  
   - No authentication needed.  
   - Connect Schedule Trigger → RSS Read.  

3. **Create News Formatting Node (Set)**  
   - Type: Set  
   - Add expression assignment:  
     ```javascript
     news_list = $items().slice(0,5).map(item => `タイトル：${item.json.title}\n概要：${item.json.contentSnippet}`).join('\n\n---\n\n')
     ```  
   - Connect RSS Read → ニュース整形.  

4. **Create AI Agent Node**  
   - Type: Langchain AI Agent  
   - Text input: `={{ $('ニュース整形').first().json.news_list }}`  
   - System message prompt:  
     ```
     You are an expert research analyst. Analyze the following news articles and output a JSON with 3 key trends and one keyword (fallback to "ふるさと納税" if none). Output ONLY the JSON.
     Format:
     {
       "summary": "Three trend summaries as bullet points",
       "keyword": "extracted keyword"
     }
     ```  
   - Connect ニュース整形 → AI Agent.  

5. **Create OpenRouter Chat Model Node**  
   - Type: Langchain LM Chat OpenRouter  
   - Insert your OpenRouter API key credential.  
   - Connect OpenRouter Chat Model → AI Agent (select as language model).  

6. **Create JavaScript Code Node**  
   - Type: Code  
   - Paste JavaScript to parse AI JSON output, clean keyword, set default "ふるさと納税" if empty:  
     ```javascript
     const items = $input.all();
     for (const item of items) {
       try {
         const aiOutput = JSON.parse(item.json.text);
         item.json.parsedData = aiOutput;
       } catch {
         item.json.error = 'Failed to parse JSON';
         item.json.parsedData = {};
       }
       let keyword = item.json.parsedData.keyword || '';
       keyword = keyword.trim().replace(/\n/g, '');
       if (keyword === '') keyword = 'ふるさと納税';
       item.json.parsedData.keyword = keyword;
     }
     return items;
     ```  
   - Connect AI Agent → Code in JavaScript node.  

7. **Create If Node**  
   - Type: If  
   - Condition: Check if `parsedData.keyword.trim()` is not empty  
   - True branch continues; False branch connects to No Operation node.  
   - Connect Code in JavaScript → If.  

8. **Create No Operation Node**  
   - Type: No Operation  
   - Connect If (false output) → No Operation.  

9. **Create Workflow Configuration Node (Set)**  
   - Type: Set  
   - Assignments:  
     - `keyword = {{ $json.parsedData.keyword.trim().replace(/\n/g, '') }}`  
     - `country = "jp"`  
     - `language = "ja"`  
   - Connect If (true output) → Workflow Configuration.  

10. **Create Google Trends API Node (HTTP Request)**  
    - Type: HTTP Request  
    - URL: `https://serpapi.com/search`  
    - Method: GET  
    - Query parameters:  
      - `engine=google_trends`  
      - `q={{ $node["Workflow Configuration"].json.keyword }}`  
      - `data_type=TIMESERIES`  
      - `geo=JP`  
      - `api_key=<your SerpApi key>` (replace with credential or parameter)  
    - Connect Workflow Configuration → Google Trends API.  

11. **Create Aggregate Node**  
    - Type: Aggregate  
    - Operation: Aggregate all item data into one  
    - Connect Google Trends API → Aggregate Research Data.  

12. **Create Report AI Agent Node**  
    - Type: Langchain AI Agent  
    - Text input: `=` (empty or pass the aggregated data as needed)  
    - System message prompt:  
      ```
      You are a market analyst. Analyze the Google Trends data for keyword "{{ $json.keyword }}". Provide a concise markdown report (~200 characters) summarizing the search volume trend over the past 12 months.
      Output markdown only.
      ```  
    - Connect Aggregate Research Data → Report AI Agent.  

13. **Create OpenRouter Chat Model1 Node**  
    - Type: Langchain LM Chat OpenRouter  
    - Insert OpenRouter API key credential.  
    - Connect OpenRouter Chat Model1 → Report AI Agent (language model).  

14. **Create Format Final Report Node (Set)**  
    - Type: Set  
    - Assignments:  
      - `report = {{ $json.output }}` (output of Report AI Agent)  
      - `keyword = {{ $('Workflow Configuration').first().json.keyword }}`  
      - `timestamp = {{ $now.toISO() }}`  
    - Connect Report AI Agent → Format Final Report.  

15. **Create Slack Message Node**  
    - Type: Slack  
    - Authentication: OAuth2 (configure with Slack API token)  
    - Channel: Set to Slack channel ID `C09V7LVKTGD` (named "ふるさと納税")  
    - Text: `={{ $json.report }}` (or mapped `slack_text` field)  
    - Connect Format Final Report → Send a message.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| ## How it works: Acts as a specialized market analyst for Japan's "Furusato Nozei" system. Automates monitoring of news, validates keyword popularity using Google Trends, and delivers strategic reports to Slack. Useful for marketers and municipal researchers to track a competitive market without manual searching.                                                                 | Sticky Note1 Content                                |
| 🛠️ Requirements: n8n v1.0+, OpenRouter API key (or swap with OpenAI/Anthropic), SerpApi key for Google Trends, Slack account with posting permissions.                                                                                                                                                                                                                                       | Sticky Note1 Content                                |
| **Regional Settings:** Workflow preset for Japan (`jp` country, `ja` language). Modify in Workflow Configuration and Google Trends API nodes for other regions.                                                                                                                                                                                                                             | Sticky Note2 Content                                |
| **Schedule:** Default trigger is 9:00 AM JST daily. Adjust Schedule Trigger node to preferred time.                                                                                                                                                                                                                                                                                          | Sticky Note3 Content                                |
| **RSS Feed:** Preconfigured to fetch “ふるさと納税” news from Japanese Google News RSS. Can be changed if needed.                                                                                                                                                                                                                                                                              | Sticky Note4 Content                                |
| **SerpApi Key:** Required for Google Trends API node to fetch data. Ensure credential is properly configured.                                                                                                                                                                                                                                                                                  | Sticky Note5 Content                                |
| **Slack Credentials:** Slack OAuth2 authentication required for Send a message node. Ensure valid permissions and channel ID.                                                                                                                                                                                                                                                                 | Sticky Note6 Content                                |
| **OpenRouter API Key:** Needed for AI Agent and Report AI Agent nodes that use OpenRouter Chat Model. Add credentials in these nodes. Alternatively, swap with other supported LLM providers if desired.                                                                                                                                                                                        | Sticky Note Content near AI nodes                   |

---

This completes the detailed technical documentation and reconstruction instructions for the "Monitor Furusato Nozei Market Trends with Google News, AI Analysis and Slack Reporting" workflow.