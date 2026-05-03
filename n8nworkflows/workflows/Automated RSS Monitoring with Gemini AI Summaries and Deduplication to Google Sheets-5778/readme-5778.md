Automated RSS Monitoring with Gemini AI Summaries and Deduplication to Google Sheets

https://n8nworkflows.xyz/workflows/automated-rss-monitoring-with-gemini-ai-summaries-and-deduplication-to-google-sheets-5778


# Automated RSS Monitoring with Gemini AI Summaries and Deduplication to Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of multiple RSS feeds, generating AI-powered summaries for new articles and storing the results in Google Sheets while avoiding duplicates. It is designed for users who want an up-to-date digest of recent articles with concise, structured summaries created by Gemini AI.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Scheduling**: Handles manual execution and scheduled triggers to start the workflow.
- **1.2 Configuration Settings**: Loads Google Sheets URLs and filtering parameters.
- **1.3 RSS Feed Retrieval and Preparation**: Reads RSS feed URLs from Google Sheets and fetches articles, appending source names.
- **1.4 Filtering & Deduplication**: Filters articles to only recent ones and checks for duplicates in Google Sheets.
- **1.5 Content Fetching and Processing**: Retrieves full article HTML content, extracts body, and converts it to Markdown.
- **1.6 AI Summarization**: Uses Gemini AI to create structured, concise summaries.
- **1.7 Output Handling**: Formats AI output and appends new article summaries to Google Sheets.
- **1.8 Workflow Control**: Includes looping and no-op nodes for flow control and termination.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

- **Overview:** Initiates the workflow either manually or on a scheduled hourly basis.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger  
- **Node Details:**  
  - *When clicking ‘Execute workflow’*  
    - Type: Manual Trigger  
    - Role: Allows manual start by user interaction  
    - Configuration: No parameters; triggers workflow on button click  
    - Output: Connects to Settings node  
    - Edge Cases: None typical; user must trigger manually  
  - *Schedule Trigger*  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow every hour  
    - Configuration: Interval set to every 1 hour  
    - Output: Connects to Settings node  
    - Edge Cases: Time zone differences may affect trigger time  

#### 1.2 Configuration Settings

- **Overview:** Defines key parameters such as Google Sheets document URLs and the time window filter for articles.
- **Nodes Involved:**  
  - Settings  
  - Sticky Note (Settings instructions)  
- **Node Details:**  
  - *Settings*  
    - Type: Set node  
    - Role: Stores configuration like Google Sheets URLs for feeds, articles, current time, and filter days  
    - Configuration:  
      - Document URL (Google Sheets document for articles)  
      - Articles Sheet URL (sheet for storing articles)  
      - RSS Feeds Sheet URL (sheet with RSS URLs)  
      - Current Time: dynamic, based on trigger or current time  
      - Last X Days: number parameter (default 31) to filter recent articles  
    - Output: Connects to Get RSS Feed List node  
    - Edge Cases: Incorrect URLs or missing parameters will cause downstream failures  
  - *Sticky Note*  
    - Provides setup instructions and Google Sheets copy link  

#### 1.3 RSS Feed Retrieval and Preparation

- **Overview:** Reads the list of RSS feeds from Google Sheets and fetches articles from each feed, annotating them with source names.
- **Nodes Involved:**  
  - Get RSS Feed List  
  - Loop Over Items  
  - Read RSS  
  - Combine Rss with source name  
  - Sticky Note1  
- **Node Details:**  
  - *Get RSS Feed List*  
    - Type: Google Sheets  
    - Role: Retrieves RSS feed URLs and names from the configured Google Sheet  
    - Configuration: Uses dynamic document and sheet names from Settings node  
    - Credentials: Google Sheets OAuth2  
    - Output: Connects to Loop Over Items  
    - Edge Cases: Sheet access errors, empty feed lists  
  - *Loop Over Items*  
    - Type: SplitInBatches  
    - Role: Processes each RSS feed entry individually to manage load and concurrency  
    - Output: Connects to Read RSS and Filter Last X Days nodes  
  - *Read RSS*  
    - Type: RSS Feed Read  
    - Role: Reads articles from the given RSS feed URL  
    - Configuration: URL taken dynamically from current item JSON, ignores SSL errors  
    - Output: Connects to Combine Rss with source name  
    - Edge Cases: Feed unavailability, malformed RSS data  
  - *Combine Rss with source name*  
    - Type: Set  
    - Role: Adds RSS source name field to each article item  
    - Configuration: Assigns “RSS NAME” from Loop Over Items node data  
    - Output: Connects back to Loop Over Items for further processing  
  - *Sticky Note1*  
    - Describes loop purpose to add source name to RSS records  

