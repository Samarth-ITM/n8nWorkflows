Curate & Translate News from RSS using Google Gemini, Sheets, and Slack

https://n8nworkflows.xyz/workflows/curate---translate-news-from-rss-using-google-gemini--sheets--and-slack-11301


# Curate & Translate News from RSS using Google Gemini, Sheets, and Slack

### 1. Workflow Overview

This workflow automates the process of curating and translating news articles from an RSS feed, using Google Gemini AI models for rewriting and translation, storing results in Google Sheets, and notifying a Slack channel. It is designed primarily for content curators, localization teams, and travel bloggers who need to monitor news, extract relevant content, translate it into multiple languages, and archive it for easy access and alerting.

The workflow is logically divided into three main blocks:

- **1.1 Fetch & Filter**: Scheduled retrieval of news articles from NHK RSS feed, limiting the number of items, and filtering articles by keywords such as "Tokyo" or "Season."
- **1.2 AI Analysis & Parsing**: Use of Google Gemini AI to rewrite the filtered articles, extract key location names, and translate the rewritten text into English, Chinese, and Korean.
- **1.3 Archive & Notify**: Formatting and cleaning the AI outputs, appending the structured data to Google Sheets, and sending formatted notifications to a Slack channel. Also, notifying Slack when articles are skipped due to keyword filtering.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Fetch & Filter

- **Overview:**  
  This block runs on a daily schedule at 20:00, fetches the latest news items from the NHK RSS feed, limits the number of items processed, and filters articles based on the presence of specific keywords ("Tokyo" or "Season") in their titles.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch RSS Feed  
  - Limit Items  
  - Filter Keywords  
  - Notify Skipped (conditional)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* scheduleTrigger  
    - *Role:* Initiates the workflow once daily at 20:00 (8 PM).  
    - *Configuration:* Interval trigger set to trigger at hour 20 every day.  
    - *Inputs:* None (start node).  
    - *Outputs:* Connected to Fetch RSS Feed.  
    - *Failure Modes:* Potential timing issues if server time zone differs from expected; no authentication required.  

  - **Fetch RSS Feed**  
    - *Type:* rssFeedRead  
    - *Role:* Fetches news items from NHK RSS feed URL `https://www.nhk.or.jp/rss/news/cat3.xml`.  
    - *Configuration:* Default options for RSS reading.  
    - *Inputs:* From Schedule Trigger.  
    - *Outputs:* To Limit Items.  
    - *Failure Modes:* Network errors, RSS feed downtime, malformed RSS.  

  - **Limit Items**  
    - *Type:* limit  
    - *Role:* Limits the number of RSS items passed downstream (default limit, not explicitly configured).  
    - *Inputs:* From Fetch RSS Feed.  
    - *Outputs:* To Filter Keywords.  
    - *Failure Modes:* None significant; empty input results in no further processing.  

  - **Filter Keywords**  
    - *Type:* if  
    - *Role:* Filters articles whose titles contain any of the keywords `"tokyo"` or `"season"` (case-sensitive).  
    - *Configuration:* OR combinator with three conditions: empty string, `"tokyo"`, `"season"` contained in the article title.  
    - *Inputs:* From Limit Items.  
    - *Outputs:* True branch to Rewrite Article; False branch to Notify Skipped.  
    - *Failure Modes:* Expression evaluation failure if article title is missing or null; logic could miss variants due to case sensitivity.  

  - **Notify Skipped**  
    - *Type:* slack  
    - *Role:* Sends a Slack message notifying that an article was skipped due to missing keywords.  
    - *Configuration:* Uses Slack OAuth2 credentials; posts to channel `C09RFVAC7FD` (`all-yuki`); message includes article title and reason for skip.  
    - *Inputs:* False branch from Filter Keywords.  
    - *Outputs:* None (end node for skips).  
    - *Failure Modes:* Slack API errors, auth token expiration, channel not found.

---

#### 2.2 Block: AI Analysis & Parsing

- **Overview:**  
  This block processes filtered articles through Google Gemini AI models. It rewrites articles into structured journalistic content, extracts the most important location name, and translates the rewritten text into English, Chinese, and Korean.

