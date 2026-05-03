Classify and Summarize WeChat Articles with GPT-4 Nano to Google Sheets and Notion

https://n8nworkflows.xyz/workflows/classify-and-summarize-wechat-articles-with-gpt-4-nano-to-google-sheets-and-notion-5933


# Classify and Summarize WeChat Articles with GPT-4 Nano to Google Sheets and Notion

### 1. Workflow Overview

This workflow automates the process of classifying and summarizing WeChat articles using GPT-4.1-nano, integrating results into Google Sheets and Notion for streamlined content management and monitoring. It is designed for users who track specific topics or individuals in WeChat public account articles, filtering relevant content, generating insightful summaries in Chinese, and archiving processed data.

The workflow consists of four main logical blocks:

- **1.1 Data Input**: Reading initial article links and RSS feed URLs from Google Sheets.
- **1.2 Data Filtering and Deduplication**: Filtering articles by publication date and removing duplicate URLs to ensure only new and relevant content is processed.
- **1.3 Content Processing and AI Analysis**: Cleaning raw HTML content, classifying article relevance by topic, and generating detailed Slack-formatted summaries using GPT-4.1-nano.
- **1.4 Output and Archiving**: Saving summarized and classified articles into Google Sheets and Notion databases for further usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Input

- **Overview:**  
  This block retrieves initial data sources by reading pre-configured links and RSS feed URLs from two Google Sheets tabs. It sets the foundation for subsequent content fetching.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Read Initial Links  
  - Read RSS Links  
  - RSS Read  
  - pubDate Processing  
  - IF (Filter by Date)  
  - Filtered Data  
  - Save Initial Data

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Connections: Outputs to "Read Initial Links" and "Read RSS Links".  
    - Edge Cases: Workflow won't run unless manually triggered.

  - **Read Initial Links**  
    - Type: Google Sheets  
    - Role: Reads a specific sheet (gid=198451233) from a Google Sheets document containing initial article links.  
    - Configuration: Uses Google Sheets OAuth2 credentials; no filters applied.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Merge" node.  
    - Edge Cases: Google Sheets API errors, credential expiration, empty sheet.

  - **Read RSS Links**  
    - Type: Google Sheets  
    - Role: Reads RSS feed URLs from another sheet (gid=0) in the same Google Sheets document.  
    - Configuration: Uses same Google Sheets OAuth2 credentials.  
    - Inputs: From Manual Trigger  
    - Outputs: To "RSS Read".  
    - Edge Cases: Similar to "Read Initial Links".

  - **RSS Read**  
    - Type: RSS Feed Read  
    - Role: Fetches feed entries from each RSS URL read previously.  
    - Configuration: Reads URL dynamically from input JSON (`rss_feed_url`), does not ignore SSL errors.  
    - Inputs: From "Read RSS Links"  
    - Outputs: To "pubDate Processing"  
    - Edge Cases: Network errors, invalid or expired RSS URLs, partial feed data, and continues on errors.

  - **pubDate Processing**  
    - Type: Set  
    - Role: Normalizes pubDate into ISO string and sets default or empty fields for other metadata.  
    - Configuration: Uses expressions to convert date formats and map fields.  
    - Inputs: From "RSS Read"  
    - Outputs: To "IF (Filter by Date)".  
    - Edge Cases: Invalid or missing dates produce empty or malformed ISO strings.

  - **IF (Filter by Date)**  
    - Type: If (Conditional)  
    - Role: Filters articles published within the last 10 days.  
    - Configuration: Compares article pubDate timestamp with current date minus 10 days.  
    - Inputs: From "pubDate Processing"  
    - Outputs: True branch to "Filtered Data", false branch is discarded.  
    - Edge Cases: Date parsing errors may exclude valid articles.

  - **Filtered Data**  
    - Type: Set  
    - Role: Assigns all relevant article metadata fields explicitly for later use.  
    - Inputs: True output from "IF (Filter by Date)"  
    - Outputs: To "Save Initial Data" and "Merge1".  
    - Edge Cases: Missing fields propagate empty values.

  - **Save Initial Data**  
    - Type: Google Sheets  
    - Role: Appends or updates article metadata into the "Save Initial Links" sheet, keyed by link URL.  
    - Configuration: Uses defined columns for link, title, pubDate, with update-on-match by link.  
    - Inputs: From "Filtered Data"  
    - Outputs: To "Merge" node.  
    - Edge Cases: Google Sheets API limits, concurrency issues, mismatched keys.

