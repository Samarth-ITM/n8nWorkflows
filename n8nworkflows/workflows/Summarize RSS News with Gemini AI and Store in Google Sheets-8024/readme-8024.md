Summarize RSS News with Gemini AI and Store in Google Sheets

https://n8nworkflows.xyz/workflows/summarize-rss-news-with-gemini-ai-and-store-in-google-sheets-8024


# Summarize RSS News with Gemini AI and Store in Google Sheets

### 1. Workflow Overview

This workflow automates the process of fetching news articles from multiple RSS feeds, summarizing their content using AI (Google Gemini), and storing the summarized results in Google Sheets. It is designed to keep a curated and up-to-date news summary repository by filtering recent articles, avoiding duplicates, and generating concise summaries for each article. The workflow is triggered either manually or on a daily schedule.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Configuration Setup:** Starts the workflow via manual or scheduled trigger and loads configuration settings including Google Sheets URLs and filtering parameters.
- **1.2 RSS Feed Retrieval and Filtering:** Reads the list of RSS feed URLs, fetches articles, and filters articles published within a recent timeframe.
- **1.3 Article Processing Loop:** Iterates over each article, checks if it already exists in the Google Sheets database, and decides whether to process it further.
- **1.4 Content Extraction and Conversion:** For new articles, fetches the full webpage content, extracts the HTML body, and converts it into Markdown format for better AI processing.
- **1.5 AI Summarization:** Sends the Markdown content to the Google Gemini AI model to generate a structured summary following a defined prompt.
- **1.6 Data Storage:** Appends or updates the summarized article data into designated Google Sheets tabs, maintaining both a complete archive and a daily summary.
- **1.7 Miscellaneous:** Includes no-operation and sticky note nodes for workflow management and documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration Setup

- **Overview:** This block initializes the workflow, triggered either manually or by a scheduled daily time. It sets up key configuration parameters such as Google Sheets document URLs, the number of days for filtering recent articles, and captures the current timestamp.
- **Nodes Involved:**  
  - Schedule Trigger  
  - When clicking ‘Execute workflow’  
  - Settings  
  - Clear sheet  
- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow daily at 6:00 AM (Asia/Kolkata timezone).  
    - Configuration: Set for daily execution at hour 6.  
    - Inputs: None  
    - Outputs: Triggers "Settings" node.  
    - Edge Cases: Timezone misconfiguration could cause unexpected trigger times.  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow for testing or on-demand runs.  
    - Inputs: None  
    - Outputs: Triggers "Settings" node.  
  - **Settings**  
    - Type: Set  
    - Role: Defines key variables, including Google Sheets document URLs for feeds, articles, today’s sheet, and last X days filter.  
    - Key Parameters:  
      - Document: URL of the Google Sheets document used for storage.  
      - Articles Sheet: Sheet URL for storing article summaries.  
      - Today Sheet: Sheet URL for daily summaries.  
      - Rss Feeds: Sheet URL listing RSS feed URLs.  
      - Current Time: Captures trigger timestamp or current time.  
      - Last X Days: Numeric filter for how many days back to consider articles.  
    - Inputs: Trigger nodes.  
    - Outputs: Triggers "Clear sheet" node.  
  - **Clear sheet**  
    - Type: Google Sheets (clear operation)  
    - Role: Clears the "Today Sheet" except for the first row to prepare for fresh data daily.  
    - Configuration:  
      - Operation: Clear sheet content.  
      - Keep First Row: True (preserves headers).  
      - Sheet Name & Document ID: Taken from Settings node.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Failure if authentication expires or sheet inaccessible; data loss if misconfigured.  
    - Inputs: Settings node.  
    - Outputs: Triggers "Get RSS Feed List" node.

---

#### 2.2 RSS Feed Retrieval and Filtering

- **Overview:** Retrieves the list of RSS feed URLs from a Google Sheet, then reads articles from each feed and filters articles published within the configured recent timeframe.
- **Nodes Involved:**  
  - Get RSS Feed List  
  - Loop Over RSS Feed  
  - Read RSS  
  - Combine Rss with source name  
  - Filter Last X Days  