#### 1.4 Filtering & Deduplication

- **Overview:** Filters articles published within the last X days and checks if articles already exist in Google Sheets to avoid duplicates.
- **Nodes Involved:**  
  - Filter Last X Days  
  - Loop Over Rss Elements  
  - Get Row for URL is in Sheets  
  - Check If Article Exists  
- **Node Details:**  
  - *Filter Last X Days*  
    - Type: Filter  
    - Role: Keeps only articles with pubDate after the calculated cutoff date (Current Time minus Last X Days)  
    - Configuration: DateTime comparison using expressions, dynamic cutoff from Settings  
    - Output: Connects to Loop Over Rss Elements  
    - Edge Cases: Articles with missing or malformed pubDate will be excluded  
  - *Loop Over Rss Elements*  
    - Type: SplitInBatches  
    - Role: Processes each filtered article individually for deduplication and further steps  
    - Output: Connects to Get Row for URL is in Sheets and End of workflow (for skipped duplicates)  
  - *Get Row for URL is in Sheets*  
    - Type: Google Sheets  
    - Role: Searches the Articles Sheet for any existing row matching the article’s link URL  
    - Configuration: Lookup by "link" column using the current article's link  
    - Credentials: Google Sheets OAuth2  
    - Output: Connects to Check If Article Exists  
    - Edge Cases: API limits, authentication errors, or missing sheet columns  
  - *Check If Article Exists*  
    - Type: If  
    - Role: Determines if the Google Sheets query returned an empty object (article does not exist)  
    - Configuration: Condition checks if the output from "Get Row for URL is in Sheets" is empty  
    - Output:  
      - True branch (article not found): proceeds to fetch article content  
      - False branch (article exists): loops back to Loop Over Rss Elements to skip  
    - Edge Cases: Expression evaluation failure if input is malformed  

#### 1.5 Content Fetching and Processing

- **Overview:** Downloads full HTML article content, extracts the main body, and transforms it to Markdown for AI processing.
- **Nodes Involved:**  
  - Get Webpage HTML Content  
  - Extract Body Content in HTML  
  - Convert HTML to Markdown  
- **Node Details:**  
  - *Get Webpage HTML Content*  
    - Type: HTTP Request  
    - Role: Fetches full HTML content of the article URL  
    - Configuration: URL from current article link, response format text  
    - Output: Connects to Extract Body Content in HTML  
    - Edge Cases: Network errors, timeouts, HTTP errors, redirects  
  - *Extract Body Content in HTML*  
    - Type: HTML Extract  
    - Role: Parses full HTML to extract only the `<body>` content, ignoring images and other elements  
    - Configuration: CSS selector `body`, returns HTML fragment  
    - Output: Connects to Convert HTML to Markdown  
    - Edge Cases: Pages without body tag, malformed HTML  
  - *Convert HTML to Markdown*  
    - Type: Markdown  
    - Role: Converts extracted HTML body into Markdown format for better AI readability  
    - Configuration: Ignores images and form elements, no block elements specified  
    - Output: Connects to Summarize Content  
    - Edge Cases: Conversion errors if input HTML is invalid or empty  

#### 1.6 AI Summarization

- **Overview:** Uses Gemini AI to generate structured, concise summaries of the article content in Markdown format.
- **Nodes Involved:**  
  - LLM Chat Model  
  - Summarize Content  
  - Format Output  