---

#### 2.2 Data Filtering and Deduplication

- **Overview:**  
  This block merges initial and RSS data, filters duplicates by link, and restores full article content for unique new entries to prevent redundant processing.

- **Nodes Involved:**  
  - Merge  
  - pubDate&link Only  
  - Filter Unique Links  
  - Merge1  
  - Restore Full Data with Code

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from "Read Initial Links" and "Save Initial Data" streams.  
    - Inputs:  
      - From "Read Initial Links"  
      - From "Save Initial Data"  
    - Outputs: To "pubDate&link Only".  
    - Edge Cases: Mismatched arrays or empty inputs.

  - **pubDate&link Only**  
    - Type: Set  
    - Role: Reduces dataset to only `pubDate` and `link` fields for deduplication checks.  
    - Inputs: From "Merge"  
    - Outputs: To "Filter Unique Links".  
    - Edge Cases: Missing or malformed link field causes filtering issues.

  - **Filter Unique Links**  
    - Type: Code  
    - Role: Custom JavaScript filters out all links appearing more than once, keeping only unique links to avoid duplicate processing.  
    - Logic: Counts link occurrences, returns items with count == 1.  
    - Inputs: From "pubDate&link Only"  
    - Outputs: To "Merge1".  
    - Edge Cases: Links with different case or whitespace treated as different; may skip near-duplicates.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines filtered unique links with filtered article data for restoring full content.  
    - Inputs:  
      - From "Filtered Data" (original article data)  
      - From "Filter Unique Links" (unique links subset)  
    - Outputs: To "Restore Full Data with Code".  
    - Edge Cases: Empty inputs cause no output.

  - **Restore Full Data with Code**  
    - Type: Code  
    - Role: Custom JavaScript to match unique links back to their full article data, retaining only those with non-empty content and duplicate instances.  
    - Logic:  
      - Tracks link counts and associated items.  
      - Selects only duplicated links with content for further processing.  
    - Inputs: From "Merge1"  
    - Outputs: To "Clean HTML Content".  
    - Edge Cases: Case sensitivity in links, empty content filtered out, may skip some edge cases.

---

#### 2.3 Content Processing and AI Analysis

- **Overview:**  
  This block cleans and extracts meaningful text content from HTML, classifies article relevance by topic using LangChain text classifier, and generates detailed Slack-formatted summaries with GPT-4.1-nano.

- **Nodes Involved:**  
  - Clean HTML Content  
  - Relevance Classification for Topic Monitoring  
  - OpenAI Chat Model1  
  - Basic LLM Chain

- **Node Details:**

  - **Clean HTML Content**  
    - Type: Code  
    - Role: Removes scripts, styles, HTML tags, comments, and extracts either meta description or main textual content from HTML, producing a clean text summary for AI input.  
    - Logic: Regular expressions to remove noise and extract content.  
    - Inputs: From "Restore Full Data with Code"  
    - Outputs: To "Relevance Classification for Topic Monitoring".  
    - Edge Cases: Malformed HTML, missing meta description, non-standard tags.

  - **Relevance Classification for Topic Monitoring**  
    - Type: LangChain Text Classifier  
    - Role: Classifies articles into "relevant" or "not_relevant" categories based on title and cleaned content.  
    - Configuration:  
      - Input text: concatenation of title and cleaned content  
      - Categories:  
        - "relevant": related to 欧阳良宜, 读书笔记, AI fields  
        - "not_relevant": unrelated articles  
      - Fallback: discard unclassified content  
    - Inputs: From "Clean HTML Content"  
    - Outputs: To "Basic LLM Chain" (on relevant articles)  
    - Edge Cases: Classification ambiguity, fallback discards data silently.

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4.1-nano model as AI engine for the LLM chain.  
    - Credentials: OpenAI API credentials required.  
    - Inputs: Used internally by "Basic LLM Chain".  
    - Edge Cases: API limit errors, network errors, rate limits.

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Generates Slack-formatted, in-depth Chinese summaries with analysis for relevant articles using GPT-4.1-nano.  
    - Configuration:  
      - Input text: cleaned article content  
      - Prompt: detailed instructions for summary structure, Slack Markdown formatting, and analytical depth.  
    - Inputs: From "Relevance Classification for Topic Monitoring"  
    - Outputs: To "Set Fields - Relevant Articles".  
    - Edge Cases: Model response failures, prompt misinterpretation, incomplete output.