- **Nodes Involved:**  
  - Rewrite Article (LangChain Agent)  
  - Extract Location (LangChain Agent)  
  - Translate Content (LangChain Agent)  
  - Gemini Chat Model, Gemini Chat Model1 (AI model references)

- **Node Details:**

  - **Rewrite Article**  
    - *Type:* langchain.agent  
    - *Role:* Uses Google Gemini to rewrite the original article content into a structured format with sections such as title, introduction, summary, background, outlook, and conclusion.  
    - *Configuration:*  
      - Input text: `{{$json.content}}` (original article content).  
      - System prompt instructs to create a neutral, objective, and easy-to-understand article with defined sections (title, intro, summary, background, outlook, summary).  
    - *Inputs:* True branch from Filter Keywords.  
    - *Outputs:* To Extract Location.  
    - *Failure Modes:* API authentication failure, text length limits, output format errors.  

  - **Extract Location**  
    - *Type:* langchain.agent  
    - *Role:* Extracts the single most important location or facility name from the original article content.  
    - *Configuration:*  
      - Input text: `{{$('Limit Items').item.json.content}}` (original article content).  
      - System prompt requests output of only one location name, no explanations.  
    - *Inputs:* From Rewrite Article.  
    - *Outputs:* To Translate Content.  
    - *Failure Modes:* Extraction ambiguity, misinterpretation, API errors.  

  - **Translate Content**  
    - *Type:* langchain.agent  
    - *Role:* Translates the rewritten article (Japanese) into English, Simplified Chinese, and Korean.  
    - *Configuration:*  
      - Input text: output from Rewrite Article (Japanese article).  
      - System prompt demands JSON output with keys `en`, `zh`, and `ko` only, forbidding extra text or Markdown.  
    - *Inputs:* From Extract Location.  
    - *Outputs:* To Format Data.  
    - *Failure Modes:* JSON formatting errors, incomplete translations, API failures.  

  - **Gemini Chat Model** and **Gemini Chat Model1**  
    - *Type:* Google Gemini PaLM API connectors  
    - *Role:* Provide the AI language model backend for Rewrite Article, Extract Location, and Translate Content nodes.  
    - *Configuration:* Each node uses the same Google Palm API credential.  
    - *Inputs:* Connected in ai_languageModel parameter for relevant AI agent nodes.  
    - *Failure Modes:* API rate limits, authentication, quota exhaustion.

---

#### 2.3 Block: Archive & Notify

- **Overview:**  
  This block formats the AI outputs, cleans markdown, extracts summaries, and consolidates data into a structured JSON object. It then appends or updates the data in Google Sheets and sends formatted notifications to Slack. It ensures that article metadata and translations are stored and communicated effectively.

- **Nodes Involved:**  
  - Format Data (Code)  
  - Save to Google Sheets  
  - Notify Slack