- **Node Details:**  
  - *LLM Chat Model*  
    - Type: Langchain OpenRouter LLM Chat Model  
    - Role: Intermediary node to send prompts to Gemini AI (Google Gemini 2.5 flash)  
    - Configuration: Model `google/gemini-2.5-flash`, temperature 0.3, response format text  
    - Credentials: OpenRouter API key  
    - Output: Connects to Summarize Content node as AI language model input  
    - Edge Cases: API quota, authentication errors, model downtime  
  - *Summarize Content*  
    - Type: Langchain Chain LLM  
    - Role: Sends detailed prompt to Gemini AI to produce structured summary with specific instructions and formatting  
    - Configuration: Complex templated prompt including article metadata and content, detailed writing guidelines, and output formatting rules (Markdown, no preambles)  
    - Output: Connects to Format Output  
    - Retry on failure enabled for robustness  
    - Edge Cases: Prompt template errors, AI response errors, timeout  
  - *Format Output*  
    - Type: Set  
    - Role: Cleans the AI response by removing any `<think>` tags from the output text  
    - Configuration: Regex replacement on `text` property  
    - Output: Connects to Append Aummary to Google Sheets  
    - Edge Cases: Unexpected AI output format may cause empty results  

#### 1.7 Output Handling

- **Overview:** Appends or updates the article and AI summary data in Google Sheets.
- **Nodes Involved:**  
  - Append Aummary to Google Sheets  
- **Node Details:**  
  - *Append Aummary to Google Sheets*  
    - Type: Google Sheets  
    - Role: Inserts or updates an article record with AI summary in the configured Articles Sheet  
    - Configuration:  
      - Maps columns: link, title, source, pubDate, ai summary, categories  
      - Uses “appendOrUpdate” operation keyed on the link column to avoid duplicates  
      - Cell format RAW to preserve Markdown formatting  
      - Dynamic document and sheet references from Settings node  
    - Credentials: Google Sheets OAuth2  
    - Output: Loops back to Loop Over Rss Elements to continue processing next article  
    - Edge Cases: API quota limits, sheet schema mismatches, connectivity issues  

#### 1.8 Workflow Control

- **Overview:** Manages workflow loops and termination points for clean execution.
- **Nodes Involved:**  
  - End of worfklow (NoOp)  
- **Node Details:**  
  - *End of worfklow*  
    - Type: NoOp  
    - Role: Marks logical end of workflow processing for articles that don't require further action (e.g., duplicates)  
    - No inputs or outputs connected besides Loop Over Rss Elements  
    - Edge Cases: None  

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                               | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                             |
|------------------------------|-------------------------------|-----------------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start of workflow                       |                               | Settings                      |                                                                                                       |
| Schedule Trigger             | Schedule Trigger              | Scheduled hourly start                         |                               | Settings                      |                                                                                                       |
| Settings                    | Set                           | Configuration parameters for sheets and filters | When clicking ‘Execute workflow’, Schedule Trigger | Get RSS Feed List             | Copy this Google Sheet : https://docs.google.com/spreadsheets/d/1i1p_DPymm8QeZrCLxs-Skz8BMSutKpUW4khzpiNBcYc/copy Enter copied URLs in settings Set up time gap for news from RSS |
| Get RSS Feed List           | Google Sheets                 | Retrieve RSS feed URLs and names               | Settings                      | Loop Over Items               | Get feeds Url's  from Google Sheet                                                                    |
| Loop Over Items              | SplitInBatches               | Process RSS feeds one by one                    | Get RSS Feed List             | Read RSS, Filter Last X Days  | Loop for adding source name to RSS record                                                             |
| Read RSS                    | RSS Feed Read                 | Fetch articles from RSS feed                    | Loop Over Items               | Combine Rss with source name  | This read RSS channel                                                                                  |
| Combine Rss with source name | Set                           | Add source name to each article                 | Read RSS                     | Loop Over Items               | Loop for adding source name to RSS record                                                             |
| Filter Last X Days          | Filter                        | Keep only recent articles in last X days       | Loop Over Items               | Loop Over Rss Elements        | This filter only news from last x days                                                                |
| Loop Over Rss Elements       | SplitInBatches               | Process articles individually for deduplication | Filter Last X Days            | Get Row for URL is in Sheets, End of worfklow |                                                                                                       |
| Get Row for URL is in Sheets | Google Sheets                 | Check if article URL already exists in sheet   | Loop Over Rss Elements        | Check If Article Exists       | Check if the record with the link exists in Google Sheets                                             |
| Check If Article Exists      | If                            | Branch based on duplicate check result         | Get Row for URL is in Sheets  | Get Webpage HTML Content (if new), Loop Over Rss Elements (if duplicate) | We check if the object is empty                                                                         |
| Get Webpage HTML Content     | HTTP Request                  | Download full article HTML content              | Check If Article Exists       | Extract Body Content in HTML  |                                                                                                       |
| Extract Body Content in HTML | HTML Extract                  | Extract main body HTML without images           | Get Webpage HTML Content      | Convert HTML to Markdown      | We need only body without images                                                                       |
| Convert HTML to Markdown     | Markdown                      | Convert HTML body to Markdown                    | Extract Body Content in HTML  | Summarize Content            | Markdown is more friendly LLM format than HTML                                                        |
| LLM Chat Model              | Langchain LLM Chat Model      | AI model interface (Gemini AI)                   | Summarize Content (LLM input) | Summarize Content (LLM output) | You can choose other LLM chat model                                                                   |
| Summarize Content           | Langchain Chain LLM           | Generate structured AI summary                    | Convert HTML to Markdown, LLM Chat Model | Format Output               |                                                                                                       |
| Format Output               | Set                           | Clean AI output text                              | Summarize Content            | Append Aummary to Google Sheets |                                                                                                       |
| Append Aummary to Google Sheets | Google Sheets               | Insert or update article and summary in sheet   | Format Output                | Loop Over Rss Elements        |                                                                                                       |
| End of worfklow             | NoOp                          | Logical endpoint for skipped articles            | Loop Over Rss Elements        |                               |                                                                                                       |
| Sticky Note                 | Sticky Note                   | Setup instructions                               |                               |                               | See notes in block 1.2                                                                                 |
| Sticky Note1                | Sticky Note                   | Loop explanation                                 |                               |                               | Loop for adding source name to RSS record                                                             |
| Sticky Note2                | Sticky Note                   | Google Sheets columns explanation                 |                               |                               | Google Sheet Template: https://docs.google.com/spreadsheets/d/1i1p_DPymm8QeZrCLxs-Skz8BMSutKpUW4khzpiNBcYc/ Articles Sheet and RSS FEEDS columns info                |
| Sticky Note3                | Sticky Note                   | Full workflow description and setup guide         |                               |                               | https://docs.google.com/spreadsheets/d/1i1p_DPymm8QeZrCLxs-Skz8BMSutKpUW4khzpiNBcYc/copy Workflow detailed description and instructions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Allow manual execution of the workflow.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval: Every 1 hour.