---

#### 2.4 Output and Archiving

- **Overview:**  
  This final block formats fields for each summarized article and stores them into Google Sheets and Notion databases for archival and further use.

- **Nodes Involved:**  
  - Set Fields - Relevant Articles  
  - Google Sheets - Add relevant article  
  - Create a database page

- **Node Details:**

  - **Set Fields - Relevant Articles**  
    - Type: Set  
    - Role: Assigns fields such as article URL, summary text, fetch timestamp, published date, and summarized flag.  
    - Inputs: From "Basic LLM Chain"  
    - Outputs: To "Google Sheets - Add relevant article" and "Create a database page".  
    - Edge Cases: Missing fields propagate empty values.

  - **Google Sheets - Add relevant article**  
    - Type: Google Sheets  
    - Role: Appends summarized article data into a dedicated Google Sheets tab for processed data.  
    - Configuration: Specifies columns (title, summary, fetched_at, etc.) with no matching keys (append only).  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Inputs: From "Set Fields - Relevant Articles"  
    - Outputs: None (terminal node).  
    - Edge Cases: API quotas, data truncation, connectivity issues.

  - **Create a database page**  
    - Type: Notion  
    - Role: Creates a new page in a Notion database representing the summarized article.  
    - Configuration: Maps article URL to title, summary to rich text, and fetch timestamp.  
    - Credentials: Notion API credentials.  
    - Inputs: From "Set Fields - Relevant Articles"  
    - Outputs: None (terminal node).  
    - Edge Cases: API limits, property mismatches, credential errors.

---

### 3. Summary Table

| Node Name                        | Node Type                   | Functional Role                         | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                                                            |
|---------------------------------|-----------------------------|---------------------------------------|-----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Manual workflow start                  | -                                 | Read Initial Links, Read RSS Links     |                                                                                                                                        |
| Read Initial Links               | Google Sheets               | Read initial article links             | When clicking ‘Execute workflow’   | Merge                                | Step 1 - Data Input📥: Read initial links and RSS feeds from Google Sheets. 📋                                                        |
| Read RSS Links                  | Google Sheets               | Read RSS feed URLs                     | When clicking ‘Execute workflow’   | RSS Read                            | Step 1 - Data Input📥: Read initial links and RSS feeds from Google Sheets. 📋                                                        |
| RSS Read                       | RSS Feed Read               | Fetch RSS articles from feeds          | Read RSS Links                    | pubDate Processing                  |                                                                                                                                        |
| pubDate Processing             | Set                         | Normalize and assign publication dates | RSS Read                         | IF (Filter by Date)                 |                                                                                                                                        |
| IF (Filter by Date)            | If                          | Filter articles published in last 10 days | pubDate Processing               | Filtered Data                      |                                                                                                                                        |
| Filtered Data                 | Set                         | Assign article metadata fields         | IF (Filter by Date)               | Save Initial Data, Merge1            |                                                                                                                                        |
| Save Initial Data             | Google Sheets               | Append/update filtered articles         | Filtered Data                    | Merge                              |                                                                                                                                        |
| Merge                        | Merge                       | Merge initial links and saved data     | Read Initial Links, Save Initial Data | pubDate&link Only                |                                                                                                                                        |
| pubDate&link Only             | Set                         | Reduce items to pubDate and link only  | Merge                            | Filter Unique Links                |                                                                                                                                        |
| Filter Unique Links           | Code                        | Filter out duplicate links              | pubDate&link Only                | Merge1                            | Step 2 - Deduplication Step 🔄: Check for already processed URLs. Skip duplicates for efficiency. 🚀                                     |
| Merge1                       | Merge                       | Merge filtered unique links and data   | Filtered Data, Filter Unique Links | Restore Full Data with Code       | Step 2 - Deduplication Step 🔄: Check for already processed URLs. Skip duplicates for efficiency. 🚀                                     |
| Restore Full Data with Code   | Code                        | Restore full article data for duplicates | Merge1                          | Clean HTML Content                |                                                                                                                                        |
| Clean HTML Content            | Code                        | Clean HTML to extract readable text    | Restore Full Data with Code      | Relevance Classification for Topic Monitoring |                                                                                                                                        |
| Relevance Classification for Topic Monitoring | LangChain Text Classifier  | Classify articles as relevant or not    | Clean HTML Content               | Basic LLM Chain                   | Step 3 - Processing⚙️: Classify articles' relevance to AI/specific persons. 🎯 AI determines relevance.                                |
| OpenAI Chat Model1            | LangChain OpenAI Chat Model | GPT-4.1-nano model for AI summarization | Used internally by Basic LLM Chain | Basic LLM Chain               | Step 3 - Processing⚙️: AI Summarization 🧠 with GPT-4.1-nano. Focus on key points & analysis.                                            |
| Basic LLM Chain               | LangChain LLM Chain         | Generate Slack-formatted article summaries | Relevance Classification for Topic Monitoring | Set Fields - Relevant Articles | Step 3 - Processing⚙️: AI Summarization 🧠 with GPT-4.1-nano. Focus on key points & analysis.                                            |
| Set Fields - Relevant Articles | Set                         | Format summarized article fields        | Basic LLM Chain                 | Google Sheets - Add relevant article, Create a database page |                                                                                                                                        |
| Google Sheets - Add relevant article | Google Sheets           | Append summarized articles to sheet     | Set Fields - Relevant Articles   | -                                 | Step 4 - Output📤: Save to Google Sheets and Notion.                                                                                   |
| Create a database page        | Notion                      | Create Notion database pages for articles | Set Fields - Relevant Articles   | -                                 | Step 4 - Output📤: Save to Google Sheets and Notion.                                                                                   |
| Sticky Note                  | Sticky Note                 | Workflow overview and setup notes       | -                               | -                                 | ## Workflow Overview\nAutomates classification and summarization of WeChat articles using GPT-4.1-nano with outputs to Sheets and Notion. |
| Sticky Note1                 | Sticky Note                 | Step 1 Data Input description            | -                               | -                                 | Step 1 - Data Input📥\nRead initial links and RSS feeds from Google Sheets. 📋                                                          |
| Sticky Note2                 | Sticky Note                 | Step 2 Deduplication description        | -                               | -                                 | Step 2 - Deduplication Step 🔄\nCheck for already processed URLs. Skip duplicates for efficiency. 🚀                                      |
| Sticky Note3                 | Sticky Note                 | Step 3 Processing description            | -                               | -                                 | Step 3 - Processing⚙️\nClassification and AI Summarization with GPT-4.1-nano. 🤖                                                       |
| Sticky Note4                 | Sticky Note                 | Step 4 Output description                | -                               | -                                 | Step 4 - Output📤\nSave to Google Sheets and Notion.                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Manual start point.  