- **Node Details:**  
  - **Get RSS Feed List**  
    - Type: Google Sheets (read)  
    - Role: Reads RSS feed URLs from a designated Google Sheet.  
    - Configuration:  
      - Document ID and Sheet Name fetched dynamically from Settings node.  
    - Credentials: Google Sheets OAuth2.  
    - Inputs: Clear sheet node.  
    - Outputs: Triggers "Loop Over RSS Feed".  
    - Notes: Sticky note "Get feeds Url's from Google Sheet".  
    - Edge Cases: Empty or malformed feed URLs, sheet access issues.  
  - **Loop Over RSS Feed**  
    - Type: Split In Batches  
    - Role: Iterates over each RSS feed URL to process them individually.  
    - Configuration: Default batching, no specific batch size defined.  
    - Inputs: Get RSS Feed List.  
    - Outputs: Two parallel branches: "Filter Last X Days" and "Read RSS".  
  - **Read RSS**  
    - Type: RSS Feed Read  
    - Role: Fetches feed items from the current RSS URL.  
    - Configuration:  
      - URL: Dynamically set to current feed's "RSS URL" field.  
      - Options: SSL errors ignored to avoid failures with invalid certificates.  
      - Execute Once: True to avoid repeated reads in the same execution.  
    - Inputs: Loop Over RSS Feed.  
    - Outputs: Triggers "Combine Rss with source name".  
    - Edge Cases: Network errors, invalid RSS formats, empty feeds.  
  - **Combine Rss with source name**  
    - Type: Set  
    - Role: Adds the RSS feed name to each article item for source tracking.  
    - Configuration: Adds field "RSS NAME" from Loop Over RSS Feed current item.  
    - Inputs: Read RSS.  
    - Outputs: Loop Over RSS Feed (loop continuation).  
    - Notes: Sticky Note "Loop for adding source name to RSS record".  
  - **Filter Last X Days**  
    - Type: Filter  
    - Role: Filters articles leaving only those published within the last X days, based on Settings.  
    - Configuration:  
      - Condition: Article 'pubDate' is after (Current Time - Last X Days).  
      - Uses date comparison expressions and dynamic date calculation.  
    - Inputs: Loop Over RSS Feed branch.  
    - Outputs: Loop Over Rss Elements node.  
    - Edge Cases: Incorrect dates, timezone mismatches, missing pubDate fields.

---

#### 2.3 Article Processing Loop

- **Overview:** Processes each filtered article individually, checking if it already exists in the Google Sheets archive to avoid duplicate summarization.
- **Nodes Involved:**  
  - Loop Over Rss Elements  
  - Get Row for URL is in Sheets  
  - Check If Article Exists  
  - End of worfklow (NoOp)  
- **Node Details:**  
  - **Loop Over Rss Elements**  
    - Type: Split In Batches  
    - Role: Iterates over each filtered article separately.  
    - Inputs: Filter Last X Days.  
    - Outputs: Two branches: to "End of worfklow" and "Get Row for URL is in Sheets".  
  - **Get Row for URL is in Sheets**  
    - Type: Google Sheets (read with filter)  
    - Role: Searches the Articles Sheet for existing records matching the article link.  
    - Configuration:  
      - Filter: Lookup by 'link' column matching current article's link.  
      - Sheet and Document IDs from Settings.  
      - Always outputs data (empty or matching row).  
    - Inputs: Loop Over Rss Elements.  
    - Outputs: Check If Article Exists.  
    - Edge Cases: Sheet access failures, link field missing, multiple matches.  
  - **Check If Article Exists**  
    - Type: If  
    - Role: Checks if the Google Sheets query result is empty, meaning article is new.  
    - Condition: Object emptiness of the "Get Row" node output.  
    - Inputs: Get Row for URL is in Sheets.  
    - Outputs:  
      - True (empty): continue processing new article with "Get Webpage HTML Content".  
      - False (exists): skips processing, loops back to Loop Over Rss Elements.  
  - **End of worfklow (NoOp)**  
    - Type: No Operation  
    - Role: Terminates processing path for articles that already exist.  
    - Inputs: Loop Over Rss Elements (branch for skipping).  
    - Outputs: None.  
    - Notes: Helps control flow for existing articles.