3. **Create Settings Node (Set)**  
   - Add fields:  
     - Document (string): URL of main Google Sheets document for articles  
     - Articles Sheet (string): URL or sheet name for articles storage  
     - Rss Feeds (string): URL or sheet name for RSS feeds list  
     - Current Time (expression): Use `$workflow.trigger === 'Schedule Trigger' ? $('Schedule Trigger').item.json.timestamp : $now`  
     - Last X Days (number): default 31 (filter window)  
   - Connect outputs of Manual Trigger and Schedule Trigger to this node.

4. **Create Get RSS Feed List Node (Google Sheets)**  
   - Operation: Read rows  
   - Document ID & Sheet Name: Use expressions referencing Settings node values  
   - Credentials: Configure Google Sheets OAuth2 API  
   - Connect Settings node output to this node.

5. **Create Loop Over Items Node (SplitInBatches)**  
   - Purpose: Process each RSS feed entry individually  
   - Connect Get RSS Feed List output here.

6. **Create Read RSS Node (RSS Feed Read)**  
   - URL: Set to `{{$json["RSS URL"]}}` dynamically from each feed  
   - Options: Enable ignore SSL errors if needed  
   - Connect Loop Over Items output here.

7. **Create Combine Rss with source name Node (Set)**  
   - Add a field “RSS NAME” with value from `{{$json["RSS NAME"]}}` of Loop Over Items node  
   - Connect Read RSS output here.

8. **Connect Combine Rss with source name output back to Loop Over Items**  
   - To continue processing enriched articles.

9. **Create Filter Last X Days Node (Filter)**  
   - Condition: `$json.pubDate` is after `$('Settings').item.json['Current Time'].toDateTime().minus($('Settings').item.json['Last X Days'], 'days').startOf('day')`  
   - Connect Loop Over Items output here.

10. **Create Loop Over Rss Elements Node (SplitInBatches)**  
    - Processes filtered articles individually  
    - Connect Filter Last X Days output here.