2. **Create 'Read Initial Links' Google Sheets Node**  
   - Document ID: Your Google Sheets document containing initial links.  
   - Sheet Name: The tab/gid number for initial links (e.g., 198451233).  
   - Operation: Read rows.  
   - Credentials: Google Sheets OAuth2.  
   - Connect from Manual Trigger.

3. **Create 'Read RSS Links' Google Sheets Node**  
   - Document ID: Same as above.  
   - Sheet Name: Tab/gid for RSS feed URLs (e.g., gid=0).  
   - Operation: Read rows.  
   - Credentials: Google Sheets OAuth2.  
   - Connect from Manual Trigger.

4. **Create 'RSS Read' Node**  
   - Type: RSS Feed Read  
   - URL: Set dynamically using `={{ $json.rss_feed_url }}` from input.  
   - Options: SSL validation enabled.  
   - Connect from 'Read RSS Links'.

5. **Create 'pubDate Processing' Set Node**  
   - Assign `pubDate` as ISO string from input date.  
   - Initialize other metadata fields (creator, title, link, author, content fields).  
   - Connect from 'RSS Read'.

6. **Create 'IF (Filter by Date)' Node**  
   - Condition: `pubDate` timestamp > current date minus 10 days.  
   - Connect from 'pubDate Processing'.

7. **Create 'Filtered Data' Set Node**  
   - Reassign all article metadata fields explicitly from input JSON.  
   - Connect from IF node's "true" output.

8. **Create 'Save Initial Data' Google Sheets Node**  
   - Document and Sheet: Same Google Sheets doc, tab for saving initial links.  
   - Operation: Append or update by matching 'link'.  
   - Columns: link, title, pubDate, etc.  
   - Credentials: Google Sheets OAuth2.  
   - Connect from 'Filtered Data'.

9. **Create 'Merge' Node**  
   - Merge inputs from 'Read Initial Links' and 'Save Initial Data'.  
   - Mode: Default.  
   - Connect 'Read Initial Links' and 'Save Initial Data' outputs to this node.