---

#### 2.4 Content Extraction and Conversion

- **Overview:** For new articles, fetches the full web page HTML, extracts the main body content (excluding images), and converts it into Markdown format to prepare for AI summarization.
- **Nodes Involved:**  
  - Get Webpage HTML Content  
  - Extract Body Content in HTML  
  - Convert HTML to Markdown  
- **Node Details:**  
  - **Get Webpage HTML Content**  
    - Type: HTTP Request  
    - Role: Fetches raw HTML content from the article’s URL.  
    - Configuration:  
      - URL taken dynamically from current article's link field.  
      - Response format: Plain text.  
    - Inputs: Check If Article Exists (True branch).  
    - Outputs: Extract Body Content in HTML.  
    - Edge Cases: Network errors, HTTP errors, redirects, content unavailable.  
  - **Extract Body Content in HTML**  
    - Type: HTML Extract  
    - Role: Extracts the <body> content only, excluding images, forms, and other non-essential elements.  
    - Configuration:  
      - CSS Selector: `body`  
      - Return value: HTML content of body.  
      - Options: Trim values, clean up text enabled.  
    - Inputs: Get Webpage HTML Content.  
    - Outputs: Convert HTML to Markdown.  
    - Notes: Sticky Note "We need only body without images".  
  - **Convert HTML to Markdown**  
    - Type: Markdown Conversion  
    - Role: Converts extracted HTML body into Markdown format for better LLM processing.  
    - Configuration:  
      - Input HTML: From extracted body content.  
      - Options: Ignore images and forms.  
    - Inputs: Extract Body Content in HTML.  
    - Outputs: Summarize Content node.  
    - Notes: Sticky Note "Markdown is more friendly LLM format than HTML".  
    - Edge Cases: Conversion errors, malformed HTML.

---

#### 2.5 AI Summarization

- **Overview:** Uses Google Gemini AI language model to generate a concise, structured summary of the article content, following a detailed prompt instructing the AI on tone, style, and structure.
- **Nodes Involved:**  
  - Summarize Content (Langchain LLM Chain)  
  - Google Gemini Chat Model1  
  - Format Output  
- **Node Details:**  
  - **Google Gemini Chat Model1**  
    - Type: Google Gemini AI Chat Model  
    - Role: Provides the AI language model backend for summarization.  
    - Credentials: Google Palm API credentials required.  
    - Inputs: Summarize Content node (as AI model).  
    - Outputs: Summarize Content node (LLM chain).  
    - Edge Cases: API authentication errors, rate limits, timeouts.  
  - **Summarize Content**  
    - Type: Langchain LLM Chain  
    - Role: Defines the prompt template and sends Markdown content to Gemini AI for summarization.  
    - Configuration:  
      - Prompt includes article metadata (date, title, author, categories).  
      - Instructions specify Narayan persona, writing guidelines, special cases, quality control, and output formatting in Markdown.  
      - Response expected as a structured summary without extraneous text.  
      - Retry on failure enabled.  
    - Inputs: Convert HTML to Markdown.  
    - Outputs: Format Output.  
  - **Format Output**  
    - Type: Set  
    - Role: Cleans AI output by removing any internal <think> tags from the summary text.  
    - Configuration: Uses regex replacement on the "text" field.  
    - Inputs: Summarize Content.  
    - Outputs: Two Google Sheets append nodes.  
    - Edge Cases: Regex failure if unexpected text format.

---

#### 2.6 Data Storage

- **Overview:** Appends or updates the summarized data into two Google Sheets tabs: one for the full archive of articles and one for the daily summary sheet.
- **Nodes Involved:**  
  - Append Aummary to Google Sheets (Archive)  
  - Append Aummary to Google Sheets1 (Today Sheet)  