11. **Create Get Row for URL is in Sheets Node (Google Sheets)**  
    - Operation: Lookup row with filter column “link” equals current item’s link  
    - Document ID & Sheet Name: Use Settings node expression values  
    - Credentials: Google Sheets OAuth2  
    - Connect Loop Over Rss Elements output here.

12. **Create Check If Article Exists Node (If)**  
    - Condition: Check if output from Get Row for URL is in Sheets is empty (no existing article)  
    - Connect Get Row for URL is in Sheets output here.

13. **Create Get Webpage HTML Content Node (HTTP Request)**  
    - URL: Use current item’s link  
    - Response Format: Text  
    - Connect If node “True” branch here.

14. **Create Extract Body Content in HTML Node (HTML Extract)**  
    - Extraction: CSS selector `body`, return HTML  
    - Connect HTTP Request output here.

15. **Create Convert HTML to Markdown Node (Markdown)**  
    - Input: HTML content from Extract Body Content in HTML  
    - Options: Ignore images and form elements  
    - Connect HTML Extract output here.

16. **Create LLM Chat Model Node (Langchain OpenRouter LLM Chat Model)**  
    - Model: `google/gemini-2.5-flash`  
    - Temperature: 0.3  
    - Credentials: OpenRouter API key configured  
    - Connect Summarize Content node input (next step) to this node as AI language model input.

17. **Create Summarize Content Node (Langchain Chain LLM)**  
    - Input Text: Construct prompt template incorporating article metadata and content with specific writing and output guidelines for Gemini AI (see detailed prompt in analysis)  
    - Retry on failure: enabled  
    - Connect Convert HTML to Markdown output and LLM Chat Model output here.

18. **Create Format Output Node (Set)**  
    - Operation: Clean AI output text removing `<think>` tags via regex replace on field `text`  
    - Connect Summarize Content node output here.

19. **Create Append Aummary to Google Sheets Node (Google Sheets)**  
    - Operation: Append or update row keyed by “link” column  
    - Map columns: link, title, source, pubDate, ai summary, categories  
    - Use RAW cell format to preserve Markdown  
    - Document ID & Sheet Name: From Settings node references  
    - Credentials: Google Sheets OAuth2  
    - Connect Format Output node output here.

20. **Connect Append Aummary to Google Sheets output back to Loop Over Rss Elements**  
    - To continue processing next articles.

21. **Create End of worfklow Node (NoOp)**  
    - Connect Loop Over Rss Elements output “duplicate” branch here  
    - Marks workflow end for skipped duplicates.

22. **Add Sticky Notes nodes with instructions and references** as per blocks above for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Copy this Google Sheets template to get started: https://docs.google.com/spreadsheets/d/1i1p_DPymm8QeZrCLxs-Skz8BMSutKpUW4khzpiNBcYc/copy. Update the Settings node with your copied URLs. Configure OAuth2 credentials for Google Sheets and OpenRouter API key for Gemini AI.                                                                                                                                                                                                                                                                                                                                                                             | Setup instructions from Sticky Note in block 1.2                                                                                |
| Google Sheets structure: Articles Sheet columns include pubDate, source, title, link, categories, ai summary. RSS FEEDS sheet columns include RSS NAME and RSS URL.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | From Sticky Note2                                                                                                                 |
| Workflow description: Monitors RSS feeds, filters recent articles, deduplicates using Google Sheets, fetches full content, converts HTML to Markdown, generates Gemini AI summaries, and saves data to Google Sheets. Supports both manual and scheduled execution.                                                                                                                                                                                                                                                                                                                                                                                               | From Sticky Note3 and workflow overview                                                                                          |
| AI prompt in Summarize Content node includes detailed instructions for Gemini AI to produce structured, concise summaries using Markdown, emphasizing clarity, objectivity, and practical insights. It handles technical, business, and case study articles with special formatting.                                                                                                                                                                                                                                                                                                                                                                               | From Summarize Content node analysis                                                                                            |
| To modify AI model or summarization style, update the LLM Chat Model or Summarize Content nodes accordingly. Adjust time filtering by changing “Last X Days” in Settings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Customization options described in Sticky Note3                                                                                  |

---

This completes the full structured documentation and reference for the "Automated RSS Monitoring with Gemini AI Summaries and Deduplication to Google Sheets" n8n workflow.