- **Node Details:**

  - **Format Data**  
    - *Type:* code  
    - *Role:*  
      - Consolidates data from multiple preceding nodes: original filtered articles, AI rewritten text, extracted locations, and translations.  
      - Cleans markdown characters (#, *) and excessive newlines in the rewritten articles.  
      - Parses JSON translation output safely, with fallback defaults.  
      - Extracts a concise summary from the AI rewritten article’s "Key Points Summary" section or defaults to the first 120 characters.  
      - Outputs a unified JSON object with keys: `title`, `url`, `location`, `short_summary`, `original_clean`, and translations (`en`, `zh`, `ko`).  
    - *Inputs:* From Translate Content.  
    - *Outputs:* To Save to Google Sheets.  
    - *Failure Modes:* Parsing errors, missing data from upstream nodes, runtime JS errors.  

  - **Save to Google Sheets**  
    - *Type:* googleSheets  
    - *Role:* Appends or updates the structured article data into a Google Sheet for archival and tracking.  
    - *Configuration:*  
      - Document ID and sheet name specified (sheet gid=0).  
      - Columns mapped: title, summary, location, en, zh, ko, url.  
      - Matching on URL to avoid duplicates.  
    - *Inputs:* From Format Data.  
    - *Outputs:* To Notify Slack.  
    - *Failure Modes:* Google OAuth2 token expiration, API quota, permission issues on the sheet.  

  - **Notify Slack**  
    - *Type:* slack  
    - *Role:* Sends a formatted Slack notification for each processed article, including translation outputs and location info.  
    - *Configuration:*  
      - OAuth2 credentials for Slack account.  
      - Posts to channel `C09RFVAC7FD` (`all-yuki`).  
      - Message includes translated content, location extracted, and original article link.  
    - *Inputs:* From Save to Google Sheets.  
    - *Outputs:* None (workflow end for successful articles).  
    - *Failure Modes:* Slack API errors, token expiration, message formatting errors.

---

### 3. Summary Table

| Node Name        | Node Type                      | Functional Role                    | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                           |
|------------------|--------------------------------|----------------------------------|--------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger | scheduleTrigger                | Starts workflow daily at 20:00   | None                     | Fetch RSS Feed           | ## 1. Fetch & Filter<br>Runs on a schedule to fetch latest RSS items and filter articles by keywords (Tokyo, Season).|
| Fetch RSS Feed   | rssFeedRead                   | Fetches NHK RSS news feed        | Schedule Trigger         | Limit Items              |                                                                                                                       |
| Limit Items      | limit                         | Limits number of items processed | Fetch RSS Feed           | Filter Keywords          |                                                                                                                       |
| Filter Keywords  | if                            | Filters articles by keywords     | Limit Items              | Rewrite Article, Notify Skipped |                                                                                                                       |
| Notify Skipped   | slack                         | Alerts Slack of skipped articles | Filter Keywords (False)  | None                     |                                                                                                                       |
| Rewrite Article  | langchain.agent               | AI rewrites article content      | Filter Keywords (True)   | Extract Location         | ## 2. AI Analysis & Parsing<br>Uses Google Gemini to rewrite news, extract locations, and translate content.           |
| Extract Location | langchain.agent               | Extracts key location name       | Rewrite Article          | Translate Content        |                                                                                                                       |
| Translate Content| langchain.agent               | Translates article into EN, ZH, KO | Extract Location      | Format Data              |                                                                                                                       |
| Gemini Chat Model| lmChatGoogleGemini            | AI model backend for rewriting   | None (ai_languageModel)  | Rewrite Article, Extract Location |                                                                                                                       |
| Gemini Chat Model1| lmChatGoogleGemini           | AI model backend for translation | None (ai_languageModel)  | Translate Content        |                                                                                                                       |
| Format Data      | code                          | Cleans, parses, and consolidates data | Translate Content      | Save to Google Sheets    | ## 3. Archive & Notify<br>Appends structured data to Sheets and notifies Slack.                                       |
| Save to Google Sheets | googleSheets              | Archives data in Google Sheets   | Format Data              | Notify Slack             |                                                                                                                       |
| Notify Slack     | slack                         | Sends Slack notification         | Save to Google Sheets    | None                     |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: scheduleTrigger  
   - Set interval to trigger once daily at hour 20 (8 PM).  
   - No credentials needed.

2. **Add Fetch RSS Feed node**  
   - Type: rssFeedRead  
   - URL: `https://www.nhk.or.jp/rss/news/cat3.xml`  
   - Connect Schedule Trigger output to this node’s input.

3. **Add Limit Items node**  
   - Type: limit  
   - Default limit (or specify as needed).  
   - Connect Fetch RSS Feed output to this node.

4. **Add Filter Keywords node (If node)**  
   - Type: if (version 2)  
   - Conditions (OR): Check if `{{ $json.title }}` contains any of: empty string, `"tokyo"`, `"season"` (case sensitive).  
   - Connect Limit Items output to this node.

5. **Add Notify Skipped node**  
   - Type: slack  
   - Configure Slack OAuth2 credentials.  
   - Select channel `C09RFVAC7FD` (`all-yuki`).  
   - Message text:  
     ```
     ==[Skipped Notification]
     The article "{{ $json.title }}" did not match the keywords (Tokyo or Season), so summarization was skipped.
     ```  
   - Connect Filter Keywords False output to this node.

6. **Add Rewrite Article node (LangChain Agent)**  
   - Type: langchain.agent  
   - Text input: `{{$json.content}}` (article content)  
   - System message prompt: instruct to rewrite article into structured journalistic format with six sections (title, intro, summary, background, outlook, summary), neutral tone, easy language.  
   - Connect Filter Keywords True output to this node.  
   - Set `Gemini Chat Model` as AI language model credential.

7. **Add Extract Location node (LangChain Agent)**  
   - Type: langchain.agent  
   - Text input: `{{$('Limit Items').item.json.content}}`  
   - System message: Extract single most important location name, no extra text.  
   - Connect Rewrite Article output to this node.  
   - Reuse `Gemini Chat Model`.

8. **Add Translate Content node (LangChain Agent)**  
   - Type: langchain.agent  
   - Text input: output JSON text from Rewrite Article node.  
   - System message: Translate input Japanese article into English, Simplified Chinese, Korean; output strictly JSON with keys `en`, `zh`, `ko`.  
   - Connect Extract Location output to this node.  
   - Use `Gemini Chat Model1` for AI model.

9. **Add Format Data node (Code node)**  
   - Type: code  
   - Copy and paste the JavaScript code provided in the original workflow.  
   - This code references `Filter Keywords`, `Rewrite Article`, `Extract Location`, and `Translate Content` nodes to combine and format data.  
   - Connect Translate Content output to this node.

10. **Add Save to Google Sheets node**  
    - Type: googleSheets  
    - Configure Google Sheets OAuth2 credentials.  
    - Set Document ID to your target spreadsheet ID.  
    - Sheet name: use `gid=0` or specify your target sheet.  
    - Mapping Mode: define columns for `title`, `summary`, `location`, `en`, `zh`, `ko`, and `url`.  
    - Matching column: `url` for append or update.  
    - Connect Format Data output to this node.

11. **Add Notify Slack node**  
    - Type: slack  
    - Configure Slack OAuth2 credentials (same as Notify Skipped).  
    - Channel: `C09RFVAC7FD` (`all-yuki`).  
    - Message Type: attachment with two parts:  
      - Part 1: original translated content and extracted location.  
      - Part 2: English, Chinese, and Korean translations from formatted data.  
    - Connect Save to Google Sheets output to this node.

12. **Add Gemini Chat Model and Gemini Chat Model1 nodes**  
    - Type: lmChatGoogleGemini  
    - Credentials: Google Palm API credentials with valid API access.  
    - Connect these nodes as AI language models to the respective langchain.agent nodes:  
      - Gemini Chat Model → Rewrite Article and Extract Location  
      - Gemini Chat Model1 → Translate Content

13. **Configure all credentials and test nodes individually**  
    - Google Sheets OAuth2 with read/write access to the target spreadsheet.  
    - Slack OAuth2 with posting rights to the specified Slack channel.  
    - Google Palm API key for Gemini AI.

14. **Run and monitor workflow**  
    - Trigger manually or wait for schedule.  
    - Verify Slack notifications and Google Sheets data correctness.  
    - Handle any API errors or rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow is designed to be multilingual and supports Japanese source articles rewritten and translated into English, Simplified Chinese, and Korean.             | Workflow description                                                                                   |
| The AI rewriting prompt includes detailed structure instructions for article clarity and reader accessibility.                                                        | Node: Rewrite Article system prompt                                                                    |
| The code node’s robust data formatting logic prevents index misalignment by referencing the filtered original article list (`Filter Keywords`) rather than limited items. | Format Data node JavaScript comments                                                                    |
| Slack channel used is `all-yuki` (channel ID: C09RFVAC7FD); change as needed for your own workspace.                                                                   | Slack notification nodes                                                                               |
| Google Sheets document ID and sheet name must be set to a valid spreadsheet with appropriate column headers for smooth operation.                                     | Save to Google Sheets node configuration                                                              |
| Google Gemini API credentials must be configured and available in n8n for AI nodes to function.                                                                         | Gemini Chat Model nodes                                                                                 |
| This workflow respects content policies and only processes publicly available news content; no illegal or protected content is handled.                                | Disclaimer from user-provided workflow metadata                                                       |
| For more on Google Gemini (PaLM API) usage in n8n, refer to official docs: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                              | External resource                                                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.