- **Node Details:**  
  - **Append Aummary to Google Sheets**  
    - Type: Google Sheets (append or update)  
    - Role: Stores summarized article data in the main Articles Sheet.  
    - Configuration:  
      - Columns mapped: link, title, source (RSS NAME), publication date (formatted), AI summary text, categories as JSON string.  
      - Matching column: link (used to avoid duplicates).  
      - Cell format: RAW.  
      - Operation: append or update.  
      - Sheet and Document IDs from Settings node.  
    - Credentials: Google Sheets OAuth2.  
    - Inputs: Format Output.  
  - **Append Aummary to Google Sheets1**  
    - Type: Google Sheets (append or update)  
    - Role: Stores summarized article data in the Today Sheet, refreshed daily.  
    - Configuration and credentials same as above, but sheet points to Today Sheet URL.  
    - Inputs: Format Output.  
    - Edge Cases: Sheet access issues, data conflicts, formatting errors.

---

#### 2.7 Miscellaneous and Flow Control

- **Nodes Involved:**  
  - End of worfklow (NoOp)  
  - Sticky Note1  
- **Node Details:**  
  - **End of worfklow**  
    - Type: No Operation  
    - Role: Terminates processing path for existing articles to prevent reprocessing.  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Documentation aid stating "Loop for adding source name to RSS record".  
    - Visually covers nodes: Combine Rss with source name and related nodes.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                          | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                  |
|----------------------------|------------------------------|----------------------------------------|-------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual workflow start                   | None                          | Settings                                   |                                                                                              |
| Schedule Trigger           | Schedule Trigger              | Scheduled workflow start (daily 6 AM) | None                          | Settings                                   |                                                                                              |
| Settings                  | Set                          | Define URLs, filter parameters          | When clicking ‘Execute workflow’, Schedule Trigger | Clear sheet                                | Set Here Your Google Sheets URLs and Last x Days Filter                                    |
| Clear sheet               | Google Sheets (clear)         | Clear Today Sheet before update         | Settings                      | Get RSS Feed List                          |                                                                                              |
| Get RSS Feed List          | Google Sheets (read)          | Retrieve RSS feed URLs                   | Clear sheet                   | Loop Over RSS Feed                         | Get feeds Url's  from Google Sheet                                                         |
| Loop Over RSS Feed         | Split In Batches              | Iterate over RSS feeds                   | Get RSS Feed List             | Filter Last X Days, Read RSS               |                                                                                              |
| Read RSS                   | RSS Feed Read                | Fetch articles from RSS feed             | Loop Over RSS Feed            | Combine Rss with source name               | This read RSS channel                                                                       |
| Combine Rss with source name | Set                         | Add RSS feed name to articles            | Read RSS                     | Loop Over RSS Feed                         | Loop for adding source name to RSS record                                                  |
| Filter Last X Days         | Filter                       | Keep articles published within last X days | Loop Over RSS Feed           | Loop Over Rss Elements                     | This filter only news from last x days                                                     |
| Loop Over Rss Elements     | Split In Batches              | Iterate over filtered articles           | Filter Last X Days            | End of worfklow, Get Row for URL is in Sheets |                                                                                              |
| Get Row for URL is in Sheets | Google Sheets (read with filter) | Check if article exists in archive       | Loop Over Rss Elements        | Check If Article Exists                    | Check if the record with the link exists in Google Sheets                                  |
| Check If Article Exists    | If                           | Determine if article is new               | Get Row for URL is in Sheets  | Get Webpage HTML Content (if new), Loop Over Rss Elements (if exists) | We check if the object is empty                                                            |
| End of worfklow           | No Operation                 | End processing for existing articles     | Loop Over Rss Elements (skip) | None                                       |                                                                                              |
| Get Webpage HTML Content   | HTTP Request                 | Fetch article webpage HTML                | Check If Article Exists       | Extract Body Content in HTML               |                                                                                              |
| Extract Body Content in HTML | HTML Extract                | Extract body content without images      | Get Webpage HTML Content      | Convert HTML to Markdown                   | We need only body without images                                                           |
| Convert HTML to Markdown   | Markdown Conversion          | Convert HTML body to Markdown             | Extract Body Content in HTML  | Summarize Content                          | Markdown is more friendly LLM format than HTML                                             |
| Google Gemini Chat Model1  | Google Gemini AI Model       | AI backend for summarization              | Summarize Content (LLM chain) | Summarize Content                          |                                                                                              |
| Summarize Content          | Langchain LLM Chain          | Prepare prompt and get AI summary         | Convert HTML to Markdown      | Format Output                              |                                                                                              |
| Format Output             | Set                          | Clean AI output text                       | Summarize Content            | Append Aummary to Google Sheets, Append Aummary to Google Sheets1 |                                                                                              |
| Append Aummary to Google Sheets | Google Sheets (append/update) | Store summary in Articles Sheet           | Format Output                | Loop Over Rss Elements                      |                                                                                              |
| Append Aummary to Google Sheets1 | Google Sheets (append/update) | Store summary in Today Sheet               | Format Output                | Loop Over Rss Elements                      |                                                                                              |
| Sticky Note1              | Sticky Note                  | Documentation note                         | None                        | None                                       | Loop for adding source name to RSS record                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   - Add a **Schedule Trigger** node named "Schedule Trigger" configured to run daily at 6:00 AM in the Asia/Kolkata timezone.