10. **Create 'pubDate&link Only' Set Node**  
    - Keep only 'pubDate' and 'link' fields.  
    - Connect from 'Merge'.

11. **Create 'Filter Unique Links' Code Node**  
    - JavaScript code to count link occurrences and keep only those with count = 1.  
    - Connect from 'pubDate&link Only'.

12. **Create 'Merge1' Node**  
    - Merge inputs from 'Filtered Data' and 'Filter Unique Links'.  
    - Connect 'Filtered Data' and 'Filter Unique Links' outputs here.

13. **Create 'Restore Full Data with Code' Node**  
    - JavaScript code to reconstruct full data for duplicated links with content.  
    - Connect from 'Merge1'.

14. **Create 'Clean HTML Content' Code Node**  
    - JavaScript to extract meaningful text from HTML content and meta description, remove scripts/styles/tags.  
    - Connect from 'Restore Full Data with Code'.

15. **Create 'Relevance Classification for Topic Monitoring' Node**  
    - Type: LangChain Text Classifier.  
    - Input: `={{ $json.title }}{{ $json.cleanedContent }}`.  
    - Categories:  
      - relevant: related to 欧阳良宜, 读书笔记, AI  
      - not_relevant: unrelated articles  
    - Fallback: discard.  
    - Connect from 'Clean HTML Content'.

16. **Create 'OpenAI Chat Model1' Node**  
    - Type: LangChain OpenAI Chat Model.  
    - Model: GPT-4.1-nano.  
    - Credentials: OpenAI API.  
    - Used internally by Basic LLM Chain.

17. **Create 'Basic LLM Chain' Node**  
    - Type: LangChain LLM Chain.  
    - Input Text: `={{ $json.cleanedContent }}`.  
    - Prompt: Detailed instructions to produce Slack-formatted Chinese summaries with analysis.  
    - Use 'OpenAI Chat Model1' as AI model.  
    - Connect from 'Relevance Classification for Topic Monitoring'.

18. **Create 'Set Fields - Relevant Articles' Node**  
    - Assign fields: article_url, summarized=YES, summary (from LLM output), fetched_at (current timestamp), publish_date (formatted).  
    - Connect from 'Basic LLM Chain'.

19. **Create 'Google Sheets - Add relevant article' Node**  
    - Document and Sheet: Google Sheets document and tab for saving processed data.  
    - Operation: Append only.  
    - Columns: title, summary, fetched_at, summarized, article_url, publish_date.  
    - Credentials: Google Sheets OAuth2.  
    - Connect from 'Set Fields - Relevant Articles'.

20. **Create 'Create a database page' Notion Node**  
    - Resource: databasePage.  
    - Database ID: Your Notion database for WeChat articles.  
    - Properties: Map article_url to title, summary to rich_text, fetched_at to rich_text.  
    - Credentials: Notion API.  
    - Connect from 'Set Fields - Relevant Articles'.

21. **Optionally add Sticky Notes for documentation and clarity** at appropriate canvas positions.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow automates classification and summarization of WeChat articles using GPT-4.1-nano with outputs to Sheets and Notion. | Workflow Overview Sticky Note in n8n canvas.                                                            |
| Step 1 - Data Input: Read initial links and RSS feeds from Google Sheets.                                      | Sticky Note1                                                                                            |
| Step 2 - Deduplication: Check for already processed URLs; skip duplicates for efficiency.                      | Sticky Note2                                                                                            |
| Step 3 - Processing: Classification and AI summarization with GPT-4.1-nano focused on key points and analysis. | Sticky Note3                                                                                            |
| Step 4 - Output: Save to Google Sheets and Notion.                                                             | Sticky Note4                                                                                            |
| Prompt used for summarization includes Slack markdown formatting instructions and is tailored for Chinese content. | Inside "Basic LLM Chain" node prompt configuration.                                                     |
| Google Sheets and Notion credentials should be configured in n8n credential manager to avoid hardcoding.       | Security best practice.                                                                                  |
| Replace Google Sheets document IDs, sheet names, RSS feed URLs, and Notion database IDs with user-specific values. | Customization requirement before running workflow.                                                      |

---

This structured documentation enables comprehensive understanding, facilitates modification or extension, and supports troubleshooting for the entire WeChat article classification and summarization workflow in n8n.