2. **Create Settings Node:**  
   - Add a **Set** node named "Settings".  
   - Configure the following fields:  
     - Document: Google Sheets document URL for storage.  
     - Articles Sheet: URL for article summaries sheet.  
     - Today Sheet: URL for daily summaries sheet.  
     - Rss Feeds: URL for sheet containing RSS feed URLs.  
     - Current Time: Expression to use trigger timestamp or current time:  
       `={{$workflow.trigger === 'Schedule Trigger' ? $('Schedule Trigger').item.json.timestamp : $now}}`  
     - Last X Days: Number, e.g., 1 (for filtering recent articles).  
   - Connect both "When clicking ‘Execute workflow’" and "Schedule Trigger" nodes to this Settings node.

3. **Clear Today Sheet:**  
   - Add a **Google Sheets** node named "Clear sheet".  
   - Operation: Clear sheet content except first row.  
   - Sheet Name and Document ID: Use expressions from the Settings node fields for "Today Sheet" and document ID respectively.  
   - Connect "Settings" node output to "Clear sheet".

4. **Get RSS Feed List:**  
   - Add a **Google Sheets** node named "Get RSS Feed List".  
   - Operation: Read rows from the sheet containing the RSS feed URLs.  
   - Document ID and Sheet Name: From Settings node fields "Document" and "Rss Feeds" respectively.  
   - Connect "Clear sheet" output to this node.

5. **Loop Over RSS Feed:**  
   - Add a **Split In Batches** node named "Loop Over RSS Feed" to process feeds one by one.  
   - Connect "Get RSS Feed List" output to this node.

6. **Read RSS:**  
   - Add an **RSS Feed Read** node named "Read RSS".  
   - URL: Set to current feed URL dynamically: `={{ $json["RSS URL"] }}`.  
   - Options: Enable "Ignore SSL errors" to avoid failures with invalid certificates.  
   - Connect one output branch of "Loop Over RSS Feed" to this node.

7. **Combine RSS with source name:**  
   - Add a **Set** node named "Combine Rss with source name".  
   - Add field "RSS NAME" with value: `={{ $('Loop Over RSS Feed').item.json["RSS NAME"] }}` (or equivalent to carry feed name).  
   - Include all other fields from incoming data.  
   - Connect "Read RSS" output to this node.  
   - Connect output back to "Loop Over RSS Feed" to continue the loop.

8. **Filter Last X Days:**  
   - Add a **Filter** node named "Filter Last X Days".  
   - Condition: Keep only articles where `pubDate` is after `Current Time - Last X Days` (use expression with date functions).  
   - Connect the other output branch of "Loop Over RSS Feed" to this node.

9. **Loop Over Rss Elements:**  
   - Add a **Split In Batches** node named "Loop Over Rss Elements" to iterate over filtered articles.  
   - Connect "Filter Last X Days" output to this node.

10. **Get Row for URL is in Sheets:**  
    - Add a **Google Sheets** node configured to read rows with filter.  
    - Sheet Name and Document ID: From Settings node fields "Articles Sheet" and "Document".  
    - Filter: Search for rows where "link" equals current article's "link".  
    - Always output data.  
    - Connect "Loop Over Rss Elements" output to this node.

11. **Check If Article Exists:**  
    - Add an **If** node named "Check If Article Exists".  
    - Condition: Check if the output from "Get Row for URL is in Sheets" is empty (object empty test).  
    - Connect "Get Row for URL is in Sheets" output to this node.

12. **End of Workflow (NoOp):**  
    - Add a **No Operation** node named "End of worfklow".  
    - Connect the False output (article exists) from the If node to this NoOp node to stop processing duplicates.

13. **Get Webpage HTML Content:**  
    - Add an **HTTP Request** node named "Get Webpage HTML Content".  
    - URL: Dynamic from current article link.  
    - Response format: Text (HTML).  
    - Connect True output (new article) of the If node to this node.

14. **Extract Body Content in HTML:**  
    - Add an **HTML Extract** node named "Extract Body Content in HTML".  
    - Operation: Extract HTML content from `<body>` tag only.  
    - Options: Trim values, clean up text.  
    - Connect "Get Webpage HTML Content" output to this node.

15. **Convert HTML to Markdown:**  
    - Add a **Markdown** node named "Convert HTML to Markdown".  
    - Input: HTML content from previous node.  
    - Options: Ignore images and forms.  
    - Connect "Extract Body Content in HTML" output to this node.

16. **Summarize Content:**  
    - Add a **Langchain LLM Chain** node named "Summarize Content".  
    - Input text: Detailed prompt including article metadata and content in Markdown.  
    - Configure prompt according to the provided multi-section instructions, including persona, writing guidelines, quality control, and structured summary format.  
    - Enable retry on failure.  
    - Connect "Convert HTML to Markdown" output to this node.

17. **Google Gemini Chat Model1:**  
    - Add a **Google Gemini AI Chat Model** node named "Google Gemini Chat Model1".  
    - Connect it as AI language model node to "Summarize Content".  
    - Configure with Google Palm API credentials.

18. **Format Output:**  
    - Add a **Set** node named "Format Output".  
    - Use regex expression to clean out any `<think>` tags from the AI response text.  
    - Connect "Summarize Content" output to this node.

19. **Append Aummary to Google Sheets (Archive):**  
    - Add a **Google Sheets** node named "Append Aummary to Google Sheets".  
    - Operation: Append or update records in "Articles Sheet" with columns: link, title, source, pubDate (formatted), ai summary, categories (JSON string).  
    - Matching column: "link".  
    - Credentials: Google Sheets OAuth2.  
    - Connect "Format Output" output to this node.

20. **Append Aummary to Google Sheets1 (Today):**  
    - Add a **Google Sheets** node named "Append Aummary to Google Sheets1".  
    - Operation: Same as above but target "Today Sheet".  
    - Connect "Format Output" output to this node.

21. **Connect both Append nodes outputs back to "Loop Over Rss Elements"** to continue processing.

22. **Add Sticky Notes:**  
    - For documentation, add sticky notes as in the original workflow, e.g. "Loop for adding source name to RSS record" over the relevant nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Use Google Sheets OAuth2 credentials to allow reading and writing sheets securely.                                      | Credential setup in n8n for Google Sheets OAuth2.                                              |
| Google Gemini (PaLM) API credentials are required for AI summarization nodes.                                           | https://developers.generativeai.google/                                                        |
| The prompt used for summarization is carefully designed with detailed instructions for tone, structure, and content.  | Embedded in the Summarize Content node configuration.                                          |
| The workflow clears the "Today Sheet" daily to maintain a fresh list of summarized articles for that day.               | Prevents data duplication and maintains focus on recent news.                                 |
| The workflow uses batch splitting to efficiently process multiple feeds and articles without overloading API requests. | n8n SplitInBatches nodes configured without specific batch size, but can be adjusted.         |
| Date filtering relies on dynamic expressions using n8n date/time functions and assumes consistent pubDate format.      | Timezone and date format consistency is critical for accurate filtering.                       |
| Markdown conversion improves AI processing by simplifying HTML content structure and removing unnecessary elements.   | Avoids issues with raw HTML parsing by the AI model.                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.